+++ 
date = 2021-01-19T13:42:07-06:00
title = "Tekton triggers and Interceptors"
description = ""
slug = "" 
tags = ["tekton", "triggers", "cel"]
categories = []
externalLink = ""
series = []
+++

Tekton [Triggers](https://tekton.dev/docs/triggers/) work by having EventListeners receive incoming webhook notifications, processing them using an Interceptor, and creating Kubernetes resources from templates if the interceptor allows it, with extraction of fields from the body of the webhook

[CEL Interceptors](https://tekton.dev/docs/triggers/eventlisteners/#cel-interceptors) can be used to filter or modify incoming events. for example if you want to truncate commit id from the webhook body

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

Applied overlay can be extracted using a extensions body in the binding. In below example `$(extensions.<overlay_key>)`

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