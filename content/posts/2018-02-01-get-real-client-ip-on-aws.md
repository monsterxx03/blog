---
title: "Get Real Client Ip on AWS"
date: 2018-02-01T15:20:37+08:00
categories:
  - tech
tags:
  - lang-en
  - AWS
  - CloudFront
  - ALB
---


If you run a webserver on AWS, get real client ip will be tricky if you didn't configure server right and write code correctly.

Things related to client real ip:

- CloudFront (cdn)
- ALB (loadbalancer)
- nginx (on ec2)
- webserver (maybe a python flask application).

Request sequence diagram will be like following:

![req](/posts/images/cf-alb-nginx.png)


User's real client ip is forwarded by front proxies one by one in head `X-Forwarded-For`.

For CloudFront:

- If user's req header don't  have `X-Forwarded-For`, it will set user's ip(from tcp connection) in `X-Forwarded-For` 
- If user's req already have `X-Forwarded-For`, it will append user's ip(from tcp connection) to the end of `X-Forwarded-For`


For ALB, rule is same as CloudFront, so the `X-Forwarded-For` header pass to nginx will be the value received from CloudFront + CloudFront's ip.


For nginx, things will be tricky depends on your config.

Things maybe involved in nginx:

- real ip module
- proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

If you didn't use real ip module, you need to pass X-Forwarded-For head explictly.

`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` will append ALB's ip to the end of `X-Forwarded-For` header received from ALB.

So `X-Forwarded-For` header your webserver received will be `user ip,cloudfront ip, alb ip`

Or you can use real ip module to trust the value passed from ALB.

Config will be:

        set_real_ip_from  10.50.0.0/16;
        real_ip_header    X-Forwarded-For;
        real_ip_recursive on;

Assume your vpc cidr range is `10.50.0.0/16`, so trust header passed from this range, it will including ALB's ip addresses (lanuched in same vpc). It will use `X-Forwarded-For` header received from ALB as real client ip.Don't use $proxy_add_x_forwarded_for in this case, otherwise the `X-Forwarded-For` webserver see will be `user ip, cloudfront ip, user ip`.

To make things clear, I use real ip module, and didn't set $proxy_add_x_forwarded_for, then my webserver will see `user ip, cloudfront ip`

If your application is flask app, there is an easy way to retrive use's ip.

        from flask import Flask, request
        from werkzeug.contrib.fixers import ProxyFix

        app = Flask('test_app')
        app.wsgi_app = ProxyFix(app.wsgi_app, num_proxies=2)

        @app.route('/')
        def hello():
            return request.remote_addr


Use `ProxyFix` middleware to tell webserver how many proxies are on front of your application, since I use real ip module, `X-Forwarded-For` don't contain ALB ip (nginx work like a real transparent proxy), so on the view of my webserver, there are two proxies on front, cloudfront and alb. Then `request.remote_addr` will get the right client ip.
