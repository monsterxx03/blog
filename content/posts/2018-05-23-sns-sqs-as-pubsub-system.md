---
title: "Use SNS & SQS to build Pub/Sub System"
date: 2018-05-23T18:05:28+08:00
categories:
    - tech
tags:
    - AWS
    - SNS
    - SQS
---

Recently, we build pub/sub system based on AWS's SNS & SQS service, record some notes.

Originally, we have an pub/sub system based on redis(use BLPOP to listen to a redis list). It's
really simple, and mainly for cross app operations. Now we have needs to enhance it to support more complex
pubsub logic, eg: topic based distribution. It don't support redivery as well, if subscribe failed to process
the message, the message will be dropped.

There're three obvious choices in my mind:

1. kafka
2. AMQP based system (rabbitmq,activemq ...) 
3. SNS + SQS

My demands for this system are:

- Support message persistence.
- Support topic based message distribution.
- Easy to manage.

The data volume won't be very large, so performance and throughput won't be critical concerns.

I choose SNS + SQS, main concerns are from operation side:

- kafka need zookeeper to support cluster.
- rabbitmq need extra configuration for HA, and AMQP model is relatively complex for programming.

So my decision is:

- application publish message to SNS topic
- Setup multi SQS queues to subscribe SNS topic
- Let different application processes to subscribe to different queues to finish its logic.

SQS and SNS is very simple, not too much to say, just some notes:

- SQS have two type queues, FIFO queue and standard queue. FIFO queue will ensure message order, and ensure exactly once delivery, tps is limited(3000/s)
  standard queue is at least once delivery, message order is not ensured, tps is unlimited. In my case, I use standard queue, order is not very important.
- SQS message size limit is 256KB.
- Use [goaws](https://github.com/p4tin/goaws) for local development, it has problem on processing message attributes, but I just use message body. messages only store in ram,
  will be cleared after restarted.
- If you failed to delivery message to sqs from sns, can setup topic's `sqs failure feedback role` to log to cloudwatch, in most case it's caused by iam permission. 
- Message in sqs can retain at most 14 days.
- Once a message is received by a client, it will be invisible to other clients in `visibility_timeout_seconds`(default 30s). It means if your client failed to process
the message, it will be redelivered after 30s.
- SQS client use long polling to receive message, set `receive_wait_time_seconds` to reduce api call to reduce fee.
- If your client failed to process a message due to bug, the message will be redelivered looply, set `redrive_policy` for the queue to limit retry count, and set a dead letter
queue to store those messages. You can decide how to handle them late.


I setup SNS and SQS via terraform, used following resources:

- aws_sns_topic
- aws_sns_topic_subscription
- aws_sqs_queue
- aws_sqs_queue_policy (iam policy to allow sns to send msg to sqs)