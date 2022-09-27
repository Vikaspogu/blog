+++ 
date = 2020-08-18
title = "Configure Jenkins pipeline job"
description = "Configure bitbucket pipeline job using Jenkins config as code"
slug = "" 
tags = ["Jenkins", "configuration-as-code-plugin", "pipeline-job"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

In one of my previous posts, I discussed configuring multibranch pipeline seed jobs using Jenkins configuration as code [plugin](https://github.com/jenkinsci/configuration-as-code-plugin)

Below is an example of configuring a declarative pipeline job from a bitbucket repo, running at midnight every day

```yaml
- script: >
    pipelineJob('sample-job') {
      definition {
          cpsScm {
            scriptPath 'job1/Jenkinsfile' ## If the Jenkins job is in a nested folder
            scm {
              git {
                  remote {
                    url 'https://github.com/Vikaspogu/sample-repo'
                    credentials 'sample-creds'
                  }
                  branch '*/master'
                  extensions {}
              }
            }
            triggers {
              cron('@midnight')
            }
          }
      }
    }
```
