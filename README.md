# Trilium Helm Chart

This is the Helm Chart for Trilium, to easily deploy Trilium on your Kubernetes cluster. This chart leverages the [bjw-s common library](https://github.com/bjw-s/helm-charts/blob/common-3.2.1/charts/library/common/values.yaml) to further increase the ease of use when deploying. Please view the previous link to see what values you can change/tweak to your needs [here](https://github.com/bjw-s/helm-charts/blob/common-3.2.1/charts/library/common/values.yaml).

Please refer to the section "Modifying Deployed Resources" below on how to customize the deployment, or refer to the examples in [the examples folder](./examples/)

Seperate from the [values.yaml](./charts/trilium/values.yaml), please also view the additional files in the [templates](./charts/trilium/templates/) folder to see the additional values that are provided to Helm, to create the Kubernetes release. These values can also be overridden, and the defaults should be completely unobtrusive to any changes that are commonly made.

If you find that a value in your release is inconsistent with those found in the [values.yaml](./charts/trilium/values.yaml) and the [bjw-s common library](https://github.com/bjw-s/helm-charts/blob/common-3.2.1/charts/library/common/values.yaml), then they are being modified in the [templates](./charts/trilium/templates/) folder. Any value changes specified by the user override any values defined within this chart.

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
  tag: 0.63.7
  pullPolicy: IfNotPresent
```

## Using Helm CLI

```bash
helm repo add trilium https://triliumnext.github.io/helm-charts
helm install --create-namespace --namespace trilium trilium trilium/trilium
```

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
          main:
            containers:
              trilium:
              image:
                repository: zadam/trilium
                tag: 0.63.7
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

### Modifying Deployed Resources

Often times, modifications need to be made to a Helm chart to allow it to operate in your Kubernetes cluster. By utilizing bjw-s's `common` library, there are quite a few options that can be easily modified.

Anything you see [here](https://github.com/bjw-s/helm-charts/blob/d9e8c23df242dd9a2dda7c3738360928526d7a20/charts/library/common/values.yaml), including the top-level keys, can be added and subtracted from this chart's `values.yaml`.

For example, if you wished to create a `serviceAccount`, refer to the values [here](https://github.com/bjw-s/helm-charts/blob/d9e8c23df242dd9a2dda7c3738360928526d7a20/charts/library/common/values.yaml#L364-L376), and override them as needed. So, to create a `serviceAccount`, you would want to add YAML below to your Helm release values:

```yaml
serviceAccount:
  create: true
```

Then, (for some reason), if you wished to change the Deployment type to `DaemonSet`, ([referencing the values here](https://github.com/bjw-s/helm-charts/blob/d9e8c23df242dd9a2dda7c3738360928526d7a20/charts/library/common/values.yaml#L96)), you could do the following:

```yaml
controllers:
  main:
    type: daemonset
```  

## Development

To use Helm in order to create the individual Kubernetes manifests needed to deploy it "by hand", you can use the following commands:

```bash
git clone https://github.com/TriliumNext/helm-charts
cd helm-chart/charts/trilium
helm dependency update
helm package .
helm template test1 . --namespace testing -f values.yaml --debug > output.yaml
```
