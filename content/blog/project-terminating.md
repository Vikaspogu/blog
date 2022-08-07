+++
title= "Deleting a OpenShift project stuck in terminating state"
date= "2019-05-15"
tags= ["OpenShift"]
+++

Recently I was faced an issue where one of my project was stuck in terminating state for days. The workaround below fixed the issue.

Export OpenShift project as an JSON Object

```bash
oc get project delete-me -o json > ns-without-finalizers.json
```

Replace below from

```yaml
spec:
  finalizers:
    - kubernetes
```

to

```yaml
spec:
  finalizers: []
```

On one of the master node, execute these commands.

```bash
kubectl proxy & PID=$!
curl -X PUT http://localhost:8001/api/v1/namespaces/delete-me/finalize \
-H "Content-Type: application/json" --data-binary @ns-without-finalizers.json
kill $PID
```
