---
title: "Kubernetes in Action Notes"
date: 2018-09-03T18:20:46+08:00
categories:
    - tech
tags:
    - lang-en
    - book
    - k8s
---

Miscellaneous notes when reading `<Kubernetes in Action>`.


## api group and api version

core api group need't specified in `apiVersion` field.

For example, `ReplicationController` is on core api group, so only:

    apiVersion: v1
    kind: ReplicationController
    ...

`ReplicationSet` is added later in `app` group, `v1beta2` version (k8s v1.8):
    
    apiVersion: apps/v1beta2            1
    kind: ReplicaSet           


https://kubernetes.io/docs/concepts/overview/kubernetes-api/


### ReplicationController VS ReplicationSet

ReplicationController is replaced by ReplicationSet, which has more expressive pod selectors.

ReplicationController's label selector only allows matching pods that include a certain label, ReplicationSet can
meet multi labels at same time. 

rs also support operator on key value: `In, NotIn, Exists, DoesNotExist`

If migrate from rc to rs, can delete rc with `--cascade=false` option, it will delete
rc only, but left pods running, then we can create a rs with same selector to make pods under management.

### DaemonSet

DaemonSet ensures pod run exact one copy on one node, useful for processes like monitor agent and log collector. Use `node-selector`
to make ds only run on specific nodes.

If node is made unschedulable, normal pods won't be scheduled to deploy on them, but ds will still be deployed to it, since ds will bypass
scheduler.

### Job

Job is used to run a single completable task.

Use `activeDeadlineSeconds` to control job timeout. `backoffLimit` define how many times a job can retry before it's defined as failed, default to 6.

### ConfigMap and Secret

If configmap is injected as env variables for container, it can't be modified after prod started. 

If need to refer to a value from configmap  in container's `cmd` and `args` fields, need to refer configmap in env, then refer to env value in those fields.

Use configmap and expose it through a volume. If configmap is updated, the file mounted in volume will be updated atomically (through symbolic link), but the update interval is long, up to 1 minute.

If only mount a sinlge file instead of whole volume, the file will not be updated! one workaround is to mount the whole volume into a different directory and then create a symbolic link pointing to the file.

The contents of a Secret's entries are shown as Base64-encoded strings, whereas those of a ConfigMap are shown in clear text. Secret entries can contain binary values, size is limited to 1MB. `stringData` field can be used for non-binary Secret data, it's `write-only`, if you do `kubectl get -o yaml`, `stringData` field will not be shown, they will be base64 encoded and display in `data` field. `secret` volume is mounted to container as tmpfs. 

### metadata

Metadata info can be exposed via env variables or `downwardAPI` volume.

`labels` and `annotations` can't be exposed via env variables, since they can be modified during pod run time.

### rolling update with Deployment


With old `ReplicationController`, there is a `kubectl rolling-update` cmd to do rolling update of pods, but it has problems:

- need client mainten connection to kube api server, if disconnected during updating, pods will be in a mid-way.
- it actually start a new ReplicationController with new image, and rolling shutdown old ones, it violates kubernetes design: declare desired state, not tell it add something or remove something.
- it will modify container's label and rc's label selector.

### Communication between all components in k8s

etcd is the only persistent store, kubernetes api server is the only component access etcd directly, other components only talk to kubernets api server.

All data in k8s is stored in etcd under `/register`:

    $ etcdctl ls /registry
      /registry/configmaps
      /registry/daemonsets
      /registry/deployments
      /registry/events
      /registry/namespaces
      /registry/pods
      ...

K8s api server just manager data stored in etcd, other clients (kubelet, kubeproxy) connect to api server through watch, if data is modified, api server will notify
those clients(eg: pod creation, service creation), then clients do the real things.

### pause container

Every pod will have a pause container, it's an infrastructure container to hold all namespaces, all user defined containers of the pod use the namespaces of it.

If pause container is killed , kubelet will recreate it and all pod's containers.

### scheduler policy

Scheduler doesn't look at how much of each individual resource is being used at the exact time of scheduling but at the sum of resources requested by the existing pods deployed on the node.

`LeastRequestedPriority`  prefers nodes with fewer requested resources.

`MostRequestedPriority` prefers nodes that have the most requested resources(to save cost on cloud infra).

