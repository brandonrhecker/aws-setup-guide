# CLAUDE.md — aws-setup-guide

Personal walkthrough for spinning up a fresh AWS account + local CDK/CLI tooling on WSL2 Ubuntu. Reusable across future personal AWS projects.

## What this repo is

A single `README.md` that documents end-to-end account setup, billing guardrails, IAM hardening, and local-tooling install (AWS CLI, CDK, Python venv, boto3). Written so a future-me on a fresh machine can repeat the process in ~45 min.

## What this repo is NOT

- Not project-specific. No application code, no CDK stacks. Those live in the *project* repos.
- Not a tutorial on writing CDK apps. Just gets you to the point where `cdk bootstrap` succeeds.

## Conventions

- Use generic placeholders: `<account-id>`, `<your-username>`, `<your-alias>`. Real account IDs/usernames stay out of the committed file.
- Keep steps numbered, code blocks copy-pasteable, and ADHD-skimmable.
- Update when something changes in AWS Console UX or when a step turns out wrong.

## When to update this

- A step breaks because AWS changed the console
- A new gotcha is discovered (add to "Common Gotchas" section)
- A new tool becomes mandatory (e.g. if Anthropic SDK or Bedrock CLI becomes standard for this user)
