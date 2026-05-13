# IAM (Identity and Access Management)

The "who can do what" layer of AWS. Every API call to every AWS service goes through IAM.

You won't usually create a dedicated *IAM service* the way you create an S3 bucket — but understanding execution roles, managed vs. inline policies, and least-privilege scoping is what separates a "I clicked Administrator and shipped it" build from a credible one.

**What you'll actually do day-to-day:**
- Attach the right execution role to each Lambda
- Write policies that allow exactly the actions a function needs, on exactly the resources it touches
- Swap a too-broad managed policy for a scoped inline policy when you tighten security
- Understand why an `AccessDenied` happened (it's almost always one missing action on one resource ARN)

---

## Concepts

| Term | Meaning |
|---|---|
| **User** | A long-lived identity for a human (you). Has a password and/or access keys. Should have MFA. |
| **Role** | A short-lived identity another AWS service assumes. Lambda functions, EC2 instances, ECS tasks all use roles. |
| **Policy** | A JSON document listing allowed (or denied) actions on resources. Attached to users, groups, or roles. |
| **Managed policy** | A reusable policy with an ARN. Can be AWS-managed (built-in like `AmazonDynamoDBReadOnlyAccess`) or customer-managed (your own reusable policy). |
| **Inline policy** | A policy embedded directly into a single user/role. No ARN, no reuse. Use for one-off scoping. |
| **Trust policy** | A special policy on a role that says *which services or accounts can assume it*. Lambda's role trusts `lambda.amazonaws.com`. |
| **Resource ARN** | The unique name of an AWS thing — e.g. `arn:aws:dynamodb:us-east-1:<account-id>:table/my-table`. Policies allow actions on specific ARNs. |
| **Action** | A specific API call, namespaced by service — e.g. `dynamodb:GetItem`, `s3:PutObject`. Wildcards allowed but discouraged. |
| **Effect** | `Allow` or `Deny`. Default is implicit Deny — you have to actively allow what you need. |
| **Principal** | *Who* the policy applies to. On role trust policies and resource policies; not on identity-based policies. |

---

## The mental model

Every API request:

1. **Who is making this request?** (the principal — a user, a role assumed by a service)
2. **What action are they trying to take?** (e.g. `dynamodb:PutItem`)
3. **On which resource?** (e.g. `arn:aws:dynamodb:us-east-1:<account-id>:table/sessions`)
4. **Is there an `Allow` statement that matches all three, with no `Deny` overriding it?**

If yes → allowed. If no → `AccessDenied`.

That's the entire model. Every IAM mistake reduces to: a missing action, a too-narrow resource ARN, or an explicit Deny somewhere.

---

## Lambda execution roles

When you create a Lambda in the console, AWS offers to auto-create a role. Take it — but understand it:

```
role: my-function-role-<random>
└── AWSLambdaBasicExecutionRole (managed)
       allows: logs:CreateLogStream, logs:PutLogEvents on this function's log group
```

That's the minimum: just enough to write to CloudWatch Logs. If your function needs DynamoDB, S3, or anything else, you have to add it.

**Three ways to add permissions**, in order of preference for a real build:

1. **Inline policy on the role** — scoped to specific resource ARNs. Best for least-privilege.
2. **Customer-managed policy attached to the role** — same scoping, but reusable across multiple roles.
3. **AWS-managed policy attached to the role** — fastest to wire up; almost always too broad.

---

## The "drop the managed, add the inline" pattern

A common first-build mistake is attaching `AmazonDynamoDBReadOnlyAccess` to a Lambda role to "just make it work." That grants read access to *every* DynamoDB table in the account — even ones the function should never see.

Tighten it like this:

### 1. Write the inline policy

`/tmp/my-app-ddb-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MyAppDDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:BatchWriteItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:<account-id>:table/my-app-characters",
        "arn:aws:dynamodb:us-east-1:<account-id>:table/my-app-sessions",
        "arn:aws:dynamodb:us-east-1:<account-id>:table/my-app-events"
      ]
    }
  ]
}
```

### 2. Detach the managed policy

```bash
aws iam detach-role-policy \
  --role-name my-function-role-abc123 \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess
```

### 3. Attach the inline policy

```bash
aws iam put-role-policy \
  --role-name my-function-role-abc123 \
  --policy-name MyAppDDBAccess \
  --policy-document file:///tmp/my-app-ddb-policy.json
```

### 4. Verify the role's final state

```bash
aws iam list-attached-role-policies --role-name my-function-role-abc123
aws iam list-role-policies          --role-name my-function-role-abc123
```

The first lists managed policy attachments (should now be just `AWSLambdaBasicExecutionRole-*`), the second lists inline policy names (should now include `MyAppDDBAccess`).

The function keeps working but the blast radius of a Lambda compromise drops from "read everything" to "read/write three specific tables." This is the kind of swap that matters in a real security review.

---

## Managed vs. inline — when to pick which

| Use managed (AWS or customer) | Use inline |
|---|---|
| Many roles need the same permissions | One role, one specific scope |
| You want a named, versioned policy you can audit centrally | You want the policy to live and die with the role |
| You're using IaC (CDK/Terraform) and want a shared policy resource | Quick one-off tightening before IaC lands |
| The policy is genuinely reusable (e.g. "developers read-only") | The resource ARNs are unique to this function |

**Heuristic:** start inline. If you find yourself copy-pasting the same JSON into a second role, promote it to a customer-managed policy.

---

## Wildcards: when they're OK and when they're not

```json
"Action": "dynamodb:GetItem",
"Resource": "arn:aws:dynamodb:us-east-1:<account-id>:table/my-app-*"
```

vs.

```json
"Action": "dynamodb:*",
"Resource": "*"
```

The first is fine — it scopes to your own tables only. The second is the same as `AdministratorAccess` for DynamoDB and should never appear in a Lambda role.

**Rule of thumb:**
- **Wildcard in Action**: only if you genuinely need every action on the resource (e.g. an admin Lambda that does table management). Otherwise enumerate.
- **Wildcard in Resource**: only if scoped by name prefix (`table/my-app-*`) or path prefix (`bucket/my-app-bucket/*`). Never bare `"*"` in app code.

---

## Common gotchas

**`AccessDenied` but the action looks right in the policy**
Check the resource ARN region + account ID. A policy allowing `arn:aws:s3:::my-bucket` won't match an attempt to read from a different bucket name, even if the names are similar.

**Policy edit doesn't take effect**
IAM is *eventually consistent*. After updating a policy, allow 5-10 seconds before the next API call sees the change.

**`iam:PassRole` denied**
Some services (Lambda, ECS, CodeBuild) require the *caller* to have permission to pass a role to the service. If you can't attach a role to a new Lambda via CLI, you're missing `iam:PassRole` on your own user.

**Implicit deny vs. explicit deny**
- Implicit deny = "no Allow statement matched." Fix by adding an Allow.
- Explicit deny = "a Deny statement matched." Wins over any Allow — even an Admin policy. Used by SCPs (Service Control Policies) in Organizations to lock things down.

**Managed policy attachments cap out at 10 per role**
You'll hit this trying to assemble a role from AWS-managed pieces. Switch to a customer-managed policy that combines what you need.

**MFA-protected actions**
You can add `aws:MultiFactorAuthPresent` conditions to deny dangerous actions unless the caller authenticated with MFA. Useful for protecting `iam:*` actions on your own user.

---

## CLI cheat sheet

| Task | Command |
|---|---|
| Who am I right now? | `aws sts get-caller-identity` |
| List a role's attached managed policies | `aws iam list-attached-role-policies --role-name <role>` |
| List a role's inline policies | `aws iam list-role-policies --role-name <role>` |
| Read an inline policy | `aws iam get-role-policy --role-name <role> --policy-name <name>` |
| Attach managed policy | `aws iam attach-role-policy --role-name <role> --policy-arn <arn>` |
| Detach managed policy | `aws iam detach-role-policy --role-name <role> --policy-arn <arn>` |
| Add/update inline policy | `aws iam put-role-policy --role-name <role> --policy-name <name> --policy-document file://policy.json` |
| Delete inline policy | `aws iam delete-role-policy --role-name <role> --policy-name <name>` |
| Simulate a policy (dry-run) | `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names dynamodb:GetItem --resource-arns <table-arn>` |

---

## Pricing

IAM is free. Policy evaluation is free. Logging IAM events to CloudTrail does cost — but only for *data events* (S3 object reads, Lambda invocations), not management events.

---

## When IAM is NOT the answer

| Scenario | Better choice |
|---|---|
| Granting access to external users (customers) | Cognito User Pools |
| Federating from your company's identity provider | IAM Identity Center (formerly SSO) |
| Cross-account access between AWS accounts | IAM roles with trust policy + AWS Organizations |
| Application-level authorization (this user can edit this resource) | Your app's own logic — IAM is for AWS API calls, not app domain logic |
