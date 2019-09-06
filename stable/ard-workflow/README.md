# ARD Workflow Helm Chart

In order to use the deployment material described here you need access to a Kubernetes server or cluster with Helm installed. For development and learning purposes [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) and [microk8s](https://microk8s.io/) can be used too.

## Scope

The solution provided here is suitable to run ARD workflows for historical data on a massive scale.

## Architecture

We run a Kubernetes `Job` with multiple parallel worker processes. As each pod is created, it picks up one unit of work from a task queue, processes it, and repeats until the end of the queue is reached.

We use [Redis](https://redis.io/) as storage service to hold the work queue and store our work items. Each work item represents one scene to be processed through an ARD workflow. In practice you would set up Redis once and reuse it for the work queues of many jobs.

## Redis Master server deployment

It's necessary to first create a *redis-values.yaml* file. As example, for a development environment you might have:

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

In order to deploy the master issue the following:

```bash
NAMESPACE=ard

RELEASEREDIS=redis

helm upgrade --install $RELEASEREDIS stable/redis \
  --namespace $NAMESPACE \
  --version=9.1.3 \
  --values redis-values.yaml
```

### Redis job definitions

The list with key `jobS2` is the work queue for Sentinel-2 ARD jobs. Add jobs with e.g.:

```bash
$ kubectl run --namespace $NAMESPACE redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:5.0.5-debian-9-r104 -- bash

I have no name!@redis-client:/$ redis-cli -h redis-master

redis-master:6379> rpush jobS2 '{"in_scene": "S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708", "inter_dir": "/data/intermediate/"}'
(integer) 1
redis-master:6379> lrange jobS2 0 -1
1) "{\"in_scene\": \"S2A_MSIL2A_20190812T235741_N0213_R030_T56LRR_20190813T014708\", \"inter_dir\": \"/data/intermediate/\"}"
```

## ARD Chart Details

By default, this chart will deploy the following:

- 3 x Sentinel-2 ARD workers that retrieve jobs from a Redis master
- 1 x Jupyter Notebook (optional) with port 80 exposed on an external LoadBalancer (default)
- All using Kubernetes Deployments

> **Tip**: See the [Kubernetes Service Type Docs](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
for the differences between ClusterIP, NodePort, and LoadBalancer.

### Installing the Chart

It's necessary to first create a *ard-values.yaml* file specific to the Kubernetes cluster and the ARD workflow that is being deployed.\
For the full set of configurable options see [values.yaml](values.yaml).

As example, for a development environment you might have:

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

For a production environment, you might instead have:

```yaml
worker:
  parallelism: 5

jupyter:
  enabled: false

aws:
  aws_access_key_id: "AKIAIOSFODNN7INVALID"
  aws_secret_access_key: "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"
```

To install the chart with the release name `s2job`:

```bash
RELEASEARD=s2job

helm upgrade --install $RELEASEARD stable/ard-workflow-s2 \
  --namespace $NAMESPACE \
  --values ard-values.yaml
```

If enabled, for access to the notebook server refer to the instructions provided by the chart once the deployment is initiated. If you need to access this information at a later time, you can issue:

```bash
helm status $RELEASEARD
```

### Job completion

A Job can be inspected for completion, e.g. by issuing:

```bash
$ kubectl get job -n ard -o wide
NAME                        COMPLETIONS   DURATION   AGE   CONTAINERS     IMAGES                          SELECTOR
s2job-ard-workflow-worker   3/1 of 3      100m       28h   ard-workflow   satapps/ard-workflow-s2:0.5.0   controller-uid=302c9874-0e24-4977-9360-bc8cfc76df96
```

Alternatively, making sure that the relevant `Pod`s are in the `Completed` status is another possible route. E.g.:

```bash
$ kubectl get pod -n $NAMESPACE --field-selector spec.restartPolicy=OnFailure -o wide
NAME                              READY   STATUS      RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
s2job-ard-workflow-worker-fsd27   0/1     Completed   0          28h   10.244.2.129   k8snode02   <none>           <none>
s2job-ard-workflow-worker-gltrk   0/1     Completed   0          28h   10.244.3.114   k8snode03   <none>           <none>
s2job-ard-workflow-worker-jnsfl   0/1     Completed   0          28h   10.244.1.98    k8snode01   <none>           <none>
```

## Cleaning up

:warning: Dangerous Zone :warning:

If you wish to undo changes to your Kubernetes cluster, simply issue the following commands:

```bash
helm delete $RELEASEREDIS --purge
helm delete $RELEASEARD --purge
kubectl delete namespace $NAMESPACE
```
