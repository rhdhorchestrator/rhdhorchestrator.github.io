---
title: Troubleshooting
date: "2024-07-25"
weight: 120
---


# Troubleshooting Guide

This document provides solutions to common problems encountered with serverless workflows.

## Table of Contents

1. [HTTP Errors](#http-errors)
2. [Workflow Errors](#workflow-errors)
3. [Configuration Problems](#configuration-problems)
4. [Workflow not showing in RHDH UI](#workflow-not-showing-in-rhdh-ui)

---

## HTTP Errors
Many workflow operations are REST requests to REST endpoints. If an HTTP error occurs then the workflow will fail and the HTTP code and message will be displayed. Here is an [example](https://github.com/rhdhorchestrator/rhdhorchestrator.github.io/blob/main/content/main/docs/serverless-workflows/409-error.png?raw=true) of the error in the UI.
Please use [HTTP codes documentation](https://developer.mozilla.org/docs/Web/HTTP/Status) for understanding the meaning of such errors.
Here are some examples:
 - `409`. Usually indicates that we are trying to update or create a resource that already exists. E.g. K8S/OCP resources.
 - `401`. Unauthorized access. A token, password or username might be wrong or expired.

 ## Workflow Errors

### Problem: Workflow execution fails

**Solution:**

1. Examine the container log of the workflow
    ```console
        oc logs my-workflow-xy73lj
    ```

### Problem: Workflow is not listed by the orchestrator plugin

**Solution:**

1. Examine the container status and logs
    ```console
        oc get pods my-workflow-xy73lj
        oc logs my-workflow-xy73lj
    ```

2. Most probably the Data index service was unready when the workflow started.
   Typically this is what the log shows:
    ```console
        2024-07-24 21:10:20,837 ERROR [org.kie.kog.eve.pro.ReactiveMessagingEventPublisher] (main) Error while creating event to topic kogito-processdefinitions-events for event ProcessDefinitionDataEvent {specVersion=1.0, id='586e5273-33b9-4e90-8df6-76b972575b57', source=http://mtaanalysis.default/MTAAnalysis, type='ProcessDefinitionEvent', time=2024-07-24T21:10:20.658694165Z, subject='null', dataContentType='application/json', dataSchema=null, data=org.kie.kogito.event.process.ProcessDefinitionEventBody@7de147e9, kogitoProcessInstanceId='null', kogitoRootProcessInstanceId='null', kogitoProcessId='MTAAnalysis', kogitoRootProcessId='null', kogitoAddons='null', kogitoIdentity='null', extensionAttributes={kogitoprocid=MTAAnalysis}}: java.util.concurrent.CompletionException: io.netty.channel.AbstractChannel$AnnotatedConnectException: Connection refused: sonataflow-platform-data-index-service.default/10.96.15.153:80
    ```

3. Check if you use a cluster-wide platform:
    ```console
       $ oc get sonataflowclusterplatforms.sonataflow.org
       cluster-platform
    ```
    If you have, like in the example output, then use the namespace `sonataflow-infra` when you look for the sonataflow services

    Make sure the Data Index is ready, and restart the workflow - notice the `sonataflow-infra` namespace usage:
    ```console
        $ oc get pods -l sonataflow.org/service=sonataflow-platform-data-index-service -n sonataflow-infra
        NAME                                                      READY   STATUS    RESTARTS   AGE
        sonataflow-platform-data-index-service-546f59f89f-b7548   1/1     Running   0          11kh

        $ oc rollout restart deployment my-workflow
    ```

### Problem: Workflow is failing to reach an HTTPS endpoint because it can't verify it

- REST actions performed by the workflow can fail the SSL certificate check if the target endpoint is signed with
a CA which is not available to the workflow. The error in the workflow pod log usually looks like this:

    ```console
        sun.security.provider.certpath.SunCertPathBuilderException - unable to find valid certification path to requested target
    ```

**Solution:**

1. If this happens then we need to load the additional CA cert into the running
   workflow container. To do so, please follow this guile from the SonataFlow guides site:
   https://sonataflow.org/serverlessworkflow/main/cloud/operator/add-custom-ca-to-a-workflow-pod.html


## Configuration Problems

### Problem: Workflow installed in a different namespace than Sonataflow services fails to start

**Solution:**
When deploying a workflow in a namespace other than the one where Sonataflow services are running (e.g., `sonataflow-infra`), there are essential steps to follow to enable persistence and connectivity for the workflow. See the following [steps](https://github.com/rhdhorchestrator/orchestrator-go-operator/blob/main/docs/release-1.6/README.md#additional-workflow-namespaces).

### Problem: sonataflow-platform-data-index-service pods can't connect to the database on startup

1. **Ensure PostgreSQL Pod has Fully Started**\
If the PostgreSQL pod is still initializing, allow additional time for it to become
fully operational before expecting the `DataIndex` and `JobService` pods to connect.
1. **Verify network policies if PostgreSQL Server is in a different namespace**\
If PostgreSQL Server is deployed in a separate namespace from Sonataflow services
(e.g., not in `sonataflow-infra` namespace), ensure that network policies in the
PostgreSQL namespace allow ingress from the Sonataflow services namespace
(e.g., `sonataflow-infra`). Without appropriate ingress rules,
network policies may prevent the `DataIndex` and `JobService` pods from
connecting to the database.

## Workflow not showing in RHDH UI

### Problem: Workflows are not showing up the in the RHDH Orchestrator UI
1. **Ensure the workflow uses `gitops` profile**\
In the RHDH Orchestrator UI, only the workflows using `gitops` profile are shown. Make sure the workflow definition and the `sonataflow` manifests are using this profile.

2. **Ensure the workflow's pod has started and is ready**\
THe first thing a workflow does when it starts is to create a schema for itself in the database (given persistence is enabled) and then it register itself to the Data Index. 
Until it was able to successfully register to the Data Index, the workflow's pod will not be ready.

3. **Ensure the workflow's pod can reach the Data Index**\
Connect to the workflow's pod and try to sent the following request to the Data Index:
```
curl -g -k  -X POST  -H "Content-Type: application/json" \
                    -d '{"query":"query{ ProcessDefinitions  { id, serviceUrl, endpoint } }"}' \
                    http://sonataflow-platform-data-index-service.sonataflow-infra/graphql
```
Use the service of the Data Index and its namespace as defined in your environment. 
Here `sonataflow-platform-data-index-service` is the service name and `sonataflow-infra` the namespace in which it is deployed.

Do the same from the RHDH pod and also make sure the workflow is reachable:
```
curl http://<workflow-service>.<workflow-namespace>/management/processes
```

4. **Ensure the Orchestrator is trying to fetch the workflow**\
In the logs of the RHDH pod, you should see logs message similar to
```
{"level":"\u001b[32minfo\u001b[39m","message":"fetchWorkflowInfos() called: http://sonataflow-platform-data-index-service.sonataflow-infra","plugin":"orchestrator","service":"backstage","span_id":"fca4ab29f0a7aef9","timestamp":"2025-08-04 17:58:26","trace_flags":"01","trace_id":"5408d4b06373ff8fb34769083ef771dd"}
```
Notice the `"plugin":"orchestrator"` that can help filtering the messages.

5. **Ensure the Data Index properties are set in the `-managed-props` ConfigMap of the workflow**\
Make sure to have:
```
kogito.data-index.health-enabled = true
kogito.data-index.url = http://sonataflow-platform-data-index-service.sonataflow-infra
...
mp.messaging.outgoing.kogito-processdefinitions-events.url = http://sonataflow-platform-data-index-service.sonataflow-infra/definitions
mp.messaging.outgoing.kogito-processinstances-events.url = http://sonataflow-platform-data-index-service.sonataflow-infra/processes
```
Those should be set automatically by the OSL operator when the Data Index service is enabled. You should have simlilar properties for the Job Services.

6. **Ensure the Workflow is registered in the Data Index**\
To check that, you may connect to the database used by the Data Index and run the following from the PSQL instance's pod:
```
$ PGPASSWORD=<psql password> psql -h localhost -p 5432 -U < user> -d sonataflow

sonataflow=# SET search_path TO "sonataflow-platform-data-index-service";
sonataflow=# select id, name from definitions;

```
You should see the workflows registered to the Data Index

7. **Ensure Data Index and Job Services are enabled**\
If the Data Index and the Job Services are not enabled in the `SontaFlowPlatform` then the Orchestrator plugin cannot fetch the available workflows. 
Make sure to have
```
services:
    dataIndex:
      enabled: true
      ...
    jobService:
      enabled: true
      ...
``` 
If not, manually edit the `SontaFlowPlatform` instance. This should trigger the re-creation of the workflow's related manifests. 

You should now make sure the properties are correctly set in the `managed-props` ConfigMap of the workflow.

8. **Ensure the RBAC permissions are set correctly**\
See [RBAC documentation](../installation/rbac.md) for detailed permission configuration.

To see if there is a permission issue, you have to set the log level to DEBUG, see https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.6/html/monitoring_and_logging/assembly-monitoring-and-logging-with-aws_assembly-rhdh-observability#configuring-the-application-log-level-by-using-the-operator_assembly-rhdh-observability