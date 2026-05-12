# CloudFront (CDN)

AWS's global content delivery network. Sits in front of an **origin** (S3 bucket, Lambda function URL, API Gateway, EC2/ALB, etc.) and serves cached responses from ~600 edge locations worldwide.

**What it's good for:**
- Serving static websites from S3 with HTTPS + caching + free egress to CloudFront
- Putting an HTTPS layer in front of API Gateway (with custom domain + WAF)
- Reducing latency for users far from your AWS region
- Streaming video, large file downloads

**Why use it (vs. just public S3):**
- **HTTPS:** S3 buckets can't easily serve custom-domain HTTPS. CloudFront can.
- **Cheaper egress:** S3 → internet is ~$0.09/GB; CloudFront → internet is ~$0.085/GB *and* S3 → CloudFront is free (origin shield bonus).
- **Security:** Bucket stays private (Block Public Access ON). CloudFront accesses via OAC.
- **Performance:** Edge caches mean Sydney users don't round-trip to N. Virginia for static files.

---

## Concepts (quick)

| Term | Meaning |
|---|---|
| **Distribution** | A CloudFront configuration unit. Maps a public URL (e.g. `d2abc.cloudfront.net`) to one or more origins. |
| **Origin** | Where the actual content lives — S3 bucket, API Gateway, ALB, EC2, or any HTTP server. |
| **OAC (Origin Access Control)** | The recommended way for CloudFront to authenticate to S3 / Lambda function URLs. Replaces old OAI. |
| **Behavior** | A rule that says "for requests matching this path pattern, use *this* origin with *these* cache settings." Distributions can have many behaviors (e.g. `/api/*` → API Gateway, everything else → S3). |
| **Cache policy** | How long CloudFront keeps responses cached at the edge. Different defaults for static (`CachingOptimized`) vs API (`CachingDisabled`). |
| **Invalidation** | Force CloudFront to evict cached responses so the next request fetches fresh from origin. |
| **Edge location** | One of ~600 globally-distributed PoPs where CloudFront serves from. |
| **Price class** | Limits which regions' edge locations are used. Cheaper classes = fewer edges = slightly higher latency for far-away users. |

---

## Creating a distribution — Console

The wizard has 4 steps (it skips the TLS step for the default `*.cloudfront.net` cert).

### Step 1 — Get started
- **Distribution name:** descriptive, no spaces (e.g. `myproject-frontend`)
- **Distribution type:** **Single website or app**
- **Route 53 managed domain:** leave blank (configure custom domain later in the Settings → CNAME / certificate fields)

### Step 2 — Specify origin
- **Origin type:** **Amazon S3** (or API Gateway, ALB, etc.)
- **S3 origin:** click **Browse S3** → pick your bucket
- **Origin path:** leave blank (set this if your content lives in a subfolder like `s3://bucket/dist/`)
- **Allow private S3 bucket access to CloudFront:** **Recommended** — auto-creates OAC + updates bucket policy. Huge time saver vs. old wizard.
- **Origin settings:** Use recommended
- **Cache settings:** Use recommended (tailored to S3 origin = `CachingOptimized`)

### Step 3 — Enable security
- **Do not enable security protections** for personal/dev projects. WAF costs ~$14/mo even idle.
- Enable WAF later if you need it (compliance, prod scale, abuse from public users).

### Step 4 — Review and create
- Verify origin + OAC settings
- **Create distribution**

### Post-creation: fix Default Root Object
The new wizard doesn't ask for it. After creation:
1. Distribution detail page → **Settings** section → **Edit**
2. **Default root object:** `index.html`
3. **Price class:** drop to **Use only North America and Europe** to save money (default is "All edge locations")
4. Save

