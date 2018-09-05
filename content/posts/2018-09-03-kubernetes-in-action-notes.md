---
title: "2018 09 03 Kubernetes in Action Notes"
date: 2018-09-03T18:20:46+08:00
draft: true
---

Miscellaneous notes when reading \<Kubenetes in Action\>.


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
