# ||WIP|| Trilium Helm Chart

This is the Helm Chart for Trilium, to easily deploy Trilium on your Kubernetes cluster. This chart leverages the [bjw-s common library](https://github.com/bjw-s/helm-charts/tree/a081de53024d8328d1ae9ff7e4f6bc500b0f3a29/charts/library/common) to further increase the ease of use when deploying.


## Example values

```yaml
image:
  repository: zadam/trilium
  tag: 0.63.5
  pullPolicy: IfNotPresent
```

## Development

To use Helm in order to create the individual Kubernetes manifests needed to deploy it "by hand", you can use the following commands:
```bash
git clone https://github.com/TriliumNext/helm-chart
cd helm-chart
helm template test1 . --namespace testing -f values.yaml --debug > output.yaml
```

