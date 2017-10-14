---
title: Build deb repository with fpm , aptly and s3
author: will
type: post
date: 2017-06-23T09:40:58+00:00
url: /2017/06/23/build-deb-repository-with-fpm-aptly-and-s3/
categories:
  - tech
tags:
  - aptly
  - fpm

---
I&#8217;m lazy, I don&#8217;t want to be deb/rpm expert, I don&#8217;t want to maintain repo server. I want as less maintenance effort as possible. 🙂

Combine tools fpm, aptly with aws s3, we can do it.

## Use fpm to convert python package to deb

[fpm][1] can transform python/gem/npm/dir/&#8230; to deb/rpm/solaris/&#8230; packages

Example:

    fpm -s python -t  deb -m xyj.asmy@gmail.com --verbose  -v 0.10.1 --python-pip /usr/local/pip Flask
    

It will transform Flask 0.10.1 package to deb. Output package will be `python-flask_0.10.1_all.deb`

Notes:

  * If python packages rely on some c libs like `MySQLdb` (libmysqlclient-dev), you need to install them on the machine to build deb binary.
  * By default fpm use easy\_install to build packages, some packages like httplib2 have permission bug with easy\_install, so I use pip 
  * By default, msgpack-python will be convert to `python-msgpack-python`, I don&#8217;t like it, so add `-n python-msgpack` to normalize the package name.
  * Some package&#8217;s dependencies&#8217; version number is not valid(eg: celery 3.1.25 deps pytz >= dev), so I replace the dependencies with `--python-disable-dependency pytz -d 'pytz >= 2016.7'`
  * fpm will not dowload package&#8217;s dependency automatically, you need to do it by your self

## Use aptly to setup deb repository

[aptly][2] can help build a self host deb repository and publish it on s3.

### gpg key

Before setup aptly, ensure you have vaild gpg key, deb reporistory need gpg key to sign packages.

Quick tips about gpg key:

  * generate gpg key: `gpg --gen-key`. on OSX, use `gpg2 --full-generate-key`
  * export private key: `gpg --export-secret-key -a "User Name" > private.key`
  * export public key: `gpg --export -a "User Name" > public.key`
  * import private key: `gpg --allow-secret-key-import --import private.key`
  * import public key: `gpg --import public.key`

Assume we&#8217;re going to setup a dep repo named `release`:

### aptly

Create local repo:

`aptly repo create -distribution=xenial -component=main release`

By default, it will create config file at `~/.aptly.conf`, and data dir `~/.aptly`

Import all deb packages in dir /mnt/debs to repo `release`

`aptly repo add release /mnt/debs/`

setup s3 bucket info in ~/.aptly.conf:

    {
       "architectures":[],
       ....
       "S3PublishEndpoints":{
          "repo.example.com":{
             "region":"us-east-1",
             "bucket":"repo.example.com",
             "acl":"public-read"
          }
       }
    }
    

Publish repo to s3:

`aptly publish repo release s3:repo.example.com:`

If you added/updated/delete some packages in local repo, update s3: `aptly publish update  xenial  s3:repo.example.com:`

## Use the repo

First, your client machine must import the gpg public key:

`apt-key add public.key`

Add repo url:

`echo 'deb http://repo.example.com xenial main' >> /etc/apt/sources.list && apt update`

Since I want my repo have high priority than system default repo, I need to setup apt preferences, create /etc/apt/preferences, with content:

    Package: *
    Pin: origin "repo.example.com"
    Pin-Priority: 1000
    

Then all packages will be installed from host `repo.example.com` first, priority 1000 means even package version in remote repo is lower than your installed version, still downgrade it.

For detail about apt preferences: `man 5 apt_preference`

 [1]: https://fpm.readthedocs.io/en/latest/
 [2]: https://www.aptly.info/