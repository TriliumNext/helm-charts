# Trilium Helm Chart

The Helm chart for [Trilium Notes](https://github.com/TriliumNext/Trilium), a hierarchical note taking application with a focus on building large personal knowledge bases.

The chart is built on the [bjw-s common library](https://github.com/bjw-s-labs/helm-charts/blob/common-5.0.1/charts/library/common/values.yaml), so every value the library supports can be set directly in this chart's values. See [Customizing the deployment](#customizing-the-deployment) below and the [examples folder](./examples/).

## Installing

From the classic Helm repository:

```bash
helm repo add trilium https://triliumnext.github.io/helm-charts
helm install trilium trilium/trilium --namespace trilium --create-namespace
```

Or from the OCI registry:

```bash
helm install trilium oci://ghcr.io/triliumnext/helm-charts/trilium --namespace trilium --create-namespace
```

Charts published to the OCI registry are signed with cosign. Verify a version with:

```bash
cosign verify ghcr.io/triliumnext/helm-charts/trilium:<version> \
  --certificate-identity-regexp 'https://github.com/TriliumNext/helm-charts/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

By default the chart creates a 20Gi PersistentVolumeClaim for your notes, so all you need is a working PVC provisioner in the cluster. The PVC is annotated so it survives `helm uninstall`.

## Upgrading from 1.x to 2.0.0

Version 2.0.0 is a breaking release. What changed:

- The bjw-s common library was upgraded from 3.3.2 to 5.0.1. The Deployment selector labels changed, and selector labels are immutable in Kubernetes, so the old Deployment has to be deleted once before upgrading.
- The container image moved from `triliumnext/notes` to `triliumnext/trilium` (the old image name no longer receives updates), and the default version is now v0.104.0.
- Health probes now use `GET /api/health-check` instead of `/login`. Trilium v0.104.0 rate limits `/login`, which made the old probes restart-loop the pod ([TriliumNext/Trilium#10617](https://github.com/TriliumNext/Trilium/issues/10617)).
- The chart now creates its own PVC by default. `persistence.data.existingClaim` is still fully supported, it is just no longer required. All 1.x installs used an existing claim, and upgrades keep using it, so your data is untouched.
- A dedicated ServiceAccount is now created for the pod, with `automountServiceAccountToken: false`.

Steps to upgrade:

```bash
# 1. Take a backup of your Trilium data (always a good idea before upgrades).

# 2. Delete the old Deployment. Your PVC and data are not affected.
kubectl --namespace <namespace> delete deployment <release-name>

# 3. Upgrade the release.
helm repo update
helm upgrade <release-name> trilium/trilium --namespace <namespace> -f your-values.yaml
```

Notes:

- If your values pin `triliumnext/notes` as the image repository, remove that override. The image name changed upstream.
- Trilium migrates its database format on the first start of v0.104.0. The migration is automatic, but it is one more reason to take the backup in step 1.
- Your values file keeps the same flat shape as before. The `configini` block, `persistence.data.existingClaim`, ingress definitions, and other common library overrides all keep working.

## Persistence

The chart provisions a `ReadWriteOnce` PVC by default (Trilium uses SQLite, so the volume must not be shared between nodes):

```yaml
persistence:
  data:
    size: 20Gi
    # storageClass: my-storage-class
```

The PVC carries the `helm.sh/resource-policy: keep` annotation, so uninstalling the release leaves your notes in place.

To bring your own PVC instead:

```yaml
persistence:
  data:
    existingClaim: my-existing-claim
```

## Configuration

### config.ini

The `configini` block renders Trilium's `config.ini` into a ConfigMap. Changing it automatically restarts the pod so the new settings take effect:

```yaml
configini:
  general:
    instanceName: ""
    # Disable authentication (for private networks, or when auth is handled
    # by another component in front of Trilium)
    noAuthentication: false
    # Disable automatic database backups
    noBackup: false
  network:
    host: "0.0.0.0"
    port: 8080
    https: false
    certPath: ""
    keyPath: ""
    trustedReverseProxy: true
```

### Environment variables

Trilium can also be configured through environment variables named `TRILIUM_<SECTION>_<KEY>`, which override the corresponding `config.ini` values:

```yaml
controllers:
  main:
    containers:
      trilium:
        env:
          TRILIUM_GENERAL_INSTANCENAME: my-trilium
          TRILIUM_NETWORK_TRUSTEDREVERSEPROXY: "true"
```

For secret values, prefer a Kubernetes Secret loaded with `envFrom`:

```yaml
controllers:
  main:
    containers:
      trilium:
        envFrom:
          - secret: trilium-secrets
```

## Permissions (UID/GID)

Trilium runs as UID/GID 1000 by default. An init container fixes the ownership of the data directory before Trilium starts, and the image entrypoint drops to the same UID/GID. To run as a different user:

```yaml
permissions:
  uid: 568
  gid: 568

defaultPodOptions:
  securityContext:
    fsGroup: 568
```

If your storage already handles ownership (or you do not want a root init container), disable it:

```yaml
controllers:
  main:
    initContainers:
      fixperms:
        enabled: false
```

## Exposing Trilium

An ingress example in the common library 5.x shape:

```yaml
ingress:
  main:
    enabled: true
    className: nginx
    annotations:
      # remove the request body size limit for large file uploads
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
    hosts:
      - host: trilium.example.com
        paths:
          - path: /
            pathType: Prefix
            service:
              identifier: main
              port: http
```

## Customizing the deployment

Anything from the [common library values](https://github.com/bjw-s-labs/helm-charts/blob/common-5.0.1/charts/library/common/values.yaml) can be set at the top level of this chart's values and is merged with the chart defaults. For example, to run as a DaemonSet:

```yaml
controllers:
  main:
    type: daemonset
```

The chart's health probes target Trilium's unauthenticated `GET /api/health-check` endpoint. If you run an older Trilium version that lacks it, override `controllers.main.containers.trilium.probes` in your values.

## GitOps

An ArgoCD Application using this chart (see [examples/argocd-application.yaml](./examples/argocd-application.yaml) for the full file):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trilium
  namespace: argocd
spec:
  project: default
  source:
    chart: trilium
    repoURL: https://triliumnext.github.io/helm-charts
    targetRevision: 2.0.0
    helm:
      values: |
        persistence:
          data:
            size: 20Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: trilium
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Development

```bash
git clone https://github.com/TriliumNext/helm-charts
cd helm-charts
helm dependency update charts/trilium
helm lint charts/trilium
helm template trilium charts/trilium > output.yaml
```

Releases are automated with [release-please](https://github.com/googleapis/release-please): pull request titles follow [Conventional Commits](https://www.conventionalcommits.org/) (they become the squash commit messages), and merging the generated release PR tags the release, publishes the chart to the GitHub release, the `https://triliumnext.github.io/helm-charts` index, and `oci://ghcr.io/triliumnext/helm-charts`, and signs the OCI artifact with cosign.
