# GitOps Repository — Flask AWS Monitor (Exam Section 8)

GitOps repository for deploying the Flask AWS monitor with **ArgoCD** across
three environments (`dev`, `qa`, `prd`) using a single **ApplicationSet**.

## Repository structure

```
gitops-repo/
├── charts/
│   └── flask-aws-monitor/        # Helm chart (deployed by ArgoCD)
├── env/
│   ├── dev/values.yaml           # dev overrides
│   ├── qa/values.yaml            # qa overrides
│   └── prd/values.yaml           # prod overrides
└── applicationsets/
    └── flask-applicationset.yaml # one ArgoCD Application per env
```

## Environment differences

| Setting        | dev        | qa         | prd          |
|----------------|------------|------------|--------------|
| replicaCount   | 1          | 2          | 3            |
| service.type   | ClusterIP  | ClusterIP  | LoadBalancer |
| image.tag      | dev        | qa         | latest       |
| CPU limit      | 250m       | 500m       | 1            |
| Memory limit   | 256Mi      | 512Mi      | 1Gi          |

## How it works

The `ApplicationSet` uses a **git generator** that scans `env/*`. For each
directory it creates an ArgoCD `Application` named `flask-<env>` that:

- deploys the chart at `charts/flask-aws-monitor`
- applies the matching `env/<env>/values.yaml`
- syncs into a namespace named after the environment (auto-created)
- self-heals and prunes automatically

## Deploy / update

```bash
# Apply the ApplicationSet once (creates dev, qa and prd Applications)
kubectl apply -f applicationsets/flask-applicationset.yaml

# Watch ArgoCD create the applications
kubectl get applications -n argocd

# From here on, GitOps: commit a change to any env/<env>/values.yaml
# and ArgoCD automatically syncs that environment.
```

## Notes

- AWS credentials are **never** committed. Provide them per cluster via a
  Secret or an untracked `aws-values.yaml`.
- Update the `repoURL` in `applicationsets/flask-applicationset.yaml` if the
  repository is hosted under a different account/name.
