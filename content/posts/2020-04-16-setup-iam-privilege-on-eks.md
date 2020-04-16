---
title: "在 eks 中正确设置 IAM 权限"
date: 2020-04-16T11:03:39+08:00
categories:
- tech
tags:
- eks
- aws
---

在代码中调用 aws api 的时候常用两种方法:

- 直接传入 aws accessKey/secretKey
- 使用 instance profile

前者一般是创建一个 IAM 用户, 绑定对应权限, 生成 keypair, 在 k8s 环境里, 把 keypair 放在 Secrets 里, 或通过环境变量注入. 好处是可以每个应用单独设置,
但需要自己管理 keypair.

后者创建一个 IAM role, 绑定对应权限, 创建 ec2 的时候选择对应的 role. 跑在该 ec2 instance 上的程序自动能拿到对应的 IAM 权限. 好处是不必自己管理 keypair,
缺点是跑在同一 server 上的程序权限都一样.

eks 1.14 里有个两全齐美的办法: serviceaccount role: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html

可以把 IAM role 绑定在 pod 使用的 ServiceAccount 上, 既避免了管理 keypair 的麻烦, 也可以 per 应用得设置权限.

## 流程

用 terraform 来演示下给[external-dns](https://github.com/kubernetes-sigs/external-dns) 设置权限的整个过程.

创建对应的 iam policy:

    resource "aws_iam_policy" "external_dns" {
        name = "external-dns"
        description = "policy for k8s external-dns"
        policy = <<EOF
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "route53:ChangeResourceRecordSets"
                    ],
                    "Resource": [
                        "arn:aws:route53:::hostedzone/XXXXXXXXXXXXX",
                    ]
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "route53:ListHostedZones",
                        "route53:ListResourceRecordSets"
                    ],
                    "Resource": [
                        "*"
                    ]
                }
            ]
        }
        EOF
    }


创建 iam role, 为了方便可以用第三方模块 [terraform-aws-iam/modules/iam-assume-role-with-oidc](https://github.com/terraform-aws-modules/terraform-aws-iam/tree/master/modules/iam-assumable-role-with-oidc)

    module "iam_assumable_role_external_dns" {
        source = "../modules/aws_iam/modules/iam-assumable-role-with-oidc"
        create_role = true
        role_name                     = "external-dns"
        provider_url                  = replace(module.eks.cluster_oidc_issuer_url, "https://", "")
        role_policy_arns = [aws_iam_policy.external_dns.arn]
        oidc_fully_qualified_subjects = ["system:serviceaccount:kube-system:external-dns"]  # 这里指向 eks 中对应的 ServiceAccount
    }

创建之后能得到 role 的 arn, 只需要在 ServiceAccount 的 annotation 里标记一下就行了.

    annotations:
        eks.amazonaws.com/role-arn: arn:aws:iam::xxxxxxxx:role/external-dns

pod 需要重新创建才能拿到 iam 权限.

## Debug

如果不生效, 先检查 iam role 注入是否成功:

    >> kubectl exec -n kube-system external-dns-xxxx-yyyy env | grep AWS

    AWS_ROLE_ARN=arn:aws:iam::xxxxxx:role/external-dns
    AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token


能看到这两个就说明注入成功了.

如果注入成功还是没权限, 那有两种可能.

程序使用的 aws sdk 版本不够, 没有实现 assume role, 检查是否满足最低版本: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html

程序运行用户权限不够, 有些 helm chart 默认以 nobody 权限运行, 但挂上去的 aws token file 是 root 权限的.  检查下 deployment 的 securityContext, 是否用了:

    securityContext:
        runAsUser: 65534

可以直接删掉, 让程序以 root 运行, 或设置 `fsGroup: 65534`, 让文件系统权限归 nobody 用户.