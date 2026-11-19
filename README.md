# ArgoCD VKS Cluster Registration - App of Apps Pattern

🎉 **Complete App of Apps setup for automated VKS cluster deployment and ArgoCD registration!**

## 🎯 What This Does

This repository contains a complete **App of Apps** implementation that:

1. **Wave 0** (0-30 min): Creates a Tanzu VKS cluster using Cluster API
2. **Wave 10** (30-35 min): Automatically registers the cluster with ArgoCD using ManagedEntity CRD

The entire process is orchestrated by ArgoCD with sync waves, ensuring the cluster is fully created before registration begins.

## 📋 Quick Start

### Deploy in 1 command:
```bash
kubectl apply -f argocd-app-of-apps.yaml
```

### Monitor progress:
```bash
argocd app watch argocd-vks-app-of-apps
```

### Verify success (15-40 minutes later):
```bash
argocd cluster list  # Should show argocd-vks-prod
```

👉 **For more details, see [QUICK_START.md](./QUICK_START.md)**

## 📁 Files in This Repository

| File | Purpose | Wave |
|------|---------|------|
| `argocd-vks/argocd-vks.yaml` | VKS Cluster definition | 0 |
| `argocd-vks/argocd-managedentity.yaml` | Cluster registration (Role, RoleBinding, ManagedEntity) | 10 |
| `argocd-app-of-apps.yaml` | Orchestration Application & AppProject | - |
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
│   ├── argocd-vks.yaml              Wave 0: Cluster creation
│   └── argocd-managedentity.yaml    Wave 10: Registration
├── argocd-app-of-apps.yaml          Orchestration
├── README.md                        (This file)
├── QUICK_START.md                   Fastest guide
├── DEPLOYMENT_GUIDE.md              Step-by-step guide  
├── APP_OF_APPS_README.md            Comprehensive docs
├── WAVE_EXECUTION_DIAGRAM.md        Visual timeline
└── FILES_SUMMARY.txt                Reference
```

## ⏱️ Timeline

```
kubectl apply -f argocd-app-of-apps.yaml
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
git add argocd-vks/ argocd-managedentity.yaml argocd-app-of-apps.yaml
git commit -m "Add App of Apps for VKS cluster"
git push

# Deploy the orchestration
kubectl apply -f argocd-app-of-apps.yaml
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
kubectl apply -f argocd-vks/argocd-vks.yaml
kubectl apply -f argocd-managedentity.yaml
```

**Pros:**
- ✅ Immediate
- ✅ No Git needed

**Cons:**
- ⚠️ Manual wave management
- ⚠️ Not repeatable

## ✅ Success Criteria

After deployment, you should see:

```bash
# Wave 0 Complete
$ kubectl get cluster argocd-vks -n kim-prod-vlryx
NAME         PHASE   AGE
argocd-vks   Ready   15m

# Wave 10 Complete
$ kubectl get managedentity argocd-vks-managed -n kim-prod-vlryx
NAME                 PHASE   AGE
argocd-vks-managed   Ready   5m

# Cluster in ArgoCD
$ argocd cluster list
SERVER                          NAME            VERSION  STATUS
https://172.16.44.33:6443       argocd-vks-prod          Successful
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
- AppProject for access control

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
Edit `argocd-vks/argocd-vks.yaml`:
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
kubectl get cluster argocd-vks -n kim-prod-vlryx -w        # Wave 0
kubectl get managedentity argocd-vks-managed -n kim-prod-vlryx -w  # Wave 10
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
kubectl apply -f argocd-app-of-apps.yaml

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
