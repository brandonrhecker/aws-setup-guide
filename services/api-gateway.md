# API Gateway

Sits in front of Lambda (or any HTTP backend) and exposes a public HTTPS URL. Handles routing, throttling, auth, CORS, request/response transformation, and optional features like usage plans + API keys.

**What it's good for:**
- Public REST/HTTP APIs backed by Lambda
- Webhook receivers
- Putting a public endpoint on top of a private VPC service
- WebSocket APIs (separate API type)

**The decision you'll make first:** HTTP API vs. REST API.

---

## HTTP API vs. REST API

| | **HTTP API** | **REST API** |
|---|---|---|
| Cost | $1.00 per 1M requests | $3.50 per 1M requests |
| Latency | ~60% lower than REST | Higher overhead |
| CORS | Built-in, simple config | Manual setup, more flexible |
| Auth | JWT (Cognito, Auth0), Lambda authorizer, IAM | All of HTTP + API keys, usage plans, mTLS |
| Request validation | No | Yes (JSON schema) |
| WAF | Via CloudFront (not direct) | Direct WAF attach |
| Caching | No | Built-in (separate cost) |
| Default choice | ✅ for new serverless backends | When you need its extra features |

**Rule of thumb:** start with HTTP API. Move to REST API only when you hit a feature you can't get elsewhere (request validation, API keys for partner access, etc.).

---

## Concepts

| Term | Meaning |
|---|---|
| **API** | A logical collection of routes + integrations. Has a unique ID like `s4dsxdhx63`. |
| **Route** | A method + path combo, e.g. `GET /api/characters`. Maps to one integration. |
| **Integration** | What handles the request — usually a Lambda function, but could be HTTP backend, AWS service action, mock response. |
| **Stage** | A named deployment environment for the API. Default stage in HTTP API is `$default` (auto-deploys). REST APIs typically use `dev` / `staging` / `prod`. |
| **Invoke URL** | The public HTTPS endpoint: `https://<api-id>.execute-api.<region>.amazonaws.com`. Each stage has its own. |
| **Event format v2.0** | The JSON payload Lambda receives. v2.0 (HTTP APIs) is cleaner than v1.0 (REST APIs). |

---

## Creating an HTTP API — Console

### Step 1: Create and configure integrations
1. API Gateway → **Create API** → **HTTP API** → **Build**
2. **API name:** `myproject-http-api`
3. **Integrations** → **Add integration**:
   - Type: **Lambda**
   - Region: `us-east-1`
   - Function: pick existing Lambda
   - Version: **2.0** (default; pick this for new projects)
4. Next

### Step 2: Configure routes
5. **Method:** `GET` (or whatever)
6. **Resource path:** `/api/characters` (full path the API exposes)
7. **Integration target:** auto-filled
8. Add more routes via "Add route" if needed
9. Next

### Step 3: Define stages
10. **Stage name:** `$default` (auto-created, auto-deploys on every change)
11. **Auto-deploy:** ON
12. Next

### Step 4: Review and create
13. Verify routes → **Create**

You'll land on the API detail page. **Invoke URL** appears at the top — that's your public endpoint.

---

## Adding routes after the fact

1. API detail page → **Develop → Routes** → **Create**
2. Method + path → **Create**
3. Click the route → **Attach integration** → pick your Lambda → **Attach**

Auto-deploy on `$default` means it's live within seconds.

---

## Event format 2.0 (what Lambda receives)

```json
{
  "version": "2.0",
  "routeKey": "GET /api/characters",
  "rawPath": "/api/characters",
  "rawQueryString": "side=dark",
  "headers": { ... },
  "queryStringParameters": { "side": "dark" },
  "pathParameters": { "slug": "Bucee_Smashcan" },   // if route has {slug}
  "body": "...",
  "isBase64Encoded": false,
  "requestContext": {
    "http": {
      "method": "GET",
      "path": "/api/characters",
      "sourceIp": "1.2.3.4"
    }
  }
}
```

Access these in Python:
```python
method = event["requestContext"]["http"]["method"]
slug   = event.get("pathParameters", {}).get("slug")
body   = json.loads(event["body"] or "{}")
```

---

## Path parameters

Add a route like `GET /api/characters/{slug}` — `{slug}` is the variable. In Lambda:
```python
slug = event["pathParameters"]["slug"]
```

Useful for RESTful resource URLs: `/api/characters/Bucee_Smashcan`.

---

## CORS

Browsers block cross-origin requests by default. Two ways to handle CORS:

### Easy way: return headers from Lambda
```python
return {
    "statusCode": 200,
    "headers": {
        "Access-Control-Allow-Origin": "*",         # or your CloudFront domain in prod
        "Access-Control-Allow-Methods": "GET,POST,OPTIONS",
        "Access-Control-Allow-Headers": "Content-Type,Authorization",
    },
    "body": json.dumps(...),
}
```

