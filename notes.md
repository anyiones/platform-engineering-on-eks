# Platform Engineering on EKS — Session Notes

**Date:** 2026-08-12/13
**Workshop:** AWS "Platform Engineering on EKS" (PEEKS reference architecture)
**Event:** aws-experience.com/emea/smb/e/a765a/platform-engineering-on-eks

## 1. Architecture Overview

Hub-and-spoke multi-cluster GitOps platform:

- **Hub cluster (`peeks-hub`)** — control plane, runs platform tooling:
  - Backstage (developer self-service portal)
  - Argo CD (GitOps continuous deployment)
  - Keycloak (SSO / identity provider, federated via AWS IAM Identity Center)
  - kro (Kubernetes Resource Orchestrator)
  - ACK controllers: IAM, EKS, EC2, ECR, S3, DynamoDB
  - Argo Workflows, Kargo, Argo Rollouts, Argo Events
  - GitLab, DevLake, JupyterHub, Ray Operator, Spark Operator

- **Spoke clusters** — workload environments:
  - `peeks-spoke-dev`
  - `peeks-spoke-prod`
  - Each auto-registered with Argo CD, running Flux/KubeVela/Crossplane, ingress-nginx, cert-manager, monitoring stack

All clusters run **EKS Auto Mode** (no manual node group / Karpenter management needed —
AWS handles compute provisioning automatically).

## 2. Key Concepts Learned

### kro (Kubernetes Resource Orchestrator)
- Lets platform teams define **ResourceGraphDefinitions (RGDs)** — reusable templates
  that compose multiple K8s/AWS resources into one simplified custom resource.
- Example: `EksclusterWithVpc` RGD — one custom resource provisions VPC, IAM roles,
  an Auto Mode EKS cluster, Argo CD registration, and Pod Identity — all from a single YAML.
- Comparable to a schema/template engine, NOT a CI/CD pipeline definition
  (contrast with `.gitlab-ci.yml`, which defines pipeline *steps*, not new resource *types*).

### ACK (AWS Controllers for Kubernetes)
- Lets you manage real AWS resources (S3, DynamoDB, IAM, EKS, EC2, ECR) via
  Kubernetes-native YAML + kubectl/Argo CD, instead of Console/Terraform/CLI.
- Each AWS service has its own controller watching for matching CRDs.

### kro + ACK combined
- kro provides the simplified developer-facing API; ACK does the actual AWS provisioning
  underneath. Together = **self-service infrastructure**.

### Argo CD
- GitOps continuous deployment engine — watches Git, keeps cluster state in sync.
- RBAC roles: ADMIN / EDITOR / VIEWER, plus project-scoped granular permissions.
- SSO via IAM Identity Center → Keycloak.

### Argo Workflows
- Container-native workflow engine for CI/CD and data pipelines on Kubernetes.

### Kargo
- Multi-stage continuous promotion layer on top of Argo CD — automates
  dev → staging → prod promotions with GitOps principles.

## 3. Hands-On Exercises Completed

