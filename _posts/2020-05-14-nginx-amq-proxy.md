---
title: NGINX as AMQ reverse proxy
tags: [Red Hat, Openshift, AMQ, Fuse]
style: 
color: 
description: Using NGINX as a reverse proxy for Red Hat AMQ
---

`The Problem:` Connecting to Red Hat AMQ through a Proxy Server from a Red Hat Fuse microservice using the camel-amqp component as the client 

`The Solution:` Installing NGINX on the Proxy Server on tcp mode, then instead of connecting the Qpid JMS client (camel-amqp) to AMQ directly, it will connect to a tcp port exposed on NGINX

## 1. Configure NGINX

In this example the Proxy Server was simulated as being an Openshift Cluster, so I deployed the NGINX container there.

The NGINX configuration is quite simple. Create a file `nginx.conf` where we are telling NGINX to expose `port 8085` and redirect to AMQ  - `proxy_pass 192.168.64.1:61616`:

```
load_module /usr/lib/nginx/modules/ngx_stream_geoip_module.so;
worker_processes auto;

events {
    worker_connections  1024;
}

stream {
    server {
        listen 8085;
        #TCP traffic will be forwarded to the specified server
        proxy_pass 192.168.64.1:61616;

        #If the proxy server has several network interfaces, you can optionally configure NGINX to use a particular source IP address when connecting to an upstream server.


    }
}
```

```
NOTE: This is an insecure connection, for more information on securing the TCP Traffic:
https://docs.nginx.com/nginx/admin-guide/security-controls/securing-tcp-traffic-upstream/
```

## 1. Deploy NGINX on Openshift



