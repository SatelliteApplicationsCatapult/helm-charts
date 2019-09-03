# Sentinel-2 ARD Workflow Helm Chart

## Chart Details

This chart will deploy the following:

- 1 x Redis master
- 3 x Sentinel-2 ARD workers that retrieve jobs from the Redis master
- 1 x Jupyter Notebook (optional) with port 80 exposed on an external LoadBalancer (default)
- All using Kubernetes Deployments

> **Tip**: See the [Kubernetes Service Type Docs](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
for the differences between ClusterIP, NodePort, and LoadBalancer.

## Installing the Chart

To install the chart with the release name `my-release`:

```bash
helm install --name my-release stable/ard-workflow-s2
```

Depending on how your cluster was setup, you may also need to specify a namespace with the following flag: `--namespace my-namespace`.
