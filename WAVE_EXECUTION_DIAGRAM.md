# Wave Execution Flow Diagram

## Application Deployment Timeline

```
Time ─────────────────────────────────────────────────────────────────────────────>

Application Created
│
├─ Wait for dependencies
│
├─ WAVE 0: VKS Cluster Creation (argocd-vks.yaml)
│  │
│  ├─ Create Cluster resource
│  │  └─ metadata:
│  │      annotations:
│  │        argocd.argoproj.io/sync-wave: "0"
│  │
│  ├─ Provision Infrastructure
│  │  ├─ Control Plane: 1 replica (best-effort-medium)
│  │  └─ Worker Nodes: 1 replica (best-effort-medium)
│  │
│  ├─ Cluster Health Checks
│  │  ├─ etcd running
│  │  ├─ API server responsive
│  │  └─ Nodes ready
│  │
│  └─ Status: Ready ✓
│     (Takes 10-30 minutes)
│
├─ Wave 0 Complete ✓ → Proceed to Wave 10
│
├─ WAVE 10: Cluster Registration (argocd-managedentity.yaml)
│  │
│  ├─ Create RBAC Role
│  │  └─ metadata:
│  │      annotations:
│  │        argocd.argoproj.io/sync-wave: "10"
│  │  
│  │  Rules:
│  │  - secrets (get, list, watch)
│  │  - configmaps (get, list, watch)
│  │  - clusters.cluster.x-k8s.io (get, list, watch)
│  │
│  ├─ Create RoleBinding
│  │  └─ Bind argocd-k8s-sa to Role
│  │
│  ├─ Create ManagedEntity
│  │  └─ metadata:
│  │      annotations:
│  │        argocd.argoproj.io/sync-wave: "10"
│  │  
│  │  spec:
│  │  - targetRef: Cluster argocd-vks (from Wave 0)
│  │  - clusterName: argocd-vks-prod
│  │
│  ├─ Operator Validation
│  │  ├─ Verify service account bound to role
│  │  ├─ Fetch cluster API endpoint
│  │  └─ Create kubeconfig
│  │
│  ├─ Create Cluster Secret
│  │  └─ cls-vks-kim-prod-vlryx-argocd-vks
│  │
│  └─ Status: Ready ✓
│     (Takes 2-5 minutes)
│
└─ ALL WAVES COMPLETE ✓
   Application Status: Synced
   Cluster registered in ArgoCD
   Ready for App deployment
```

## Detailed Wave 0 Sequence

```
Wave 0: Cluster Creation
├─ T+0s: Cluster CRD created
├─ T+5s: Topology controller observes new cluster
├─ T+10s: Infrastructure provider begins provisioning
├─ T+30s: VM creation starts
│  ├─ Control plane VM (1x)
│  └─ Worker VMs (1x)
├─ T+2m: VMs powered on, OS boots
├─ T+4m: Kubernetes components installed on control plane
├─ T+5m: Worker nodes join cluster
├─ T+7m: CNI and system pods deployed
├─ T+10m: Cluster networking ready
├─ T+12m: Health checks pass
└─ T+30m: Cluster Status = Ready ✓
   (Typical: 10-30 minutes depending on infrastructure speed)
```

## Detailed Wave 10 Sequence

```
Wave 10: Cluster Registration
├─ T+0s: Wave 0 detected as complete
├─ T+5s: Role created
├─ T+7s: RoleBinding created
├─ T+10s: ManagedEntity CRD created
├─ T+15s: Operator controller observes ManagedEntity
├─ T+20s: Operator validates argocd-k8s-sa exists
├─ T+25s: Operator checks RBAC binding
│  └─ Condition: SupervisorServiceAccountBound = True ✓
├─ T+30s: Operator fetches cluster info from Wave 0 Cluster
│  └─ Condition: ClusterInfoFetched = True ✓
├─ T+40s: Operator generates kubeconfig credentials
├─ T+45s: Operator creates cluster secret in ArgoCD
│  └─ Condition: SecretReady = True ✓
├─ T+50s: Operator validates all conditions
│  └─ Condition: Ready = True ✓
└─ T+60s: ManagedEntity Phase = Ready ✓
   (Typical: 2-5 minutes)
```

## Dependency Relationship

```
Wave 10 Resources
     │
     ├─ Role
     ├─ RoleBinding (references Role)
     └─ ManagedEntity
            └─ depends on
                └─ Wave 0: Cluster (argocd-vks)
                     └─ depends on
                         └─ Infrastructure Provider (VMware vSphere)
```

## Success Criteria

✅ **Wave 0 Success**
```
$ kubectl get cluster argocd-vks -n kim-prod-vlryx
NAME         PHASE   AGE
argocd-vks   Ready   15m
```

✅ **Wave 10 Success**
```
$ kubectl describe managedentity argocd-vks-managed -n kim-prod-vlryx | grep -A2 "Phase:"
Phase: Ready
```

✅ **Overall Success**
```
$ argocd cluster list
SERVER                          NAME            VERSION  STATUS      MESSAGE
https://172.16.44.33:6443       argocd-vks-prod          Successful
```

## Failure Points and Recovery

| Failure Point | Cause | Recovery |
|---|---|---|
| Wave 0 hangs | Infrastructure unavailable | Check vSphere capacity, networking |
| Wave 10 fails (RBAC) | Missing RoleBinding | Re-apply argocd-managedentity.yaml |
| Wave 10 fails (SA not bound) | Role not created | Verify Role exists, trigger reconciliation |
| ManagedEntity stuck Pending | Cluster not Ready | Wait for Wave 0 to complete |
| Cluster secret not created | Operator error | Check operator logs, restart operator |

## Rollback Procedure

```bash
# Option 1: Delete entire Application (cascading)
kubectl delete application argocd-vks-app-of-apps -n argocd

# Option 2: Delete in reverse wave order
kubectl delete managedentity argocd-vks-managed -n kim-prod-vlryx
kubectl delete cluster argocd-vks -n kim-prod-vlryx
```

## Wave Number Reference

- **Wave 0**: Initial infrastructure
- **Wave 1-9**: Available for other prerequisites
- **Wave 10**: Registration/post-creation tasks
- **Wave 11+**: Application deployment, additional configurations

This allows flexibility for adding more resources at different stages!
