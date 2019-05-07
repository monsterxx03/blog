---
title: "kubeconfig 和 aws profile 管理的一些 tips"
date: 2019-05-07T18:36:38+08:00
draft: true
categories:
  - tech
tags:
  - k8s
---

我有两个 EKS 集群 (sandbox + production), 这两个集群分处两个 aws 帐号中.
所以管理的时候也需要两套 aws credential.

同时我用 [helm-secrets](https://github.com/futuresimple/helm-secrets) 来管理 helm charts
中需要加密的一些配置. helm-secrets 只是 [sops](https://github.com/mozilla/sops) 的一个 shell 
wrapper, 实际加密是通过 sops 进行的.

sops 支持 aws KMS, gcp KMS, azure key vault.. 等加密服务. 我用的是 aws KMS, 在 KMS 里创建一个 key,
授权允许我这个 iam 帐号能用它来进行加解密.

这带来了一个问题, kubectl 和 helm-secrets 都需要 aws credential, 如果两边用的不一样就会执行失败.


我统一使用 aws 的 [named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) 
来管理 credential. 不在环境变量里设 aws 的 access key/secret key(如果设置了, 优先级比 named profile 高)

目录结构:

    ~/.aws/
        credential
        config


credentail 中内容:

    [sandbox]
    aws_access_key_id = xxx
    aws_secret_access_key = yyy

    [production]
    aws_access_key_id = aaa
    aws_secret_access_key = bbb

config 中内容:


    [profile sandbox]
    region=us-west-1

    [profile production]
    region=us-west-2

一般来说, 多个 eks 集群的信息可以写在 ~/.kube/config 中, 通过 `kubectl config use-context` 来切换.

我更喜欢一个集群一个配置文件, 通过环境变量 `KUBECONFIG` 来指定使用哪个配置文件.

假设我把所有 kubconfig 放在 `~/.kube/configs/` 下:

    ~/.kube/configs/
        production
        sandbox

使用 eks 时, kubectl 是通过 iam-authenticator-aws 来进行认证的, kubeconfig 示例配置:


    users:
    - name: arn:aws:eks:us-west-1:123456:cluster/sandbox
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1alpha1
          args:
          - token
          - -i
          - sandbox
          command: aws-iam-authenticator
          env:
          - name: AWS_PROFILE
            value: sandbox

通过环境变量让 aws-iam-authenticator 使用指定的 profile.

所以只要同时控制 `KUBECONFIG` 和 `AWS_PROFILE` 这两个环境变量就达成了同时切换 aws profile 和 eks cluster 的目的. 

写个 shell 函数就行啦:


    function eks {
        local config_dir=~/.kube/configs
        export KUBECONFIG=$config_dir/$1
        export AWS_PROFILE=$1
    }

使用:

- eks production
- eks sandbox

当有多个集群的时候, 操作时候经常会忘记自己目前的 context 是那个 cluster (还有对应的 aws profile). 可以把信息显示到 zsh
的提示符上.

在 .zshrc 中加入:


    function aws_prof {
        echo "%{$fg_bold[blue]%}aws:(%{$fg[yellow]%}${AWS_PROFILE}%{$fg_bold[blue]%})%{$reset_color%} "
    }

    function k8s_ctx {
        echo "%{$fg_bold[blue]%}k8s:(%{$fg[yellow]%}${KUBECONFIG##*/}%{$fg_bold[blue]%})%{$reset_color%} "
    }

    export PROMPT="\$(aws_prof)\$(k8s_ctx)$PROMPT"

效果如下:

![](/posts/images/zsh-k8s.png)

因为偶尔还是可能单独切换 aws profile 的, 所以两个分开显示了.
