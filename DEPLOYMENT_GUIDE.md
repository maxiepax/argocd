# Deployment Guide: App of Apps for VKS Cluster Registration

## 🎯 Objective
Deploy `infra-vks-01`, register it with ArgoCD, and request VKS add-ons using
sync waves. See [add-on setup](./APP_OF_APPS_README.md#vks-add-ons).

## 📦 What You Have

Main files:

```
infra-vks-01.yaml
├─ Wave 0: Creates VKS cluster
└─ Time: 10-30 minutes to Ready state

*-addon.yaml
└─ Wave 10: Requests cert-manager, contour, and prometheus

argocd-managedentity.yaml
├─ Wave 10: Registers the cluster to ArgoCD
├─ Includes: Role, RoleBinding, ManagedEntity
└─ Time: 2-5 minutes

argocd-app-of-apps.yaml
├─ Orchestration Application
└─ Syncs the cluster and registration files using the default project

argocd-vks-appproject.yaml
└─ Optional separate AppProject

Documentation
├─ QUICK_START.md (fastest way to deploy)
├─ APP_OF_APPS_README.md (comprehensive guide)
├─ WAVE_EXECUTION_DIAGRAM.md (visual timeline)
└─ FILES_SUMMARY.txt (detailed reference)
```

## 🚀 Deployment: Two Options

### Option A: GitOps (Recommended) ⭐
**Pros:** Full automation, version-controlled, self-healing
**Cons:** Requires Git push

```bash
# 1. Add files to your Git repository
git add argocd-vks/ argocd-app-of-apps.yaml
git commit -m "Add App of Apps for VKS cluster registration"
git push

# 2. Deploy the orchestration Application
kubectl apply -n argocd -f argocd-app-of-apps.yaml

# 3. Watch progress
argocd app wait argocd-vks-app-of-apps --sync -v
```

Alternatively, paste the entire `argocd-app-of-apps.yaml` into the ArgoCD UI's
Application YAML editor. The optional AppProject must be created separately and
selected in the Application's `spec.project` if it is to be used.
For kubectl commands, replace `argocd` with your ArgoCD installation namespace
if it differs; this is separate from the destination namespace.

### Option B: Direct kubectl (Quick test)
**Pros:** Immediate, no Git needed
**Cons:** One-time, less GitOps-y

```bash
# 1. Apply cluster creation
kubectl apply -f argocd-vks/infra-vks-01.yaml

# 2. Wait 10-30 minutes for cluster to be Ready
kubectl get cluster infra-vks-01 -n kim-prod-vlryx -w

# 3. Apply registration
kubectl apply -f argocd-vks/argocd-managedentity.yaml
kubectl apply -f argocd-vks/cert-manager-addon.yaml
kubectl apply -f argocd-vks/contour-addon.yaml
kubectl apply -f argocd-vks/prometheus-addon.yaml

# 4. Wait 2-5 minutes for registration
kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx -w
```

## 📊 Real-Time Monitoring

### Watch Application Sync
```bash
# Option 1: ArgoCD CLI (best view)
argocd app watch argocd-vks-app-of-apps

# Option 2: kubectl watch
kubectl get application argocd-vks-app-of-apps -n argocd -w

# Option 3: Detailed status
kubectl describe application argocd-vks-app-of-apps -n argocd
```

### Monitor Wave 0: Cluster Creation
```bash
# Terminal 1: Watch cluster status
kubectl get cluster infra-vks-01 -n kim-prod-vlryx -w

# Terminal 2: Get detailed info
watch kubectl describe cluster infra-vks-01 -n kim-prod-vlryx

# Terminal 3: Check conditions (when Wave 0 is done, Phase: Ready appears)
kubectl get cluster infra-vks-01 -n kim-prod-vlryx -o jsonpath='{.status.phase}'
```

### Monitor Wave 10: Cluster Registration
```bash
# Terminal 1: Watch registration status
kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx -w

# Terminal 2: Get detailed info
watch kubectl describe managedentity infra-vks-01-managed -n kim-prod-vlryx

# Terminal 3: Check phase (when Wave 10 is done, Phase: Ready appears)
kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx -o jsonpath='{.status.phase}'
```

### Verify Final Success
```bash
# Check cluster appears in ArgoCD
argocd cluster list

# Should show: infra-vks-01 in the list with status "Successful"
```

## ⏱️ Timeline & What to Expect

### T+0: Deployment Start
```
kubectl apply -n argocd -f argocd-app-of-apps.yaml
Application created: argocd-vks-app-of-apps
```

### T+0-5 seconds: Initial Sync
```
Application status: Syncing
└─ Starting Wave 0 resources
```

### T+5-30 minutes: Wave 0 - Cluster Creation
```
Resources being created:
├─ Cluster CRD (immediate)
└─ VM provisioning (10-30 minutes)

Watch: kubectl get cluster -w
Expected progression:
├─ Phase: Provisioning (first 5 minutes)
├─ Phase: Provisioned (5-20 minutes)
└─ Phase: Ready (20-30 minutes total)
```

### T+30-35 minutes: Wave 0 Complete → Wave 10 Starts
```
ArgoCD detects Wave 0 completion
└─ Automatically starts Wave 10 resources
```

### T+35-40 minutes: Wave 10 - Cluster Registration
```
Resources being created:
├─ Role (immediate)
├─ RoleBinding (immediate)
└─ ManagedEntity (5-10 seconds)

Watch: kubectl get managedentity -w
Expected progression:
├─ ManagedEntity created
├─ Conditions evaluating
└─ Phase: Ready (2-5 minutes total)
```

### T+40 minutes: Complete ✅
```
Application Status: Synced
├─ Wave 0: Succeeded ✓
├─ Wave 10: Succeeded ✓
└─ Cluster available in ArgoCD ✓

Command: argocd cluster list
├─ infra-vks-01 (status: Successful)
```

## ✅ Success Checklist

### Application Level
- [ ] `kubectl get application argocd-vks-app-of-apps -n argocd`
  - Status: Synced
  - Sync Result: Successful

### Wave 0: Cluster Creation
- [ ] `kubectl get cluster infra-vks-01 -n kim-prod-vlryx`
  - Phase: Ready
  - Status: Provisioned

- [ ] `kubectl get nodes` (on the new cluster when kubeconfig is available)
  - At least 1 control plane node
  - At least 1 worker node
  - All nodes: Ready

### Wave 10: Cluster Registration
- [ ] `kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx`
  - Phase: Ready

- [ ] `kubectl describe managedentity infra-vks-01-managed -n kim-prod-vlryx`
  - ArgoCDExists: True ✓
  - ClusterInfoFetched: True ✓
  - SupervisorServiceAccountBound: True ✓
  - SecretReady: True ✓
  - Ready: True ✓

- [ ] `kubectl get secret cls-vks-kim-prod-vlryx-infra-vks-01 -n kim-prod-vlryx`
  - Secret exists

### ArgoCD Integration
- [ ] `argocd cluster list`
  - infra-vks-01 appears in list
  - Status: Successful

## 🔧 Troubleshooting During Deployment

### "Application stuck in Syncing"
```bash
# Check application events
kubectl describe application argocd-vks-app-of-apps -n argocd

# Check specific resource status
kubectl get cluster infra-vks-01 -n kim-prod-vlryx
kubectl describe cluster infra-vks-01 -n kim-prod-vlryx
```

### "Wave 0 not progressing"
```bash
# Check cluster events
kubectl describe cluster infra-vks-01 -n kim-prod-vlryx | grep Events -A 20

# Check infrastructure
# - Verify vSphere has available capacity
# - Check network connectivity
# - Review vSphere task logs
```

### "Wave 10 fails: SA not bound"
```bash
# Verify rolebinding exists
kubectl get rolebinding argocd-k8s-sa-rolebinding -n kim-prod-vlryx

# If missing, re-apply the manifest
kubectl apply -f argocd-vks/argocd-managedentity.yaml

# Force reconciliation
kubectl patch managedentity infra-vks-01-managed -n kim-prod-vlryx \
  --type merge -p '{"metadata":{"annotations":{"restartedAt":"'$(date -Iseconds)'"}}}'
```

### "Cluster not appearing in ArgoCD cluster list"
```bash
# 1. Verify ManagedEntity is Ready
kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx -o jsonpath='{.status.phase}'
# Should show: Ready

# 2. Verify cluster secret exists
kubectl get secret cls-vks-kim-prod-vlryx-infra-vks-01 -n kim-prod-vlryx -o yaml

# 3. Force ArgoCD to reload clusters
argocd cluster list --refresh

# 4. If still missing, restart ArgoCD application controller
kubectl rollout restart deployment/argocd-application-controller -n argocd
```

## 📋 Post-Deployment Tasks

Once everything is synced and ready:

### 1. Test Cluster Access
```bash
# Get cluster endpoint
kubectl get managedentity infra-vks-01-managed -n kim-prod-vlryx \
  -o jsonpath='{.status.clusterEndpointURL}'

# Test access via ArgoCD (should work without issues)
argocd cluster info <cluster-name>
```

### 2. Create Your First Application
```bash
# Option A: Using ArgoCD CLI
argocd app create my-first-app \
  --repo https://github.com/maxiepax/argocd \
  --path my-app \
  --dest-server https://172.16.44.33:6443 \
  --dest-namespace default

# Option B: Using Application CRD
kubectl apply -f - << 'YAML'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-first-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/maxiepax/argocd
    path: my-app
    targetRevision: HEAD
  destination:
    server: https://172.16.44.33:6443
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
YAML
```

### 3. Commit to Git
```bash
# If using GitOps, verify everything is in Git
cd <your-argocd-repo>
git status
git add .
git commit -m "VKS cluster deployment completed successfully"
git push
```

## 🎓 Learning Resources

For deeper understanding:

1. **ArgoCD Sync Waves**
   - Docs: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
   - Understanding: Waves execute sequentially by number
   - Use case: Ensure resources are created in correct order

2. **App of Apps Pattern**
   - Docs: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps
   - Understanding: One Application manages other Applications
   - Use case: Multi-environment deployments, GitOps at scale

3. **VMware Operator for ArgoCD**
   - Understanding: ManagedEntity automates cluster registration
   - Use case: Manage VKS clusters with ArgoCD

## 🚨 Important Notes

⚠️ **Network Connectivity**
- Wave 0 requires vSphere connectivity
- Wave 10 requires ArgoCD instance to be running
- Both require the vSphere namespace to be accessible

⚠️ **Resource Requirements**
- Cluster creation reserves VM resources
- Storage requirement: depends on VM count and workloads
- Ensure vSphere cluster has sufficient capacity

⚠️ **Timing Considerations**
- Don't interrupt Wave 0 during cluster creation
- Wave 10 starts automatically after Wave 0 completes
- Total deployment time: 15-40 minutes

## 📞 Support & Next Steps

For more details:
- **Quick reference**: See `QUICK_START.md`
- **Comprehensive guide**: See `APP_OF_APPS_README.md`
- **Visual timeline**: See `WAVE_EXECUTION_DIAGRAM.md`
- **File descriptions**: See `FILES_SUMMARY.txt`

Ready to deploy? Run:
```bash
kubectl apply -n argocd -f argocd-app-of-apps.yaml
```

Then monitor with:
```bash
argocd app watch argocd-vks-app-of-apps
```

🎉 Enjoy your automated VKS cluster deployment!
