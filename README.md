# DWP Platform Kubernetes GitOps

This repository contains Kubernetes manifests for the DWP platform in `env/` and a Helm chart managed by Argo CD. The Helm chart is the intended deployment source; the existing environment manifests are retained temporarily for migration and comparison.

## Repository layout

```text
charts/dwp-platform/       Helm chart and environment-specific values
argocd/bootstrap.yaml      One-time Argo CD application bootstrap
argocd/apps/               Argo CD environment applications
env/{dev,qa,stage,prod}/   Existing manifests; do not apply after Argo CD cutover
```

The chart deploys the auth, blog, profile, and UI services, their Services, and the existing NetworkPolicies. It does not create Namespace resources. The Argo CD Applications set each destination namespace and use `CreateNamespace=true`.

## Argo CD setup

The Argo CD instance must have access to this Git repository and manage the cluster hosting these environments. The application definitions currently target the `main` branch and the in-cluster API server; adjust those settings if your deployment uses a different branch or cluster.

1. Install/configure External Secrets Operator and configure its AWS Secrets Manager access.
2. Ensure each environment namespace has a `dwp-platform-secrets` Secret, created by External Secrets Operator, with these keys:

   - `auth-datasource-username`, `auth-datasource-password`, `auth-jwt-secret`
   - `blog-datasource-username`, `blog-datasource-password`, `blog-jwt-secret`
   - `profile-db-user`, `profile-db-password`, `profile-jwt-secret`

   Secret values are intentionally not stored in this repository or Helm values.
3. Bootstrap Argo CD from a machine configured to access the cluster:

   ```sh
   kubectl apply -f argocd/bootstrap.yaml
   ```

   The bootstrap Application manages the non-production ApplicationSet and the production Application. Dev, QA, and stage are configured for automated sync; production requires a manual sync.
4. Review the Argo CD diff for each environment and confirm its resources and secrets before syncing. Do not have another deployment process apply the same resources once Argo CD takes ownership.

The chart values in `charts/dwp-platform/values-*.yaml` preserve the current environment-specific database URLs and image tags. Update an environment's values file to promote new image tags through Git review.

Before syncing production, verify the production database endpoint: the current production manifests refer to an RDS hostname containing `nonprod`, and the chart preserves that existing setting rather than guessing a replacement.

## Local chart validation

Run these commands from the repository root:

```sh
helm lint charts/dwp-platform -f charts/dwp-platform/values-dev.yaml
helm template dwp-platform-dev charts/dwp-platform \
  --namespace dwp-dev \
  --values charts/dwp-platform/values-dev.yaml
```

Repeat with `qa`, `stage`, or `prod` as needed. Helm rendering is for validation; Argo CD is the intended deployment controller.

## Migration notes

The files under `env/` are the current manifest baseline and are not automatically deleted or applied by the Helm chart. Compare Helm-rendered resources with the existing environment resources, then cut over one environment at a time. Once Argo CD owns an environment, stop applying that environment's old manifests and remove them only after the cutover has been verified.
