+++
title = 'Setting Up Cloudflared Tunnel on OpenShift'
date = 2024-03-11T16:03:46-04:00
draft = false
tags = ["OpenShift", "Cloudflared"]
+++

## Introduction

Cloudflare's Tunnel, powered by Cloudflared, provides a secure way to expose your web applications to the internet without exposing your origin server's IP address. This guide walks us through the steps to set up a Cloudflared tunnel on OpenShift, a popular container platform.

{{< alert >}}
This setup will expose the OpenShift console and Ingress routes only, not the API server, for `oc login`
{{< /alert >}}

### Prerequisites

- An OpenShift cluster, either on-premises or in a cloud provider
- Cloudflare account
- Basic knowledge of OpenShift and Cloudflare

### Install Cloudflared on Your Local Machine

Download and [Install](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-local-tunnel/#1-download-and-install-cloudflared) the Cloudflared binary for your operating system from the Cloudflare website.

```bash
brew install cloudflared
```

Verify the installation

```bash
$ cloudflared version
cloudflared version 2024.2.1 (built 2024-02-20T16:25:25Z)
```

### Create a Cloudflare Tunnel

Authenticate Cloudflared: A browser window will open and prompt you to log in to your Cloudflare account. After logging in, select your hostname.

```bash
cloudflared tunnel login
```

Create a tunnel and give it a name. From the command's output, note the Tunnel's UUID and the path to your Tunnel's credentials file.

```bash
cloudflared tunnel create openshift
```

### Configure Cloudflared for OpenShift

Create an OpenShift secret from the Tunnel's credentials file.

```bash
oc create secret generic cloudflared --from-file ~/.cloudflared/<tunnel_id>.json
```

To deploy the cloudflared tunnel image, I'll be using a helm [chart](https://github.com/xUnholy/charts)

Create a `values.yaml` with a cloudflared configuration.

```yaml
replicaCount: 3
image:
  repository: docker.io/cloudflare/cloudflared
  tag: "2024.2.1"
cloudflared:
  tunnelID: "<REPLACE_WITH_TUNNEL_ID>"
  existingSecret: cloudflared
  ingress:
    - hostname: "oauth-openshift.apps.<DOMAIN>"
      originRequest:
        noTLSVerify: true
      service: https://oauth-openshift.openshift-authentication
    - hostname: "*.apps.<DOMAIN>"
      originRequest:
        noTLSVerify: true
      service: https://router-internal-default.openshift-ingress.svc.cluster.local
    - service: http_status:404
```

Install helm chart

```bash
helm repo add xunholy-charts https://xunholy.github.io/charts/
helm install cloudflared xunholy-charts/cloudflared -f values.yaml -n networking
```

Cloudflared will authenticate with Cloudflare using the credentials in the configuration file and establish a secure tunnel to your OpenShift service.

Verify that cloudflared pods are running.

```bash
oc get pods
NAME                          READY   STATUS    RESTARTS   AGE
cloudflared-fd8478945-l7xhc   1/1     Running   0          119m
cloudflared-fd8478945-v9nq5   1/1     Running   0          119m
cloudflared-fd8478945-wt2z7   1/1     Running   0          119m
```

### Create a DNS record

Follow [this](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/routing-to-tunnel/dns/#create-a-dns-record) guide to create a DNS record for the tunnel. For the CNAME record name, add `*.apps.<DOMAIN_NAME>` with the target as `<TUNNEL_ID>,cfargotunnel.com`

### Verify the Tunnel

Open a web browser and navigate to the OpenShift console.

Congratulations! You have successfully set up a Cloudflared tunnel on OpenShift, which allows you to expose your web application securely to the internet.
