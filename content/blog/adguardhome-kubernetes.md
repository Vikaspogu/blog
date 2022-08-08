+++ 
date = 2020-08-19
title = "AdGuard on Kubernetes"
description = "Deploying adguard home on kubernetes"
slug = "" 
tags = ["adguard", "kubernetes", "ad-blocker"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

## AdGuard

[AdGuard Home](https://github.com/AdguardTeam/AdGuardHome/) is a network-wide software for blocking ads & tracking

Adguard is similar to Pi-Hole with more features. Comparison of [Adguard to Pi-Hole](https://github.com/AdguardTeam/AdGuardHome#how-does-adguard-home-compare-to-pi-hole)

The below configuration has worked for me. I am a big fan of helm charts and I'll be using [AdGuard chart](https://github.com/billimek/billimek-charts/tree/master/charts/adguard-home)

### Configure chart

Pull chart locally

```bash
helm repo add billimek https://billimek.com/billimek-charts/
helm fetch billimek/adguard-home
```

Update `deployment` to use `hostNetwork`

```yaml
#templates/deployment.yaml
...
  spec:
    hostNetwork: true
    securityContext:
...
```

Enable `configAsCode`, update `bind_host` to Kubernetes host IP in `values.yaml`

```yaml
# values.yaml
configAsCode:
  enabled: true
  config:
    bind_host: 192.168.0.101
    bind_port: 3000
    dns:
      bind_host: 192.168.0.101
```

Update `securityContext` to run as a `privileged` pod, drop all capabilities and add `NET_BIND_SERVICE`

```yaml
# values.yaml
securityContext:
  privileged: true
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

Add `nodeSelector` to assign pod to that node

```yaml
# values.yaml
nodeSelector:
  kubernetes.io/hostname: node3
```

Deploy helm chart

```bash
helm install adguard-home
```

Wait for AdGuard pods

```bash
kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
adguard-adguard-home-67975f7768-v6bg9   1/1     Running   0          43h
```

### Configure your devices to use your AdGuard Home

Now, once we've established that AdGuard Home is deployed, you can use it on other computers in your network by changing their system DNS settings to use Kubernetes node's IP address (which is 192.168.0.101 in our case).
