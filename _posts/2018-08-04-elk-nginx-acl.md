---
layout: post
title: Elasticsearch with Kibana ACLs through Nginx
tags:
- elasticsearch
- nginx
- aws
---

You need Nginx as a reverse proxy ACL manager for your Elasticsearch cluster if:

1. You are using [AWS Elasticsearch](https://aws.amazon.com/elasticsearch-service/). You would have 
noticed by now that as of this writing their Kibana UI does not come with [X-Pack plugins for
Role based access controls](https://www.elastic.co/guide/en/x-pack/current/authorization.html).

This means that all the users in your org can:

1. Create/read/update/delete searches, visualizations, dashboards.
2. Mess around with the admin controls.

2.  Conversely if you run your own Elasticsearch cluster and you want to try out simple 
multi-level ACL (read-only, normal and admin user) without paying for the 
[X-Pack feature](https://www.elastic.co/subscriptions), then you will need to put something 
in between your users and Kibana UI to allow/block operations.

[This](https://github.com/saurabh-hirani/docker-elk-nginx-acl) is the setup. Follow the README and 
profit. Check [nginx/config/kibana.conf](https://github.com/saurabh-hirani/docker-elk-nginx-acl/blob/master/nginx/config/kibana.conf)
for details on how I implemented the ACLs. Basically it was a cycle of going through the calls 
made by Kibana UI and blocking/allowing the right calls for the right users. Simple and effective. 
