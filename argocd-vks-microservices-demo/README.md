# VKS cluster and Online Boutique with Argo CD

This demo manages the `infra-vks-01` VKS cluster, its Argo CD
registration, VKS add-ons, and the Online Boutique demo using an App of Apps.
The Supervisor namespace is `kim-prod-vlryx`; the workload namespace is
`online-boutique`.

## Quick start

### Prerequisites

- An Argo CD instance on the Supervisor, with access to this Git repository
  and the Supervisor destination configured in `argocd-app-of-apps.yaml`.
- VKS, the VMware Operator for Argo CD, and the `argocd-k8s-sa` service
  account in `kim-prod-vlryx`. The Supervisor must provide the `AddonInstall`
  CRD and compatible cert-manager, contour, and prometheus add-on releases.
- Sufficient vSphere capacity, network connectivity, the configured storage
  class, and the Kubernetes version and VM class specified in the cluster
  manifest. The workload cluster also needs access to its container registries.

### Deploy

1. Commit and push the manifests to the repository referenced by the parent
   Application. The manifests use `HEAD` as their target revision.
2. In the Argo CD UI, create an Application using the YAML editor and paste
   `argocd-app-of-apps.yaml`. It is a single Application document. Alternatively,
   using a Supervisor context with access to the Argo CD installation namespace:

   From this directory:

   ```bash
   kubectl apply -n kim-prod-vlryx -f argocd-app-of-apps.yaml
   ```

   The parent manifest omits `metadata.namespace`; use the actual Argo CD
   installation namespace if it differs. The parent uses `kim-prod-vlryx` for
   both the registered destination name and namespace, as supported by this
   VCF Automation environment.

3. Monitor the parent and verify that `infra-vks-01` appears in Argo CD:

   ```bash
   argocd app get argocd-vks-app-of-apps
   argocd cluster list
   ```

For an existing parent Application, changes to `argocd-app-of-apps.yaml` must
also be applied to that Application; it does not manage itself. When moving the
source directory, pause automated sync, push the new directory, update the parent
Application to use the new path, and review its diff before resuming sync.
Automatic pruning can delete resources that disappear from the selected source.

## Wave diagram

```text
argocd-vks-app-of-apps  ->  Supervisor / kim-prod-vlryx
  |
  +-- Wave 0:  Cluster infra-vks-01
  |
  +-- Wave 10: Role + RoleBinding + ManagedEntity
  |            cert-manager + contour + prometheus AddonInstalls
  |
  +-- Wave 20: Application infra-vks-01-online-boutique
                 |
                 +-- Deployments and Services -> infra-vks-01 / online-boutique
```

Waves order resource application; they do not by themselves guarantee that
the cluster, registration, or add-ons are ready. Verify Argo CD has suitable
health assessments for the relevant CRDs and check their status before relying
on wave ordering. Add-on reconciliation and workload readiness are separate
from creating the `AddonInstall` resources.

## Repository layout

```text
argocd/
  argocd-vks-microservices-demo/
    README.md                           This guide
    argocd-app-of-apps.yaml             Parent Application
    argocd-vks-appproject.yaml          Optional AppProject
    infrastructure/
      infra-vks-01.yaml                 Cluster (wave 0)
      cert-manager-addon.yaml           VKS add-on (wave 10)
      contour-addon.yaml                VKS add-on (wave 10)
      prometheus-addon.yaml             VKS add-on (wave 10)
    argocd-managedentity.yaml           Registration and RBAC (wave 10)
    application/
      online-boutique-app.yaml          Child Application (wave 20)
      deployments.yaml                  Online Boutique Deployments
      services.yaml                     Online Boutique Services
  argocd-vm/                            Separate Argo CD VM demo
```

The parent selects infrastructure, registration, and `*-app.yaml` Application
manifests. The child selects the workload directory but excludes `*-app.yaml`,
so it does not deploy its own Application manifest into the workload cluster.

The cluster manifest currently specifies one control plane and two workers
using `best-effort-medium`. The `ManagedEntity` registers the cluster as
`infra-vks-01`. The parent and child Applications use the `default` project.
The `argocd-vks-appproject.yaml` file is optional and is not used unless
configured separately; its namespace and permissions must match the intended
Argo CD installation and destinations.

## Monitoring and troubleshooting

Use the **Supervisor context** for cluster, registration, and add-on status:

```bash
kubectl get cluster infra-vks-01 -n kim-prod-vlryx
kubectl describe managedentity infra-vks-01-managed -n kim-prod-vlryx
kubectl get addonreleases -n vmware-system-vks-public
kubectl get addoninstall,clusteraddon -n kim-prod-vlryx
kubectl describe addoninstall infra-vks-01-contour -n kim-prod-vlryx
```

For add-on failures, describe the corresponding `infra-vks-01-cert-manager`
or `infra-vks-01-prometheus` AddonInstall as well. If registration fails,
check the ManagedEntity conditions and the `argocd-k8s-sa` RoleBinding.

Switch to the **infra-vks-01 workload context** to inspect the application:

```bash
kubectl get pods,services -n online-boutique
kubectl get service frontend-external -n online-boutique
kubectl describe pod -n online-boutique <pod-name>
kubectl describe nodes
```

The `frontend-external` Service requests a LoadBalancer address. Pending pods
may indicate insufficient allocatable CPU or memory, node taints, image pull
failures, or Pod Security admission errors; inspect pod events and node
allocated resources. Redis uses an `emptyDir` volume, so its data is ephemeral.

## Changes and deletion

Edit the manifests and push changes for automated sync. Change worker count,
VM class, network, or Kubernetes version in `infra-vks-01.yaml`; ensure the
Supervisor has enough capacity and the selected VM class is available.

**Automatic pruning and self-healing are enabled.** Removing a managed
resource from Git can delete it on sync. The parent Application also has a
cascading-deletion finalizer, so deleting it can delete managed resources,
including the VKS cluster. The Online Boutique child Application has no
cascading-deletion finalizer; deleting it alone does not delete its workloads.
Review the Argo CD deletion and pruning behavior before removing either.

## References

- [Argo CD sync waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [Argo CD App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#app-of-apps)
- [VMware VKS consumption model example](https://github.com/vmware/vks-consumption-models/tree/main/argocd-app-of-apps)
