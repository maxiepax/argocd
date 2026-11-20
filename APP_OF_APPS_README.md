# ArgoCD VKS Cluster Registration - App of Apps Pattern

This directory contains manifests for a VKS cluster, its registration with ArgoCD, and VKS add-ons.

## Architecture

```
argocd-vks-app-of-apps (Main Application)
├── Wave 0: Create infra-vks-01
│
└── Wave 10: Cluster Registration to ArgoCD (argocd-managedentity.yaml)
    ├── Create Role for ArgoCD service account
    ├── Create RoleBinding
    ├── Create a ManagedEntity for the cluster
    └── Request cert-manager, contour, and prometheus
```

## Files

### `infra-vks-01.yaml`
- **Purpose**: Defines the VKS cluster to be created
- **Wave**: 0 (First wave - initial resource creation)
- **API Group**: `cluster.x-k8s.io/v1beta2`
- **Kind**: Cluster
- **Contains**:
  - Cluster network configuration (pods, services, serviceDomain)
  - Topology specification with cluster class and version
  - Control plane configuration (1 replica)
  - Worker node pool configuration (1 replica)
  - VM class and storage class settings

### `cert-manager-addon.yaml`, `contour-addon.yaml`, and `prometheus-addon.yaml`
- **Purpose**: Request VKS-managed add-ons for `infra-vks-01`
- **Wave**: 10
- **Kind**: AddonInstall, created on the Supervisor

### `argocd-managedentity.yaml`
- **Purpose**: Registers the VKS cluster to ArgoCD
- **Wave**: 10 (Second wave - happens after cluster is ready)
- **Contains**:
  - `Role`: RBAC role for `argocd-k8s-sa` service account to access cluster resources
  - `RoleBinding`: Binds the role to the service account
  - A `ManagedEntity` resource that:
    - References the VKS cluster (wave 0)
    - Creates a cluster secret in ArgoCD
    - Makes the cluster available as a deployment target
    - Enables App of Apps pattern on the cluster

### `argocd-app-of-apps.yaml`
- **Purpose**: Main orchestration Application, syncing the cluster and registration YAML files in wave order
- **Project**: Uses the existing `default` AppProject

### `argocd-vks-appproject.yaml`
- **Purpose**: Optional `vks-cluster-registration` AppProject, provided separately
- The Application does not use this project unless `spec.project` is changed.

## Prerequisites

1. **VMware Operator for ArgoCD 1.2.0+** installed in your vSphere environment
2. **ArgoCD instance** installed and its Application namespace known
3. **argocd-k8s-sa** ServiceAccount must exist (created by VMware Operator)
4. **Git repository access**: Application references this GitHub repository
5. **Permissions**:
   - Can create resources in `kim-prod-vlryx` vSphere namespace
   - Can create Roles and RoleBindings in that namespace
   - Can create Applications in ArgoCD namespace

The Application targets the registered Supervisor proxy endpoint rather than
`https://kubernetes.default.svc`. If the endpoint differs in your environment,
update the Application's `spec.destination.server` and, if used, the optional
AppProject's `spec.destinations` to match the server registered in ArgoCD.

## Deployment

### Option 1: Deploy via kubectl (Direct)

```bash
# Apply all resources in correct order
kubectl apply -f argocd-vks/infra-vks-01.yaml
# Wait for the cluster to become ready before applying registration and add-ons.
kubectl apply -f argocd-vks/argocd-managedentity.yaml
kubectl apply -f argocd-vks/cert-manager-addon.yaml
kubectl apply -f argocd-vks/contour-addon.yaml
kubectl apply -f argocd-vks/prometheus-addon.yaml
```

### Option 2: Deploy via ArgoCD (Recommended - Full App of Apps)

To use the ArgoCD UI, create an Application using the YAML editor and paste the
entire `argocd-app-of-apps.yaml` file. It contains one Application document.
Commit and push the source manifests before syncing.
The Application's metadata namespace is omitted so the UI can use its own
namespace. For kubectl, replace `argocd` below with the ArgoCD installation
namespace if it differs; this is separate from the destination namespace.

```bash
# Apply the App of Apps Application
kubectl apply -n argocd -f argocd-app-of-apps.yaml

# Monitor the application
argocd app get argocd-vks-app-of-apps
argocd app wait argocd-vks-app-of-apps --sync

# Watch real-time progress
argocd app wait argocd-vks-app-of-apps --sync -v
```

The separate `argocd-vks-appproject.yaml` is optional and is not created by
applying the Application. To use it, create the AppProject separately and set
the Application's `spec.project` to `vks-cluster-registration`.

## VKS Add-ons

