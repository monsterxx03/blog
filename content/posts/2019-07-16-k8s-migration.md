---
title: "2019 07 16 K8s Migration"
date: 2019-07-16T12:32:08+08:00
draft: true
---

nginx-ingress controller: https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md#service-upstream

uwsgi preStop hook 

uwsgi http-keepalive should be `1` https://github.com/unbit/uwsgi/issues/2018

datadog agent conntrack:

    old:  /host/proc/net/nf_conntrack
    new:  /host/proc/sys/net/netfilter/nf_conntrack_count   https://github.com/DataDog/integrations-core/commit/0497a14bbd1c0601f230c3e9f4a81aa2eed5ec7b#diff-dd651adb689f92282ee490f6dd77a033L418
