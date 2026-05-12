# AWS Personal Setup Guide

A step-by-step walkthrough for setting up a fresh AWS account + local tooling for a personal serverless project. Optimized for **WSL2 Ubuntu** but most steps work on any Linux/macOS.

Following this guide end-to-end takes ~45 minutes (including waits) and costs **$0** (free tier covers everything we install).

---

## Prerequisites

You should already have:

- **AWS account** created at https://aws.amazon.com/ (the root email/password combo)
- **Authenticator app** on your phone (LastPass Authenticator, Google Authenticator, Authy, etc.)
- **WSL2 Ubuntu** (or any Linux box) with terminal access
- **Node.js** installed (via NVM is fine — `node -v` should return v18+)
- **Python 3.10+** (`python3 --version`)

Verify:
```bash
node --version    # v18+
python3 --version # 3.10+
```

---

## Phase 0a: Account Hardening (Console)

**Do these in the AWS Console before installing anything locally.** This is the part that prevents bill shock + account takeover.

### Step 1: Set a spending limit FIRST

Before *anything* else. Takes 2 minutes.

1. Sign in to the AWS Console as **root** (you'll switch off root in step 3)
2. Search **"Billing"** in the top search bar → open **Billing and Cost Management**
3. Left sidebar → **Budgets** → **Create budget**
4. **Use a template (simplified)** → **Monthly cost budget**
5. Budget amount: **$5** (raise later if you actually deploy real workloads)
6. Email: your email
7. Create

You'll now get an email at 85% / 100% / 100%-forecasted of the threshold. Cheap insurance.

**Also enable Free Tier alerts:** Billing → **Billing preferences** → check **"Receive Free Tier usage alerts"** → save.

### Step 2: Enable MFA on the root account

Root account with no MFA = single biggest "I lost my AWS account" risk.

1. Top-right account name → **Security credentials**
2. Find **Multi-factor authentication (MFA)** → **Assign MFA device**
3. Device name: anything (e.g. `phone-mfa`)
4. Pick **Authenticator app**
5. Scan the QR code with LastPass Authenticator (or whichever app) → enter **two consecutive** 6-digit codes → submit

**Important:** Don't store the MFA seed in the same password vault as your AWS password. If your vault is compromised, both factors are gone. Either use a dedicated authenticator app on your phone, or a hardware key (YubiKey).

### Step 3: Create an IAM admin user (stop using root)

Root is nuclear. Use it only for billing + account-level operations. For everything else, use an IAM user.

1. Search **IAM** → left sidebar **Users** → **Create user**
2. Username: pick one (e.g. `<your-name>`)
3. Check **Provide user access to AWS Management Console** → **I want to create an IAM user**
4. Set a password (different from root)
5. Next → **Attach policies directly** → check **`AdministratorAccess`**
6. Create user
7. On the success screen, **copy the sign-in URL** — looks like:
   ```
   https://<account-id>.signin.aws.amazon.com/console
   ```
   **Bookmark it.** This is your daily sign-in URL going forward.

### Step 4: Enable MFA on the IAM user

Same process as step 2, but for the IAM user.

1. **Sign out of root** → sign in to the IAM URL as your new IAM user
2. Top-right username → **Security credentials**
3. **Assign MFA device** → Authenticator app → scan QR → submit two codes

### Step 5: Switch your default region to us-east-1

Most AWS tutorials assume **us-east-1 (N. Virginia)**. It also has the cheapest data egress and most service support.

Top-right region dropdown → **US East (N. Virginia)**. Bookmark this — every console action defaults to whatever region is selected.

### (Optional) Step 6: Set an account alias

Replaces the ugly 12-digit ID in your sign-in URL.

IAM → **Dashboard** → top-right **Account Alias** → **Create**.

URL becomes `https://<your-alias>.signin.aws.amazon.com/console` — much easier to remember.

---

## Phase 0b: Local tooling (WSL/Ubuntu)

Three tools to install: **AWS CLI**, **AWS CDK**, **boto3** (in a Python venv).

### Step 1: AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
unzip /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install
aws --version
```

Expected output: `aws-cli/2.x.x Python/3.x.x Linux/...`

### Step 2: AWS CDK (Node tool — works for any language)

CDK CLI is a Node tool even when your CDK app is written in Python.

```bash
npm install -g aws-cdk
cdk --version
```

Expected output: `2.x.x (build ...)`

### Step 3: Python venv + boto3 + CDK libs

Modern Ubuntu/Debian (PEP 668) blocks system-wide pip. **Always use a venv.**

**One-time setup:**
```bash
sudo apt install -y python3-pip python3.14-venv  # adjust version: python3.12-venv, python3.13-venv, etc.
```

> Match the venv package to your `python3 --version`. If you have Python 3.12, install `python3.12-venv`. If 3.14, `python3.14-venv`.

**Per-project setup:**
```bash
cd ~/projects/<your-project>
python3 -m venv .venv
source .venv/bin/activate
pip install boto3 aws-cdk-lib constructs
```

Verify:
```bash
python3 -c "import boto3; print(boto3.__version__)"
```

**Activate the venv whenever you work on this project:**
```bash
source ~/projects/<your-project>/.venv/bin/activate
```

Add `.venv/` to your `.gitignore` — never commit it.

### Step 4: Create an access key + configure the CLI

1. Console (signed in as your IAM user) → top-right username → **Security credentials**
2. Scroll to **Access keys** → **Create access key**
3. Use case: **Command Line Interface (CLI)** → check the acknowledgement → Next → Create
4. **Copy both values immediately** (you only see the secret once)

Then locally:
```bash
aws configure
```

Paste when prompted:
- **AWS Access Key ID:** `<paste>`
- **AWS Secret Access Key:** `<paste>`
- **Default region name:** `us-east-1`
- **Default output format:** `json`

These get saved to `~/.aws/credentials` and `~/.aws/config`. Never commit those files.

**Test it works:**
```bash
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/<your-username>"
}
```

If you see your account ID + IAM user ARN, you're authenticated.

### Step 5: CDK bootstrap

One-time per account/region. Creates a small S3 bucket + IAM roles CDK needs to deploy stacks. Costs ~$0.

```bash
cdk bootstrap aws://<your-account-id>/us-east-1
```

Get your account ID from `aws sts get-caller-identity` if you forget. Takes ~2 minutes.

---

## Verification Checklist

Run through this to confirm everything's wired up before you start building:

- [ ] $5 budget alert configured (Billing → Budgets)
- [ ] Free Tier usage alerts enabled
- [ ] Root MFA enabled (Security credentials shows "Assigned" MFA device)
- [ ] IAM admin user created with `AdministratorAccess`
- [ ] IAM user MFA enabled
- [ ] Signed in as the IAM user (top-right shows username, not "root user")
- [ ] Region set to **us-east-1** (top-right dropdown)
- [ ] (Optional) Account alias set
- [ ] `aws --version` returns 2.x
- [ ] `cdk --version` returns 2.x
- [ ] `python3 -m venv .venv` works in a project folder
- [ ] `aws sts get-caller-identity` returns your IAM user ARN
- [ ] `cdk bootstrap` completed for us-east-1

If everything's checked, you're ready to write CDK stacks.

---

## Cost Guardrails (Don't Skip)

| Setting | Why | Where |
|---|---|---|
| **Budget at $5/mo** | Catches runaway costs early | Billing → Budgets |
| **Free Tier alerts** | Warns at 85% of free tier limits | Billing → Billing preferences |
| **Never enable NAT Gateway "to see what it does"** | $32/mo idle, even with zero traffic | VPC console |
| **CloudWatch log retention: 7 days** | Default is "never expire" — expensive at scale | CloudWatch → Log groups → set retention per group |
| **Default region: us-east-1** | Cheapest egress; most service support | Top-right dropdown |

---

## Common "Why doesn't this work?" Gotchas

**`aws configure` says command not found**
The install added `/usr/local/bin/aws` — make sure that's in your `$PATH`. Open a new shell tab if needed.

**`pip install boto3` errors with "externally-managed-environment"**
PEP 668 — you forgot to activate the venv. Run `source .venv/bin/activate` first.

**`python3 -m venv .venv` says "ensurepip is not available"**
Install the venv package matching your Python version: `sudo apt install python3.14-venv` (substitute your version).

**`cdk bootstrap` says "no credentials"**
Run `aws sts get-caller-identity` first to confirm CLI auth works. If that fails, re-run `aws configure`.

**Region dropdown keeps reverting to a non-us-east-1 region**
Some services (Bedrock, certain Lambda triggers) are region-specific — always check top-right before clicking around in the console.

**IAM user sign-in URL got lost**
It's `https://<your-account-id>.signin.aws.amazon.com/console`. Account ID is in `aws sts get-caller-identity` output, or top-right of console when signed in as root.

