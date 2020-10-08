+++ 
date = 2020-10-08T09:17:20-05:00
title = "Helm join strings in a named template"
description = "How to join strings in a named template"
slug = "" 
tags = []
categories = ["helm"]
externalLink = ""
series = []
+++

Key benefits of Helm is that it helps reduce the amount of configuration a user needs to provide to deploy applications to Kubernetes. With Helm, we can have a single chart that can deploy all the microservices. 

### Problem:

Create a unique service account for each microservice, so all microservices don't share same service account. In our example, we will append the release name with the service account name.

* Using join function

```
{{ (list .Values.serviceAccount.name (include "openjdk.fullname" .) | join "-") }} 
```

* Adding fail condition

```
{{- if .Values.serviceAccount.name -}} 
  {{ (list .Values.serviceAccount.name (include "openjdk.fullname" .) | join "-") }} 
{{- else -}} 
  {{- fail "Please enter a name for service account to create." }} 
{{- end -}}
```

Full example

```
{{- define "openjdk.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}} 
  {{- if .Values.serviceAccount.name -}} 
    {{ (list .Values.serviceAccount.name (include "openjdk.fullname" .) | join "-") }} 
  {{- else -}} 
    {{- fail "Please enter a name for service account to create." }} 
  {{- end -}}
{{- else -}} 
  {{- if .Values.serviceAccount.name -}} 
    {{ (list .Values.serviceAccount.name (include "openjdk.fullname" .) | join "-") }} 
  {{- else -}} 
    "default" 
  {{- end -}}
{{- end -}}
{{- end -}}
```

## Resources

- [Sprig functions](https://github.com/Masterminds/sprig)