The three `AddonInstall` resources request cert-manager, contour, and prometheus
using VKS add-on management, following the [reference manifests](https://github.com/vmware/vks-consumption-models/tree/main/argocd-app-of-apps/clusters/cl01).
Each selects only the cluster labeled `cluster.x-k8s.io/cluster-name: infra-vks-01`.
The Supervisor must provide the AddonInstall CRD and compatible add-on releases.

On the Supervisor, check availability and reconciliation:

```bash
kubectl get addonreleases -n vmware-system-vks-public
kubectl get addoninstall -n kim-prod-vlryx
kubectl get clusteraddon -n kim-prod-vlryx
kubectl describe addoninstall infra-vks-01-cert-manager -n kim-prod-vlryx
kubectl describe addoninstall infra-vks-01-contour -n kim-prod-vlryx
kubectl describe addoninstall infra-vks-01-prometheus -n kim-prod-vlryx
```

An ArgoCD sync applies the requests but does not itself prove installation
succeeded. Check the add-on statuses and availability in the workload cluster.

## Wave Execution Timeline

The cluster is in wave 0. ArgoCD advances between waves based on resource
health; verify that your ArgoCD installation has an appropriate health
assessment for the Cluster CRD before relying on it to wait for readiness.

### Wave 0 Execution
1. VKS Cluster resource is created
2. Kubernetes control plane begins provisioning
3. Worker nodes are created and joined to cluster
4. Cluster health checks run
5. Status moves to "Ready" state

**Duration**: 10-30 minutes depending on infrastructure

### Wave 10 Execution (Triggered after Wave 0 completes)
1. Role for `argocd-k8s-sa` is created with necessary permissions
2. RoleBinding connects the service account to the role
3. ManagedEntity resource is created
4. AddonInstall resources are created
5. VMware Operator validates service account permissions
6. Operator creates kubeconfig and credentials
7. Cluster secret is created in ArgoCD
8. ManagedEntity reaches "Ready" status
9. Cluster appears in ArgoCD's cluster list

**Duration**: 2-5 minutes

## Monitoring Status

### Check Application Status
```bash
# Check overall application status
kubectl get application argocd-vks-app-of-apps -n argocd -w

# Detailed application information
argocd app describe argocd-vks-app-of-apps
```

### Check Wave 0: Cluster Creation
```bash
# Watch cluster provisioning
kubectl get cluster infra-vks-01 -n kim-prod-vlryx -w

# Detailed cluster status
kubectl describe cluster infra-vks-01 -n kim-prod-vlryx
```

### Check Wave 10: Cluster Registration
```bash
# Check ManagedEntity status
kubectl get managedentity -n kim-prod-vlryx
kubectl describe managedentity infra-vks-01-managed -n kim-prod-vlryx

# Check generated cluster secret
kubectl get secret -n kim-prod-vlryx | grep infra-vks-01
kubectl get secret cls-vks-kim-prod-vlryx-infra-vks-01 -n kim-prod-vlryx -o yaml
```

### Check ArgoCD Cluster Registration
```bash
# List clusters in ArgoCD
argocd cluster list

# Check specific cluster details
argocd cluster get <cluster-name>
```

## Expected Status Conditions

### ManagedEntity Conditions

| Condition | Type | Status | Description |
|-----------|------|--------|-------------|
| ArgoCDExists | | True | ArgoCD instance found |
| ClusterInfoFetched | | True | Cluster API endpoint retrieved |
| SupervisorServiceAccountBound | | True | Service account validated with RBAC |
| SecretReady | | True | ArgoCD cluster secret created |
| Ready | | True | All conditions satisfied |

**Phase**: Should show `Ready` when complete

## Troubleshooting

### ManagedEntity stuck in "Pending"
```bash
kubectl describe managedentity infra-vks-01-managed -n kim-prod-vlryx
```
Check conditions for specific failure reasons.

### Service Account Not Bound Error
```bash
# Verify role exists
kubectl get role argocd-k8s-sa-role -n kim-prod-vlryx

# Verify rolebinding exists
kubectl get rolebinding argocd-k8s-sa-rolebinding -n kim-prod-vlryx

# Force reconciliation
kubectl patch managedentity infra-vks-01-managed -n kim-prod-vlryx \
  --type merge -p '{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt":"'$(date -Iseconds)'"}}}'
```

### Cluster Not Appearing in ArgoCD
1. Verify ManagedEntity status is "Ready"
2. Check cluster secret was created: `kubectl get secret -n kim-prod-vlryx | grep cls-vks`
3. Restart ArgoCD application controller: `kubectl rollout restart deployment/argocd-application-controller -n argocd`
4. Force ArgoCD to reload clusters: `argocd cluster list --refresh`

## Customization

### Changing Cluster Configuration
Edit `infra-vks-01.yaml`:
- `vmClass`: VM machine class (best-effort-small, best-effort-medium, etc.)
- `version`: Kubernetes version
- `replicas`: Control plane and worker replicas
- `storageClass`: vSAN storage policy
- `cidrBlocks`: Pod and service IP ranges

### Changing Registration Project
Edit `argocd-managedentity.yaml`:
- `argoCDProject`: Change from "default" to custom ArgoCD project
- `namespaces`: Restrict which namespaces ArgoCD can manage

### Changing Wave Timing
To change the execution delay between waves, adjust the annotation in the manifest:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "10"  # Change this number
```

## Security Considerations

1. **Service Account Permissions**: The Role grants read-only access to cluster resources, appropriate for discovery
2. **RBAC Isolation**: Resources are namespace-scoped, not cluster-wide
3. **Secret Management**: ArgoCD-generated cluster secrets are stored in the vSphere namespace
4. **Project Isolation**: The Application uses the `default` project unless configured to use a separate AppProject

## References

- [ArgoCD Sync Waves Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [VMware Operator for ArgoCD](https://docs.vmware.com/en/VMware-Operator-for-ArgoCD/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps)
- [Cluster API - Cluster CRD](https://cluster-api.sigs.k8s.io/)

## Maintenance

### Updating Cluster Configuration
1. Modify `infra-vks-01.yaml` with new settings
2. Commit to Git repository
3. ArgoCD will automatically detect and sync changes (if automated sync is enabled)

### Removing the Cluster
```bash
# Delete the Application (cascading delete)
kubectl delete application argocd-vks-app-of-apps -n argocd

# Or manually delete resources in reverse wave order
kubectl delete managedentity infra-vks-01-managed -n kim-prod-vlryx
kubectl delete cluster infra-vks-01 -n kim-prod-vlryx
```

## Support

For issues with:
- **VKS Cluster Creation**: Check VMware vSphere logs
- **ManagedEntity Registration**: Check VMware Operator for ArgoCD logs
- **ArgoCD Integration**: Check ArgoCD application controller logs