---

## What NOT to Do

- **Don't keep using the root account.** Use the IAM user. Root only for billing, account closure, and a few account-level operations.
- **Don't commit `~/.aws/credentials` or `~/.aws/config`.** They hold your access key + secret in plaintext. Even private repos.
- **Don't commit `.venv/`.** Big, OS-specific, recreatable.
- **Don't store the MFA seed in the same vault as your AWS password.** Defeats the point of 2FA.
- **Don't enable NAT Gateway or VPN to "see what it does."** Idle cost ~$32/mo.
- **Don't pick a non-us-east-1 region as your default.** Unless you have a specific reason — tutorials and Stack Overflow answers assume us-east-1.

---

## What's Next

Once Phase 0 is complete, you have a working AWS account + local CDK environment. Typical next steps for a serverless web app:

1. **Phase 1:** Static frontend on S3 + CloudFront
2. **Phase 2:** First Lambda + API Gateway endpoint
3. **Phase 3:** DynamoDB tables
4. **Phase 4:** Wire all API routes through Lambda
5. **Phase 5:** Auth (Cloudflare Access or Cognito)
6. **Phase 6:** Domain + HTTPS (Route 53 + ACM)
7. **Phase 7:** CI/CD via GitHub Actions + OIDC (no static AWS keys in repo)
8. **Phase 8:** CloudWatch dashboards + alarms

---

## Per-service guides

Once Phase 0 is done, see [`services/`](./services/) for per-service walkthroughs (console + CLI commands, security defaults, pricing notes, common gotchas):

- [**S3**](./services/s3.md) — object storage; bucket creation, file upload, security, pricing
- [**CloudFront**](./services/cloudfront.md) — CDN; distribution setup, OAC, cache invalidation, default-root-object gotcha
- [**Lambda**](./services/lambda.md) — functions-as-a-service; runtimes, handler shape, CLI vs console deploys, paste-indent gotcha
- [**API Gateway**](./services/api-gateway.md) — HTTP API vs REST API decision, routes/integrations/stages, v2.0 event format

(More added as I touch each service.)

---

## References

- AWS CLI install: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- AWS CDK Python guide: https://docs.aws.amazon.com/cdk/v2/guide/work-with-cdk-python.html
- IAM best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- AWS Free Tier: https://aws.amazon.com/free/
- PEP 668 (why pip is locked down): https://peps.python.org/pep-0668/
