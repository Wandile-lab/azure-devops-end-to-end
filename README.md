# azure-devops-end-to-end — CI/CD & gated deployments (GitHub Actions)

## Summary
An end-to-end CI/CD pipeline using GitHub Actions that reproduces Azure DevOps–style governance: gated deployments, environment-scoped secrets, human approvals, and auditable artifacts without requiring a cloud subscription. The pipeline demonstrates CI, gated CD, environment-scoped secrets, human approvals, and verifiable artifacts.

GitHub → primary runner and workflows.
Azure DevOps → conceptual mapping for enterprise readers.
Microsoft Azure → referenced as the original target in the project (no subscription available).

```md
**Enterprise equivalence:** 
This project mirrors common Azure DevOps controls (environments, approvals, service connections) using GitHub-native primitives, making it suitable for governance demos and learning without Azure access.
```

## Quick Start 

```bash
git clone https://github.com/<you>/azure-devops-end-to-end
cd azure-devops-end-to-end
git push origin main
```

## What I built (TL;DR)

CI: YAML-based workflow that runs on push/PR, installs deps, runs tests, fails fast.

Policy check: tiny required-template-check job verifies template.yaml exists and parses YAML.

Branch protection: main protected so merges require the required-template-check status to pass and a PR review.

CD (gated): separate deploy-pipeline workflow with build → deploy_test → deploy_prod jobs. deploy_prod uses environment: production so it pauses for explicit approval.

Secrets & scope: production secrets are stored as environment secrets (not repo-level) and only released to runs that target the environment after approval.

Artifact proof: deployment writes DEPLOY_SUMMARY.txt and uploads it as an artifact for auditors/reviewers to download.

## Why this matters

Keeps the same control plane as Azure DevOps: automated checks + human gate before production.

Demonstrates secure secret scoping, auditable approvals, and deterministic workflows without needing a cloud subscription.

Easily switchable to a real cloud auth method later (OIDC / service principal) with minimal YAML changes.

## Key decisions (and the why)

Split CI and CD (separate workflows) — CI validates code and policy; CD performs deploys. Separating simplifies debugging and aligns with best practice.

Use GitHub Environments + environment secrets — enforces approvals and restricts secrets to approved runs. Mirrors Azure environment scoping.

Small, focused jobs — fail fast, quick feedback (unit tests on push, heavier integration runs scheduled or on PR).

Demo fallback mode — without Azure subscription, deploy job conditionally does a harmless demo deploy (DEMO secret) and creates an artifact. This preserves the entire governance flow for demos.

Pin major versions of actions (e.g., actions/checkout@v4, actions/setup-python@v4) — reduce surprise breakage.

## Challenges (and fixes)

No Azure subscription → I initially planned on using Azure. Fix: implement demo deploy fallback and environment-scoped secrets so the pipeline behavior is identical for governance testing.

YAML indentation error (python heredoc) → Actions rejected the file. Fix: correct indentation and use actions/setup-python so Python blocks run cleanly.

template.yaml not detected → check file path; it must live at the path the validator expects (root by default). Fix: add template.yaml to repo root on main.

Policy enforcement gap → branch-protection was added after the workflow had an initial run (status checks must exist first). Fix: commit and run the validator before adding branch protection.

## How it works — core files & snippets

1) Template check workflow (.github/workflows/required-template-check.yml)

   Runs on push/PR to main

   Ensures template.yaml exists and parses with pyyaml

   Fail-fast status that becomes a required status check for main

2) Deploy pipeline (.github/workflows/deploy-pipeline.yml)

   Jobs: build → deploy_test (environment: test) → deploy_prod (environment: production)

   deploy_prod has environment: production so GitHub pauses and requires approver action

   deploy_prod loads secrets from the production environment and runs:

   Demo path when AZURE_SUBSCRIPTION_ID=DEMO

   Real path (commented) to replace with az commands when real creds available

   Uploads demo-deploy/DEPLOY_SUMMARY.txt via actions/upload-artifact@v4

## Security & best practices 

- Never echo secrets to logs. Always write secrets to $GITHUB_ENV or use them directly in commands that don’t print them.

- Use environment secrets for production, not repo-level secrets — environment secrets are only released to runs that target the environment and are gated by approvals.

- Prefer OIDC / workload identity federation over long-lived client secrets when you have control of the Azure tenant. If OIDC is not possible, use a least-privilege service principal and store its secret as an environment secret.

- Pin actions to a major version (e.g., @v4) to avoid unexpected breaking changes. Consider SHA pinning for critical workflows.

- Require a run before branch protection — status checks must exist (have run at least once) before they can be selected in branch protection.

- Least privilege for RBAC — scope cloud roles (resource-group level) rather than subscription-wide Owner where possible.

- Artifact metadata — include branch, commit SHA, and github.run_id in DEPLOY_SUMMARY.txt so artifacts are self-describing.

## Mapping to Azure DevOps (quick reference)

|Azure DevOps                            |Concept	GitHub Actions equivalent
|----------------------------------------|-----------------------------------------------------------------------------------
|Pipeline (YAML)	                       |Workflow YAML (.github/workflows/*.yml)
|Environments → Approvals & checks	     |GitHub Environments + required reviewers
|Service Connection (ARM)	               |OIDC federation (azure/login) or environment secrets (service principal fallback)
|Branch policies: Build validation	     |Branch protection → required status checks
|Pipeline artifacts	                     |actions/upload-artifact

## How to reproduce 

- Add required-template-check.yml and commit to main → run it.

- Add branch protection for main and require required-template-check status + 1 PR review.

- Create test and production environments under Settings → Environments. Add required reviewers for production.

- Add environment secrets for production (e.g., AZURE_* = DEMO).

- Add deploy-pipeline.yml with environment: set on deploy jobs. Commit.

- Approve the paused deploy_prod run and inspect logs and artifact.

## Evidence & what I produced

- Pipeline run that demonstrates: build, test, test-deploy, prod-deploy (paused), approval, resume, artifact upload.

- demo-deploy/DEPLOY_SUMMARY.txt artifact containing deployment timestamp 
