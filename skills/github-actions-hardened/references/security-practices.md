# Security Practices Reference

Load this file when generating deployment workflows, cloud integrations, or any workflow that handles credentials.

---

## OIDC Authentication (Preferred Over Static Secrets)

OIDC tokens are scoped to a single run and expire in minutes. They leave no long-lived credential to rotate, audit, or accidentally leak.

### AWS

**Step 1 — Create the OIDC provider in AWS IAM (once per account):**
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

**Step 2 — Create an IAM Role with this trust policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:ref:refs/heads/main"
      }
    }
  }]
}
```

Replace `OWNER/REPO` with the actual repository slug. Scope to a specific branch/tag when possible — avoid `repo:OWNER/REPO:*` in production.

**Step 3 — Workflow usage:**

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::ACCOUNT_ID:role/GitHubActionsRole
      role-session-name: GitHubActions-${{ github.run_id }}
      aws-region: us-east-1
```

### Azure

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

Configure a Federated Credential on your Azure App Registration pointing to the GitHub OIDC issuer (`https://token.actions.githubusercontent.com`).

### Google Cloud

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: 'projects/PROJECT_ID/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID'
      service_account: 'sa-name@project-id.iam.gserviceaccount.com'
```

---

## Secrets Management

**Do:**
- Store secrets in GitHub Secrets (repository, environment, or organization level)
- Use environment-specific secrets (`environment: production`) for deployment jobs
- Pass secrets to shell commands via `env:`, never via direct `${{ secrets.X }}` interpolation in `run:`

**Don't:**
- Echo or log secret values
- Store secrets in workflow files or repository code
- Use the same secret across environments with different trust levels

```yaml
# SAFE — value is masked and treated as data by the shell
- name: Deploy
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: ./deploy.sh

# RISKY — may leak in verbose logging or process list
- run: curl -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" https://api.example.com
```

### Masking dynamically generated values

```yaml
- name: Generate ephemeral token
  run: |
    TOKEN=$(generate-token)
    echo "::add-mask::$TOKEN"
    echo "DEPLOY_TOKEN=$TOKEN" >> $GITHUB_ENV
```

---

## Runner Security

**Never use self-hosted runners for public repositories.** Any GitHub user can open a PR that triggers a workflow. On a self-hosted runner that workflow runs on your infrastructure and can steal credentials, access internal networks, or mine crypto.

If self-hosted runners are required (e.g., specific hardware, on-prem access):
- Use them **only for private repositories**
- Run jobs inside ephemeral containers or VMs, not directly on the host
- Require approval for first-time contributors in repository settings
- Never store secrets or credentials directly on the runner machine

---

## Supply Chain Security for Container Workflows

For workflows that build and push container images, consider:

**Image signing with Sigstore/Cosign (keyless):**

```yaml
permissions:
  id-token: write   # Required for keyless signing

steps:
  - uses: sigstore/cosign-installer@v3

  - name: Sign image
    run: |
      cosign sign --yes \
        ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
```

**SBOM generation:**

```yaml
- uses: anchore/sbom-action@v0
  with:
    image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json
```

These are optional additions for compliance-scoped projects; not required for every container workflow.
