# ArgoCD VKS Cluster Registration - App of Apps Pattern

🎉 **Complete App of Apps setup for automated VKS cluster deployment and ArgoCD registration!**

## 🎯 What This Does

This repository contains a complete **App of Apps** implementation that:

1. **Wave 0**: Creates `infra-vks-01` using Cluster API
2. **Wave 10**: Registers the cluster and requests cert-manager, contour, and prometheus

`infra-vks-01` has one fixed worker. See [add-on setup and verification](./APP_OF_APPS_README.md#vks-add-ons).
ArgoCD sync waves require an appropriate Cluster health assessment to wait for
readiness before registration.

## 📋 Quick Start

### Deploy in 1 command:
```bash
kubectl apply -n argocd -f argocd-app-of-apps.yaml
```

Alternatively, paste the entire `argocd-app-of-apps.yaml` into the ArgoCD UI's
Application YAML editor. Commit and push the source manifests before syncing.
For kubectl commands, replace `argocd` with your ArgoCD installation namespace
if it differs; this is separate from the destination namespace.

### Monitor progress:
```bash
argocd app watch argocd-vks-app-of-apps
```

### Verify success (15-40 minutes later):
```bash
argocd cluster list  # Should show infra-vks-01
```

👉 **For more details, see [QUICK_START.md](./QUICK_START.md)**

## 📁 Files in This Repository

| File | Purpose | Wave |
|------|---------|------|
| `argocd-vks/infra-vks-01.yaml` | VKS Cluster, one fixed worker | 0 |
| `argocd-vks/argocd-managedentity.yaml` | RBAC and cluster registration | 10 |
| `argocd-vks/*-addon.yaml` | cert-manager, contour, and prometheus AddonInstalls | 10 |
| `argocd-app-of-apps.yaml` | Orchestration Application (uses the `default` project) | - |
| `argocd-vks-appproject.yaml` | Optional separate AppProject | - |
| **Documentation** | | |
| `QUICK_START.md` | ⚡ Fastest way to deploy | - |
| `DEPLOYMENT_GUIDE.md` | 📋 Step-by-step deployment with monitoring | - |
| `APP_OF_APPS_README.md` | 📖 Comprehensive guide with troubleshooting | - |
| `WAVE_EXECUTION_DIAGRAM.md` | 📊 Visual timeline and sequences | - |
| `FILES_SUMMARY.txt` | 📝 Detailed file descriptions | - |
| `README.md` | 👈 This file | - |

## 🗂️ Structure

```
argocd/
├── argocd-vks/
│   ├── infra-vks-01.yaml              Wave 0: Cluster creation
│   ├── argocd-managedentity.yaml    Wave 10: Registration
│   └── *-addon.yaml                 Wave 10: Add-on installations
├── argocd-app-of-apps.yaml          Orchestration
├── argocd-vks-appproject.yaml       Optional AppProject
├── README.md                        (This file)
├── QUICK_START.md                   Fastest guide
├── DEPLOYMENT_GUIDE.md              Step-by-step guide  
├── APP_OF_APPS_README.md            Comprehensive docs
├── WAVE_EXECUTION_DIAGRAM.md        Visual timeline
└── FILES_SUMMARY.txt                Reference
```

## ⏱️ Timeline

```
kubectl apply -n argocd -f argocd-app-of-apps.yaml
           ↓
Wave 0: Create VKS Cluster (10-30 min)
├─ Cluster CRD created
├─ VMs provisioned
├─ Kubernetes setup
└─ Cluster Ready ✓
           ↓
Wave 10: Register with ArgoCD (2-5 min)
├─ Create RBAC Role
├─ Create RoleBinding
├─ Create ManagedEntity
└─ Cluster registered ✓
           ↓
Complete! Cluster available in ArgoCD (15-40 min total)
```

## 🚀 Two Deployment Options

### Option 1: GitOps (Recommended) ⭐
```bash
# Add to your repository
git add argocd-vks/ argocd-app-of-apps.yaml
git commit -m "Add App of Apps for VKS cluster"
git push

# Deploy the orchestration
kubectl apply -n argocd -f argocd-app-of-apps.yaml
```

**Pros:**
- ✅ Full GitOps workflow
- ✅ Version-controlled
- ✅ Self-healing
- ✅ Repeatable

**Cons:**
- ⚠️ Requires Git push

### Option 2: Direct kubectl
```bash
kubectl apply -f argocd-vks/infra-vks-01.yaml
# Wait for the cluster to become ready before registering it and applying add-ons.
kubectl apply -f argocd-vks/argocd-managedentity.yaml
kubectl apply -f argocd-vks/cert-manager-addon.yaml
kubectl apply -f argocd-vks/contour-addon.yaml
kubectl apply -f argocd-vks/prometheus-addon.yaml
```

**Pros:**
- ✅ Immediate
- ✅ No Git needed

**Cons:**
- ⚠️ Manual wave management
- ⚠️ Not repeatable

## ✅ Success Criteria

After deployment, check the cluster and registration. For example:

```bash
# Wave 0 Complete
$ kubectl get cluster infra-vks-01 -n kim-prod-vlryx
NAME         PHASE   AGE
infra-vks-01   Ready   15m

# Wave 10 Complete
$ kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx
NAME                 PHASE   AGE
infra-vks-01-managed   Ready   5m

# Cluster in ArgoCD
$ argocd cluster list
SERVER                          NAME            VERSION  STATUS
https://172.16.44.33:6443       infra-vks-01          Successful
```

## 📖 Documentation Guide

**Choose your path:**

- 🏃 **Just want to deploy?** → [QUICK_START.md](./QUICK_START.md)
- 👣 **Step-by-step guide?** → [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md)
- 📚 **Learn everything?** → [APP_OF_APPS_README.md](./APP_OF_APPS_README.md)
- 📊 **Visual learner?** → [WAVE_EXECUTION_DIAGRAM.md](./WAVE_EXECUTION_DIAGRAM.md)
- 🔍 **Need details?** → [FILES_SUMMARY.txt](./FILES_SUMMARY.txt)

## 🔑 Key Concepts

### Sync Waves
ArgoCD's sync waves ensure resources are created in order:
- Wave 0: Creates the cluster
- Waits for Wave 0 to complete
- Wave 10: Registers with ArgoCD

### App of Apps Pattern
One Application (`argocd-vks-app-of-apps`) manages:
- VKS Cluster creation
- Cluster registration resources

The Application uses the `default` AppProject. The optional
`argocd-vks-appproject.yaml` must be created separately and selected in
`spec.project` to use it.

### ManagedEntity
VMware Operator CRD that:
- Registers VKS cluster to ArgoCD
- Manages cluster credentials
- Creates kubeconfig secret

## 🎓 Prerequisites

1. **VMware Operator for ArgoCD 1.2.0+** installed
2. **ArgoCD instance** running in `kim-prod-vlryx` namespace
3. **vSphere Namespace** `kim-prod-vlryx` with:
   - Sufficient VM resources
   - Network connectivity
   - `argocd-k8s-sa` service account
4. **Git repository** (if using GitOps option)

## 🛠️ Customization

### Change namespace
Update `kim-prod-vlryx` in all files:
```bash
sed -i 's/kim-prod-vlryx/your-namespace/g' *.yaml argocd-vks/*.yaml
```

### Add more waves
Create new resources with `argocd.argoproj.io/sync-wave: "15"` annotation

### Change cluster config
Edit `argocd-vks/infra-vks-01.yaml`:
- `vmClass`: VM machine class
- `version`: Kubernetes version
- `replicas`: Control plane/worker replicas
- Network settings, storage class, etc.

See [APP_OF_APPS_README.md](./APP_OF_APPS_README.md) for detailed customization.

## 📊 Real-Time Monitoring

```bash
# Watch application sync
argocd app watch argocd-vks-app-of-apps

# Or use kubectl
kubectl get application argocd-vks-app-of-apps -n argocd -w

# Check specific waves
kubectl get cluster infra-vks-01 -n kim-prod-vlryx -w        # Wave 0
kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx -w  # Wave 10
```

## 🚨 Troubleshooting

**Wave 0 hangs?**
- Check vSphere capacity
- Verify network connectivity
- Review vSphere task logs

**Wave 10 fails?**
- Verify rolebinding exists
- Check ManagedEntity conditions
- Restart VMware Operator if needed

**Cluster not in ArgoCD?**
- Verify ManagedEntity is Ready
- Check cluster secret exists
- Force ArgoCD to reload clusters: `argocd cluster list --refresh`

👉 **Full troubleshooting guide in [APP_OF_APPS_README.md](./APP_OF_APPS_README.md)**

## 🎯 Next Steps

After successful deployment:

1. **Test cluster access**: Deploy an application
2. **Create more applications**: Build your App of Apps
3. **Commit to Git**: Make everything repeatable
4. **Add more waves**: Deploy additional components

## 📚 Resources

- [ArgoCD Sync Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps)
- [VMware Operator for ArgoCD](https://docs.vmware.com/en/VMware-Operator-for-ArgoCD/)
- [Cluster API](https://cluster-api.sigs.k8s.io/)

## 🤝 Contributing

This is part of the maxiepax/argocd repository. To improve:

1. Test the deployment
2. Report issues with specific error messages
3. Suggest improvements or additional waves

## 📝 License

Part of maxiepax/argocd repository.

## 🎉 You're Ready!

```bash
# Deploy
kubectl apply -n argocd -f argocd-app-of-apps.yaml

# Monitor
argocd app watch argocd-vks-app-of-apps

# Verify (after 15-40 minutes)
argocd cluster list
```

Enjoy your automated VKS cluster deployment! 🚀

---

**Questions?** Check the documentation files:
- Quick help: [QUICK_START.md](./QUICK_START.md)
- Full guide: [APP_OF_APPS_README.md](./APP_OF_APPS_README.md)
- Visual guide: [WAVE_EXECUTION_DIAGRAM.md](./WAVE_EXECUTION_DIAGRAM.md)
