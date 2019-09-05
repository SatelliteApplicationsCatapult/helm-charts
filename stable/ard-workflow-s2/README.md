# Sentinel-2 ARD Workflow Helm Chart

In order to use the deployment material described here you need access to a Kubernetes server or cluster with Helm installed. For development and learning purposes [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and [microk8s](https://microk8s.io/) can be used too.

For the full set of configurable options see [values.yaml](values.yaml).

## Chart Details

This chart will deploy the following:

- 3 x Sentinel-2 ARD workers that retrieve jobs from a Redis master
- 1 x Jupyter Notebook (optional) with port 80 exposed on an external LoadBalancer (default)
- All using Kubernetes Deployments

> **Tip**: See the [Kubernetes Service Type Docs](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
for the differences between ClusterIP, NodePort, and LoadBalancer.

## Installing the Chart

It's necessary to first create a *ard-values.yaml* file specific to the Kubernetes cluster where the ARD workflow is being deployed. As example:

```yaml
jupyter:
  enabled: false

aws:
  aws_access_key: "AKIAIOSFODNN7INVALID"
  aws_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYINVALIDKEY"
```

To install the chart with the release name `my-release`:

```bash
helm install --name my-release stable/ard-workflow-s2 \
  --values ard-values.yaml
```

Depending on how your cluster was setup, you may also need to specify a namespace with the following flag: `--namespace my-namespace`.
