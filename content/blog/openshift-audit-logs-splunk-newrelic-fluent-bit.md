---
title: "Forward Openshift Audit Logs to Splunk using Newrelic Fluent Bit"
date: "2023-01-11T12:52:13-06:00"
comments: false
socialShare: true
toc: false
---

It is common for organizations to send logs to multiple systems—for example, container logs to NewRelic, and audit logs to Splunk for the InfoSec team. 

Deploying [NewRelic](https://github.com/newrelic/helm-charts/tree/master/charts/nri-bundle) using helm chart will also deploy the NewRelic logging deamonset, which utilizes a custom Fluent Bit image with a NewRelic output [plugin](https://github.com/newrelic/newrelic-fluent-bit-output) to forward logs easily. Since the NewRelic Fluent Bit image uses an upstream fluent bit as a [base](https://github.com/newrelic/newrelic-fluent-bit-output/blob/master/Dockerfile#L19) image, it is pre-baked with the Splunk [plugin](https://github.com/fluent/fluent-bit#output-plugins).

Now, let's look at how to configure both NewRelic for container logs and Splunk for audit logs using the helm chart.

Three things have to be configured in helm chart values

* Configure [tail](https://docs.fluentbit.io/manual/pipeline/inputs/tail) input for audit log paths
* Update the `Match` tag in the Kubernetes [filter](https://docs.fluentbit.io/manual/pipeline/filters/kubernetes) to match all records that start with `audit.*`
* Configure Splunk [output](https://docs.fluentbit.io/manual/pipeline/outputs/splunk) with HEC token and host details

### Configure audit input

Input Plugins gather information from different sources; some collect data from log files. `tail` input plugin allows monitoring one or several text files. It has similar behavior to the `tail -f` shell command.

In this case, we want to tail `/var/log/audit/audit.log`; to configure new input in NewRelic logging. Add `tail` input for `audit` logs in `values.yaml` file under the `fluentBit` config -> inputs section, as shown below.

```yaml
fluentBit:
  config:
    inputs: |
      [INPUT]
          Name              tail
          Tag               kube.*
          Path              ${PATH}
          Parser            ${LOG_PARSER}
          DB                ${FB_DB}
          Mem_Buf_Limit     7MB
          Skip_Long_Lines   On
          Refresh_Interval  10
      [INPUT]
          Name              tail
          Tag               audit.*
          Path              "/var/log/audit/audit.log"
          Parser            ${LOG_PARSER}
          DB                ${FB_DB}
          Mem_Buf_Limit     7MB
          Skip_Long_Lines   On
          Refresh_Interval  10
```

### Update Match rule

A `Match` represents a simple rule to select Events whose Tags match a defined rule.

In this scenario, we are checking the `audit.*` tags. Only one unique filter is allowed; we'll append it to the existing `kube.*` tag.

```yaml
fluentBit:
  config:
      [FILTER]
          Name                kubernetes
          Match               kube.* audit.*
          Kube_URL            https://kubernetes.default.svc.cluster.local:443
          Buffer_Size         ${K8S_BUFFER_SIZE}
          K8S-Logging.Exclude ${K8S_LOGGING_EXCLUDE}
      [FILTER]
          Name           record_modifier
          Match          *
          Record         cluster_name ${CLUSTER_NAME}
```

### Configure Splunk output

The `output` interface allows the definition of destinations for the data. For example, our destination for audit logs is `Splunk`.

To configure new output in NewRelic logging. Add `splunk` output for `audit.*` logs in the `values.yaml` file under the `fluentBit` config -> outputs section, as shown below.

```yaml
fluentBit:
  config:
    outputs: |
      [OUTPUT]
          Name           newrelic
          Match          *
          licenseKey     ${LICENSE_KEY}
          endpoint       ${ENDPOINT}
          lowDataMode    ${LOW_DATA_MODE}
          Retry_Limit    ${RETRY_LIMIT}
      [OUTPUT]
          Name           splunk
          Match          audit.*
          Host           splunk-hec.host
          Port           443
          TLS            On
          Splunk_Token   ${SPLUNK_TOKEN} #Create a secret with Splunk token and mount into container
```

Finally, recreate the logging pods. Audit logs should be routed to Splunk along with container logs to NewRelic.
