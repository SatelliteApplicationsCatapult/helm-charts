# Open Data Cube Routine Product Generation Helm Chart

In order to use the deployment material described here you need access to a Kubernetes server or cluster with Helm installed. For development and learning purposes [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and [microk8s](https://microk8s.io/) can be used too.

## Scope

The solution provided here is suitable to generate summary products leveraging an [Open Data Cube](https://www.opendatacube.org/) instance and a distributed [Dask](https://dask.org/) cluster.

## Architecture

We run a Kubernetes `Job` with a single worker `Pod`. As the `Pod` is created, it picks up one unit of work from a task queue, processes it, and repeats until the end of the queue is reached.

We use [Redis](https://redis.io/) as storage service to hold the work queue and store our work items. Each work item represents one product to be generated.

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

For the full set of configurable options see [values.yaml](https://github.com/helm/charts/blob/master/stable/redis/values.yaml).

In order to deploy the Master issue the following:

```bash
NAMESPACEODCPROD=odc-routine-products

RELEASEREDIS=redis

helm upgrade --install $RELEASEREDIS stable/redis \
  --namespace $NAMESPACEODCPROD \
  --version 9.1.3 \
  --values values-redis.yaml
```

## Job definitions

The work queue can be populated with job definitions either manually or programmatically.

### Manual job definition

The list with key `jobProduct` is the work queue for all work items. Insert items with e.g.:

```bash
$ kubectl run --namespace $NAMESPACEODCPROD redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:5.0.5-debian-9-r104 -- bash

I have no name!@redis-client:/$ redis-cli -h redis-master

redis-master:6379> rpush jobProduct '{"job_code": "geomedian", "product": "ls8_usgs_sr_scene", "query_x_from": "2130000.0", "query_y_from": "3499700.0", "query_x_to": "2233300.0", "query_y_to": "3600300.0", "query_crs": "EPSG:3460", "time_from": "2019-01-01", "time_to": "2019-12-31", "output_crs": "EPSG:3460", "prefix": "common_sensing/fiji/landsat_8_geomedian/2019"}'
(integer) 1
redis-master:6379> lrange jobProduct 0 -1
1) '{\"job_code\": \"geomedian\", \"product\": \"ls8_usgs_sr_scene\", \"query_x_from\": \"2130000.0\", \"query_y_from\": \"3499700.0\", \"query_x_to\": \"2233300.0\", \"query_y_to\": \"3600300.0\", \"query_crs\": \"EPSG:3460\", \"time_from\": \"2019-01-01\", \"time_to\": \"2019-12-31\", \"output_crs\": \"EPSG:3460\", \"prefix\": \"common_sensing/fiji/landsat_8_geomedian/2019\"}'
```

For [mass insertion](https://redis.io/topics/mass-insert) you can use e.g.:

```bash
$ kubectl run --namespace $NAMESPACEODCPROD redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:5.0.5-debian-9-r104 -- bash

I have no name!@redis-client:/$ cat <<EOF | redis-cli -h redis-master --pipe
rpush jobProduct '{"job_code": "geomedian", "product": "ls8_usgs_sr_scene", "query_x_from": "2130000.0", "query_y_from": "3499700.0", "query_x_to": "2233300.0", "query_y_to": "3600300.0", "query_crs": "EPSG:3460", "time_from": "2019-01-01", "time_to": "2019-12-31", "output_crs": "EPSG:3460", "prefix": "common_sensing/fiji/landsat_8_geomedian/2019"}'
...
EOF
```

### Automatic job definition

See the [Using Kubernetes section of the Job insertion Docker image](https://github.com/SatelliteApplicationsCatapult/ard-docker-images/blob/master/job-insert/README.md#using-kubernetes) within the [ard-docker-images repo](https://github.com/SatelliteApplicationsCatapult/ard-docker-images).

## Chart Details

By default, this Chart will deploy the following:
- 1 x Worker

### Installing the Chart

Make Helm aware of the [Satellite Applications Catapult Helm chart repository](https://satelliteapplicationscatapult.github.io/helm-charts/), so you will be able to install the ARD chart from it without having to use a long URL name:

```bash
helm repo add satapps https://satelliteapplicationscatapult.github.io/helm-charts
helm repo update

helm search odc-product-generation
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
satapps/odc-product-generation  0.3.0           0.0.90          A Helm chart for generating routine EO products with Kube...
```

It's then necessary to create a *values-worker.yaml* file specific to the Kubernetes cluster, Open Data Cube and Dask deployments in use.\
For the full set of configurable options see [values.yaml](values.yaml).

As example, we might have:

```yaml
image:
  repository: "satapps/odc-products"
  tag: "0.0.91"
  pullPolicy: IfNotPresent

env:
  - name: PYTHONUNBUFFERED
    value: "0"
  - name: AWS_VIRTUAL_HOSTING
    value: "FALSE"
  - name: AWS_S3_ENDPOINT_URL
    value: "http://s3-uk-1.sa-catapult.co.uk"

aws:
  accessKeyId: "AKIAIOSFODNN7INVALID"
  secretAccessKey: "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"
```

To install the Chart with the release name `geomedian`:

```bash
RELEASEODCPROD=geomedian

helm upgrade --install $RELEASEODCPROD satapps/odc-product-generation \
  --namespace $NAMESPACEODCPROD \
  --version 0.1.0 \
  --values values-worker.yaml
```

## Default Configuration

The following tables list the configurable parameters of the Chart and their default values.

### Data Cube DB Connection Parameters

| Parameter             | Description | Default                                              |
| ----------------------| ------------| -----------------------------------------------------|
| `datacubeDB.host`     | Host        | `datacubedb-postgresql.datacubedb.svc.cluster.local` |
| `datacubeDB.database` | DB name     | `datacube`                                           |
| `datacubeDB.user`     | User        | `postgres`                                           |
| `datacubeDB.password` | Password    | `localuser1234`                                      |

### Redis Master Connection Parameters

| Parameter          | Description | Default        |
| -------------------| ------------| ---------------|
| `redisServiceHost` | Host        | `redis-master` | 

### Dask Cluster Connection Parameters

| Parameter           | Description | Default                                 |
| --------------------| ------------| ----------------------------------------|
| `daskSchedulerHost` | Host        | `dask-scheduler.dask.svc.cluster.local` | 

### Worker

| Parameter          | Description                     | Default                |
| -------------------| --------------------------------| -----------------------|
| `image.repository` | Container image name            | `satapps/odc-products` |
| `image.tag`        | Container image tag             | `0.0.90`               |
| `image.pullPolicy` | Container image pull policy     | `IfNotPresent`         |
| `resources`        | Container resources             | `{}`                   |
| `env`              | Container environment variables | `{}`                   |
| `nodeSelector`     | nodeSelector                    | `{}`                   |
| `tolerations`      | Tolerations                     | `[]`                   |
| `affinity`         | Container affinity              | `{}`                   |
| `ttlSeconds`       | TTL in seconds after finished   | `nil`                  |

### AWS

| Parameter             | Description    | Default                                    |
|-----------------------|----------------|--------------------------------------------|
| `aws.accessKeyId`     | AWS key id     | `AKIAIOSFODNN7INVALID`                     |
| `aws.secretAccessKey` | AWS secret key | `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` |

## Logging

In order to extract the logs from all workers, issue:

```bash
for pod in $(kubectl get pods -n $NAMESPACEODCPROD -l component=worker -o name); do kubectl logs $pod -n $NAMESPACEODCPROD; done
```

## Cleaning up

:warning: Dangerous Zone :warning:

If you wish to undo changes to your Kubernetes cluster, simply issue the following commands:

```bash
helm delete $RELEASEREDIS --purge
helm delete $RELEASEODCPROD --purge
kubectl delete namespace $NAMESPACEODCPROD
```
