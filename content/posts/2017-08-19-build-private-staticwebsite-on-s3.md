---
title: Build private static website on S3
author: will
date: 2017-08-19T07:28:16+00:00
url: /2017/08/19/build-private-staticwebsite-on-s3/
categories:
  - tech
tags:
  - lang-en
  - AWS
  - S3

---
Build static website on S3 is very easy, but by default, it can be accessed by open internet.It will be super helpful if we can build website only available in VPC. Then we can use it to host internal deb repo, doc site&#8230;

Steps are very easy, you only need VPC endpoints and S3 bucket policy.

AWS api is open to internet, if you need to access S3 in VPC, your requests will pass through VPC&#8217;s internet gateway or NAT gateway. With VPC endpoints(can be found in VPC console), your requests to S3 will go through AWS&#8217;s internal network. Currently, VPC endpoints only support S3, support for dynamodb is in test.

To restrict S3 bucket only available in your VPC, need to set bucket policy (to host static website, enable static website support first). At first, I didn&#8217;t check doc, try to restrict access by my VPC ip cidr, but it didn&#8217;t work, I need to restrict by VPC endpoint id:

    {
      "Version": "2012-10-17",
      "Id": "Policy1415115909152",
      "Statement": [
        {
          "Sid": "Access-to-specific-VPCE-only",
          "Principal": "*",
          "Action": "s3:GetObject",
          "Effect": "Allow",
          "Resource": ["arn:aws:s3:::my_secure_bucket",
                       "arn:aws:s3:::my_secure_bucket/*"],
          "Condition": {
            "StringEquals": {
              "aws:sourceVpce": "vpce-1a2b3c4d"
            }
          }
        }
      ]
    }
    

BTW, if you can config bucket policy restrict on VPC directly, with VPC endpoint you can limit to subnets. Details can be found in doc: http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-endpoints-s3.html
