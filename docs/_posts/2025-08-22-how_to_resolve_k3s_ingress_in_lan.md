---
layout: single
toc: true
toc_sticky: true
title:  "how to resolve k3s ingress in lan"
categories: [dns, ingress, k3s, external-dns]
tags: [dns, ingress, k3s, external-dns]
---

# 使用本地dns服务解析k3s ingress
## k3s ingress介绍
k3s service资源类型主要分为以下三种
* clusterip: 只能在内部集群中访问
* nodeport: 将内部端口映射到k3s节点的指定端口，然后添加iptable规则，实现外部通过集群任意一个节点的指定端口进行访问
* loadbalance: 对外暴露一个统一的 IP，自动把请求分发到后端 Service，内部环境需要安装MetalLB等插件
其中clusterip不能直接通过外部访问，需要借助ingerss才能实现，ingress的功能主要类似反向代理，定义ingress规则，通过统一暴露的端口，将流量转发给内部服务监听的端口。
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: music.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: navidrome  # 你上面 ClusterIP Service 的名字
                port:
                  number: 4533
    - host: grafana.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana  # 你上面 ClusterIP Service 的名字
                port:
                  number: 80
    - host: alertmanager.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-alertmanager   # 你上面 ClusterIP Service 的名字
                port:
                  number: 9093
    - host: prometheus.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-prometheus   # 你上面 ClusterIP Service 的名字
                port:
                  number: 9090
    - host: jellyfin.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jellyfin   # 你上面 ClusterIP Service 的名字
                port:
                  number: 8096
```
## 需要解决的问题
定义好ingress规则后，我们就能通过traefik或者nginx等ingress controller将外部流量路由到集群内部的服务。但是由于集群大多只是局域网内部使用，所声明的域名并不是公网域名，如果要通过这些私有域名实现访问k3s集群内部的服务，最简单的方式往往需要手动修改本地主机的hosts文件，添加这些私有域名的dns记录（指向ingress controller暴露的ip地址），但是终端可能很多，且这种手动的方式不利于维护，如何实现自动解析则是需要考虑的问题。
## 解决方案
这里我们采用搭建本地dns服务的方式实现，大概思路是通过往本地dns添加对应的记录（私有域名-ingress controller暴露的ip地址），然后在路由器添加本地dns服务地址，通过dhcp下发到局域网的终端，从而实现私有域名的dns解析。本地dns服务部署可以使用bind9或者其它方案，具体参考对应的部署文档，这里不再多说，终端获取dns地址这个也没什么难度（通过路由器配置自动下发到终端或者手动配置终端），那现在需要解决的问题就是如何在添加ingress规则时，向本地dns服务添加对应的记录。这里我们可以通过借助external-dns来避免手动维护。
### external-dns
ExternalDNS 是一个 Kubernetes 控制器（Controller），可以把集群内的服务（Service）和 Ingress 的域名信息同步到 外部 DNS 服务（如 AWS Route53、Cloudflare、阿里云 DNS 等）。可以自动创建、更新、删除 DNS 记录，无需手动去 DNS 控制台操作。由于我们搭建的本地dns服务，因此需要通过RFC2136 协议实现。
* 部署示例（官方有提供对应的helm chart）
```
# rfc2136.host是本地dns服务所在的ip地址，注意rfc2136.tsigKeyname，rfc2136.tsigAlgorithm，rfc2136.tsigSecret是rfc2136的认证信息，需要在本地dns上配置同样的信息以实现维护dns记录时的认证
helm install external-dns bitnami/external-dns    --set provider=rfc2136   --set rfc2136.host=192.168.31.100   --set rfc2136.port=53   --set rfc2136.zone=pengdake.xyz   --set rfc2136.tsigSecret=QLErbZUtNtEsCk5a4ExIyP6uM1WgVqXc1FxRyCDGaS4=  --set rfc2136.tsigKeyname=externaldns   --set rfc2136.tsigAlgorithm=hmac-sha256   --set policy=upsert-only --set txtOwnerId=k3s
```
部署完成后，我们就可以在ingress规则声明中添加external-dns.alpha.kubernetes.io/target信息，实现ingress规则变更时，自动维护本地dns的记录
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    external-dns.alpha.kubernetes.io/target: "192.168.31.94,192.168.31.95,192.168.31.96,192.168.31.98"
    # 如果要 https 可以加 websecure
    # traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  rules:
    - host: music.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: navidrome  # 你上面 ClusterIP Service 的名字
                port:
                  number: 4533
    - host: grafana.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana  # 你上面 ClusterIP Service 的名字
                port:
                  number: 80
    - host: alertmanager.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-alertmanager   # 你上面 ClusterIP Service 的名字
                port:
                  number: 9093
    - host: prometheus.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-prometheus   # 你上面 ClusterIP Service 的名字
                port:
                  number: 9090
    - host: jellyfin.pengdake.xyz    # 域名
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jellyfin   # 你上面 ClusterIP Service 的名字
                port:
                  number: 8096

```
