+++ 
date = 2021-01-19
title = "Tekton triggers and Interceptors"
description = ""
slug = "" 
tags = ["tekton", "triggers", "cel"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

Tekton [Triggers](https://tekton.dev/docs/triggers/) work by having EventListeners receive incoming webhook notifications, processing them using an Interceptor, and creating Kubernetes resources from templates if the interceptor allows it, with the extraction of fields from the body of the webhook

[CEL Interceptors](https://tekton.dev/docs/triggers/eventlisteners/#cel-interceptors) can filter or modify incoming events. For example, you can truncate the commit id from the webhook body.

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: shared-listener
  namespace: default
spec:
  serviceAccountName: build-bot
  triggers:
    - name: shared-pipeline-trigger
      interceptors:
        - cel:
            overlays:
            - key: intercepted.commit_id_short
              expression: "body.head_commit.id.truncate(7)"
      bindings:
        - ref: pipeline-binding
      template:
        ref: pipeline-template
```

The applied overlay uses an extension body in the binding. As shown in below example as `$(extensions.<overlay_key>)`

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: pipeline-binding
  namespace: default
spec:
  params:
  - name: git-repo-name
    value: $(body.repository.name)
  - name: git-repo-url
    value: $(body.repository.url)
  - name: image-tag
    value: $(extensions.intercepted.commit_id_short)
```
