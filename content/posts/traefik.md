---
title: "I Ran Traefik in Production for a Year Before I Really Understood It"
description: "The questions I should have asked before copying my first Middleware: how Traefik differs from NGINX beyond the YAML, which traffic it actually sees, what happens to a request between the entrypoint and the pod, why some changes need a restart, what a TLS 1.3 handshake really does with the certificate, and why we run more than one instance per cluster."
dateString: October 2026
draft: false
tags: ["Traefik", "Kubernetes", "TLS", "DevOps", "SRE"]
weight: 2
cover:
    image: "/blog/traefik/traefik.png"
---

For about a year, Traefik was the thing I copied. A new app needed a hostname, so I took the Ingress from the last app, changed the host and the Service name, and it worked. It needed an IP allow-list, so I found a Middleware that did that on another cluster and pasted it in. Across the customer clusters I run at Obmondo this got me surprisingly far. Traefik is forgiving of people who do not read the docs, and I took full advantage of that.

Then I had to explain our setup to someone outside the team, and I could not get past the YAML. I could say what each object was called. I could not say what happened to a request between the load balancer and the pod, why one change to Traefik needed a rollout and another did not, or what the certificate in the Secret was actually for during the handshake.

So I went back and asked the questions I should have asked at the start. This post is those questions, in the order I asked them. If you have ever copied a Middleware without reading what it does, this is for you.

## Is Traefik just a reverse proxy like NGINX?

Yes. Same job. One entry point in front of many backends, routing on hostname and path, TLS handled in one place instead of in every app, policies like redirects and allow-lists applied centrally.

The difference is how it gets its configuration. Classic NGINX reads a config file at startup, and changes to that file do nothing until you send it a reload. The reload itself is clean: the master process checks the new file, starts new workers, and tells the old ones to finish their in-flight requests and exit. But somebody or something has to write the file and send the signal. Traefik has no file to write. It watches a source of truth, Docker or Kubernetes or a few others, and when that source changes it rebuilds its routing table in memory. The docs describe this as hot-reloaded with no request interruption or connection loss.

A concrete example makes the gap obvious. Say you are putting a blog container on a Docker host at `blog.example.com`, with HTTPS from Let's Encrypt.

With NGINX, the upstream points at the container, so its address goes into the config. Then a server block for the redirect, a server block for HTTPS, and certbot to get the certificate and edit the config for you:

```nginx
upstream blog {
    server 172.18.0.5:2368;
}

server {
    listen 80;
    server_name blog.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name blog.example.com;
    ssl_certificate     /etc/letsencrypt/live/blog.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/blog.example.com/privkey.pem;

    location / {
        proxy_pass http://blog;
    }
}
```

Then `nginx -t && nginx -s reload`. Certbot's packaged timer runs `certbot renew` on a schedule, and you add a deploy hook so NGINX reloads when a renewal lands. When the container is recreated and comes up on a different IP, or when you scale it to two, you edit the upstream and reload again.

With Traefik on the same host, the per-app configuration is two labels on the container:

```yaml
services:
  blog:
    image: ghost:5
    labels:
      traefik.http.routers.blog.rule: Host(`blog.example.com`)
      traefik.http.routers.blog.tls.certresolver: le
```

The certificate resolver named `le` and the redirect from port 80 to 443 are set once on the Traefik instance, not per app. Traefik sees the container start through the Docker socket, reads the labels, picks the container's exposed port, requests a certificate, and starts routing. Recreate the container and the new IP is picked up. Scale it and both replicas get traffic. Remove it and the route goes away.

Now the honest part. In Kubernetes the gap is much smaller than this example suggests, because ingress-nginx does exactly the wrapping I just described by hand. It watches Ingress objects, renders an `nginx.conf` from a template, and reloads NGINX when the config changes; it even skips the reload when only the endpoints behind an upstream change, by pushing the new list into a Lua handler inside NGINX. And cert-manager takes care of certificates for either controller: it talks to Let's Encrypt, stores the certificate and key in a Secret, and renews before expiry. In a cluster, both tools give you declarative routing and automatic certificates.

What remains is a difference in design. ingress-nginx is a reload-based proxy with an automation layer wrapped around it. Traefik is dynamic from the ground up; nothing in it works by regenerating a file. For most workloads you will not feel that day to day. It shows up in the edges: how often routes change, what a bad config does, how the provider model composes. Those are the next questions.

