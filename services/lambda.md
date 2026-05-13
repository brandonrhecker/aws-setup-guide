# Lambda (Functions-as-a-Service)

Run code without managing servers. You upload a function (Python, Node, Go, Java, etc.), AWS runs it on demand when triggered — by an HTTP request via API Gateway, a file landing in S3, a DynamoDB stream event, a scheduled cron, etc.

**What it's good for:**
- HTTP API backends behind API Gateway
- Event-driven processing (resize image when uploaded to S3; react to DynamoDB changes)
- Scheduled jobs (cron replacement)
- Glue code between AWS services
- Anything where idle = $0 matters more than constant low-latency

**What it's NOT good for:**
- Workloads >15 min (Lambda hard max — use Step Functions or ECS for longer)
- Sustained high traffic where cold starts hurt UX (consider provisioned concurrency, or just use containers)
- WebSocket servers with persistent connections (use API Gateway WebSocket APIs or ECS)
- GPU workloads (use SageMaker / EC2)

---

## Concepts

| Term | Meaning |
|---|---|
| **Function** | The unit. A name, a runtime, code, an execution role, and config (memory, timeout). |
| **Runtime** | Language environment AWS maintains (`python3.14`, `nodejs22.x`, `provided.al2023` for custom runtimes). |
| **Handler** | The entrypoint AWS calls — string like `module.function_name`. Default for Python: `lambda_function.lambda_handler`. |
| **Event** | JSON payload Lambda receives describing what triggered the invocation. Shape depends on the trigger (API Gateway v2 event ≠ S3 event ≠ scheduled event). |
| **Context** | Object Lambda passes alongside event — request ID, time remaining, log group, etc. Rarely needed. |
| **Execution role** | IAM role the function assumes. Defines *what AWS resources* the function is allowed to touch. |
| **Layers** | Reusable code/library packages shared across multiple functions. Up to 5 layers per function. |
| **Cold start** | First invocation of a function (or after idle) requires AWS to spin up a new execution environment. Adds 100ms-2s depending on runtime + package size. |
| **Warm invocation** | Subsequent calls within ~5-15 min reuse the same execution environment. Single-digit ms overhead. |

---

## Creating a function — Console

1. Top search bar → **Lambda** → **Create function**
2. **Author from scratch** (skip blueprints and container images for now)
3. **Function name:** descriptive (e.g. `myproject-api`)
4. **Runtime:** latest **Python 3.x** under "Latest supported"
5. **Permissions → Custom settings** (expand):
   - Leave **Custom execution role** OFF → AWS creates a new role with **basic Lambda permissions** (CloudWatch Logs write only)
   - We'll attach more policies later when the function needs DynamoDB/S3/Bedrock
6. **Architecture:** **x86_64** (ARM64 is slightly cheaper but a few libs are still x86-only)
7. Leave **Function URL**, **VPC**, **Code signing**, **KMS** off
8. **Create function**

You land on the function detail page with a **default Hello World handler** already deployed.

---

## Editing code — two ways

### In-console editor (small functions only)

1. **Code** tab → edit `lambda_function.py` in the browser
2. Click **Deploy** ← THIS is what publishes the code. Just typing saves a draft.
3. Then **Test** to invoke

**Gotcha — paste indents:** The in-console editor sometimes adds stray leading whitespace when pasting Python code from chat / docs. Symptom: `SyntaxError: unexpected indent` on line 1. Workaround: select-all + delete the entire file first, retype, or use the CLI upload below.

### CLI upload (the real-world way)

```bash
# 1. Write your code locally
mkdir -p lambda_handlers
# (edit lambda_handlers/lambda_function.py)

# 2. Zip it (use Python's zipfile if `zip` isn't installed)
cd lambda_handlers
python3 -c "import zipfile; zipfile.ZipFile('lambda_function.zip','w').write('lambda_function.py')"
# Or: zip lambda_function.zip lambda_function.py

# 3. Upload
aws lambda update-function-code \
  --function-name myproject-api \
  --zip-file fileb://lambda_function.zip
```

For functions with dependencies (`boto3` is built into Lambda, but say you need `requests`):

```bash
pip install requests -t .   # install into current dir
zip -r lambda.zip .          # zip everything
aws lambda update-function-code --function-name myproject-api --zip-file fileb://lambda.zip
```

For large deps (>10 MB), use **Lambda Layers** instead.

---

## Handler shape for API Gateway HTTP API (v2.0 events)

