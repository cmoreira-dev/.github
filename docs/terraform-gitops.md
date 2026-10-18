# terraform-gitops (reusable workflow)

Plan on pull request, apply on push to the default branch behind an approval, for the
`terraform/<folder>` of a `gitops.<app>` repo. AWS only, runs on GitHub-hosted runners (no LAN).

## One-time setup per repo

1. Add the repo to `repos` in `iac.homelab-live-infra/aws/cmoreira-dev/us-east-1/gitops-terraform-roles/terragrunt.hcl`
   (its `state_key_prefix` and the IAM user name prefixes it may create). Merging applies it.
2. Repo variables: `TF_PLAN_ROLE_ARN` and `TF_APPLY_ROLE_ARN` (outputs `plan_role_arns` / `apply_role_arns` of that unit).
3. Repo environment `production` with required reviewers: that is the approval gate, and the apply role only
   trusts that environment.
4. Add the caller below as `.github/workflows/terraform.yml`.

## Caller

```yaml
name: terraform
on:
  pull_request:
    paths: ["terraform/**"]
  push:
    branches: [main]
    paths: ["terraform/**"]
jobs:
  ecr-pull-iam:
    uses: cmoreira-dev/.github/.github/workflows/terraform-gitops.yml@main
    with:
      directory: terraform/ecr-pull-iam
      plan-role-arn: ${{ vars.TF_PLAN_ROLE_ARN }}
      apply-role-arn: ${{ vars.TF_APPLY_ROLE_ARN }}
```

One job per terraform root. A folder that is not in the caller is never applied by CI.

## Guardrails

- The plan role trusts only the `pull_request` context and is read-only (state read of its own prefix plus IAM reads on its own users; no ReadOnlyAccess, since that can read every state file).
- The apply role trusts only the `production` environment, has no IAM beyond users with its own name prefixes,
  and every user it creates must carry the `gitops-terraform-boundary` policy (a ceiling: ECR read, SSM read,
  product S3 buckets, queues, mail). Extend the boundary deliberately.
- Private repos only: a plan from a fork must never get a role.
- `tofu-version` must match live-infra and platform-bootstrap.