### Configured CORS (HTTP API)
API console → **CORS** tab → fill in allowed origins/methods/headers → save. API Gateway then handles preflight `OPTIONS` requests automatically.

The configured approach is cleaner for complex setups; Lambda-response headers are fine for simple cases.

**In prod, don't use `Access-Control-Allow-Origin: *`.** Lock it to your specific frontend domain (e.g. `https://d2vnfp4c8wburu.cloudfront.net`).

---

## Stages

**HTTP API** uses `$default` by default — auto-deploys. You can add named stages for multi-environment setups.

**REST API** requires explicit deploys to a stage every time you change anything. Slower workflow but explicit.

For personal projects, `$default` is fine.

---

## Custom domain names

Built-in: `https://<api-id>.execute-api.us-east-1.amazonaws.com` (ugly but works).

For `https://api.mydomain.com`:
1. ACM cert in `us-east-1` for the domain (Route 53 DNS validation)
2. API Gateway → **Custom domain names** → **Create**
3. Map the custom domain to your API (Stage = `$default`)
4. Route 53 A-record (alias) → the API Gateway domain

Covered in Phase 7 of the AWS setup playbook.

---

## Common operations cheat sheet

```bash
# List APIs
aws apigatewayv2 get-apis --query 'Items[].{Id:ApiId,Name:Name,Endpoint:ApiEndpoint}'

# Get an API's details
aws apigatewayv2 get-api --api-id <api-id>

# List routes for an API
aws apigatewayv2 get-routes --api-id <api-id>

# Create a route + attach integration (faster than console)
aws apigatewayv2 create-integration \
  --api-id <api-id> \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:lambda:us-east-1:<acct>:function:<func> \
  --payload-format-version 2.0
aws apigatewayv2 create-route \
  --api-id <api-id> \
  --route-key "GET /api/characters/{slug}" \
  --target integrations/<integration-id>

# Test
curl https://<api-id>.execute-api.us-east-1.amazonaws.com/api/characters

# Tail logs (uses CloudWatch — API GW must have logging enabled)
aws logs tail /aws/apigateway/<api-id> --follow
```

---

## Pricing

**HTTP API:**
- $1.00 per 1M requests
- Free tier: **1M requests/month for the first 12 months only**
- No idle cost

**REST API:**
- $3.50 per 1M requests
- Same 1M free-tier
- Optional caching: $0.02-$3.80/hour depending on cache size

**For order-d20 at 10 users:** $0/month, easily.

**Watch out:** Data transfer charges still apply for response bytes returned to the internet ($0.09/GB).

---

## Common gotchas

**`{"message":"Not Found"}` from a freshly-created route**
The route exists but no integration is attached. API console → Routes → click the route → Attach integration.

**`{"message":"Internal Server Error"}` after invocation**
Usually a Lambda error. Check CloudWatch Logs for the function. Could be:
- Permissions error (Lambda execution role missing something)
- Code exception
- Lambda returning wrong response shape (must be `{"statusCode": ..., "body": "..."}`)

**CORS preflight (OPTIONS) fails but GET works in Postman**
Browsers issue a preflight `OPTIONS` request before the real call. Either configure CORS in the API console, or add an `OPTIONS /api/{proxy+}` route that returns CORS headers.

**Lambda receives a string when you expected JSON**
`event["body"]` is always a string. JSON-decode it yourself: `body = json.loads(event["body"] or "{}")`.

**v1.0 event format breaks code expecting v2.0**
HTTP APIs default to v2.0. REST APIs use v1.0. The field names differ (`requestContext.identity.sourceIp` vs `requestContext.http.sourceIp`). Stick with HTTP API + v2.0 for new projects.

**Path params disappear**
Route must use `{paramName}` (curly braces). `event["pathParameters"]` is `None` if there are no params; defensive `event.get("pathParameters") or {}` avoids `AttributeError`.

**Stage `$default` doesn't show up in REST API console**
That's an HTTP API feature only. REST API requires named stages with explicit deploys.

**API Gateway charges even for 4xx/5xx responses**
You pay per request regardless of outcome. Don't expose the API URL publicly without throttling configured if abuse is a concern.

---

## When API Gateway is NOT the right answer

| Scenario | Better choice |
|---|---|
| Internal-only API (in VPC) | ALB or VPC link |
| One Lambda + you don't care about routing | Lambda Function URL (cheaper, simpler) |
| GraphQL | AppSync |
| Massive scale + cost-sensitive | ALB → Lambda (cheaper above ~30M req/mo) |
| Need WebSocket | API Gateway WebSocket APIs (separate API type) or AppSync |
