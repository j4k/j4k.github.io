---
layout: post
title:  "Using socket.io on Elastic Beanstalk with nginx reverse proxy"
date:   2015-03-14 00:47:41
categories: deployment
---


This weekend I was trying out elastic beanstalk to deploy a node twitter stream sentiment analysis tool I am building as a side project. The end project is to visualise this data in cool ways. It parses tweets from twitter's garden hose and matches certain keywords in locations and then pushes them out to the browser over a socket with sentiment data attached.

However on deployment I could not get web sockets connected so I could push data out to the browser. Default node hosting for EB is nginx reverse proxied to a node process. Disabling the nginx proxy saw my websockets come to life. This is obviously not ideal for production environments. After some searching in the docs, I found you can use `files.config` file in `.ebextensions` directory to deploy.

I added this to this file:

{% highlight %}
files:
    "/etc/nginx/conf.d/websocketupgrade.conf" :
        mode: "000755"
        owner: root
        group: root
        content: |
             proxy_set_header        Upgrade         $http_upgrade;
             proxy_set_header        Connection      "upgrade";
{% endhighlight %}

This creates a configuration file for nginx that upgrades a websockets initial request to a TCP connection. Commiting this to the repo and running `eb deploy` saw my sockets working. Nice and easy!

Elastic beanstalk can be configured in a variety of ways, the docs were pretty useful and can be found [here][ebdocs].

[ebdocs]: http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-ec2.html#customize-containers-format-files