## resource management

Resource requests sum of all pods on a node can't be larger than node's allocatable resource, but resource limits can overcommitted.

If 100% of node's resources are used up, containers will be killed.

If container exceeds cpu limit, it won't be killed, and won't get more cpu time than configured.

If container tries to allocate memory over its limit, process is killed.If pod's restart policy is `Always` or `OnFailure`, it will be restarted immediately.If it keeps
being killed, restart time delay will be increased(10s, 20s, 40s, 80s, 160s, 300s), pod status will be `CrashLoopBackOff`.

Container always see node's memory and cpu, won't be aware of its limit.

Problems:

- If you run a java program, jvm will set maximum heap size based on host's total memory instead of memory available for the container, which means it maybe OOM killed.
- If program looks up cpu numbers to decide spawn how many workers, maybe over spawn too many workers.

To get real cpu limit, can use downward api to pass in cpu limit, or look `/sys/fs/cgroup/cpu/cpu.cfs_quota_us` and `/sys/fs/cgroup/cpu/cpu.cfs_period_us`.

Pod's QoS classes:

- BestEffort (lowest priority). all containers in pod don't have `requests` and `limits` set.
- Burstable. `limits` not match `requests`. 
- Guaranteed (highest).  `requests` and `limits` bot set for all containers in pod and equal. If requests is not set, default to limit.

Pod with lower priority will be killed first if resource is not enough.

If two pods at same QoS class, the one with higher ram percentage usage will be killed first.

`LimitRange` can be used to valid pod spec, if pod requires more resources than available, api server will reject requst(it works on individual pod/container), without `LimitRange`, api server will accept pod, but never schedule it. `LimitRange` can be used to prevent users from creating pods bigger than any node in cluster.

`ResourceRuote` can be used to limit total amount of resources available in a namespace.But it has no effect on existing pods.

kubelet contains an agent called cAdvisor, it will collect statistics usage of node and containers, we can run heapster centrally to gather those data.

Note:
    
    cAdvisor and heapster only hold usage data for a short time, if historical monitor data is needed, we need usual monitor tools like influxdb, datadog, NewRelic

After heapster is running:

    kubectl  top node # get node runtime metrics
    kubectl top pod --all--namespaces # get pods runtime metrics from all namespace

`kubectl top` get  metrics from heapster, so the metrics may delay for several minutes, following error is okay:

    W0312 22:12:58.021885   15126 top_pod.go:186] Metrics not available for pod default/kubia-3773182134-63bmb, age: 1h24m19.021873823s

With `--containers` option, you can see metrics for every container instead of pods.

### HorizonalPodAutoScaler (HPA)

Managed by Horizonal controller. Check pod metrics, calculats the number of replicas required to meet target metric value configured, then adjusts replicas field on target resource(Deployment, ReplicaSet, ReplicationController, StatefulSet).

HPA query Heapster through REST apis to get metrics.

Pods -> cAdvisor -> Heapster -> HorizonalPodAutoscaler

To test autoscaler, we can manually create load:

    kubectl run -it --rm --restart=Never loadgenerator --image=busybox -- sh -c "while true; do wget -O - -q http://kubia.default; done"

Initially, HPA can only scale based on cpu usage, in 1.8 memory-based was introduced.Also works on custom metrics. 

When configure HPA, metrics type can be `Pods`, `Resource`, `Object`.

If metrics is defined as `Object`, autoscaler obtains a single metric from the single object.

Cluster Autoscaler can be used to auto scale nodes on cloud infrastructure: https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler

Situations when nodes won't be relinquished: 

- If system pods are running(only, but except DaemonSet) on the node.
- If an unmanaged pod running on the node.
- A pod with local storage is running on the node.

A node will only be returned to cloud provider if Cluster Autoscaler knows the pods running on the node will be rescheduled to other nodes.

If a node is selected to shut down, it will be marked as unscheduled first, then all pods on it will be rescheduled to other nodes.

`kubectl cordon <node>` marks node as unschedulable, but do nothing with pods running on the node.

`kubectl drain <node>` markds node as unschedulable then evits all pods from the node.

`PodDisruptionBudget` can be used to ensure minium number of pods when rescheduling pods.
