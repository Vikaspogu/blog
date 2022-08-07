+++ 
date = 2020-11-04
title = "Syslog with Loki Promtail"
description = "Setup rsyslog in k3s with promtail chart"
slug = "" 
tags = ["loki", "rsyslog"]
categories = []
externalLink = ""
series = []
+++

# Loki Promtail

Enable syslog, do this on each host and replace `target` IP (and maybe `port`) with you syslog `externalIP` that is in helm values for [promtail](https://github.com/grafana/loki/tree/master/production/helm/promtail)

```yaml
promtail:
  serviceMonitor:
    enabled: true
  extraScrapeConfigs:
    - job_name: syslog
      syslog:
        listen_address: 0.0.0.0:1514
        label_structured_data: yes
        labels:
          job: "syslog"
      relabel_configs:
        - source_labels: ["__syslog_connection_ip_address"]
          target_label: "ip_address"
        - source_labels: ["__syslog_message_severity"]
          target_label: "severity"
        - source_labels: ["__syslog_message_facility"]
          target_label: "facility"
        - source_labels: ["__syslog_message_hostname"]
          target_label: "host"
  syslogService:
    enabled: true
    type: LoadBalancer
    port: 1514
    externalIPs:
        - 192.168.42.155
```

Create file `/etc/rsyslog.d/50-promtail.conf` with the following content:

```bash
module(load="omprog")
module(load="mmutf8fix")
action(type="mmutf8fix" replacementChar="?")
action(type="omfwd" protocol="tcp" target="192.168.42.155" port="1514" Template="RSYSLOG_SyslogProtocol23Format" TCP_Framing="octet-counted" KeepAlive="on")
```

Restart rsyslog and view status

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

In Grafana, on the explore tab, you should now be able to view you hosts logs, e.g. this query `{host="k3s-master"}`