**Without a default root object, hitting `https://distribution.cloudfront.net/` returns 403** (it doesn't know which file to return for `/`).

Distribution deployment takes **5-15 minutes** the first time. Status: `Deploying` → `Enabled`.

---

## Creating a distribution — CLI / CDK

**CLI** is verbose and rarely used directly. Prefer CDK:

```python
# CDK Python (conceptual)
from aws_cdk import aws_cloudfront as cf, aws_cloudfront_origins as origins

distribution = cf.Distribution(self, "Frontend",
    default_root_object="index.html",
    default_behavior=cf.BehaviorOptions(
        origin=origins.S3BucketOrigin.with_origin_access_control(bucket),
        viewer_protocol_policy=cf.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cache_policy=cf.CachePolicy.CACHING_OPTIMIZED,
    ),
    price_class=cf.PriceClass.PRICE_CLASS_100,  # NA + EU
)
```

That replaces the entire console wizard in ~10 lines.

---

## Pattern: one distribution, multiple origins (static + API)

A common architecture: serve the static frontend from S3 *and* the `/api/*` routes from API Gateway *through the same CloudFront domain*. Saves having to deal with CORS between frontend and API.

**How to set it up:**
1. Distribution already exists with S3 as default origin (Phase 1)
2. Add API Gateway as a **second origin** (no OAC — API Gateway is public HTTPS)
3. Add a **behavior** with path pattern `/api/*` → that new origin, with:
   - Cache policy: **`CachingDisabled`** (don't cache API responses)
   - Origin request policy: **`AllViewerExceptHostHeader`** (critical — see gotcha below)
   - Allowed methods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE

**Behavior precedence** matters — CloudFront matches behaviors top-down by path pattern. The wildcard `Default (*)` should always be last. New behaviors created via console default to precedence 0 (highest), which is what you want for `/api/*`.

This pattern lets your frontend just call `fetch("/api/characters")` — same origin, no CORS headers needed.

---

## OAC: how CloudFront reads a private S3 bucket

**Old way (avoid):** Public S3 bucket. Bypasses CloudFront entirely if someone finds the S3 URL. Common data-leak vector.

**Modern way:** Bucket is private (Block Public Access ON). CloudFront has an **Origin Access Control** identity. The bucket policy grants `s3:GetObject` *only* to your specific CloudFront distribution.

The new wizard's "Allow private S3 bucket access to CloudFront: Recommended" option does all of this automatically. The bucket policy ends up looking like:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<bucket-name>/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
      }
    }
  }]
}
```

You almost never write this by hand anymore — let the wizard or CDK generate it.

---

## Cache invalidation

CloudFront caches responses at edge locations. When you update files in S3, edges keep serving the *old* cached version until either:
- Cache TTL expires (default several hours-to-days)
- You issue an **invalidation**

### Invalidate via CLI

```bash
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```

Use `/*` to invalidate everything (fast and brutal). Or be surgical:
```bash
--paths "/index.html" "/static/css/*"
```

### Invalidate via Console

Distribution → **Invalidations** tab → **Create invalidation** → paste paths.

### Cost

- **First 1,000 paths/month: free**
- After that: $0.005 per path

`/*` counts as 1 path, not "every file." So `--paths "/*"` is the cheap-and-simple default.

### When to invalidate

| When | Need to invalidate? |
|---|---|
| Deploying new HTML/CSS/JS to S3 | **Yes** — otherwise users see old version |
| Adding new files (no existing version) | Usually no — first request will fetch and cache |
| Changing bucket policy or OAC | No — these aren't cached |
| Changing CloudFront settings (default root object, etc.) | No — these aren't cached responses |

### Versioned filenames vs invalidation

Modern frontends often skip invalidation entirely by versioning filenames (`main.abc123.css`, `app.def456.js`). New deploy = new filenames = no cache collision. Vite, webpack, Next.js do this automatically. For hand-written static sites, invalidation is fine.

---

## Common operations cheat sheet

```bash
# List distributions
aws cloudfront list-distributions --query 'DistributionList.Items[].{Id:Id,Domain:DomainName,Status:Status}'

# Get details on one
aws cloudfront get-distribution --id <DIST_ID>

# Invalidate everything
aws cloudfront create-invalidation --distribution-id <DIST_ID> --paths "/*"

# Check invalidation status
aws cloudfront list-invalidations --distribution-id <DIST_ID>

# Disable a distribution (required before deletion)
# 1. get current config + ETag
aws cloudfront get-distribution-config --id <DIST_ID>
# 2. set Enabled=false, then update-distribution
# 3. wait for deploy
# 4. delete-distribution

# View bucket policy that OAC generated
aws s3api get-bucket-policy --bucket <bucket-name> --output text | jq .
```

---

## Pricing

**Free tier (always free, not just first 12 months):**
- 1 TB data transfer out/month
- 10 million requests/month
- 1,000 invalidations/month

**After free tier (us-east-1 → North America):**
- **Data transfer out:** ~$0.085/GB (cheaper than S3 direct)
- **HTTPS requests:** ~$0.0120 per 10,000
- **Invalidations beyond 1,000:** $0.005 per path

**At 10 D&D-party users:** $0/month, always.

**Price class trade-off:**
| Class | Coverage | Cost factor |
|---|---|---|
| All edge locations | Best performance globally | 1.0× (most expensive) |
| **NA + EU** | Most users | ~0.8× |
| NA only | Cheapest, slowest for international | ~0.6× |

For personal projects with NA users, **NA + EU** is the sweet spot.

---

## Common gotchas

**`/` returns 403 after deployment**
Default root object isn't set. Distribution → Settings → Edit → `index.html`.

**CSS/JS 404s even though files exist in S3**
The HTML references paths that don't match the bucket layout. Example: HTML says `<link href="/static/css/main.css">` but you uploaded files to the bucket root (so the file is at `/css/main.css`, not `/static/css/main.css`). **Bucket path layout must match HTML hrefs.**

Fix by either:
- Re-uploading with the matching prefix (`aws s3 sync ./static/ s3://bucket/static/`)
- Or rewriting HTML to use bucket-root paths

Then invalidate CloudFront so cached 404 responses don't persist.

**Updated file in S3 but CloudFront still serves old version**
Edge cache. Run `aws cloudfront create-invalidation --distribution-id X --paths "/*"`. Wait 1-2 min.

**"Access Denied" XML from CloudFront**
Either:
- Bucket policy doesn't grant CloudFront's OAC access (re-run the wizard or check policy)
- The path doesn't exist in S3 (404 disguised as 403 by S3 default behavior)
- The viewer hit a path that maps to no behavior (rare; happens with multi-origin setups)

**Custom domain (CNAME) requires ACM cert in us-east-1**
Even if your distribution is global, the ACM certificate for the CNAME *must* be in `us-east-1`. Other regions are silently invalid. Bites people setting up custom domains from a non-us-east-1 account.

**OAC vs OAI**
OAI (Origin Access Identity) is the legacy mechanism. OAC (Origin Access Control) is the modern replacement — supports more services (Lambda function URLs, etc.) and uses SigV4 signing. Always use OAC for new distributions.

**Disabling vs deleting a distribution**
You can't delete an Enabled distribution. Disable it first → wait for deployment to "Deployed" status → then delete. Takes ~10-15 min total.

**Logs cost real money**
Standard logging delivers access logs to an S3 bucket — fine, but at scale you may want to disable it or move to real-time logs with sampling.

**API Gateway origin returns 403 `ForbiddenException` when accessed via CloudFront, but works directly**
The `AllViewer` managed origin-request policy forwards the viewer's `Host` header (your CloudFront domain) to API Gateway. API Gateway doesn't recognize that host and returns 403.

**Fix:** Use the managed origin-request policy **`AllViewerExceptHostHeader`** for any behavior whose origin is API Gateway / ALB / Lambda Function URL. CloudFront then rewrites Host to the origin's hostname.

Behavior → Edit → Origin request policy → **AllViewerExceptHostHeader** → Save. ~5 min redeploy.

---

## When CloudFront is NOT the right answer

| Use case | Better choice |
|---|---|
| Internal-only API (in VPC) | ALB + Route 53 private hosted zone |
| Single-region, low-latency reads | Direct API Gateway or ALB (skip CDN) |
| Streaming video at scale | MediaPackage / MediaLive + CloudFront (CF is part of it, but you need the media services too) |
| Strict edge compute logic | Lambda@Edge (Lambda functions running at CloudFront edge) — separate feature, complex |