### Cluster verification
\`\`\`bash
aws eks list-clusters
# Output: peeks-hub, peeks-spoke-dev, peeks-spoke-prod
\`\`\`

### Login flow (SSO chain: App -> IAM Identity Center -> Keycloak)
\`\`\`bash
# Env vars set via workshop's custom zshrc (not default shell!)
source ~/environment/platform-on-eks-workshop/hack/.zshrc

echo \$ARGOCD_URL
echo \$WORKFLOWS_URL
echo \$KARGO_URL
echo \$BACKSTAGE_URL

# Credentials
echo "Username: user1"
echo "Password: \$USER1_PASSWORD"
\`\`\`

**Lesson learned:** these URL env vars are defined in a *workshop-specific* \`.zshrc\`
(\`~/environment/platform-on-eks-workshop/hack/.zshrc\`), separate from the default
shell profile — must be sourced manually in fresh terminal sessions.

### Self-service S3 bucket provisioning via Backstage
Flow: Backstage form -> kro \`S3Bucket\` custom resource -> ACK S3 controller -> real AWS bucket -> Argo CD tracks it as an Application.

\`\`\`bash
# Generate unique bucket name
echo "Bucket Name: peeks-s3-kro-\$AWS_ACCOUNT_ID-\$(uuidgen | tr -d - | tr '[:upper:]' '[:lower:]' | cut -c1-10)"
\`\`\`

Steps performed in Backstage UI:
1. Create -> search "S3" -> "S3 Bucket Template ACK with KRO" -> CHOOSE
2. Filled: Bucket Name, Environment=dev, Region=us-west-2, Namespace=default, Config=standard
3. REVIEW -> Create

**Validation:**
\`\`\`bash
aws s3 ls | grep peeks-s3-kro
# Confirmed: bucket created successfully in AWS
\`\`\`

Also validated via Argo CD — application \`peeks-s3-kro-xxxxx\` showed Synced/Healthy.

## 4. Key IAM/AWS Concepts Reinforced

- **IAM Role** = temporary, assumable set of permissions (vs. IAM User = permanent
  credentials). Workshop terminal ran as \`assumed-role/...\` throughout — no long-lived keys.
- **\`aws sts get-caller-identity\`** — quick way to check which identity/role you're
  currently operating as.
- CloudFormation nested stacks can fail silently at the top level — real errors often
  live in CodeBuild logs (when a custom resource Lambda triggers a CodeBuild project).

## 5. Not Reached This Session (Future Follow-up)

- [ ] Argo Rollouts — canary deployments with quality gates
- [ ] Kargo — full dev→prod promotion walkthrough (logged in, didn't run a promotion)
- [ ] Polyglot app deployment (Java/Rust/NextJS apps across environments)
- [ ] Kiro / GenAI integration for platform code generation
- [ ] DORA metrics and observability dashboards

## 6. Comparison Notes (from discussion)

| Tool/File | Purpose | Analogy |
|---|---|---|
| \`.gitlab-ci.yml\` | CI/CD pipeline steps | A recipe |
| kro \`ResourceGraphDefinition\` | New K8s custom resource type/schema | A cookie cutter |
| \`odoo.yaml\` (Odoo apps manifest) | Static data/config list | A shopping list |
| Rancher | Cluster lifecycle management | Control panel for clusters |
| Harvester | VM/HCI virtualization on K8s | Different domain (infra, not app deploy) |
| Argo CD | GitOps app deployment | What's *running* matches what's *in Git* |

## 7. Related Earlier Session: EKS Fundamentals

See companion repo: \`AWS-EKS-fundamentals\` — covered node/cluster basics,
AWS Load Balancer Controller, NLB vs ALB, and EKS Auto Mode/Karpenter internals.

## 9. Rust Backend — Full Deployment (Completed)

Successfully deployed the Unicorn E-Commerce Rust backend end-to-end via the
Backstage-provisioned CI/CD pipeline and kro `AppmodService` custom resource.

### Root causes debugged and fixed (real production-style issues):

1. **Missing `${AWS_ACCOUNT_ID}` substitution** — the deployment manifest template
   included a placeholder that our initial `sed` pass didn't cover, causing kro
   to construct an invalid IAM role ARN. ACK's IAM controller correctly rejected
   it as AccessDenied. Fix: added the missing `sed` substitution and verified
   with a regex check for any remaining `<...>` or `${...}` placeholders before
   committing.

2. **Missing stable/preview Kubernetes Services** — kro's `AppmodService` RGD
   does not create plain `Service` objects; Argo Rollouts requires a
   `stableService` and `previewService` to exist for canary traffic routing.
   The Rollout was stuck in `Degraded`/`InvalidSpec` state referencing a
   nonexistent service. Fix: manually appended two `Service` manifests to the
   deployment YAML, matching the label selector Argo Rollouts actually expected
   (confirmed via `kubectl get rollout ... -o jsonpath='{.spec.selector.matchLabels}'`
   rather than guessing).

### Verification

\`\`\`bash
kubectl get pods -n team-rust
# rust-microservice-dev-xxxx   1/1   Running

curl --request GET -sLk --url "http://\${DNS_DEV}/rust-app/collection/FRONT_PAGE" | jq '.'
# Returns real product data from DynamoDB
\`\`\`

### Key lesson

When a kro-managed app abstraction (\`AppmodService\`) doesn't create every
resource a dependent controller (Argo Rollouts) needs, check the *consuming*
resource's actual spec/selector rather than assuming template patterns from
a previous exercise will match exactly — label names can differ between app
instances even when using the same RGD.
