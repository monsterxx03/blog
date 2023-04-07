---
title: "About Me"
date: 2016-01-01T23:49:46+08:00
---

Work with tech, belive in magic, live on fun :)

### Side projects

- https://github.com/monsterxx03/gospy  non-Invasive goroutine inspector
- https://github.com/monsterxx03/snet  transparent tcp proxy(mainly used in China), works on Linux & MacOS


## Work expenrience

### 2021.6 ~ Now  XSky Ltd (https://www.xsky.com/)

Work as senior architect, participated in the design and development of HCI(Hyperconverged infrastructure) system from 0 to 1.

It's a vm scheduling and management system.

Main tech stack:
- Postgresql as DB, use logical replication as msg queue.
- VictoriaMetrics for monitoring.
- Etcd for leader election.
- code in golang

Major achievements:
- Write a terraform provider for the product, which generates the terraform schema definition from an OpenAPI spec using code generation, uses reflection to implement generic Terraform CRUD.
- Designed and developed a HA solution for Postgresql based on Patroni(by client side service discovering)
- Build alerting system based on VictoriaMetrics, support both metrics & event based alerting.
- Design and implement the upgrade path for the product.


### 2014.11 ~ 2021.5 Glow Inc (https://glowing.com)

Work as company's infrastructure architect:
- Responsible for AWS-based infrastructure design, selection, maintenance, and refactoring. System capacity planning, cost management.
- Identify bottlenecks in system performance and architecture scalability, refactor existing code, increase user base by 10X while keeping IT cost increase to only two times.
- Regular team training and technical sharing, as well as code quality control.

#### Server Infra

- Develop and achieve SLOs for the business, including:
    - Availability: service uptime >=99.99%
    - Quality: p95 latency < 200ms
    - Core service p95 latency < 50ms.
- Implement infrastructure as code and use Terraform to codify the management of AWS infrastructure components such as VPC, Autoscaling Group, IAM permissions, DMS tasks, RDS, etc. All historical changes should be tracking.
- Fully migrating the architecture to EKS (AWS-managed Kubernetes) and upgrading it from version 1.10 to 1.18 without any downtime, such as containerizing the business applications and refactoring the monitoring and alerting solution from Datadog to Prometheus+Grafana.
- Based on Jenkins, using Groovy to write pipelines, achieve self-service deployment and rollback of business, and integrate SonarQube to ensure code quality control for the team.
- Database selection and optimization, e.g., using MySQL on RDS or AWS Aurora for relational databases, DynamoDB for key-value databases, Sentinel Redis for message queue brokers, Redis Cluster for caching, and ElasticSearch for search and analytics
- Performing daily troubleshooting and performance tuning in the production environment, and improving related toolchains, such as using Sentry for bug reporting, Jaeger for OpenTracing, and FluentBit + Loki + Syslog-ng for logging.
- Developing a management tool for EKS worker node groups using Golang, enabling seamless migration of stateless applications (such as web services and some message queue consumers) and stateful applications (such as Redis Sentinel HA and Redis Cluster).
- Assisting the company in achieving HIPAA compliance, which primarily involves implementing technical measures such as data-at-rest encryption, log anonymization, and access control for internal data.

### Code framework design and refactor

- Upgrading a Python project with over 600,000 lines of code from Python 2.7 to Python 3.6, performing a gradual rollout to production within three months without any downtime and without adding extra burden to business development. This was achieved by using SonarQube to perform syntax checks in the CI phase to prevent developers from submitting Python 2.7 incompatible code.
- Splitting a code repository was necessary as the early codebase consisted of over a dozen Python repositories that imported each other, leading to significant deployment and troubleshooting issues. To address this, the API frontend layer and business RPC layer were cleanly separated, and the RPC client was generated through code to synchronize with each repository (using AST traversal for syntax checks).
- Refactor the deduplication module in the business with Bloom filter to reduce the latency and storage of related business to around 1/10 of the original.

### Data warehourse 
- Build a data warehouse based on Redshift/Spectrum and S3
- Design and implement ETL process using DMS, fluentd, argo-workflow, and Spark.
- Refactor an early complex reporting system (with over 10,000 lines of Python code) by separating the definition and calculation logic of metrics. After the transformation, the core Python code was reduced to 1,000 lines, and the rest was written in YAML. The system had not been significantly modified for five years. The purpose of the refactoring was to enable PAs (Product Analysts) who are not proficient in Python to define and launch business metrics independently.


### 2013.1 ~ 2014.10  UnitedStack(previous ustack.com, current tfcloud.com)

Main work is build a privatized OpenStack distribution, including:
- Rebuild openstack console to replace official horizon project (angular + django)
- Add multi region support for keystone
- Union event bus based on rabbitmq (pipeline event in openstack to third part system)

### 2012.01 ~ 2012.12 Sina (sina.com.cn)

Custom development based on OpenStack projects, e.g: keystone, swift, keystone


*email*: xyj.asmy@gmail.com
