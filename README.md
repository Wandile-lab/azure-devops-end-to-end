# azure-devops-end-to-end
Rough Draft


I configured a YAML-based CI that triggers on push/PR, uses hosted runners to install dependencies, starts required services (when needed), runs the test suite, and fails fast so broken code doesn’t reach main. I secure credentials via repository secrets and keep jobs small for fast feedback.

Azure DevOps Environments and Approvals were implemented using GitHub Actions Environments due to lack of Azure subscription. The same deployment gating, secret scoping, and human approval controls were demonstrated and verified via production deployment pause and artifact generation

I used GitHub Environments to gate deployments: jobs that target environment: production require named reviewers and the CI workflow to pass; branch protection enforces required status checks so nothing merges without green CI.

I wrote a CI gate: a small Actions workflow checks template.yaml exists and parses; branch protection requires that workflow to pass before merging to main. It gives the same policy enforcement as Azure's required-template check but in GitHub’s model.

I kept running into errors when running the workflow and at first, the issue was caused by incorrect indentation of the python block which caused it not to wrap correctly  so the Actions rejected the whole workflow

Another issue i ran into was the  workflow not detecting the template.yaml so i created in the repo root on main and re-ran the check. 

Next I made GitHub block merges into main unless that tiny CI job (the template validator) is green and a reviewer has approved the PR by implementing the check as an Actions workflow and enforcing it via branch protection in the repo settings of GitHub.

I gated deployments by targeting GitHub Environments from deployment jobs. Test deploys run automatically; prod deploys pause until an approved reviewer authorizes the job—same control as Azure DevOps environment approvals.

I split CI and CD. CI validates repo policy and templates. CD is a separate workflow with test and production deployment jobs; production targets a protected GitHub Environment so deployments pause until approval.

I replaced the Azure DevOps service connection with GitHub Actions + workload identity federation (OIDC). I configured an Entra app with a federated credential for the repo, assigned a scoped role, and used azure/login to exchange a short-lived OIDC token for Azure access — no long-lived secrets. WHY: Environment secrets are only released to runs that target the environment and are gated by approvals. That mirrors Azure’s service-connection scoping.

I implemented a gated CD job in GitHub Actions that targets a protected production environment and conditionally performs a demo deploy when cloud creds are absent, preserving the same approval & audit model as Azure DevOps

I gated production with a protected GitHub Environment; deployment jobs target environment: production so runs pause until an approver authorizes the deployment—then the job continues with the scoped secrets, preserving auditability and least privilege.

I added a gated CD job that produces an artifact on successful deploy so stakeholders can download the build output; this preserves auditability and mirrors Azure DevOps artifacts

SUmmary:
I implemented gated CD: jobs target GitHub Environments so deployments pause for human approval; post-approval they run with scoped environment secrets and produce an artifact naming the run — mirroring Azure DevOps release governance.”
(GitHub ↔ Azure DevOps)
