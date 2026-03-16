# GitOps Demo

Simple test application managed by ArgoCD on OpenShift.

## Contents
- `manifests/namespace.yaml` - Namespace
- `manifests/deployment.yaml` - Nginx deployment (2 replicas)
- `manifests/service.yaml` - ClusterIP Service
- `manifests/route.yaml` - OpenShift Route

## Test GitOps
Change `replicas` or the HTML content in `deployment.yaml`, push to Git, and watch ArgoCD sync.
