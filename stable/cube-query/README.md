# Cube Query Helm Chart

In order to use the deployment material described here you need access to a Kubernetes server or cluster with Helm installed. For development and learning purposes [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and [microk8s](https://microk8s.io/) can be used too.

## Scope

The solution provided here is suitable to deploy the API to produce on-demand data products, leveraging an [Open Data Cube](https://www.opendatacube.org/).

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
```

For a production environment, we might have instead:

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
    storageClass: "fast"
    size: "1Gi"
```

For the full set of configurable options see [values.yaml](https://github.com/bitnami/charts/blob/master/bitnami/redis/values.yaml).

In order to deploy the Master issue the following:

```bash
NAMESPACE=cubequery

kubectl create namespace $NAMESPACE

RELEASEREDIS=redis

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm upgrade --install $RELEASEREDIS bitnami/redis \
  --namespace $NAMESPACE \
  --version 10.7.16 \
  --values values-redis.yaml
```

## Chart Details

By default, this Chart will deploy the following:

- 3 x workers
- 1 x server with port 5000 exposed on an external LoadBalancer (default)
- All using Kubernetes Deployments

> **Tip**: See the [Kubernetes Service Type Docs](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
for the differences between ClusterIP, NodePort, and LoadBalancer.

### Installing the Chart

Make Helm aware of the [Satellite Applications Catapult Helm chart repository](https://satelliteapplicationscatapult.github.io/helm-charts/), so you will be able to install the ARD chart from it without having to use a long URL name:

```bash
helm repo add satapps https://satelliteapplicationscatapult.github.io/helm-charts
helm repo update
```

It's then necessary to create a *values-cubequery.yaml* file specific to the Kubernetes cluster and architecture in use.\
For the full set of configurable options see [values.yaml](values.yaml).

As example, for a development environment we might have:

```yaml
server:
  env:
    - name: APP_DEBUG
      value: "True"
    - name: APP_RESULT_DIR
      value: "/data/"

worker:
  replicaCount: 1
  env:
    - name: APP_DEBUG
      value: "True"
    - name: APP_RESULT_DIR
      value: "/data/"
    - name: AWS_NO_SIGN_REQUEST
      value: "YES"
    - name: AWS_VIRTUAL_HOSTING
      value: "FALSE"
    - name: AWS_S3_ENDPOINT
      value: "s3-uk-1.sa-catapult.co.uk"
```

For a production environment, we might have instead:

```yaml
server:
  usersConf: |
    # User, password hash (bcrypt), allowed addresses
    basic,$2b$12$LJxr6ilLLg.wSfiFGLp1w.s1A6H23z/6oV6qlcA/4..HoZNjMi/Uy,127.0.0.1;10.*
  env:
    - name: APP_RESULT_DIR
      value: "/data/"

worker:
  replicaCount: 7
  env:
    - name: APP_RESULT_DIR
      value: "/data/"
    - name: AWS_VIRTUAL_HOSTING
      value: "FALSE"
    - name: AWS_S3_ENDPOINT
      value: s3-uk-1.sa-catapult.co.uk
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