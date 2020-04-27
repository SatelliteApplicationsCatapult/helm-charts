# Cube Query Helm Chart

In order to use the deployment material described here you need access to a Kubernetes server or cluster with Helm installed. For development and learning purposes [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and [microk8s](https://microk8s.io/) can be used too.

## Scope

The solution provided here is suitable to deploy the API to produce data products from an Open Data Cube.

## Redis Master server deployment

It's necessary to first create a *values-redis.yaml* file. As example, for a development environment we might have:

```yaml
## Cluster settings
cluster:
  enabled: false

## Use password authentication
usePassword: false

##
## Redis Master parameters
##
master:
  ## Enable persistence using Persistent Volume Claims
  ## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
  ##
  persistence:
    enabled: false

## Redis config file
## ref: https://redis.io/topics/config
##
configmap: |-
  # Enable AOF https://redis.io/topics/persistence#append-only-file
  appendonly yes
  # Disable RDB persistence, AOF persistence already enabled.
  save ""
```

For a production environment, we might have instead:

```yaml
## Cluster settings
cluster:
  enabled: true

## Use password authentication
usePassword: false

##
## Redis Master parameters
##
master:
  ## Enable persistence using Persistent Volume Claims
  ## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
  ##
  persistence:
    enabled: true
    storageClass: "fast"
    size: "1Gi"

## Redis config file
## ref: https://redis.io/topics/config
##
configmap: |-
  # Enable AOF https://redis.io/topics/persistence#append-only-file
  appendonly yes
  # Disable RDB persistence, AOF persistence already enabled.
  save ""
```

For the full set of configurable options see [values.yaml](https://github.com/helm/charts/blob/master/stable/redis/values.yaml).

In order to deploy the Master issue the following:

```bash
NAMESPACE=cubequery

RELEASEREDIS=redis

helm upgrade --install $RELEASEREDIS stable/redis \
  --namespace $NAMESPACE \
  --version 9.1.3 \
  --values values-redis.yaml
```

## Chart Details

By default, this Chart will deploy the following:

- 3 x workers
- 1 x server with port 5000 exposed on an external LoadBalancer (default)
- All using Kubernetes Deployments

> **Tip**: See the [Kubernetes Service Type Docs](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
for the differences between ClusterIP, NodePort, and LoadBalancer.

### Installing the Chart - WIP

It's then necessary to create a *values-ard.yaml* file specific to the Kubernetes cluster and architecture in use.\
For the full set of configurable options see [values.yaml](values.yaml).

As example, for a development environment we might have:

```yaml
worker:
  env:
    - name: AWS_NO_SIGN_REQUEST
      value: "YES"
    - name: AWS_VIRTUAL_HOSTING
      value: "FALSE"
    - name: AWS_S3_ENDPOINT
      value: s3-uk-1.sa-catapult.co.uk

server:
  env:
    - name: APP_DEBUG
      value: "true"
```

For a production environment, we might have instead:

```yaml
worker:
  replicaCount: 7
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: component
              operator: In
              values:
              - worker
          topologyKey: "kubernetes.io/hostname"

aws:
  accessKeyId: "AKIAIOSFODNN7INVALID"
  secretAccessKey: "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"
```
