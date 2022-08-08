+++ 
date = 2020-08-18
title = "Configure jenkins pipeline job"
description = "Configure bitbucket pipeline job using jenkins config as code"
slug = "" 
tags = ["jenkins", "configuration-as-code-plugin", "pipeline-job"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

In one of my previous posts, I have discussed configuring multibranch pipeline seed jobs using Jenkins configuration as code [plugin](https://github.com/jenkinsci/configuration-as-code-plugin)

This is an example of configuring a declarative pipeline job from bitbucket repo, running at midnight every day

```yaml
- script: >
    pipelineJob('sample-job') {
      definition {
          cpsScm {
            scriptPath 'job1/Jenkinsfile' ## If jenkins job is in a nested folder
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
