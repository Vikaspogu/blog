+++ 
date = 2020-03-07T14:31:32-05:00
title = "Pi Hole on k3s cluster"
description = "Deploying Pi-Hole on k3s cluster"
slug = "" 
tags = ["pi-hole","k3s"]
categories = []
externalLink = ""
series = []
+++

The [Pi-hole](https://pi-hole.net/) is a fantastic tool that blocks DNS requests to ad servers. That means you can surf the web without having to look at ads on every page.

### PiHole in Kubernetes

After some google search, i have found this [amazing helm repo](https://github.com/ChrisPhillips-cminion/pihole-helm) which will install pi-hole using a [helm](https://helm.sh/) chart.

I have made few updates to chart like ingress, address values; Overriding default pi-hole variables like `WEBPASSWORD` and `TZ` as a container environment variables in deployment resource.

```yaml
configData: |-
  server=/local/192.168.1.1
  address=/.vikaspogu.com/192.168.1.132
ingress:
  host: pi-hole.vikaspogu.com
```

```
env:
- name: TZ
    value: "America/New_York"
- name: WEBPASSWORD
    value: "somepassword"
```

### Install chart

- Create a new namespace (optional)

```bash
#Create a new namespace
kubectl create ns pi-hole
```

- Install chart in namespace

```bash
cd pihole-helm
helm install pi-hole .
```

- Wait to pods

```bash
kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
pi-hole-pihole-5bb56b5bd-b2wl7   1/1     Running   0          61m
```

- Navigate to ingress route in my case (pi-hole.vikaspogu.com) and login with `WEBPASSWORD` used in deployment

![Pi-Hole UI](/images/pi-hole.png)

### Configure devices to use Pi-hole as their DNS server?

Configuring Verzion FiOS router to use Pi-Hole as DNS server:

- On the top

  - Click My Network

- On the left

  - Click Network Connections

- Click Broadband Connection (Ethernet/Coax)>Settings

  - Click the drop down for DNS Server and select 'Use The Following DNS Server Addresses'

  - Type in the static IP Address of your pi (Or Pi-hole server)

  - Click Apply
