+++ 
date = 2020-07-06
title = "OpenShift Jenkins configuration via JCasC plugin"
description = "Configuring OpenShift jenkins via jenkins configuration as code plugin"
slug = "" 
tags = ["jenkins","OpenShift","jcasc"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

Deploying Jenkins on Kubernetes provides significant benefits over a standard VM-based deployment. For example, we are gaining the ability to have project-specific Jenkins slaves (agents) on demand instead of having a pool of VMs idle waiting for a job.

In vanilla Kubernetes, we can deploy Jenkins using Helm, and In OpenShift, we can deploy Jenkins via the developer's catalog.

As everyone has experienced, setting up Jenkins is a complex process. Both Jenkins and its plugins require some tuning and Configuration, with dozens of parameters to set within the web UI manage section.

## Jenkins Config as code

[Configuration as code](https://github.com/jenkinsci/configuration-as-code-plugin) gives you an _opinionated_ way to configure Jenkins based on `yaml` files

This post will cover Jenkins configuration code on OpenShift.

- Mount Jenkins config as a configmap and load configuration
- Updated configuration file to automate the creation of
  - credentials
  - script approval signatures
  - shared libraries
  - multibranch pipeline seed jobs

### Install Plugin

First, let us install `configuration-as-code` plugins in Jenkins. In OpenShift, you can easily install plugins by adding the `INSTALL_PLUGINS` environment variable to the deployment config

```yaml
....
env:
  - name: INSTALL_PLUGINS
    value: "configuration-as-code:1.35,configuration-as-code-support:1.18,configuration-as-code-groovy:1.1
```

### Create ConfigMap

Create a configmap from jenkins-config yaml and mount it as a volume at `/var/jenkins_config` location. Configuration can be now loaded from `/var/jenkins_config/jenkins-config.yaml` path

```bash
oc create configmap jenkins-config --from-file jenkins-config.yaml
```

```yaml
....
    volumeMounts:
      - mountPath: /var/jenkins_config
        name: jenkins-config
dnsPolicy: ClusterFirst
restartPolicy: Always
volumes:
  - name: jenkins-config
    configMap:
      name: jenkins-config
```

### Mount Secret

Since we can have Jenkins configuration in the git repo, we don't want to hardcode the secrets as it is a security risk. The easiest way to automate credentials in Jenkins configuration is to create an OpenShift secret, and add that secret as an environment variable to the deployment config

```bash
oc create secret generic github-ssh --from-file=ssh-privatekey=github-ssh/
oc create secret generic jenkins-credentials --from-literal=OCP_SA="qwerty26sds99ie9kcsd"  --from-literal=GITHUB_PASSWORD="dummypassword"
```

Mount secrets into the container

```yaml
....
env:
....
- name: OCP_SA
  valueFrom:
    secretKeyRef:
      name: jenkins-credentials
      key: OCP_SA
- name: GITHUB_SSH
  valueFrom:
    secretKeyRef:
      name: github-ssh
      key: ssh-privatekey
- name: GITHUB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: jenkins-credentials
      key: GITHUB_PASSWORD
....
```

```yaml
credentials:
  system:
    domainCredentials:
      - credentials:
          - fileSystemServiceAccountCredential:
              id: "1a12dfa4-7fc5-47a7-aa17-cc56572a41c7"
              scope: GLOBAL
          - basicSSHUserPrivateKey:
              description: "github ssh credentials"
              id: "github-ssh"
              privateKeySource:
                directEntry:
                  privateKey: ${GITHUB_SSH}
              scope: GLOBAL
              username: "github-user"
          - OpenShiftToken:
              id: "jenkins-ocp-token"
              scope: GLOBAL
              secret: ${OCP_SA}
          - usernamePassword:
              description: "github user credentials"
              id: "github-user"
              password: ${GITHUB_PASSWORD}
              scope: GLOBAL
              username: "github-user"
```

### Example Configs

#### Security script approval

```yaml
security:
  scriptApproval:
    approvedSignatures:
      - "method java.text.DateFormat parse java.lang.String"
```

#### Global shared library

```yaml
unclassified:
  globalLibraries:
    libraries:
      - defaultVersion: "master"
        name: "shared-pipeline"
        retriever:
          modernSCM:
            scm:
              git:
                credentialsId: "github-ssh"
                id: "shared-pipeline"
                remote: "git@github.com:vikaspogu/jenkins-shared.git"
```

#### Multibranch job

Multibranch seed job with periodic polling, traits for branch discovery

```yaml
jobs:
  - script: >
      multibranchPipelineJob("sample-repo") {
        branchSources {
          branchSource {
            source {
              github {
                //This is a unique identifier; if not set, the pipeline will be 
                //indexed, and jobs will be kicked off every time new config is applied
                id("5bb970c2-766b-4588-8cf8-e077bfec23a0")
                credentialsId("github-user")
                repoOwner("vikaspogu")
                repository("sample-repo")
                traits {
                  cloneOptionTrait {
                    extension {
                      shallow (false)
                      noTags (false)
                      reference (null)
                      depth(1)
                      honorRefspec (false)
                      timeout (10)
                    }
                  }
                }
              }
            }
          }
        }
        configure {
          def traits = it / sources / data / 'jenkins.branch.BranchSource' / source / traits
          traits << 'com.cloudbees.jenkins.plugins.bitbucket.BranchDiscoveryTrait' {
            strategyId(3) // detect all branches -refer the plugin source code for various options
          }
        }
        configure {
          def traits = it / sources / data / 'jenkins.branch.BranchSource' / source / traits
          traits << 'com.cloudbees.jenkins.plugins.bitbucket.OriginPullRequestDiscoveryTrait' {
            strategyId(2)
          }
        }
        configure {
          def traits = it / sources / data / 'jenkins.branch.BranchSource' / source / traits
          traits << 'com.cloudbees.jenkins.plugins.bitbucket.TagDiscoveryTrait' {}
        }
        orphanedItemStrategy {
          discardOldItems {
            numToKeep(15)
          }
        }
        triggers {
          periodic(2)
        }
      }
```
