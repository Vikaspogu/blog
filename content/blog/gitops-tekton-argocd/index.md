+++ 
date = 2021-01-14
title = "GitOps with Tekton and ArgoCD"
description = "build and release pipeline with tekton, argocd in gitops fashion"
slug = "tekton-argocd-gitops" 
tags = ["argocd", "tekton", "gitops", "OpenShift-pipelines"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

In this post, I will:

- Use Tekton to build and publish image to docker registry
- Trigger a Tekton pipeline from GitHub
- Use Argo to deploy application

![alt argo-flow-1](gitops-architecture.png)

## Getting Started

_**Assumptions:**_

- You have a running OpenShift 4 cluster with ArgoCD, and OpenShift Pipelines installed. If not, you follow instructions to [OpenShift Pipelines](https://docs.OpenShift.com/container-platform/4.6/pipelines/installing-pipelines.html) and [ArgoCD Operator](https://argocd-operator.readthedocs.io/en/latest/install/OpenShift/)
- Understand of basic argocd and tekton concepts

Create an OpenShift project called node-web-project

```bash
oc new-project node-web-project
```

### Using Private Registries

Create a docker secret with registry authentication details

```bash
oc create secret docker-registry container-registry-secret \
  --docker-server=$CONTAINER_REGISTRY_SERVER \
  --docker-username=$CONTAINER_REGISTRY_USER \
  --docker-password=$CONTAINER_REGISTRY_PASSWORD -n node-web-project
```

Create a service account named `build-bot`

```bash
oc create sa -n node-web-project build-bot
serviceaccount/build-bot created
```

Add docker secret `container-registry-secret` to newly created service account `build-bot`

```bash
oc patch serviceaccount build-bot \
  -p '{"secrets": [{"name": "container-registry-secret"}]}'
serviceaccount/build-bot patched
```

Verify if the service account has the secret added:

```bash
oc get sa -n node-web-project build-bot -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-01-12T03:34:32Z"
  name: build-bot
  namespace: node-web-project
  resourceVersion: "53879"
  selfLink: /api/v1/namespaces/node-web-project/serviceaccounts/build-bot
  uid: 628067fd-91d1-4cdd-b6a6-88b4f7280ff0
secrets:
- name: container-registry-secret
- name: build-bot-token-8nl2v
```

### Create Pipeline

A Pipeline is a collection of Tasks that you want to run as part of your workflow. Each Task in a Pipeline is executed in a pod and by default they run in parallel. However, you can specify the order by using `runAfter`

There are 3 Tasks in below pipeline

- Clone source code
- Build and push image using `Buildah` cluster task
- Syncing argo deployment

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: node-web-app-pipeline
  namespace: node-web-project
spec:
  workspaces:
    - name: shared-workspace
  params:
    - name: node-web-app-source
    - name: node-web-app-image
  tasks:
    - name: fetch-repository
      taskRef:
        kind: ClusterTask
        name: git-clone
      workspaces:
      - name: output
        workspace: shared-workspace
      params:
      - name: url
        value: $(params.node-web-app-source)
    - name: build-and-publish-image
      params:
        - name: IMAGE
          value: $(params.node-web-app-image)
        - name: TLSVERIFY
          value: 'false'
      taskRef:
        kind: ClusterTask
        name: buildah
      runAfter:
        - fetch-repository
      workspaces:
      - name: source
        workspace: shared-workspace
    - name: sync-application
      taskRef:
        name: argocd-task-sync-and-wait
      params:
        - name: application-name
          value: node-web-app
        - name: flags
          value: --insecure --grpc-web
        - name: argocd-version
          value: v1.7.11
      runAfter:
        - build-and-publish-image
```

### Triggers

Now that the pipeline is setup, we need to setup webhook from GitHub to trigger pipeline. We need to create following resources:

- A `TriggerTemplate` act as a blueprint for pipeline resources. It can also be used to define parameters that can then be substituted anywhere within the resource template(s) being defined

  ```yaml
  apiVersion: triggers.tekton.dev/v1alpha1
  kind: TriggerTemplate
  metadata:
    name: pipeline-template
    namespace: node-web-project
  spec:
    params:
      - name: git-repo-url
      - name: git-repo-name
      - name: git-revision
      - name: image-name
    resourcetemplates:
      - apiVersion: tekton.dev/v1beta1
          kind: PipelineRun
          metadata:
          generateName: $(tt.params.git-repo-name)-pipeline-run-
          namespace: node-web-project
          spec:
          params:
          - name: source
              value: $(tt.params.git-repo-url)
          - name: image
              value: $(tt.params.image-name)
          pipelineRef:
              name: node-web-app-pipeline
          serviceAccountName: build-bot
          timeout: 1h0m0s
          workspaces:
          - name: shared-workspace
              persistentVolumeClaim:
              claimName: tekton-workspace-pvc
  ```

- A `TriggerBinding` binds the incoming event data to the template (i.e. git url, repo name, revision, etc....)

  ```yaml
  apiVersion: triggers.tekton.dev/v1alpha1
  kind: TriggerBinding
  metadata:
    name: pipeline-binding
    namespace: node-web-project
  spec:
    params:
      - name: git-repo-name
        value: $(body.repository.name)
      - name: git-repo-url
        value: $(body.repository.url)
      - name: git-revision
        value: $(body.head_commit.id)
      - name: image-name
        value: docker.io/vikaspogu/$(body.repository.name):$(body.head_commit.id)
  ```

- An `EventListener` that will create a pod application bringing together a binding and a template

  ```yaml
  apiVersion: triggers.tekton.dev/v1alpha1
  kind: EventListener
  metadata:
    name: el-node-web-app
    namespace: default
  spec:
    serviceAccountName: build-bot
    triggers:
      - name: node-web-app-trigger
        bindings:
          - ref: pipeline-binding
        template:
          ref: pipeline-template
  ```
  
- An OpenShift Route to expose the Event Listener

  ```bash
  $ oc expose svc el-node-web-app
  $ echo "URL: $(oc get route el-node-web-app --template='http://{{.spec.host}}')"
  http://el-node-web-app-node-web-project.apps.cluster-7d51.sandbox659.opentlc.com
  ```

You can learn about [Tekton Triggers](https://tekton.dev/docs/triggers/) and [OpenShift Pipelines](https://docs.OpenShift.com/container-platform/4.6/pipelines/understanding-OpenShift-pipelines.html)

### Create ArgoCD App for Web App Resources

[Create an ArgoCD application via the GUI or command line](https://argoproj.github.io/argo-cd/getting_started/)

![alt argo-pipeline](argo-node-web-app.png)

- Project: default
- cluster: (URL Of your OpenShift Cluster)
- namespace should be the name of your OpenShift Project
- repo url: git repo
- Target Revision: Head
- PATH: deployment
- AutoSync Disabled

### Configure Webhooks

You will now need to configure GitHub webHook to Tekton Event Listener to start a tekton build from git push

![alt webhooks](tekton-webhook.png)

### Make a code change and commit, look at build

1. Push an empty commit to repo

    ```bash
    git commit -m "empty-commit" --allow-empty && git push
    ```

2. In OpenShift Console you should see a pipeline run

    ![alt webhooks](gittrigger.png)

3. Once pipeline is finished. Use OpenShift route to verify app

## Workspace PVC

To use same PVC wit multiple pipelines, a `ReadWriteMany` pvc has to be created

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tekton-workspace-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```
