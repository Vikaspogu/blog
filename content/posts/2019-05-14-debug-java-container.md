+++
title="Debugging a Java application in OpenShift"
date="2019-05-14"
tags=["OpenShift", "Java"]
+++

This post will discuss debugging a JAVA application running inside a container. First, you must set the `JAVA_DEBUG` environment variable inside the container to `true` and configure port forwarding so that you can connect to your application from a remote debugger.

**Note:** If you are using `tomcat` image replace `JAVA_DEBUG` environment variable to `DEBUG`

Using the oc command, list the available deployment configurations:

```bash
$ oc get dc
```

Set the `JAVA_DEBUG` environment variable in the deployment configuration of your application to `true`, which configures the JVM to open the port number `5005` for debugging.

```bash
$ oc set env dc/MY_APP_NAME JAVA_DEBUG=true
```

**Note:** Disabling the health checks is not mandatory but it is recommended because a pod could be restarted while the process is paused during remote debugging. You can remove the readiness check to prevent an unintended restart.

Redeploy the application if it is not set to redeploy automatically on configuration change.

```bash
$ oc rollout latest dc/MY_APP_NAME
```

Configure port forwarding from your local machine to the application pod. List the currently running pods and find one containing your application. `$LOCAL_PORT_NUMBER` is an unused port number of your choice on your local machine. Remember this number for the remote debugger configuration.

**Note:** If you are using `tomcat` image replace the port `5005` to `8000`

```bash
$ oc get pod
NAME                            READY     STATUS      RESTARTS   AGE
MY_APP_NAME-3-1xrsp             1/1       Running     0          6s
...
$ oc port-forward MY_APP_NAME-3-1xrsp $LOCAL_PORT_NUMBER:5005
```

Create a new debug configuration for your application in `IntelliJ` IDE:

1. Click Run → Edit Configurations
2. In the list of configurations, add Remote. This creates a new remote debugging configuration
3. Enter a suitable name for the configuration in the name field
4. Set the port field to the port number that your application is listening on for debugging
5. Click Apply

   ![intellij-debug](/images/intellij_debug.png)

6. Click Run -> Debug -> Select Profile

   ![Debugger connected](/images/intellij_connect.png)

When you are done debugging, unset the `JAVA_DEBUG` environment variable in your application pod.

```bash
$ oc set env dc/MY_APP_NAME JAVA_DEBUG-
```
