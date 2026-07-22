# Changelog

## [2.0.0](https://github.com/TriliumNext/helm-charts/compare/trilium-1.3.0...trilium-2.0.0) (2026-07-22)


### ⚠ BREAKING CHANGES

* The chart now uses the bjw-s common library 5.0.1. The Deployment selector labels changed, so the existing Deployment must be deleted before upgrading (kubectl delete deployment <release-name>). The image moved from triliumnext/notes to triliumnext/trilium (v0.104.0), probes moved to /api/health-check, and the chart now creates a PVC by default (persistence.data.existingClaim is still supported and no longer required).

### Features

* migrate chart to bjw-s common 5.0.1 and Trilium v0.104.0 ([68d5458](https://github.com/TriliumNext/helm-charts/commit/68d5458d266d298c9cec57b1b50f9ea08757dd90))
