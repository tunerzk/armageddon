Lab 2B-Honors++
CloudFront Validators, RefreshHit, and Conditional Requests
What this lab teaches (in one sentence)

A CDN can revalidate cached objects without fully refetching them — and that’s not a bug.

1. Mental Model (Teach This First)
CloudFront cache outcomes you care about:

| x-cache value                | Meaning                                                                  |
| ---------------------------- | ------------------------------------------------------------------------ |
| `Hit from cloudfront`        | Served entirely from cache                                               |
| `Miss from cloudfront`       | Fetched from origin                                                      |
| `RefreshHit from cloudfront` | Cached object existed, but CloudFront **revalidated** it with the origin |
| `Error from cloudfront`      | Origin or edge error                                                     |


Key insight
RefreshHit means:
    CloudFront did go to the origin
    But used conditional headers (If-None-Match, If-Modified-Since)
    Origin said 304 Not Modified
    CDN reused cached body

➡️ Lower bandwidth than a Miss, higher latency than a Hit

This is normal when:
    TTL expires
    Cache-Control: max-age is short
    Origin sends validators (ETag / Last-Modified)

2. Required App Changes (EC2 Origin)
Students must modify one static or semi-static endpoint (HTML or JSON) so it:
Sends validators
At least one of:
    ETag
    Last-Modified

Example (conceptual)
    GET /static/index.html
    Response headers:

    Cache-Control: public, max-age=30
    ETag: "chewie-v1"

3. CloudFront Setup Assumptions
This lab assumes:
    Origin-driven caching (Lab 2B-Honors)
    Managed policy: UseOriginCacheControlHeaders
    TTL = controlled by origin
    No invalidation yet

4. The Failure Injection (Instructor / Santa)
Injection A — “Why am I seeing RefreshHit every time?”
Steps:
    1) Set origin to:


    Cache-Control: public, max-age=5
    ETag: "chewie-v1"

Do not change content
Students repeatedly curl every ~6–10 seconds

5. Student Investigation (Required)
Step 1 — Observe headers
<img width="1493" height="272" alt="image" src="https://github.com/user-attachments/assets/d325f8af-7040-49fd-b81f-0984263a49a0" />
<img width="972" height="513" alt="image" src="https://github.com/user-attachments/assets/98a6f846-a450-48ed-9199-de9f1a720101" />


    curl -i https://chewbacca-growl.com/static/index.html | sed -n '1,25p'

Repeat after TTL expires.
Expected sequence:
    First request → Miss
    Next few within TTL → Hit
    After TTL → RefreshHit

Step 2 — Identify validators
Students must find:
    ETag or
    Last-Modified
<img width="1458" height="222" alt="image" src="https://github.com/user-attachments/assets/de11eded-8fea-4eab-955e-1d723566092f" />

Step 3 — Explain RefreshHit (written)
Students must state:

    “CloudFront had a cached copy, but TTL expired.
    It sent a conditional request to the origin using validators.
    The origin returned 304 Not Modified, so CloudFront reused the cached body.”

    A RefreshHit happens when CloudFront already has a cached object, but the cache entry has expired. Instead of immediately fetching the full object again, CloudFront performs a conditional request to the origin using validators like ETag or Last‑Modified.
  CloudFront had a cached copy, but the TTL expired.
It sent a conditional request to the origin using validators.
The origin returned 304 Not Modified, so CloudFront reused the cached body. 
CacheHit: 5 ms
RefreshHit: 25 ms

If they say “CloudFront is broken” → ❌

6. Proving It’s NOT a Full Miss
Change nothing, wait for TTL

    sleep 6
    curl -i https://chewbacca-growl.com/static/index.html
<img width="1513" height="352" alt="image" src="https://github.com/user-attachments/assets/4fee4626-f7bc-403b-b0dd-558dac321d73" />


