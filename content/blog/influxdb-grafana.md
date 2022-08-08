---
title: "Configure Influxdb with Grafana"
date: "2022-08-08T15:32:00-05:00"
comments: false
socialShare: true
toc: false
tags: ["influxdb", "grafana"]
---

I recently started using proxmox and I chose to monitor metrics using InfluxDB and Grafana which are already deployed on my Kubernetes cluster. During that process, I encountered two issues:

- Authentication
- "Database not found"

## Authentication

For the Authentication issue, I found out that you have to use the `Authorization` header with the `Token yourAuthToken` value to access the bucket. The header name is configured in the `jsonData` field and the header value should be configured in `secureJsonData` for [grafana helm chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana) values.

```yaml
secureJsonData:
  httpHeaderValue1: 'Token ${INFLUXDB_TOKEN}'
jsonData:
  httpHeaderName1: 'Authorization'
```

Complete `datasources` helm values

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    deleteDatasources:
      - name: Loki
        orgId: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-operated:9090
        access: proxy
        isDefault: true
      - name: InfluxDB
        type: influxdb
        url: "http://influxdb.default.svc.cluster.local:80"
        database: "default"
        secureJsonData:
          httpHeaderValue1: 'Token ${INFLUXDB_TOKEN}'
        access: proxy
        jsonData:
          httpHeaderName1: 'Authorization'
          httpMode: GET
```

## "Database not found"

This one took me a while to figure out. In InfluxDB v1, data is stored in databases and retention policies. In InfluxDB v2, data is stored in buckets. Because InfluxQL uses the v1 data model, before querying in InfluxQL, a bucket must be mapped to a database and retention policy. Find more information [here](https://docs.influxdata.com/influxdb/v2.0/query-data/influxql/?t=InfluxDB+API#map-unmapped-buckets).

I am using `InfluxDB API` method to manually create DBRP mappings for unmapped buckets as shown below.

```bash
curl --request POST http://localhost:8086/api/v2/dbrps \
  --header "Authorization: Token YourAuthToken" \
  --header 'Content-type: application/json' \
  --data '{
        "bucketID": "00oxo0oXx000x0Xo",
        "database": "example-db",
        "default": true,
        "orgID": "00oxo0oXx000x0Xo",
        "retention_policy": "example-rp"
      }'
```

Make sure to update Host, Token, bucketID, database, orgID, retention_policy with your values.

Lastly, go to Grafana, choose the InfluxDB data source, and try to run a query to see that the setting up process was successful.
