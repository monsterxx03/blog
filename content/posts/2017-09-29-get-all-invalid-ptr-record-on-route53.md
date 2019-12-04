---
title: 'Get all invalid PTR record on  Route53'
author: will
date: 2017-09-29T08:55:18+00:00
url: /2017/09/29/get-all-invalid-ptr-record-on-route53/
categories:
  - tech
tags:
  - lang-en
  - AWS
  - route53

---
I use autoscaling group to manage stateless servers. Servers go up and down every day.

Once server is up, I will add a PTR record for it&#8217;s internal ip. But when it&#8217;s down, I didn&#8217;t cleanup the PTR record. As times fly, a lot of invalid PTR records left in Route53.

To cleanup those PTR records realtime, you can write a lambda function, use server termination event as trigger. But how to cleanup the old records at once?

Straightforward way is write a script to call AWS API to get a PTR list, get ip from record, test whether the ip is live, if not, delete it.

Since use awscli to delete a Route53 record is very troublesome (involve json format), you&#8217;d better write a python script to delete them. I just demo some ideas to collect those records via shell.

You can do it in a single line, but make things clear and easy to debug, I split it into several steps.

## Get PTR record list

    aws route53 list-resource-record-sets  --hosted-zone-id xxxxx --query "ResourceRecordSets[?Type=='PTR'].Name" |  grep -Po '"(.+?)"' | tr -d \" > ptr.txt
    

ptr.txt will contain lines like:

    1.0.0.10.in-addr.arpa.
    2.0.0.10.in-addr.arpa.
    

## Get ip list from PTR records

    cat ptr.txt | while read -r line ; do echo -n $line | tac -s. | cut -d. -f3- | sed 's/.$//' ; done > ip.txt
    

ip.txt:

    10.0.0.1
    10.0.0.2
    

## Filter out invalid ips

Please do it on server where you can access internal ips.

    cat ip.txt | while read -r line  ; do ping -W1 -c1 $line > /dev/null  2>&1 || echo $line  >> invalid_ip.txt ; done
    

Then invalid_ip.txt will have all ips not valid

## Transfer invalid ipts to PTR records

    cat invalid_ip.txt  | while read -r line; do echo -n $line. | tac -s. && echo in-addr.arpa ; done > ptr_del.txt
