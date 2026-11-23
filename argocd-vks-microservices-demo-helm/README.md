# VKS and Online Boutique with Argo CD and Helm

This is the Helm version of the VKS microservices demo. The parent Argo CD
Application renders the chart in this directory; the wave-20 child Application
deploys the same Online Boutique manifests as the non-Helm demo.

## Quick start

1. Fork or copy this repository. Install Helm locally and ensure your Supervisor
   has VKS, the VMware Operator for Argo CD, the `argocd-k8s-sa` service
   account in your Supervisor namespace, and the required add-on releases.
   The Argo CD instance must be able to access the repository, the registered
   Supervisor destination, and the registered VKS cluster.
2. Edit [values.yaml](values.yaml): set the repository URL and revision, the
   Supervisor namespace and its registered proxy endpoint, and the VKS
   settings for your environment. The defaults describe the original demo
   environment and will generally need to be changed. Ensure the `default`
   project, or your selected project, allows the repository, destinations, and
   resources. The child Application is created in the Supervisor namespace,
   so Argo CD must allow Applications in that namespace.
3. Commit and push the changes to the revision configured in `values.yaml`.
   The bootstrap and Argo CD must use the **same committed values**. Do not
   use local-only `--set` or uncommitted values: Argo CD reads `values.yaml`
   from the configured repository and revision.
4. From this directory, render the single bootstrap Application and paste the
   output into the Argo CD UI's YAML editor:

   ```bash
   helm template vks-bootstrap ./bootstrap -f values.yaml
   ```

   Alternatively, apply it using a Supervisor context with access to the
   Argo CD Application namespace:

   ```bash
   helm template vks-bootstrap ./bootstrap -f values.yaml \
     | kubectl apply -n <argocd-application-namespace> -f -
   ```

   The bootstrap Application omits `metadata.namespace`, matching the
   original demo. Select the appropriate namespace in the UI or command.
5. Monitor the parent Application and verify the cluster registration:

   ```bash
   argocd app get argocd-vks-app-of-apps
   argocd cluster list
   ```

The Supervisor endpoint must already be registered in Argo CD; it is not
necessarily `https://kubernetes.default.svc`. The VKS class, version, VM class,
storage class, networks, and add-on releases must be available and suitable
for the target Supervisor.

## Wave diagram

```text
argocd-vks-app-of-apps -> Supervisor / configured namespace
  |
  +-- Wave 0:  VKS Cluster
  +-- Wave 10: Role, RoleBinding, ManagedEntity, and three AddonInstalls
  +-- Wave 20: Online Boutique Application -> VKS / workload namespace
```

Waves order resource application but do not guarantee readiness. Configure and
verify suitable Argo CD health assessments for the VKS and registration CRDs
before relying on waves to gate later resources.

## Changes and deletion

Commit and push changes to `values.yaml` for Argo CD to pick up. The parent
does **not** manage its own bootstrap manifest: changes to its destination,
project, repository, revision, or chart path also require re-rendering and
updating the live parent Application. For these changes, pause automated sync,
commit and push the values, update the parent, and review the diff before
resuming sync.

This chart uses the same Application and resource names as the non-Helm demo.
Do not deploy both against the same environment as independent demos. To
migrate an existing parent, pause automated sync, push the chart and values,
update the parent using the rendered bootstrap manifest, review the diff, and
then resume sync. Changing names, namespaces, or destinations can cause
resources to be replaced or deleted.

Automated pruning and self-healing are enabled. Deleting the parent Application
can cascade to managed resources, including the cluster. The child Application
has no cascading-deletion finalizer. Online Boutique's `frontend-external`
Service requests a LoadBalancer address; Redis data is ephemeral (`emptyDir`).
