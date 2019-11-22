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
NAMESPACE=ard

RELEASEREDIS=redis

helm upgrade --install $RELEASEREDIS stable/redis \
  --namespace $NAMESPACE \
  --version=9.1.3 \
  --values values-redis.yaml
```

### Redis job definitions

The list with key `jobS2` is the work queue for Sentinel-2 ARD work items. Insert items with e.g.:

```bash
$ kubectl run --namespace $NAMESPACE redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:5.0.5-debian-9-r104 -- bash

I have no name!@redis-client:/$ redis-cli -h redis-master

redis-master:6379> rpush jobS2 '{"in_scene": "S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708", "s3_bucket": "pds-satapps", "s3_dir": "fiji/Sentinel_2/"}'
(integer) 1
redis-master:6379> lrange jobS2 0 -1
1) "{\"in_scene\": \"S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708\", \"s3_bucket\": \"pds-satapps\", \"s3_dir\": \"fiji/Sentinel_2/\"}"
```

For [mass insertion](https://redis.io/topics/mass-insert) you can use e.g.:

```bash
$ kubectl run --namespace $NAMESPACE redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:5.0.5-debian-9-r104 -- bash

I have no name!@redis-client:/$ cat <<EOF | redis-cli -h redis-master --pipe
rpush jobS2 '{"in_scene": "S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708", "s3_bucket": "pds-satapps", "s3_dir": "fiji/Sentinel_2/"}'
...
EOF
```

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
```

It's then necessary to create a *values-ard.yaml* file specific to the Kubernetes cluster and the ARD workflow that is being deployed.\
For the full set of configurable options see [values.yaml](values.yaml).

As example, for a development environment we might have:

```yaml
worker:
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
```

For a production environment, we might have instead:

```yaml
worker:
  parallelism: 5
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: component
            operator: In
            values:
            - worker
        topologyKey: "kubernetes.io/hostname"
#    podAntiAffinity:
#      preferredDuringSchedulingIgnoredDuringExecution:
#      - weight: 1
#        podAffinityTerm:
#          labelSelector:
#            matchExpressions:
#            - key: component
#              operator: In
#              values:
#              - worker
#          topologyKey: "kubernetes.io/hostname"

jupyter:
  enabled: false

aws:
  aws_access_key_id: "AKIAIOSFODNN7INVALID"
  aws_secret_access_key: "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"
```

To install the Chart with the release name `s2job`:

```bash
RELEASEARD=s2job

helm upgrade --install $RELEASEARD satapps/ard-campaign \
  --namespace $NAMESPACE \
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
| `worker.image.tag`        | Container image tag             | `0.5.0`                   |
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
| `jupyter.image.tag`        | Container image tag             | `0.5.0`                           |
| `jupyter.image.pullPolicy` | Container image pull policy     | `IfNotPresent`                    |
| `jupyter.service.type`     | k8s service type                | `LoadBalancer`                    |
| `jupyter.service.port`     | k8s service port                | `80`                              |
| `jupyter.replicaCount`     | k8s deployment replicas         | `1`                               |
| `jupyter.resources`        | Container resources             | `{}`                              |
| `jupyter.env`              | Container environment variables | `{}`                              |
| `jupyter.tolerations`      | Tolerations                     | `[]`                              |
| `jupyter.nodeSelector`     | nodeSelector                    | `{}`                              |
| `jupyter.affinity`         | Container affinity              | `{}`                              |

### AWS

| Parameter               | Description    | Default                                    |
|-------------------------|----------------|--------------------------------------------|
| `aws_access_key_id`     | AWS key id     | `AKIAIOSFODNN7INVALID`                     |
| `aws_secret_access_key` | AWS secret key | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYINVALIDKEY` |

### Job completion

A `Job` can be inspected for completion, e.g. by issuing:

```bash
$ kubectl get job -n $NAMESPACE -o wide
NAME                        COMPLETIONS   DURATION   AGE   CONTAINERS     IMAGES                          SELECTOR
s2job-ard-campaign-worker   3/1 of 3      100m       28h   ard-campaign   satapps/ard-workflow-s2:0.5.0   controller-uid=302c9874-0e24-4977-9360-bc8cfc76df96
```

Alternatively, making sure that the relevant `Pod`s are in the `Completed` status is another possible route. E.g.:

```bash
$ kubectl get pod -n $NAMESPACE --field-selector spec.restartPolicy=OnFailure -o wide
NAME                              READY   STATUS      RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
s2job-ard-campaign-worker-fsd27   0/1     Completed   0          28h   10.244.2.129   k8snode02   <none>           <none>
s2job-ard-campaign-worker-gltrk   0/1     Completed   0          28h   10.244.3.114   k8snode03   <none>           <none>
s2job-ard-campaign-worker-jnsfl   0/1     Completed   0          28h   10.244.1.98    k8snode01   <none>           <none>
```

### Logging

In order to extract the logs from all workers, issue:

```bash
for pod in $(kubectl get pods -n $NAMESPACE -l component=worker -o name); do kubectl logs $pod -n $NAMESPACE; done
```

## Cleaning up

:warning: Dangerous Zone :warning:

If you wish to undo changes to your Kubernetes cluster, simply issue the following commands:

```bash
helm delete $RELEASEREDIS --purge
helm delete $RELEASEARD --purge
kubectl delete namespace $NAMESPACE
```

## TODO
- In order to automatically add work items to a processing queue we could use the `redis-client` Docker image and inject such work items by means of a `ConfigMap`. The command executed by the image upon starting would then be something like: `cat /data/work-items.list | redis-cli -h redis-master --pipe`. One would have to make sure that the Redis Master server deployment is active before attempting to add work items.
