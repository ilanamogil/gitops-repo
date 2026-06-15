# GitOps Repository (Exam Section 8)

GitOps repository for deploying applications with **ArgoCD** across three
environments (`dev`, `qa`, `prd`). The primary application (`flask-aws-monitor`)
and the `stress-app` are each driven by a single **ApplicationSet** (git
generator), and a bonus **Redis** supporting service is managed by a standalone
ArgoCD `Application`.

## Repository structure

```
gitops-repo/
├── charts/                          # Helm charts (the "what to deploy")
│   ├── flask-aws-monitor/
│   ├── stress-app/
│   └── redis/
├── flask-aws-monitor/               # per-env values (the "how, per environment")
│   ├── dev/values.yaml
│   ├── qa/values.yaml
│   └── prd/values.yaml
├── stress-app/
│   ├── dev/values.yaml
│   ├── qa/values.yaml
│   └── prd/values.yaml
├── applicationsets/
│   ├── flask-applicationset.yaml    # one ArgoCD Application per env
│   └── stress-applicationset.yaml   # one ArgoCD Application per env
└── applications/
    └── redis-application.yaml        # bonus: standalone supporting service
```

Charts (reusable templates) are kept separate from environment **values**, so
the same chart is deployed three times with different configuration.

## Environment differences

**flask-aws-monitor**

| Setting       | dev       | qa        | prd          |
|---------------|-----------|-----------|--------------|
| replicaCount  | 1         | 2         | 3            |
| service.type  | ClusterIP | ClusterIP | LoadBalancer |
| image.tag     | dev       | qa        | latest       |
| CPU limit     | 250m      | 500m      | 1            |
| Memory limit  | 256Mi     | 512Mi     | 1Gi          |

**stress-app**

| Setting       | dev   | qa    | prd   |
|---------------|-------|-------|-------|
| replicaCount  | 1     | 2     | 3     |
| CPU workers   | 1     | 2     | 4     |
| vm workers    | 1     | 1     | 2     |
| vm bytes      | 64M   | 128M  | 256M  |
| CPU limit     | 250m  | 500m  | 1     |
| Memory limit  | 128Mi | 256Mi | 512Mi |

## How it works

Each `ApplicationSet` uses a **git generator** that scans `<app>/*` (e.g.
`flask-aws-monitor/*`). For every environment directory it finds, it creates an
ArgoCD `Application` named `<app>-<env>` that:

- deploys the chart at `charts/<app>`
- applies the matching `<app>/<env>/values.yaml` (referenced relative to the
  chart path: `../../<app>/{{path.basename}}/values.yaml`)
- syncs into a namespace named after the environment (auto-created via
  `CreateNamespace=true`)
- self-heals and prunes automatically

Redis is deployed as a single `Application` (the `charts/redis` chart into the
`redis` namespace) to demonstrate managing a supporting service the same way.

## Deploy / update

```bash
# Apply the ApplicationSets / Applications once.
kubectl apply -f applicationsets/flask-applicationset.yaml
kubectl apply -f applicationsets/stress-applicationset.yaml
kubectl apply -f applications/redis-application.yaml

# Watch ArgoCD create the applications (flask-dev/qa/prd, stress-dev/qa/prd, redis)
kubectl get applications -n argocd

# From here on it's GitOps: commit a change to any <app>/<env>/values.yaml
# and ArgoCD automatically syncs that environment.
```

### Validate a change locally before committing

```bash
# Render exactly what ArgoCD will apply for one environment
helm template flask-dev charts/flask-aws-monitor \
  -f flask-aws-monitor/dev/values.yaml --namespace dev

helm lint charts/flask-aws-monitor charts/stress-app charts/redis
```

## Notes

- AWS credentials are **never** committed. Provide them per cluster via a
  Secret or an untracked `aws-values.yaml` (see `.gitignore`).
- Update the `repoURL` in the manifests under `applicationsets/` and
  `applications/` if the repository is hosted under a different account/name.
- To run Redis per-environment too, mirror the stress-app pattern: add
  `redis/<env>/values.yaml` files and a `redis-applicationset.yaml`.
