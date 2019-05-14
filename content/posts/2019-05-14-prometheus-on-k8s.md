---
title: "Prometheus on K8S"
date: 2019-05-14T13:52:02+08:00
categories:
    - tech
tags:
    - prometheus
    - k8s
    - eks
---

# Why move to prometheus?

把生产环境迁移到 k8s 的第一步是要搞定监控, 目前线上监控用的是商业的 datadog, 在 container 环境下
datadog 监控还要按 container 数目收费, 单 host 只有 10 个的额度, 超过要加钱, 高密度部署下很不划算.
一个 server 跑 20 个以上 container 是很正常的事情, 单台 server 的监控费用立马翻倍.

tracing 这块之前用的也是 datadog, 但太贵了,一直也想换开源实现, 索性监控报警也换了, 踩一把坑吧.

vendor lock 总是不爽的...

# Metrics in k8s

先不提 prometheus, k8s 中 metrics 来源有那么几个:


## metrics-serever

[metrics-server](https://github.com/kubernetes-incubator/metrics-server) (取代 heapster), 从 node 
上 kubelet 的 [summary api](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/stats/v1alpha1/types.go) 抓取数据(node/pod 的 cpu/memory 信息), kubectl top 和 kube-dashboard 的 metrics 数据来源就是它, horizontal pod
autoscaler 做 scale up/down 决策的数据来源也是它, metrics-server 只在内存里保留 node 和 pod 的最新值.

kubelet 以前是内置了 [cadvisor](https://github.com/google/cadvisor) 来获取　container 信息, 从1.7 开始正式换成 [CRI](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md).

metrics-server 通过 k8s aggregator 把接口注册到 k8s apiserver, 访问 k8s apiserver 的 `/apis/metrics.k8s.io/v1beta1/` 的时候请求会被代理到 metrics-server 的 pod. 

prometheus 并不通过 metrics-server 来获取 cpu/ram 信息.

## kube-state-metrics

[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) 会监听 k8s apiserver, 获取 cluster 中 object的状态变化(deployment, node, endpoint, pod...), 完整的列表: https://github.com/kubernetes/kube-state-metrics/tree/master/docs, 在内存中保存这些 object 的最新状态.  

prometheus 可以通过 kube-state-metrics 来获取 k8s 集群信息.


# 部署 prometheus

在 k8s 上部署　prometheus, 我碰到了以下选择:

1. 直接部署 prometheus 的 helm chart:　https://github.com/helm/charts/tree/master/stable/prometheus
2. 通过 prometheus-operator 的 helm chart 部署: https://github.com/helm/charts/tree/master/stable/prometheus-operator
3. 通过 https://github.com/coreos/kube-prometheus 部署 prometheus-operator

2和3 选项比较接近,都是通过 prometheus-operator 来部署 prometheus, 里面还包括了一堆预设的 prometheus record rule, alerting rule, grafana dashboard. 实际上 2 中的 rules 和 dashboard 也是从３中同步过来的.

prometheus-operator 和直接部署 prometheus 区别是 operator 把 prometheus, alertmanager server 的配置, 还有scape config, record/alert rule 包装成了 k8s 中的 crd:

    kubectl get crd 

    NAME                                    CREATED AT
    alertmanagers.monitoring.coreos.com     2019-05-05T09:37:38Z
    prometheuses.monitoring.coreos.com      2019-05-05T09:37:39Z
    prometheusrules.monitoring.coreos.com   2019-05-05T09:37:39Z
    servicemonitors.monitoring.coreos.com   2019-05-05T09:37:39Z


我是选择了方式2, 部署 prometheus-operator 的 helm chart.


比如想获取 prometheus 监控的所有对现象, 不需要去看茫茫长的配置文件, 直接:

    kubectl get servicemonitor --all-namespaces

    NAMESPACE   NAME                                            AGE
    default     grafana                                         4h
    default     prom-op-prometheus-operato-alertmanager         4d
    default     prom-op-prometheus-operato-apiserver            4d
    default     prom-op-prometheus-operato-coredns              4d
    default     prom-op-prometheus-operato-kube-state-metrics   4d
    default     prom-op-prometheus-operato-kubelet              4d
    default     prom-op-prometheus-operato-node-exporter        4d
    default     prom-op-prometheus-operato-operator             4d
    default     prom-op-prometheus-operato-prometheus           4d


修改 crd 之后, operator 监控到 crd 的修改, 生成一份 prometheus 的 配置文件, gzip 压缩后存成 k8s secrets(因为 etcd d的 value 有1M 的大小限制, 所以这里压缩了), 查看生成的完整配置文件:

    kubectl get secret prometheus-prom-op-prometheus-operato-prometheus -o json | jq -r '.data."prometheus.yaml.gz"' | base64 -d | gzip -d


alertmanager 的配置同理(没压缩):

    kubectl get secret  alertmanager-prom-op-prometheus-operato-alertmanager -o json | jq -r '.data."alertmanager.yaml"' | base64 -d


prometheus 和 alertmanager 的 pod 中都有 监控 configmap/secrets 的 container, 如果挂载的 配置文件发生了修改(k8s 同步 configmap 需要1分钟左右), 会让 server 进程 reload 配置文件.


grafana 我是单独部署的, 没有用 prometheus-operator chart弄, 因为需要修改一些细节参数(resources, pvc...), kube-prometheus 中的 dashboard: https://github.com/coreos/kube-prometheus/blob/master/manifests/grafana-dashboardDefinitions.yaml 可以拿来直接用.


## 监控 etcd, kubelet, kube-controller, kube-scheduler, coredns

在 operator 的 chart 中可以 enable 对应的选项, 因为 eks 上 etcd, kube-controller, kube-scheduler 是托管的, 没法直接访问, 所以我 disable 了.

看下监控 kubelet 具体干了什么:

    kubectl get servicemonitor  prom-op-prometheus-operato-kubelet -o yaml

    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      labels:
        ...
      name: prom-op-prometheus-operato-kubelet
    spec:
      endpoints:
      - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
        honorLabels: true
        port: https-metrics
        scheme: https
        tlsConfig:
          caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecureSkipVerify: true
      ...
      jobLabel: k8s-app
      namespaceSelector:
        matchNames:
        - kube-system
      selector:
        matchLabels:
          k8s-app: kubelet


哦, prometheus-operator 在 kube-system 里替我们创建了一个 endpoint 指向 kubelet 的 metrics 端口, address 是 node 的 ip

    kubectl get ep -n kube-system -l k8s-app=kubelet  -o yaml 

    apiVersion: v1
    items:
    - apiVersion: v1
      kind: Endpoints
      metadata:
        labels:
          k8s-app: kubelet
          ...
      subsets:
      - addresses:
        - ip: 10.0.0.100
          targetRef:
            kind: Node
            name: ip-10-0-0-100.ec2.internal
            uid: xxx
        ports:
        - name: http-metrics
          port: 10255
          protocol: TCP
        ...
    kind: List

前面说过 prometheus 不通过 metrics-server 拿 cpu/ram 数据, 实际是直接从 kubelet 拿的.

## blackbox exporter

prometheus 是基于 pull 的监控系统, 它通过 k8s 的 service discovery 获取要监控的对象, 然后定时爬取对象的 `metrics_path`,  默认 `/metrics`

所以一个 service 要能被 prometheus 监控必须使用 [instrumenting client](https://prometheus.io/docs/instrumenting/clientlibs/) 来提供 prometheus 格式的 数据.

但是有很多第三方程序或者内部的老系统并没有提供 prometheus 的交互(`/metrics`), 我们又不能/不想改系统源码, 怎么让 prometheus 监控他们呢?

可以专门写一个 exporter 来暴露一些特定 metrics,  也可以用 [blackbox-exporter](https://github.com/prometheus/blackbox_exporter/) 来简单得检查对象是否存活, 其实它就是一个代理, 通过 url 参数传递你要访问的系统的url, 它访问后检查 status code 告诉访问者结果, 具体使用看文档.

假设我们部署了 [superset](https://github.com/apache/incubator-superset), 为它创建一个 servicemonitor (prometheus-operator 会将它转换成prometheus 中的 `scrape_config`):


    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: superset
      namespace: monitor
      labels:
        release: prom-op
    spec:
      selector:
        matchLabels:
          app: prometheus-blackbox-exporter
      endpoints:
      - port: http
        path: /probe
        params:
          module: ["http_2xx"]
          target: ["superset.analytics"]
        relabelings:
        - targetLabel: "job"
          replacement: "superset"
        - targetLabel: "namespace"
          replacement:  "analytics"
        - targetLabel: "service"
          replacement: "superset"
        - sourceLabels: ["__param_target"]
          targetLabel: "instance"
      namespaceSelector:
        matchNames:
        - monitor

namespaceSelector 是 blackbox-exporter 运行的namepsace, 再通过 selector 定位到 endpoint.

    kubectl get ep -n monitor -l app=prometheus-blackbox-exporter

    NAME                                                  ENDPOINTS         AGE
    prom-blackbox-exporter-prometheus-blackbox-exporter   10.0.0.200:9115   23h


endpoints 那段定义了传递给 endponit 的参数, 所以发起检查的时候等价于:

    curl http://10.0.0.200:9115?module=http_2xx&target=superset.analytics

superset.analytics 是 superet 在 k8s 中的 service name (处在 analytics namespace 中)

因为我们访问的是 blockbox-exporter, 没有直接访问 superset, 获取到的 label 都不对, 可以通过 relabel config
将 job, namespace, service 重命名成 superset 的信息.
