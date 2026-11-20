# Quick Start: App of Apps for VKS Cluster Registration

## 🚀 Single Command Deployment

```bash
# Apply the App of Apps Application to ArgoCD
kubectl apply -n argocd -f argocd-app-of-apps.yaml
```

Alternatively, paste the entire `argocd-app-of-apps.yaml` into the ArgoCD UI's
Application YAML editor. Commit and push the source manifests before syncing.
For kubectl, replace `argocd` with your ArgoCD installation namespace if it
differs; this is not the destination namespace.
The Application uses the `default` project; `argocd-vks-appproject.yaml` is
optional and must be created and selected separately.

That's it! ArgoCD will now orchestrate:

1. **Wave 0**: Create `infra-vks-01`
2. **Wave 10**: Register it and request cert-manager, contour, and prometheus

See [add-on setup and verification](./APP_OF_APPS_README.md#vks-add-ons).

## 📊 Monitor Progress

```bash
# Watch the application sync
argocd app wait argocd-vks-app-of-apps --sync -v

# Or check status manually
kubectl get application argocd-vks-app-of-apps -n argocd -w
```

## ✅ Verify Success

### Cluster Creation (Wave 0)
```bash
kubectl get cluster infra-vks-01 -n kim-prod-vlryx
```
Expected: `STATUS: Ready`

### Cluster Registration (Wave 10)
```bash
kubectl describe managedentity infra-vks-01-managed -n kim-prod-vlryx
```
Expected: `Phase: Ready`

### ArgoCD Cluster List
```bash
argocd cluster list
```
Expected: `infra-vks-01` appears in the list

## 📁 Files Overview

| File | Purpose | Wave |
|------|---------|------|
| `infra-vks-01.yaml` | VKS Cluster definition | 0 |
| `*-addon.yaml` | VKS add-on installations | 10 |
| `argocd-managedentity.yaml` | Cluster registration | 10 |
| `argocd-app-of-apps.yaml` | Orchestration | - |
| `argocd-vks-appproject.yaml` | Optional separate AppProject | - |
| `APP_OF_APPS_README.md` | Detailed documentation | - |

## ⏱️ Expected Timeline

- **Wave 0** (Cluster Creation): 10-30 minutes
- **Wave 10** (Registration): 2-5 minutes after Wave 0 completes

## 🔧 Troubleshooting

See [APP_OF_APPS_README.md](./APP_OF_APPS_README.md) for detailed troubleshooting guide.

## 🎯 Next Steps

Once registration is complete, you can:

1. **Create Applications** on the new cluster
2. **Set up App of Apps** for multi-cluster deployments
3. **Configure additional waves** for application deployment

For more details, see [APP_OF_APPS_README.md](./APP_OF_APPS_README.md)