Evidence to collect:
    x-cache: RefreshHit from cloudfront
    Response body unchanged
    Latency slightly higher than Hit
   <img width="1421" height="298" alt="image" src="https://github.com/user-attachments/assets/70177772-7cea-47a3-b381-ac25059d6700" />
   <img width="1433" height="317" alt="image" src="https://github.com/user-attachments/assets/18f0aca6-a690-4c6e-b7fc-e453079017f8" />
   <img width="1487" height="100" alt="image" src="https://github.com/user-attachments/assets/ad3893ad-5dc4-4bd0-8eb5-3944945e453e" />
   <img width="1487" height="100" alt="image" src="https://github.com/user-attachments/assets/a99332bc-7d17-4163-bc7c-8df218df8ecd" />
<img width="1487" height="100" alt="image" src="https://github.com/user-attachments/assets/27c9777e-cb1a-41ab-ac39-dc556b505e5c" />



 

Explain:
    Bandwidth saved
    Origin load reduced
    Still correct behavior

    A RefreshHit means CloudFront’s cached object has expired, but instead of downloading the entire file again, CloudFront sends a conditional request to the origin using:
If-None-Match (ETag)
If-Modified-Since (Last‑Modified)
A 304 response contains no body, so CloudFront does not re-download the file.
Why this saves bandwidth?
CloudFront only transfers a tiny 304 response instead of the full object.

A RefreshHit still requires CloudFront to contact the origin, but the work the origin does is minimal:
It checks the ETag or Last‑Modified timestamp. It returns a lightweight 304 response.
It does not read the file from disk.
It does not regenerate or re-serve the object.
Why this reduces load:
No heavy I/O. No large response body. No CPU-intensive processing. No repeated file reads.
The origin handles a quick metadata check instead of a full object retrieval.
The origin does not resend the file. The user still gets the cached body from CloudFront’s edge location.
This dramatically reduces data transfer between CloudFront and the origin.

A RefreshHit is exactly what HTTP caching is designed to do.
It proves: CloudFront’s cached copy was expired.
CloudFront validated the object with the origin. The origin confirmed the object was unchanged.
CloudFront reused the cached body. The user still received the correct, up‑to‑date content.
Why this is correct: It follows HTTP RFC 7232 (conditional requests). It ensures freshness without unnecessary downloads.
It maintains consistency between CloudFront and the origin. It optimizes performance while preserving correctness.

7. Second Injection — “Why won’t my change show up?”
Injection B — Change content BUT forget ETag
Instructor changes content but:
    Keeps same ETag
    Or forgets to update validators

Expected:
    CloudFront keeps getting 304
    Users see stale content
    RefreshHit continues forever

Student Fix Options (must justify one)

| Fix                  | When appropriate        |
| -------------------- | ----------------------- |
| Update ETag          | Correct fix             |
| Update Last-Modified | Acceptable              |
| Invalidate object    | Emergency only          |
| Increase TTL         | If content truly stable |


8. Controlled Invalidation (Tie-in to Honors+)
Now students are allowed to invalidate one path:

    aws cloudfront create-invalidation \
      --distribution-id <DIST_ID> \
      --paths "/static/index.html"


You must explain:
    Why invalidation works
    Why it’s not the preferred fix
    Why updating validators is better


9. Log Interpretation (Advanced)
If ALB access logs are enabled
Students should observe:
    Conditional requests (If-None-Match)
    304 responses from origin
    Lower response sizes

If CloudFront standard logs are enabled (optional)
    Explain fields:
    x-edge-result-type = RefreshHit
    sc-status = 304

10. Grading Rubric (Honors++)
Pass (must have all)
    Correct explanation of RefreshHit
    Evidence of validators
    Correct interpretation of headers
    Did NOT invalidate blindly

Honors++
  Explains tradeoff between:
    TTL length
    Revalidation frequency
    Latency vs freshness
  Chooses correct fix for Injection B

Auto-Fail
    “Invalidate everything”
    “CloudFront bug”
    No evidence cited

11. Add to Auto-IR (Bonus)
If Auto-IR is enabled:
    Bedrock report may say: “CloudFront cache anomaly”
    Student must override with:
        “Observed RefreshHit behavior due to short TTL + validators; system operating as designed.”

This reinforces the ethics rule:
    --> AI summarizes. Humans interpret.

Deliverables:
12. One-Paragraph Takeaway (You Must Write)
    --> “What does RefreshHit mean, and why is it often better than a Miss?”
If they can answer this cleanly, they’re ahead of most working engineers.

