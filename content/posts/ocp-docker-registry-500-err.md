+++ 
date = 2019-12-18
title = "Permission denied pushing to Openshift Registry"
description = "Permission denied on pushing build images into openshift internal registry"
slug = "" 
tags = ["Openshift"]
categories = []
externalLink = ""
series = []
+++

Recently, i ran into a issue where pushing images to the docker registry after a build fails

```bash
Pushing image docker-registry.default.svc:5000/simple-go-build/simple-go:latest ...
Registry server Address:
Registry server User Name: serviceaccount
Registry server Email: serviceaccount@example.org
Registry server Password: <<non-empty>>
error: build error: Failed to push image: received unexpected HTTP status: 500 Internal Server Error
```

Registry pods logs show `permission denied`

```bash
err.code=UNKNOWN err.detail="filesystem: mkdir /registry/docker/registry/v2/repositories/simple-go-build/simple-go/_uploads/c34415b4-c6d8-42ba-9854-aee449efd984: permission denied"
```

One of the Red Hat solutions article suggested to verify the file ownership of the files, directories in the volume and compare it to the uid of the registry

Changing the owner recursively to the uid of the registry, fixed the issue

```bash
root@master# chown -R 1001 /exports/registry/docker/
```