```python
import json

def lambda_handler(event, context):
    # event is the API Gateway v2 payload — contains:
    #   event["requestContext"]["http"]["method"]   -> "GET", "POST", etc.
    #   event["rawPath"]                            -> "/api/characters"
    #   event["pathParameters"]                     -> {"slug": "..."}  (if you used path params)
    #   event["queryStringParameters"]              -> {"foo": "bar"}
    #   event["body"]                               -> request body (string; JSON-decode it)

    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",  # CORS
        },
        "body": json.dumps({"message": "ok"}),
    }
```

The response shape is exactly: `statusCode`, `headers` (optional), `body` (must be a string — JSON-encode dicts yourself).

For non-API triggers (S3 events, CloudWatch scheduled events, DynamoDB streams), the event shape is different. Print `event` once and check CloudWatch Logs to see what you actually get.

---

## Testing

### Console
**Test** tab → **Create new event** → name it → leave the default template JSON → **Invoke**. Response box shows return value + CloudWatch logs inline.

### CLI
```bash
aws lambda invoke \
  --function-name myproject-api \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  /tmp/lambda-out.json && cat /tmp/lambda-out.json
```

The `--cli-binary-format raw-in-base64-out` flag is needed in AWS CLI v2 — without it, the payload arg is base64-decoded.

---

## Execution role basics

Every function has an IAM role it assumes when invoked. Console default = `AWSLambdaBasicExecutionRole` (CloudWatch Logs only).

To grant DynamoDB read access (Phase 3 of order-d20):
1. Lambda console → function → **Configuration** tab → **Permissions** → click the role name (opens IAM)
2. **Add permissions** → **Attach policies** → search `AmazonDynamoDBReadOnlyAccess` → attach

**Resist the urge to attach `AdministratorAccess`.** That's the #1 way junior engineers create AWS data leaks. Lambda runs *attacker-influenced input* (anything that comes through API Gateway is user-controlled). Grant only what each function needs.

For tighter scoping, write an inline policy that allows access only to specific table ARNs.

---

## Common operations cheat sheet

```bash
# Update code
aws lambda update-function-code \
  --function-name myfunc \
  --zip-file fileb://lambda.zip

# Update config (timeout, memory, env vars)
aws lambda update-function-configuration \
  --function-name myfunc \
  --timeout 30 \
  --memory-size 512 \
  --environment 'Variables={STORAGE_BACKEND=dynamodb,LOG_LEVEL=INFO}'

# Invoke
aws lambda invoke \
  --function-name myfunc \
  --cli-binary-format raw-in-base64-out \
  --payload '{"key": "value"}' \
  /tmp/out.json

# Tail logs (live)
aws logs tail /aws/lambda/myfunc --follow

# List functions
aws lambda list-functions --query 'Functions[].FunctionName'

# Get current config
aws lambda get-function-configuration --function-name myfunc

# Delete
aws lambda delete-function --function-name myfunc
```

---

## Pricing

**Free tier (always free, not just first 12 months):**
- 1 million requests/month
- 400,000 GB-seconds compute/month (≈ 3.2 million seconds at 128 MB)

**After free tier (us-east-1, x86_64):**
- **Requests:** $0.20 per million
- **Compute:** $0.0000166667 per GB-second
  - A 128 MB function running 100ms costs `128/1024 * 0.1 * $0.0000166667` ≈ **$0.0000002** per invocation
- **ARM64:** ~20% cheaper than x86_64

**For order-d20 at 10 users:** essentially $0/month.

**At scale:** Lambda gets expensive vs. containers around ~50% CPU utilization sustained. Below that, Lambda is cheaper. Above it, ECS Fargate or EC2 wins.

---

## Common gotchas

**"Hello from Lambda!" still appears after I changed the code**
You typed but didn't click **Deploy** in the console editor. Deploy = save *and* publish to the runtime.

**`SyntaxError: unexpected indent` on line 1**
Paste-into-browser-editor added invisible leading whitespace. Workaround: select-all-delete then retype, or use CLI upload.

**Permission denied talking to DynamoDB / S3 / etc.**
Execution role doesn't have the policy attached. Configuration → Permissions → role name → IAM → Attach policies.

**Function times out at exactly 3 seconds**
That's the default timeout. Configuration → General config → bump to 30s or whatever's needed (max 15 min).

**Cold starts feel slow**
Default 128 MB → small CPU allocation. CPU scales with memory in Lambda. Bump to 512 MB or 1024 MB to make a Python cold start drop from 2s to ~200ms. Costs roughly the same per request because faster = less compute-time billed.

**"Image with x architecture cannot be loaded" / library compatibility**
You picked `arm64` but pip-installed wheels for `x86_64`. Pick one and stick with it. For most pure-Python deps, both work.

