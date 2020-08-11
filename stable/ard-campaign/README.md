# ARD Processing Campaign Helm Chart

In order to use the deployment material described here you need access to a Kubernetes server or cluster with Helm installed. For development and learning purposes [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and [microk8s](https://microk8s.io/) can be used too.

## Scope

The solution provided here is suitable to run ARD processing campaigns for historical data on a massive scale.

## Architecture

We run a Kubernetes `Job` with multiple parallel worker `Pod`s. As each `Pod` is created, it picks up one unit of work from a task queue, processes it, and repeats until the end of the queue is reached.

We use [Redis](https://redis.io/) as storage service to hold the work queue and store our work items. Each work item represents one scene to be processed through an ARD workflow. In practice we would set up Redis once and reuse it for the work queues of multiple job types.

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

For the full set of configurable options see [values.yaml](https://github.com/helm/charts/blob/master/stable/redis/values.yaml).

In order to deploy the Master issue the following:

```bash
NAMESPACEARD=ard

kubectl create namespace $NAMESPACEARD

RELEASEREDIS=redis

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm upgrade --install $RELEASEREDIS bitnami/redis \
  --namespace $NAMESPACEARD \
  --version 10.7.16 \
  --values values-redis.yaml
```

## Job definitions

Per-mission and per-product work queues can be populated with job definitions either manually or programmatically.

For all available missions and products see the [relevant section](https://github.com/SatelliteApplicationsCatapult/helm-charts/tree/master/stable/ard-campaign#other-missions-and-products) of this document.

### Manual job definition

The list with key `jobS2` is the work queue for Sentinel-2 ARD work items. Insert items with e.g.:

```bash
$ kubectl run --namespace $NAMESPACEARD redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:5.0.5-debian-9-r104 -- bash

I have no name!@redis-client:/$ redis-cli -h redis-master

redis-master:6379> rpush jobS2 '{"in_scene": "S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708", "s3_bucket": "public-eo-data", "s3_dir": "fiji/Sentinel_2/"}'
(integer) 1
redis-master:6379> lrange jobS2 0 -1
1) "{\"in_scene\": \"S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708\", \"s3_bucket\": \"public-eo-data\", \"s3_dir\": \"fiji/Sentinel_2/\"}"
```

For [mass insertion](https://redis.io/topics/mass-insert) you can use e.g.:

```bash
$ kubectl run --namespace $NAMESPACEARD redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:5.0.5-debian-9-r104 -- bash

I have no name!@redis-client:/$ cat <<EOF | redis-cli -h redis-master --pipe
rpush jobS2 '{"in_scene": "S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708", "s3_bucket": "public-eo-data", "s3_dir": "fiji/Sentinel_2/"}'
...
EOF
```

### Automatic job definition

See the [Using Kubernetes section of the Job insertion Docker image](https://github.com/SatelliteApplicationsCatapult/ard-docker-images/blob/master/job-insert/README.md#using-kubernetes) within the [ard-docker-images repo](https://github.com/SatelliteApplicationsCatapult/ard-docker-images).

## ARD Chart Details

By default, this Chart will deploy the following:

- 3 x Sentinel-2 ARD workers that retrieve work items from a Redis Master
- 1 x Jupyter Notebook (optional) with port 80 exposed on an external LoadBalancer (default)
- All using Kubernetes Deployments

> **Tip**: See the [Kubernetes Service Type Docs](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
for the differences between ClusterIP, NodePort, and LoadBalancer.

### Installing the Chart

Make Helm aware of the [Satellite Applications Catapult Helm chart repository](https://satelliteapplicationscatapult.github.io/helm-charts/), so you will be able to install the ARD chart from it without having to use a long URL name:

```bash
helm repo add satapps https://satelliteapplicationscatapult.github.io/helm-charts
helm repo update

helm search ard-campaign
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
satapps/ard-campaign    0.7.0           1.2.2           A Helm chart for deploying ARD processing campaigns with ...
```

It's then necessary to create a *values-ard.yaml* file specific to the Kubernetes cluster and the ARD workflow that is being deployed.\
For the full set of configurable options see [values.yaml](values.yaml).

As example, for a development environment we might have:

```yaml
worker:
  image:
    repository: "satapps/ard-workflow-s2"
    tag: "1.2.2"
  parallelism: 1
  env:
    - name: AWS_NO_SIGN_REQUEST
      value: "YES"

jupyter:
  enabled: true
  service:
    type: NodePort
  env:
    - name: AWS_NO_SIGN_REQUEST
      value: "YES"
    - name: AWS_S3_ENDPOINT_URL
      value: "http://s3-uk-1.sa-catapult.co.uk"

gcp:
  privateKey: "-----BEGIN PRIVATE KEY-----\\n...\\n-----END PRIVATE KEY-----"
  clientEmail: "invalid@invalid.iam.gserviceaccount.com"
```

For a production environment, we might have instead:

```yaml
worker:
  image:
    repository: "satapps/ard-workflow-s2"
    tag: "1.2.2"
  parallelism: 14
  env:
    - name: LOGLEVEL
      value: "ERROR"
    - name: AWS_S3_ENDPOINT_URL
      value: "http://s3-uk-1.sa-catapult.co.uk"
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

jupyter:
  enabled: false

gcp:
  privateKey: "-----BEGIN PRIVATE KEY-----\\n...\\n-----END PRIVATE KEY-----"
  clientEmail: "invalid@invalid.iam.gserviceaccount.com"

aws:
  accessKeyId: "AKIAIOSFODNN7INVALID"
  secretAccessKey: "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"
```

To install the Chart with the release name `s2job`:

```bash
RELEASEARD=s2job

helm upgrade --install $RELEASEARD satapps/ard-campaign \
  --namespace $NAMESPACEARD \
  --version 0.7.0 \
  --values values-ard.yaml
```

If enabled, for access to the notebook server refer to the instructions provided by the Chart once the deployment is initiated. If you need to access this information at a later time, you can issue:

```bash
helm status $RELEASEARD
```

## Default Configuration

The following tables list the configurable parameters of the Chart and their default values.

### Worker

| Parameter                 | Description                     | Default                   |
| --------------------------| --------------------------------| --------------------------|
| `worker.image.repository` | Container image name            | `satapps/ard-workflow-s2` |
| `worker.image.tag`        | Container image tag             | `1.2.2`                   |
| `worker.image.pullPolicy` | Container image pull policy     | `IfNotPresent`            |
| `worker.parallelism`      | k8s job parallelism             | `3`                       |
| `worker.resources`        | Container resources             | `{}`                      |
| `worker.env`              | Container environment variables | `{}`                      |
| `worker.tolerations`      | Tolerations                     | `[]`                      |
| `worker.nodeSelector`     | nodeSelector                    | `{}`                      |
| `worker.affinity`         | Container affinity              | `{}`                      |

### Jupyter

| Parameter                  | Description                     | Default                           |
|----------------------------|---------------------------------|-----------------------------------|
| `jupyter.enabled`          | Include optional Jupyter server | `true`                            |
| `jupyter.image.repository` | Container image name            | `satapps/ard-workflow-s2-jupyter` |
| `jupyter.image.tag`        | Container image tag             | `1.2.2`                           |
| `jupyter.image.pullPolicy` | Container image pull policy     | `IfNotPresent`                    |
| `jupyter.service.type`     | k8s service type                | `LoadBalancer`                    |
| `jupyter.service.port`     | k8s service port                | `80`                              |
| `jupyter.replicaCount`     | k8s deployment replicas         | `1`                               |
| `jupyter.resources`        | Container resources             | `{}`                              |
| `jupyter.env`              | Container environment variables | `{}`                              |
| `jupyter.tolerations`      | Tolerations                     | `[]`                              |
| `jupyter.nodeSelector`     | nodeSelector                    | `{}`                              |
| `jupyter.affinity`         | Container affinity              | `{}`                              |

### ASF

| Parameter      | Description     | Default           |
|----------------|-----------------|-------------------|
| `asf.username` | ASF login name  | `invalidusername` |
| `asf.password` | ASF password    | `invalidpassword` |

### Copernicus Hub

| Parameter             | Description                | Default           |
|-----------------------|----------------------------|-------------------|
| `copernicus.username` | Copernicus Hub login name  | `invalidusername` |
| `copernicus.password` | Copernicus Hub password    | `invalidpassword` |

### GCP

| Parameter         | Description      | Default                                                         |
|-------------------|------------------|-----------------------------------------------------------------|
| `gcp.privateKey`  | GCP private key  | `-----BEGIN PRIVATE KEY-----\\n...\\n-----END PRIVATE KEY-----` |
| `gcp.clientEmail` | GCP client email | `invalid@invalid.iam.gserviceaccount.com`                       |

### AWS

| Parameter             | Description    | Default                                    |
|-----------------------|----------------|--------------------------------------------|
| `aws.accessKeyId`     | AWS key id     | `AKIAIOSFODNN7INVALID`                     |
| `aws.secretAccessKey` | AWS secret key | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYINVALIDKEY` |

## Job completion

A `Job` can be inspected for completion, e.g. by issuing:

```bash
$ kubectl get job -n $NAMESPACEARD -o wide
NAME                        COMPLETIONS   DURATION   AGE   CONTAINERS     IMAGES                          SELECTOR
s2job-ard-campaign-worker   3/1 of 3      100m       28h   ard-campaign   satapps/ard-workflow-s2:1.2.2   controller-uid=302c9874-0e24-4977-9360-bc8cfc76df96
```

Alternatively, making sure that the relevant `Pod`s are in the `Completed` status is another possible route. E.g.:

```bash
$ kubectl get pod -n $NAMESPACEARD -l component=worker -o wide
NAME                              READY   STATUS      RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
s2job-ard-campaign-worker-fsd27   0/1     Completed   0          28h   10.244.2.129   k8snode02   <none>           <none>
s2job-ard-campaign-worker-gltrk   0/1     Completed   0          28h   10.244.3.114   k8snode03   <none>           <none>
s2job-ard-campaign-worker-jnsfl   0/1     Completed   0          28h   10.244.1.98    k8snode01   <none>           <none>
```

## Logging

In order to extract the logs from all workers, issue:

```bash
for pod in $(kubectl get pods -n $NAMESPACEARD -l component=worker -o name); do kubectl logs $pod -n $NAMESPACEARD; done
```

## Cleaning up

:warning: Dangerous Zone :warning:

If you wish to undo changes to your Kubernetes cluster, simply issue the following commands:

```bash
helm delete $RELEASEREDIS -n $NAMESPACEARD
helm delete $RELEASEARD -n $NAMESPACEARD
kubectl delete namespace $NAMESPACEARD
```

## Other missions and products

### Landsat

For Landsat 4, 5, 7, and 8, jobs are defined as per example below:

```bash
rpush jobLS '{"in_scene": "https://edclpdsftp.cr.usgs.gov/orders/espa-User.Name@Domain-12202019-045818-299/LT040900641989052601T2-SC20191220121726.tar.gz", "s3_bucket": "public-eo-data", "s3_dir": "solomonislands/landsat_4/"}'
```

Configuration options would be along these lines for a production system:

```yaml
worker:
  image:
    repository: "satapps/ard-workflow-ls"
    tag: "1.2.1"
  parallelism: 28
  env:
    - name: LOGLEVEL
      value: "ERROR"
    - name: AWS_S3_ENDPOINT_URL
      value: "http://s3-uk-1.sa-catapult.co.uk"
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

jupyter:
  enabled: false

aws:
  accessKeyId: "AKIAIOSFODNN7INVALID"
  secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYINVALIDKEY"
```

### Sentinel-1

For Sentinel-1, jobs are defined as per example below:

```bash
rpush jobS1 '{"in_scene": "S1A_IW_GRDH_1SDV_20200210T064005_20200210T064040_031186_0395FE_3FBF", "s3_bucket": "public-eo-data", "s3_dir": "fiji/sentinel_1/", "ext_dem": "ancillary_products/SRTM1Sec/SRTM30_Fiji_E.tif"}'
```

Configuration options would be along these lines for a production system:

```yaml
worker:
  image:
    repository: "satapps/ard-workflow-s1"
    tag: "1.2.3"
  parallelism: 14
  env:
    - name: LOGLEVEL
      value: "ERROR"
    - name: AWS_S3_ENDPOINT_URL
      value: "http://s3-uk-1.sa-catapult.co.uk"
    - name: S1_PROCESS_P1A
      value: "/utils/cs_s1_pt1_bnr_Orb_Cal_ML.xml"
    - name: S1_PROCESS_P2A
      value: "/utils/cs_s1_pt2A_TF.xml"
    - name: S1_PROCESS_P3A
      value: "/utils/cs_s1_pt3A_TC_db.xml"
    - name: S1_PROCESS_P4A
      value: "/utils/cs_s1_pt4A_Sm_Bm_TC_lsm.xml"
    - name: SNAP_GPT
      value: "/opt/snap/bin/gpt"
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

jupyter:
  enabled: false

asf:
  username: "invaliduser"
  password: "invalidpassword"

copernicus:
  username: "invaliduser"
  password: "invalidpassword"

aws:
  accessKeyId: "AKIAIOSFODNN7INVALID"
  secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYINVALIDKEY"
```

### Sentinel-2 L1C to L2A preparation

For Sentinel-2 L1C to L2A preparation, jobs are defined as per example below:

```bash
rpush jobS2 '{"in_scene": "S2A_MSIL1C_20190524T221941_N0207_R029_T60KXD_20190524T234151", "s3_bucket": "public-eo-data", "s3_dir": "fiji/sentinel_2/"}'
```

Configuration options would be along these lines for a production system:

```yaml
worker:
  image:
    repository: "satapps/ard-workflow-s2-l1c"
    tag: "1.2.5"
  parallelism: 14
  env:
    - name: LOGLEVEL
      value: "ERROR"
    - name: AWS_S3_ENDPOINT_URL
      value: "http://s3-uk-1.sa-catapult.co.uk"
    - name: SEN2COR_8
      value: "/Sen2Cor-02.08.00-Linux64/bin/L2A_Process"
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

jupyter:
  enabled: false

copernicus:
  username: "invaliduser"
  password: "invalidpassword"

gcp:
  privateKey: "-----BEGIN PRIVATE KEY-----\\n...\\n-----END PRIVATE KEY-----"
  clientEmail: "invalid@invalid.iam.gserviceaccount.com"

aws:
  accessKeyId: "AKIAIOSFODNN7INVALID"
  secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYINVALIDKEY"
```

### Water classification

For water classification products, jobs are defined as per example below:

```bash
rpush jobWater '{"optical_yaml_path": "fiji/landsat_8/LC08_L1TP_072069_20130522/datacube-metadata.yaml", "s3_bucket": "public-eo-data", "s3_dir": "fiji/landsat_8_water/"}'
```

Configuration options would be along these lines for a production system:

```yaml
worker:
  image:
    repository: "satapps/ard-workflow-water-classification"
    tag: "1.2.1"
  parallelism: 28
  env:
    - name: LOGLEVEL
      value: "ERROR"
    - name: AWS_S3_ENDPOINT_URL
      value: "http://s3-uk-1.sa-catapult.co.uk"
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

jupyter:
  enabled: false

aws:
  accessKeyId: "AKIAIOSFODNN7INVALID"
  secretAccessKey: "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"
```

## TODO
- We could include an optional [job insert](https://github.com/SatelliteApplicationsCatapult/ard-docker-images/tree/master/job-insert) worker in the deployment.
