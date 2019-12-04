---
title: Infrastructure as Code
author: will
date: 2017-04-21T16:25:07+00:00
url: /2017/04/21/infrastructure-as-code/
categories:
  - tech
tags:
  - lang-en
  - AWS
  - server-infra

---
Create virtual resource on AWS is very convenient, but how to manage them will be a problem when your size grow.

You will come to:

  * How to explain the detail online settings for your colleagues (like: how our prod vpc is setup?what&#8217;s the DHCP option set?), navigate around AWS console is okay, but not convenient.
  * Who did what to which resource at when? AWS have a service called `Config`, can be used to track this change, but if you want to make things as clear as viewing git log, still a lot of works to do. 

Ideally, we should manage AWS resources like code, all changes kept in VCS, so called `Infrastructure as Code`.

I&#8217;ve tried three ways to do it:

  * ansible
  * CloudFormation
  * terraform

In this article, I'll compare them, however, the conclusion is to use terraform 🙂
  
## Ansible

Provision tools, like ansible/chef/puppet, all can be used to create aws resources, but they have some common problems:

  * Hard to track changes after bootstrap.
  * No confident what it will do to existing resources.

For example, I define a security group in ansibble:

    ec2_group:
      name: "web"
      description: "security group in web"
      vpc_id: "vpc-xxx"
      region: "us-east-1"
      rules:
        - proto: tcp
          from_port: 80
          to_port: 80
          cidr_ip: 0.0.0.0/0
    

It will create a security group named &#8220;web&#8221; in vpc-xxx. At first glance, it&#8217;s convenient and straightforward.

After several months, we will come to some problems:

  * If someone deleted `web` from console, how can I know?
  * If someone changed the name `web` in ansible, this code will create a complete new security group, and I&#8217;ll lose track of original `web`.
  * If someone change the rules, to make `80-80` to `443-443`, what it will do? delete original rule and add a new one? or just create a new rule? Actually, you don&#8217;t know, must test it before apply those changes to any resources. 

The root cause is ansible lack a mapping between its code and real online resources.

Those provision tools only declare they want some specific resources, if resources exist, they will try to update them, however not all attributes of AWS resources support updating after creation.

So after first creating, I really don&#8217;t have confidence whether this code work anymore.

## CloudFormation

CloudFormation is AWS&#8217;s official solution. It also use yaml as configuration format, along with a fancy web interface named `CloudFormation Designer`.

But from my experience, it really sucks&#8230;.

### Annoying syntax

The template syntax is very complex and hard to read, reading the doc for 10 mins, I think you will be lost.

    Resources:
      Ec2Instance:
        Type: AWS::EC2::Instance
        Properties:
          SecurityGroups:
          - Ref: InstanceSecurityGroup
          KeyName: mykey
          ImageId: ''
      InstanceSecurityGroup:
        Type: AWS::EC2::SecurityGroup
        Properties:
          GroupDescription: Enable SSH access via port 22
          SecurityGroupIngress:
          - IpProtocol: tcp
            FromPort: '22'
            ToPort: '22'
            CidrIp: 0.0.0.0/0
    

It took me whole day to accept its syntax, and now I don&#8217;t want to touch it anymore.

### Update policy is too strong.

The main problem is it forbids update resources in stack manually.Otherwise next stack update will come to error.

But real world will never be so clear.

Every resource have a lot of properties, some of them can be updated smoothly, but changes to some of them need to recreate the resource.

Eg: update RDS DBInstance&#8217;s `StorageEncrypted` attribute will require to replace the original db.

Last time when I&#8217;m doing HIPAA compliance for my company, I need to migrate my existing RDS to encrypted one without downtime, the flow is very complex.

Simply change property in CloudFormation will delete my DB&#8230;My ideal way is do some changes to resources manually and change CloudFormation templates to match the new resources. However, with CloudFormation you can&#8217;t do it simply.

Another wired things is some attributes can be changed in console directly, but with CloudFormation, it will require a recreation of resource, like: RDS db port&#8230;

Last time, when I checking, change ELBs associated with AutoScaling group also requires destroy the autoscaling group first, it&#8217;s crazy. But now it&#8217;s can be updated dynamically.

And for complex prod environment, the fancy designer interface don&#8217;t have many use &#8230;.

But there is still some case you can use CloudFormation: create similar and short lifetime dev environment for many teams. If the data in the environment is not important, you just need repeatable clear environment, CloudFormation deserves take a look.

## Terraform

Okay, let&#8217;s make a conclusion why tracking infrastructure on AWS is hard:

  * Dependence between resources is complex
  * How can we predict our changes before apply them?
  * Interaction with human operation or existing resource.

[terraform][1] havn&#8217;t solved those problems perfectly, but it&#8217;s more usable compared with others so far.

I won&#8217;t explain how to use terraform too much, just talk about the good part I&#8217;m using it.

### Dependence management

terraform use the simple `DAG` concept to management dependences, which I alrady used in ELT job management a lot.

The core concept is there shouldn&#8217;t be any loop in state transformation, your resource just change from one state to another, but need to wait for its dependent resource change done first.

    resource "aws_key_pair" "master" {
      key_name   = "master"
      public_key = "...."
    }
    
    resource "aws_instance" "web0" {
      ami = "ami-xxxx"
      instance_type = "m3.medium"
      key_name = "${aws_key_pair.master.key_name}"
    }
    

The dependence relationship here is implict, ec2 instance web0 requires key pair to be created first, so when running, terraform will create the key pair first, then create instance. Also, the relationship can be defined with attribute `depends_on` explictly.

### Predictable change

CloudFormation have `ChangeSet`, but it&#8217;s differences between two templates, not compared with real online resources (it assume all changes happen within CloudFormation).

terraform can generate execution plan by `terraform plan`, it&#8217;s configuration declare the desired state those resource should have, it will compare with real resource,
  
and show your what will happen if you do `terraform apply`.

The output have three color, red means this resource will be destroied, yellow means update, green menas create a new one. If there&#8217;s a `+/-` mark with green msg, it will destroy existing resource and create a new one.

How terraform map online resource with its code?

With `state` data. I use the default local tfstate file backend.

In my above example, `master` and `web0` are unique marks in terraform, every resource type can&#8217;t have two resources with same name,
  
the terraform name mark and real resource id mapping is in tfstate file, so we can know exactly which instance `web0` means, won&#8217;t have the problem we have in ansible.

If you delete definition in terraform config file, since the state file still tracking it, `terraform plan` will show you&#8217;re going to delete the resource. Then it&#8217;s upon to you, whether to delete the reosurce via terraform, or make the resource untrack by delete the data in tfstate file.

### Handle the dirty world

Since our infrastructure have existed long time ago. I need to make terraform work with those resources.

`terraform import` can import existing resources. But only resources will unique id is importable.

If you do a complex migration for live servers manually, the real instance id has changed, you can just delete related data in tfstate file, and reimport the resource again.

### Modular composite multi resources

Create a vpc is complex, involved resources are: vpc, public/private subnet,dhcp options, internet gateway, nat gateway, route table.

Make vpc creation a module can reuse code, provide it different parameters create different environments. I did something similar like this `https://github.com/terraform-community-modules/tf_aws_vpc`, so when using, I can define a vpc like this:

    module "vpc" {
      source         = "../modules/aws_vpc"
      cidr           = "10.100.0.0/16"
      azs            = ["us-west-1a", "us-west-1c"]
      internal_domain = "test.com"
      nat_eip_id     = "eip-xxx"
      public_subnets = ["10.100.1.0/24", "10.100.2.0/24"]
      app_subnets    = ["10.100.10.0/24", "10.100.20.0/24"]
      data_subnets   = ["10.100.30.0/24", "10.100.40.0/24"]
    }
    

You can also import online resource into a module.

### Handle unimportable resources

Resources like vpc route table rule, can&#8217;t be imported, but you can write your config and make sure they&#8217;re right, then do a `terraform apply` operation, terraform will not change online settings if they&#8217;re matching, just update local state file.

Some relationships between resources are also defined as resources in terraform, like ebs device mapping. They can&#8217;t be imported as well,
  
If you create them manually, you can only write the config with same settings manually and do a `terraform apply` to let terraform update state file.

It&#8217;s not good and a little dangerous, but seems no better way so far.

 [1]: http://terraform.io
