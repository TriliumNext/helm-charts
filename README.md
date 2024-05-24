# Trilium Helm Chart

This is the Helm Chart for Trilium, to easily deploy Trilium on your Kubernetes cluster. This chart leverages the [bjw-s common library](https://github.com/bjw-s/helm-charts/blob/common-3.1.0/charts/library/common/values.yaml) to further increase the ease of use when deploying. Please view the previous link to see what values you can change/tweak to your needs [here](https://github.com/bjw-s/helm-charts/blob/common-3.1.0/charts/library/common/values.yaml).

## Requirements

- A working Kubernetes cluster.
- A PVC provisioner.
  - If you don't have one, but have something that serves an NFS share, take a look at the following
    - [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
    - [democratic-csi](https://github.com/democratic-csi/democratic-csi)

## Deploying

```bash
helm repo add trilium https://triliumnext.github.io/helm-charts
helm install --create-namespace --namespace trilium trilium trilium/trilium -f values.yaml
```

### Example values

Below are some examples of what you could provide for the chart's values.

```yaml
image:
  repository: zadam/trilium
  tag: 0.63.6
  pullPolicy: IfNotPresent
```

## Using Helm CLI

## Using GitOps

If you want to use GitOps, essentially using a Git repository as the single source of truth for the applications in your cluster, you can use tools such as ArgoCD or Flux. Below is an example of what an "Application" that creates a Helm release in ArgoCD looks like:

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
    repoURL: https://trilium-next.github.io/helm-charts
    targetRevision: 0.0.1
    helm:
      values: |
  controllers:
    trilium:
   containers:
     trilium:
    image:
      repository: zadam/trilium
      tag: 0.63.6
      pullPolicy: IfNotPresent
    env:
      key: "value"

  persistence:
    data:
   enabled: true
   type: persistentVolumeClaim
   existingClaim: my-claim-1
  destination:
    server: "https://kubernetes.default.svc"
    namespace: apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true 
```

## Development

To use Helm in order to create the individual Kubernetes manifests needed to deploy it "by hand", you can use the following commands:

```bash
git clone https://github.com/TriliumNext/helm-chart
cd helm-chart
helm template test1 . --namespace testing -f values.yaml --debug > output.yaml
```
