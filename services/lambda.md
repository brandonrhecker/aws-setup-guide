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
