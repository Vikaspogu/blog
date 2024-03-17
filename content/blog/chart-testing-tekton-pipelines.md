+++
title = "Helm Chart Testing on OpenShift Pipelines"
date = 2024-03-17T08:39:20-04:00
draft = false
+++

## Introduction

OpenShift Pipelines is a cloud-native CI/CD solution built on Tekton that allows you to define and run pipelines for your applications. Helm charts are a convenient way to package and deploy applications on Kubernetes, including OpenShift. To ensure the quality and reliability of Helm charts, it's important to perform tests on them. In this blog, we will explore using the [Chart Testing](https://github.com/helm/chart-testing/tree/main) tool to perform tests on Helm charts within OpenShift Pipelines.

### Prerequisites

Before we begin, make sure you have the following prerequisites installed:

- OpenShift Cluster
- OpenShift Pipelines Operator installed
- Understanding of OpenShift Pipelines and Tasks

## Chart Testing Tasks

> `ct` is the the tool for testing Helm charts. It is meant to be used for linting and testing pull requests. It automatically detects charts changed against the target branch.

The `ct install` command includes `helm install` and `helm test`; `helm test` performs port-forwarding to validate the installation's success. Since the chart testing commands are run inside a pod, we need to ensure it can connect to the cluster to perform those tasks. If the pod cannot connect to the cluster, you'll see the error below.

```bash
The connection to the server localhost:8080 was refused - did you specify the right host or port?
Error printing details: failed waiting for process: exit status 1
Error printing logs: failed running process: exit status 1
Deleting release "deploy-jgyw6ew05y"...
release "deploy-jgyw6ew05y" uninstalled
------------------------------------------------------------------------------------------------------------------------
 ✖︎ deploy => (version: "0.1.0", path: "deploy") > failed running process: exit status 1
------------------------------------------------------------------------------------------------------------------------
failed installing charts: failed processing charts
Error: failed installing charts: failed processing charts
```

To solve the above error, we'll follow below steps:
- Create a new service account and secret token from the service account.
- Grant appropriate permissions to the service account.
- Create a kubeconfig file using the kubeconfig creator task. By Mounting the service account token as an environment variable from the secret in the task
- This task's output will be the kubeconfig file stored in the workspace and shared across tasks.

Create a new service account

```bash
oc create sa ct-helm-task
```

Create a [token](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-long-lived-api-token-for-a-serviceaccount) secret from serviceaccount, 

```yaml
oc apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: ct-helm-task
  annotations:
    kubernetes.io/service-account.name: ct-helm-task
type: kubernetes.io/service-account-token
EOF
```

Grant permissions to the service account

```bash
oc policy add-role-to-user edit -z ct-helm-task -n pipelines-helm
```

The task will use the `kubeconfigwriter` image and the provided parameters to create a `kubeconfig` file that can be used by other tasks in the pipeline to access the target cluster. The kubeconfig will be placed at `/workspace/<workspace-name>/kubeconfig`.

{{< highlight yaml "linenos=table,hl_lines=39 45-50" >}}
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  annotations:
    tekton.dev/categories: 'Deployment, Kubernetes'
    tekton.dev/displayName: kubeconfig creator
    tekton.dev/pipelines.minVersion: 0.12.1
    tekton.dev/platforms: 'linux/amd64,linux/s390x,linux/ppc64le'
    tekton.dev/tags: deploy
  name: kubeconfig-creator
