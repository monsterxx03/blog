---
title: "Centralized Logging on K8S"
date: 2019-05-26T14:27:38+08:00
categories:
    - tech
tags:
    - fluentd
    - k8s
    - eks
    - server-infra
---


搞定了监控, 下一步在 k8s 上要做的是中心化日志, 大体看了下, 感兴趣的有两个选择: ELK 套件, 或fluent-bit + fluentd.


ELK 那套好处是, 可以把监控和日志一体化, filebeat 收集日志, metricbeat 收集 metrics, 统一存储在 ElasticSearch 里, 通过第三方项目[elastalert](https://github.com/Yelp/elastalert)
可以做报警，也能在 kibana 里集成界面. 坏处是 ElasticSearch 存储成本高, 吃资源. 我们对存储的日志使用需求基本就是 debug, 没有特别复杂的BI需求, 上一整套 ELK 还是太重了.

选择 fluent-bit + fluentd 还有个好处是, 之前内部有套收集 metrics(用于统计 DAU, retention 之类指标) 的系统本来就是基于 fluentd 的, 用这套就不用改 metrics 那边的 ETL 了.

fluent-bit 和 fluentd 比, 主要好处是纯 C 写的(fluentd 是 C+ruby), 性能很好, 资源消耗低, 支持 fluentd 的 forward 协议, input/output 的种类支持没有 fluentd 那么多. 所以实际部署时候
fluent-bit 作为 daemonset 部署在所有节点上, fluentd 作为中心化的 aggregation layer, 定时把日志刷到 s3 上去, fluent-bit 和 fluentd 之间通过 fluentd 的 forward 协议传输日志.

## fluent-bit 的配置

用 tail input 监控日志文件


      [INPUT]
          Name tail
          Tag  varlog.secure.${NODE_NAME}
          Path /var/log/secure
          DB  /mnt/tail-secure-state.db

DB 参数指定一个文件路径, fluent-bit 会创建一个 sqlite 数据库记录自己读到文件哪里了, 否则 fluent-bit 重启的时候会把日志从头再发送一遍, 这个路径要用 hostPath volume 挂进容器, 否则更新的时候就丢啦.

`${NODE_NAME}` 是从环境变量里读取, 可以在 daemonset 的spec 里用以下方式注进 pod 的环境变量中:

    
    env:
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName


注意默认 fluent-bit 会把制定路径的日志全部从头到尾读一边, 如果量很大会对 fluentd 造成很大压力, 可以指定 `Ignore_Older` 参数跳过比较老的日志文件.


收集 container 日志:
    
    [INPUT]
        Name tail
        Tag kube.*
        Path /var/log/containers/*.log
        Parser docker
        Docker_Mode On
        DB /tail-db/tail-kube-sate.db

    
    [FILTER]
        Name kubernetes
        Match kube.*
        K8S-Logging.Parser On
        K8S-Logging.Exclude On
        Labels Off
        Annotations Off

`Docker_Mode` 用于处理 docker 可能对 日志做的多行切割. 这里加了个 filter 来对从容器中收集到的日志打上 k8s 中 container, pod, nanmespace 相关 context 信息.

fluent-bit 还能通过 pod 上的 annotation 来控制对 pod 输出日志的控制.

    fluentbit.io/parser:  apache   # 使用 apache parse 来解析日志, 该 parse 必须事先在 fluent-bit 的配置文件中定义
    fluentbit.io/exclude: "true"  # 让 fluent-bit 不收集该 pod 的日志

`K8S-Logging.Parser` 和 `K8S-Logging.Exclude` 用来开启这两个特性.

输出到 fluentd:

    [OUTPUT]
        Name  forward
        Match *
        Host  fluentd.logging
        Port  24224

## fluentd 的配置


之前发来的 varlog tag, 格式是 varlog.secure.ip-10-1-1-1.ec2.internal, fluentd 的 tag 以 `.` 分隔, 这里设置在本地文件缓存，每小时把数据刷到 s3 上去. `${tag[0]}` 这样的语法可以引用到 tag 中的各个部分.

    <match varlog.*.**>
      @type s3
      s3_bucket s3-log-bucket
      s3_region  us-east-1
      path fluentd/${tag[0]}/${tag[1]}/
      time_slice_format "%Y/%m/%d/"
      s3_object_key_format %{path}%{time_slice}%H_${tag[2]}_%{index}.%{file_extension}
      store_as gzip_command
      use_server_side_encryption AES256

      <buffer tag,time>
        @type file
        path /var/log/fluentd/varlog_buffer/
        timekey 1h
        timekey_wait 5m
        timekey_use_utc true
        chunk_limit_size 128M
      </buffer>
    </match>

默认 fluentd 压缩 gzip 是在进程内压缩, 因为 ruby GIL 的原因, 没法利用上多核, 设置 `gzip_command` 后会通过 shell 调用 gzip 命令压缩, 这样就能用多核啦(container 中需要安装 gzip).

其他 tag 的配置类似, 根据需求配置 s3 上路径就行了.

## 查询

暂时的查询方式是从 s3 上把数据 dump 下来，手工 grep. 更进一步可以把存到 s3 上的日志转化成 parquet, 在 redshift spectrum 中创建 external table 指向 s3 上文件, 就能方便 team 里其他人用 sql 来查询日志了.
