---
title: "Access sensitive variables on AWS lambda"
date: 2018-02-28T21:45:23+08:00
categories:
    - tech
tags:
    - lang-en
    - AWS
    - lambda
---

AWS lambda is convenient to run simple serverless application, but how to access sensitive data in code? like password,token...

Usually, we inject secrets as environment variables, but they're still visable on lambda console. I don't use it in aws lambda.

The better way is use [aws parameter store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) as configuration center. It can work with KMS to encrypt your data.

Code example:

    client = boto3.client('ssm')
    resp = client.get_parameter(
        Name='/redshift/admin/password',
        WithDecryption=True
    )
    resp:

        {
            "Parameter": {
                "Name": "/redshift/admin/password",
                "Type": "SecureString",
                "Value": "password value",
                "Version": 1
            }
        }


Things you need to do to make it work:

- Create a new KMS key
- Use new created KMS key to encrypt your data in parameter store.
- Set a execution role for your lambda function.
- In the KMS key's setting page, add the lambda execution role to the list which can read this KMS key. 

Then your lambda code can access encrypted data at runtime, and you needn't set aws access_key/secret_key, lambda execution role enable access to data in parameter store.

BTW, parameter store support hierarchy(at most 15 levels), splitted by `/`. You can retrive data under same level in one call, deltails can be found in doc, eg: http://boto3.readthedocs.io/en/latest/reference/services/ssm.html#SSM.Client.get_parameters_by_path