**Lambda invocations succeed but nothing logs**
The execution role is missing `logs:CreateLogStream` + `logs:PutLogEvents`. The auto-created basic role includes these. If you swapped roles, re-add `AWSLambdaBasicExecutionRole`.

**Code change deployed but stale code still runs for a few seconds**
Lambda's deploy is technically eventually consistent. Brief overlap (~seconds) between old + new execution environments. Almost never an issue in practice.

**Lambda can read public internet but not VPC-internal resources**
Default: Lambda runs in AWS-managed VPC with internet egress. To reach private RDS/EC2/etc., put Lambda in your VPC — but then you lose internet unless you also add a NAT Gateway ($32/mo idle). Avoid VPC-attached Lambdas unless absolutely required.

**Default timeout is 3 seconds**
Lambda's default function timeout is 3s, which is too short for almost any real workload that talks to an upstream API. Symptom: function returns `Task timed out after 3.00 seconds` while the upstream API was happily about to respond. Bump to 10-30s for typical HTTP-backed work; the Lambda hard max is 15 min. `aws lambda update-function-configuration --function-name <fn> --timeout 30`.

**boto3 DDB resource API returns `Decimal`, not `int`/`float`**
If your function uses `boto3.resource("dynamodb")` (the high-level API) and JSON-encodes the result, numbers come out as `Decimal` objects which `json.dumps` can't serialize. The classic workaround is a custom `default=` function that coerces `Decimal` → `int` (when whole) or `float`. Frontends doing math on event payloads will silently break otherwise — strings concat instead of summing. See [DynamoDB guide](./dynamodb.md) for the encoder snippet.

---

## Pattern: Lambda Layers

Layers are reusable ZIP packages of code/libraries that you attach to one or more Lambda functions. Up to 5 layers per function. Each layer's contents are extracted into `/opt/` at runtime, and the language runtime's import path is set up to find them.

**When to use a Layer:**
- A heavyweight dependency (`requests`, `numpy`, `Pillow`) used across multiple functions
- A shared helper module ("common code" used by every function in your app)
- Anything that bloats the function zip and slows code deploys

**When not to bother:**
- A one-off function — the dependency lives fine in the function zip
- Tiny pure-stdlib code with no third-party deps

### Building a Python Layer

The Layer zip must use a specific directory structure: `python/` at the root (Lambda extracts to `/opt/python` and `PYTHONPATH` is set so imports just work).

```bash
LAYER_DIR=/tmp/my-deps-layer
rm -rf $LAYER_DIR && mkdir -p $LAYER_DIR/python

# Install your deps into python/ instead of the system site-packages.
pip install --target $LAYER_DIR/python requests

# zip from inside the layer dir; the zip root must be python/, not python/...
cd $LAYER_DIR
python3 -c "
import zipfile, os
with zipfile.ZipFile('layer.zip', 'w', zipfile.ZIP_DEFLATED) as z:
    for root, _, files in os.walk('python'):
        for f in files:
            full = os.path.join(root, f); z.write(full, full)
"
```

If you have `zip` installed: `zip -r layer.zip python` works too.

### Publishing + attaching

```bash
aws lambda publish-layer-version \
  --layer-name my-deps \
  --description "requests + transitive deps" \
  --zip-file fileb:///tmp/my-deps-layer/layer.zip \
  --compatible-runtimes python3.14 \
  --compatible-architectures x86_64
# → returns LayerVersionArn

aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:us-east-1:<account-id>:layer:my-deps:1
```

Each `publish-layer-version` creates a *new immutable version* (the `:1` suffix). You don't update a Layer in place — you publish v2, v3, etc., and update the function to point at the new ARN.

### Gotchas

- **Architecture mismatch:** if you `pip install` on a non-Linux/x86_64 machine, some libraries (anything with C extensions — `pydantic-core`, `numpy`, `Pillow`) will install the wrong wheel. Use Docker (`public.ecr.aws/sam/build-python3.14`) or AWS CloudShell to build cross-compat Layers.
- **Total deploy size limit:** function code + all attached Layers must be ≤ 250 MB unzipped (50 MB zipped on direct upload, 250 MB via S3). Stripping `tests/`, `*.dist-info`, and `__pycache__` helps.
- **Layer + function zip override:** if both contain the same module, the function zip wins. Useful for emergency hotfixes; confusing if unintentional.

---

## Pattern: Bundling app code as a package alongside `lambda_function.py`

When your Lambda needs to import shared modules from elsewhere in your codebase (a `lib/` package with serializers, models, etc.), you can't just `pip install` them — they're not on PyPI. Bundle them straight into the function zip as a package.

**Layout in the function zip:**

```
lambda_function.py          ← entrypoint
lib/
  __init__.py
  models.py
  serializers.py
```

