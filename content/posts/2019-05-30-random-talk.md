---
title: "Random Talk"
date: 2019-05-30T18:48:23+08:00
categories:
    - tech
tags:
    - lang-en
    - server-infra
---

Just some random complains and notes about server infra management. I think those are my motivations to move to kubernetes.

Won't explain k8s or docker in detail, and how they solve those problems in this post.

## Infrastructure level(on AWS)

We use following services provided by AWS.

- Compute:
    - EC2
    - AutoScaling Group
    - Lambda
- network:
    - VPC (SDN network)
    - DNS (route53)
    - CDN (CloudFront)
- Loadbalancer:
    - ELB (L4)
    - NLB (L4, ELB successor, support static IP)
    - ALB (L7)
- Storage:
    - EBS (block storage)
    - EFS (hosted NFS)
    - RDS(MySQl/PostgreSQL ...)
    - Redshift (data warehouse)
    - DynamoDB (KV)
    - S3 (object storage)
    - Glacier (cheap archive storage)
- Web Firewall (WAF)
- Monitor (CloudWatch)
- DMS (ETL)
...

For infra management, in early days, we just click, click, click... or write some simple scripts to call AWS api.

With infra resources growing, management became complex, a concept called `Infrastructure as Code` rising.

AWS provides CloudFormation as orchestration tool, but we use [terraform](https://www.terraform.io/) (for short: CloudFormation sucks, for long: [Infrastructure as Code](https://blog.monsterxx03.com/2017/04/21/infrastructure-as-code/))

So far, not bad.(tweak those services internally is another story... never belive `work out of box`)

## Application level

- configuration management (setup nginx, jenkins, redis, twemproxy, ElasticSearch or WTF..)
- CI/CD
- dependency management

They're complicated, people developped bunch of tools to handle: puppet, chef, ansible, saltstack ...

They're great and working, but writing correct code still a challenge when changes involves:

- Upgrade server os from ubuntu12.04 to 18.04?  Change OS from ubuntu to CentOS?
    - upgrade dependency, and configurations (it's a hell!)
    - Deploy an upgraded new server into cluster, do AB testing with old server: if boom!! ok,lets rollback else rollout to all.
- Previously, we use single nginx to do request dispatch, now we need setup 2 in every available zone for HA purpose, and do DNS round robin for them?
    - Actully, it depends on how you setup previous nginx, in a dedicated autoscaling group? or on a dedicated server? or share a server with other componments?
- Service A & B have low traffic before, setup dedicated servers for them is wastefully, so they share servers, but we need split them for now?
    - Setup new group for B, configure loadbalacer rule to redirect service B traffic to it. (okay, resuing existing lb or setup new one? depends...)
    - Change deployment flow, since B is on different servers.
    - Clean up B's legacy configure on A, or setup a new group for A, and migrate A? really depends...
- ElasticSearch have 3 nodes before, we have bunch of new features rely on it, scale to 5 nodes?
    - Bootstrap two new servers
    - Create 2 EBS volume and attach to them 
    - Provision them to install ElasticSearch
    - Make them join existing cluster. (be careful about CPU & IO pressure during shards reallocation).

Hard to test, hard to rollback, because every line of code has side effect, depends on how you break things, fix method is different.

Then people invented some principles for high level design: idempotent, [12 factor app](https://12factor.net/).

But real live sucks:

- legacy service without maintenance, but still needs.
- I installed this software from a ubuntu PPA two years ago(wrote in ansible playbook), Now need to migrate to another server, but provision failed! because PPA's gpg key expired...Okay, I should maintain my own deb repo.
- High available is great, but this function is not so important now, for budget reason ... Or we need make it on live tomorrow, do a simple setup! 6 months later, WTF...
- I'm not familiar with it before, so ... made some stupid configuration, now migration is a problem.
- Need to upgrade ansible version.

Hmm, python(2 or 3?), ruby, nodejs, java... different language, different runtime, want different versions for a lib installed on same machine.

Okay, it's python, we have GIL problem, we use gevent to do concurrency, but want to take use of multi cores on single server? Prefork by uwsgi/gunicorn, or deploy multi single process behind loadbalancer. 

How to do gracefully cycle(or called hot code reload)? It's a prefork application, and have supports for it, do `kill -HUP {pid}` is simple. No builin support? take down from loadbalancer , restart, and attach again, batch by batch.

Making change is complicated, and involves a lot of efforts, but it's not the hardest part. How to apply changes and not affect other systems is a problem. 

## K8S advantage: Resource allocation and self-healing

Application configuration or possible migration tasks are complex but still controllable if you take enough time.

If you have written great chef recipes or ansible playbooks, it won't be your first reason to migrate to k8s. Actually, I'm not a big fan of docker, even I know it solves problems.

For me, the biggest point is resource management, with container and k8s, we can binpack multi applications on single server to take advantage of resource, simple to scale when needs (with k8s deployment & HPA). Won't affect each other (if configured right).

When people saying `server down`, there're bunch of reasons:

- applications overcommits resource(OOM, out of disk, 100% CPU usage ...)
- Network issue(dns, network partition)
- aws [instance reirement](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-retirement.html) without notice.
- underlying hardware issues make vm dead.
- ...

Some of them we can do nothing(eg: network related), just post a ticket, wait for AWS to solve.

Some of them are missing pieces in monitoring and alerting(disk usage), we should get alerting notificaitons and resize manually.

OOM and 100% CPU usage are lack of resource management, depends on your application, it may can be scaled out or not.

With k8s, we can limit application using reasonable cpu and ram, not affecting other processes.

For those unexpected instance level problems, k8s can reschedule pods to health nodes automatically. It's so called self-healthing.

## Control complex

k8s solves some problems, but it's complex, and need understand a lot of concepts to make applications run on it, no magic.

If you're still confusing whether use k8s or not, think twice, list your current pain points, are they worth the efforts?