spec:
  params:
    - default: ''
      description: name of the cluster
      name: name
      type: string
    - description: address of the cluster
      name: url
      type: string
    - default: '<CLUSTER_URL>'
      description: bearer token for authentication to the cluster
      name: token
      type: string
    - default: 'true'
      description: >-
        to indicate server should be accessed without verifying the TLS
        certificate
      name: insecure
      type: string
  steps:
    - args:
        - '-clusterConfig'
        - >-
          { "name":"$(params.name)", "url":"$(params.url)",
          "username":"$(params.username)", "password":"$(params.password)",
          "cadata":"$(params.cadata)",
          "clientKeyData":"$(params.clientKeyData)",
          "clientCertificateData":"$(params.clientCertificateData)",
          "namespace":"$(params.namespace)", "token":"$(SA_TOKEN)",
          "Insecure":$(params.insecure) }
        - '-destinationDir'
        - $(workspaces.output.path)
      command:
        - /ko-app/kubeconfigwriter
      env:
        - name: SA_TOKEN
          valueFrom:
            secretKeyRef:
              key: token
              name: ct-helm-task
      image: >-
        gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/kubeconfigwriter@sha256:b2c6d0962cda88fb3095128b6202da9b0e6c9c0df3ef6cf7863505ffd25072fd
      name: write
      resources: {}
  workspaces:
    - name: output
{{< /highlight >}}

Finally, create a task using the `chart-testing` image. We'll export the `KUBECONFIG` variable with the kubeconfig location from the workspace to the helm chart testing task.


{{< highlight yaml "linenos=table,hl_lines=38" >}}
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: helm-ci-task
spec:
  description: >-
    This task can be used to replace fields in YAML files. For example for altering helm charts on GitOps repos.
  workspaces:
    - name: source
      description: A workspace that contains the file which needs to be altered.
  params:
    - name: image
      type: string
      description: The chart testing image to use.
      default: quay.io/helmpack/chart-testing:v3.7.1
    - name: chart-dirs
      type: string
      description: directory of the charts
      default: "charts/"
    - name: charts
      type: string
      description: name of the chart to test
      default: ""
    - name: namespace
      type: string
      description: namespace to run ci tests
      default: "helm-tekton-pipeline"
    - name: ct-config-file
      type: string
      description: config file for running ct commands
      default: "ct.yaml"
  steps:
    - name: ct-script
      image: quay.io/helmpack/chart-testing:v3.5.1
      workingDir: $(workspaces.source.path)
      script: |
        ## export kubeconfig file from previous step
        export KUBECONFIG="$(workspaces.source.path)/kubeconfig"
        ct lint-and-install --chart-dirs $(params.chart-dirs) --charts $(params.charts) --namespace $(params.namespace) --config $(params.ct-config-file)
{{< /highlight >}}

## Setting Up OpenShift Pipeline

Create a pipeline resource to perform `git-clone` of the helm chart repository, then create a `kubeconfig` file and `chart-testing` task to run lint and install to validate the chart changes.

{{< highlight yaml "linenos=table,hl_lines=33-44" >}}
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: helm-ct-pipeline
spec:
  workspaces:
  - name: shared-workspace
  params:
  - name: git-url
    type: string
    description: url of the git repo for the code of deployment
  - name: git-revision
    type: string
    description: revision to be used from repo of the code for deployment
    default: master
  tasks:
  - name: fetch-repository
    taskRef:
      name: git-clone
      kind: ClusterTask
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.git-url)
    - name: subdirectory
      value: ""
    - name: deleteExisting
      value: "true"
    - name: revision
      value: $(params.git-revision)
  - name: kubeconfig-creator
    params:
      - name: url
        value: 'https://api.cluster.example.com:6443'
    runAfter:
      - fetch-repository
    taskRef:
      kind: Task
      name: kubeconfig-creator
    workspaces:
      - name: output
        workspace: shared-workspace
  - name: run-ct-test
    taskRef:
      name: helm-ci-task
      kind: Task
    params:
    - name: charts
      value: deploy
    workspaces:
    - name: source
      workspace: shared-workspace
    runAfter:
    - kubeconfig-creator
{{< /highlight >}}

## Conclusion

In this blog, we have demonstrated how to perform tests on Helm charts using the Chart Testing tool within OpenShift Pipelines. By setting up a pipeline with tasks for linting and testing, you can ensure the quality and reliability of your Helm charts before deploying them to your OpenShift cluster.