**In `lambda_function.py`:**

```python
from lib.models import MyModel
from lib.serializers import to_dict
```

**Build script (run before `update-function-code`):**

```bash
cd lambda_handlers
rm -rf build && mkdir -p build/lib
cp lambda_function.py build/
cp ../shared/lib/__init__.py ../shared/lib/models.py ../shared/lib/serializers.py build/lib/
cd build
python3 -c "
import zipfile, os
with zipfile.ZipFile('../lambda_function.zip', 'w', zipfile.ZIP_DEFLATED) as z:
    for root, _, files in os.walk('.'):
        for f in files:
            full = os.path.join(root, f); z.write(full, os.path.relpath(full, '.'))
"
```

Then `aws lambda update-function-code --zip-file fileb://lambda_function.zip` as usual.

### Why not symlinks?

Symlinks survive zip but Lambda's unzip resolves them at extract — sometimes pointing at paths that don't exist in the runtime. Just copy the files.

### Why not a Layer for app code?

You can — but Layers are immutable per version. App code changes a lot during development; you'd burn through Layer versions and have to update the function each time. Use Layers for stable deps, bundle app code in the function zip.

### Mixing the two

A realistic real-world setup:

- **Layer:** `requests`, `boto3` extras, any third-party deps (rarely changes)
- **Function zip:** `lambda_function.py` + bundled `lib/` package (changes often)

Code deploys are fast (small zip), shared deps don't bloat every function.

---

## Pattern: One Fat Lambda + Internal Dispatch

For small/personal serverless backends, the simplest architecture is **one Lambda that handles every `/api/*` request**, with a single API Gateway route `ANY /api/{proxy+}` pointing at it. Internal dispatch (regex on `event["rawPath"]` + method) routes to per-endpoint handler functions.

**Benefits:**
- One CloudWatch log group to tail
- One execution role to manage
- Warm container shared across endpoints (fewer cold starts overall)
- Adding a route = one entry in a table + one handler function

**Costs:**
- Whole API breaks if the module fails to import (syntax error, missing dep)
- Larger zip if you have many endpoints with diverse deps
- IAM scoping is per-function — granting DDB write to one route means *every* route can write

For order-d20 at 10 users, this is the right shape. At >5 engineers and >50 endpoints, split into per-route Lambdas (or use AWS SAM/CDK to do that for you).

### Template

```python
import json, os, re
import boto3
from boto3.dynamodb.types import TypeDeserializer

DDB = boto3.client("dynamodb")
DESER = TypeDeserializer().deserialize


def _response(status, body):
    return {
        "statusCode": status,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",
        },
        "body": json.dumps(body, default=str),
    }


def list_characters(event, context, params):
    resp = DDB.scan(TableName="my-table")
    return _response(200, [
        {k: DESER(v) for k, v in item.items()}
        for item in resp.get("Items", [])
    ])


def get_character(event, context, params):
    slug = params["slug"]
    resp = DDB.get_item(TableName="my-table", Key={"slug": {"S": slug}})
    if "Item" not in resp:
        return _response(404, {"error": f"not found: {slug}"})
    return _response(200, {k: DESER(v) for k, v in resp["Item"].items()})


# (METHOD, regex with named groups, handler)
ROUTES = [
    ("GET",  r"/api/characters",                 list_characters),
    ("GET",  r"/api/characters/(?P<slug>[^/]+)", get_character),
    # Add more routes here ↑
]


def dispatch(method, path):
    for m, pattern, handler in ROUTES:
        if m != method:
            continue
        match = re.fullmatch(pattern, path)
        if match:
            return handler, match.groupdict()
    return None, None


def lambda_handler(event, context):
    method = event["requestContext"]["http"]["method"]
    path = event["rawPath"]
    handler, params = dispatch(method, path)
    if handler is None:
        return _response(404, {"error": f"no route for {method} {path}"})
    try:
        return handler(event, context, params)
    except Exception as e:
        return _response(500, {"error": str(e), "where": handler.__name__})
```

Pair this with API Gateway: a single `ANY /api/{proxy+}` route → this Lambda. Every request lands at `lambda_handler`, gets dispatched.

---

## When Lambda is NOT the right answer

| Scenario | Better choice |
|---|---|
| Long-running job (>15 min) | Step Functions, ECS, Batch |
| Constant high traffic | ECS Fargate or EC2 (Lambda cost crosses over) |
| WebSocket server | API Gateway WebSocket APIs, AppSync |
| Stateful service | ECS / EC2 |
| GPU inference | SageMaker / EC2 g4/g5 |
| Hot-path low-latency (< 50 ms p99) | ECS with warmed containers |
