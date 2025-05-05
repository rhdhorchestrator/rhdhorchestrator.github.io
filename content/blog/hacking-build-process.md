---
date: 2025-04-28
title: 'Hacking The Build and Deployment Process of Serverless Workflows'
---

# Hacking The Build and Deployment Process of Serverless Workflows

In this guide, we'll dive under the hood of the serverless workflow build process, examine how it works internally, and learn how to take control of it when needed.

## The Problem

We're working with a [repository](https://github.com/masayag/poc-kafka-logic-operator) that contains an issue with the standard workflow build script described in our [previous post](./building-and-deploying-workflows.md). When running the script, you'll encounter [an error](https://github.com/apache/incubator-kie-tools/issues/3084) - the manifests can't be generated due to an incompatible workflow spec, specifically with the `eventRef` element.

Interestingly, if you run the workflow using `mvn clean quarkus:dev`, it starts successfully. This reveals a discrepancy between:

- The sonataflow-runtimes library that runs the workflow
- The spec definition used by the kn-workflow v1.35 tool to generate manifests

Instead of using the standard script, we'll explore a workaround to deploy the [lock-flow workflow](https://github.com/masayag/poc-kafka-logic-operator/blob/main/callback-flow/src/main/resources/lock.sw.yaml) to an OCP cluster using the gitops profile.

## Generating Manifests with the kn-workflow CLI

The [kn-workflow CLI](https://mirror.openshift.com/pub/cgw/serverless-logic/latest/) serves multiple purposes in development, testing, and deployment. For our needs, we'll use it solely to generate Kubernetes manifests.

> **Note:** The current latest version (v1.35) has a bug that affects manifest generation. We'll use an enhanced version from [this repository](https://github.com/rhdhorchestrator/orchestrator-demo/releases/tag/v1.36-kn-workflow) that includes a fix.

First, download the kn-workflow CLI and add it to your `$PATH`. Then generate the manifests:

```bash
TARGET_IMAGE=<image:tag> # e.g. quay.io/orchestrator/demo-poc-kafka-logic:latest
git clone https://github.com/masayag/poc-kafka-logic-operator
cd poc-kafka-logic-operator/callback-flow/src/main/resources
kn-workflow gen-manifest --profile gitops --image $TARGET_IMAGE
```

After running this command, you should see these files in the manifests directory:

```bash
manifests
├── 01-configmap_lock-flow-props.yaml
└── 02-sonataflow_lock-flow.yaml
```

These files alone won't be sufficient for a successful deployment. We'll make additional changes after building the workflow image.

For now, let's verify the `02-sonataflow_lock-flow.yaml` contains a reference to our workflow image (required for the gitops profile):

```yaml
  podTemplate:
    container:
      image: quay.io/orchestrator/demo-poc-kafka-logic:latest
      resources: {}
```

## Building the Workflow Image Using Dockerfile

Let's build the workflow image using podman on a Linux machine. Navigate to the `poc-kafka-logic-operator/callback-flow` directory and run:

```bash
podman build \
  -f ../../orchestrator-demo/docker/osl.Dockerfile \
  --tag $TARGET_IMAGE \
  --platform linux/amd64 \
  --ulimit nofile=4096:4096 \
  --build-arg QUARKUS_EXTENSIONS="\
    org.kie:kie-addons-quarkus-persistence-jdbc:9.102.0.redhat-00005,\
    io.quarkus:quarkus-jdbc-postgresql:3.8.6.redhat-00004,\
    io.quarkus:quarkus-agroal:3.8.6.redhat-00004, \
    io.quarkus:quarkus-smallrye-reactive-messaging-kafka" \
  --build-arg MAVEN_ARGS_APPEND="\
    -DmaxYamlCodePoints=35000000 \
    -Dkogito.persistence.type=jdbc \
    -Dquarkus.datasource.db-kind=postgresql \
    -Dkogito.persistence.proto.marshaller=false" \
    .
```

Note that for this particular workflow, we needed to include an additional Quarkus extension: `io.quarkus:quarkus-smallrye-reactive-messaging-kafka`.

Once built, push the image to your registry:

```bash
podman push $TARGET_IMAGE
```

## Editing the Manifests

Now we need to enhance our manifests to make them production-ready.

### Enable Persistence

In production environments, we want to enable persistence for our workflow. This can be configured either at the platform level or per workflow. See the [Sonataflow documentation for enabling persistence](https://sonataflow.org/serverlessworkflow/latest/cloud/operator/using-persistence.html#configuring-persistence-using-the-sonataflowplatform-cr) for more details.

For our workflow, persistence configuration is required since we included Quarkus persistence extensions in the build command.

Edit `src/main/resources/manifests/02-sonataflow_lock-flow.yaml` and add this section at the same level as the `podTemplate`:

```yaml
  persistence:
    postgresql:
      secretRef:
        name: sonataflow-psql-postgresql
        userKey: postgres-username
        passwordKey: postgres-password
      serviceRef:
        name: sonataflow-psql-postgresql
        port: 5432
        databaseName: sonataflow
        databaseSchema: lock-flow
```

Also, add the following property to the configmap to enable Flyway migrations:

```yaml
kie.flyway.enabled = true
```

### Add Secrets for TLS Support

To enable TLS, we need to mount a secret with the truststore to the workflow deployment. Add this under the `podTemplate` property:

```yaml
  podTemplate:
    container:
      image: quay.io/orchestrator/demo-poc-kafka-logic:latest
      volumeMounts:
        - name: truststore-volume
          mountPath: /deployment/certs
          readOnly: true
      resources: {}
    volumes:
      - name: truststore-volume
        secret:
          secretName: kafka-truststore
```

Let's also update the configmap with TLS configuration:

```yaml
    # TLS support
    kafka.bootstrap.servers=<kafka-bootstrap-server>:9093
    mp.messaging.connector.smallrye-kafka.security.protocol=SSL
    # Specify the enabled TLS protocols (forcing TLSv1.2)
    mp.messaging.connector.smallrye-kafka.ssl.enabled.protocols=TLSv1.2

    # Truststore configuration (so the client can validate the broker's certificate)
    mp.messaging.connector.smallrye-kafka.ssl.truststore.location=/deployment/certs/truststore.jks
    mp.messaging.connector.smallrye-kafka.ssl.truststore.password=password
```

For mTLS (mutual TLS), similar adaptations would be needed.

## Deploying the Manifests to the Cluster

Here's the complete configmap with all our changes:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: lock-flow-props
  namespace: sonataflow-infra
  labels:
    app: lock-flow
    app.kubernetes.io/component: serverless-workflow
    app.kubernetes.io/managed-by: sonataflow-operator
    app.kubernetes.io/name: lock-flow
    sonataflow.org/workflow-app: lock-flow
    sonataflow.org/workflow-namespace: sonataflow-infra
data:
  application.properties: |
    kie.flyway.enabled = true

    # Note the topic property; If your broker has different topic names you can change this property.
    mp.messaging.incoming.lock-event.connector=smallrye-kafka
    mp.messaging.incoming.lock-event.value.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
    mp.messaging.incoming.lock-event.topic=lock-event

    mp.messaging.incoming.release-event.connector=smallrye-kafka
    mp.messaging.incoming.release-event.value.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
    mp.messaging.incoming.release-event.topic=release-event

    mp.messaging.outgoing.released-event.connector=smallrye-kafka
    mp.messaging.outgoing.released-event.value.serializer=org.apache.kafka.common.serialization.StringSerializer
    mp.messaging.outgoing.released-event.topic=released-event

    mp.messaging.outgoing.notify-event.connector=smallrye-kafka
    mp.messaging.outgoing.notify-event.value.serializer=org.apache.kafka.common.serialization.StringSerializer
    mp.messaging.outgoing.notify-event.topic=notify-event

    quarkus.kafka.devservices.enabled=false

    # TLS support
    kafka.bootstrap.servers=kafka-cluster-3-kafka-bootstrap.orchestrator.svc:9093
    mp.messaging.connector.smallrye-kafka.security.protocol=SSL
    # Specify the enabled TLS protocols (forcing TLSv1.2)
    mp.messaging.connector.smallrye-kafka.ssl.enabled.protocols=TLSv1.2

    # Truststore configuration (so the client can validate the broker's certificate)
    mp.messaging.connector.smallrye-kafka.ssl.truststore.location=/deployment/certs/truststore.jks
    mp.messaging.connector.smallrye-kafka.ssl.truststore.password=password
```

Make sure to update the Kafka configuration, referenced secret path, and password to match your environment.

Here's the complete SonataFlow CR with our additions:

```yaml
  podTemplate:
    container:
      image: quay.io/orchestrator/demo-poc-kafka-logic:latest
      volumeMounts:
        - name: truststore-volume
          mountPath: /deployment/certs
          readOnly: true
      resources: {}
    volumes:
      - name: truststore-volume
        secret:
          secretName: kafka-truststore      
  persistence:
    postgresql:
      secretRef:
        name: sonataflow-psql-postgresql
        userKey: postgres-username
        passwordKey: postgres-password
      serviceRef:
        name: sonataflow-psql-postgresql
        port: 5432
        databaseName: sonataflow
        databaseSchema: lock-flow
```

Now we can deploy the manifests to the cluster. The files are numbered for a reason - the configmap must be applied first, otherwise, the SonataFlow operator will generate an empty configmap for the workflow:

```bash
oc create -f manifests/01-configmap_lock-flow-props.yaml -n sonataflow-infra
oc create -f manifests/02-sonataflow_lock-flow.yaml -n sonataflow-infra
```

Let's watch for the workflow pod status to verify it's running:

```bash
oc get pod -n sonataflow-infra -l app=lock-flow
```

You should see output like this:

```
NAME                        READY   STATUS    RESTARTS   AGE
lock-flow-5cbd5dbdb-2g7wz   1/1     Running   0          13m
```

## Testing the Workflow

Now we can test the workflow by emitting cloud events to trigger it. Note that in this example, we're not using the Orchestrator to invoke workflows, since it can only invoke workflows via HTTP endpoints. However, we can monitor the workflow's progress from the workflow runs tab.

### Setting Up the Kafka Producer with TLS

To invoke the workflow, we'll use Kafka's producer script. Since we're using TLS, we need to have the `truststore.jks` file available.

> In this example, we're using Kafka installed on an OpenShift cluster with the Strimzi operator.

First, copy or mount the `truststore.jks` file to the Kafka cluster broker pod. Then create a properties file at `/tmp/client-ssl.properties`:

```bash
security.protocol=SSL
ssl.truststore.location=/tmp/truststore.jks
ssl.truststore.password=password
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
ssl.protocol=TLS
```

### Triggering the Workflow

Now invoke the Kafka producer:

```bash
./bin/kafka-console-producer.sh \
  --broker-list kafka-cluster-3-kafka-bootstrap:9093 \
  --topic lock-event \
  --producer.config /tmp/client-ssl.properties
```

This command opens an interactive session where you can paste Kafka messages.

#### Step 1: Send the Lock Event

To start a workflow, first paste the content of the [lock-event](https://github.com/masayag/poc-kafka-logic-operator/blob/main/kafka-messages/lock-event.json):

```json
{
  "specversion": "1.0",
  "id": "db16ff44-5b0b-4abc-88f3-5a71378be171",
  "source": "http://dev.local",
  "type": "lock-event",
  "datacontenttype": "application/json",
  "time": "2025-03-07T15:04:32.327635-05:00",
  "lockid": "03471a81-310a-47f5-8db3-cceebc63961a",
  "data": {
    "name": "The Kraken",
    "id": "03471a81-310a-47f5-8db3-cceebc63961a"
  }
}
```

Once sent, you can verify that the workflow is in an "Active" state in the Orchestrator plugin.

#### Step 2: Send the Release Event

Next, paste the content of the [release-event](https://github.com/masayag/poc-kafka-logic-operator/blob/main/kafka-messages/release-event.json) to continue the workflow to its completion:

```json
{
  "specversion": "1.0",
  "id": "0e375e93-9846-4ba4-a9f0-ef8663e5a307",
  "source": "http://dev.local",
  "type": "release-event",
  "datacontenttype": "application/json",
  "time": "2025-03-07T15:09:32.327635-05:00",
  "lockid": "03471a81-310a-47f5-8db3-cceebc63961a",
  "data": {
    "name": "Release The Kraken",
    "id": "03471a81-310a-47f5-8db3-cceebc63961a"
  }
}
```

After sending this event, you can observe the workflow complete its execution in the Orchestrator dashboard.

## Conclusion

By following this guide, you've learned how to:

1. Work around build process issues in serverless workflows
2. Generate and customize Kubernetes manifests for workflow deployment
3. Enable critical production features like persistence and TLS security
4. Test Kafka-based workflows using cloud events

These techniques allow you to deploy complex workflows even when standard tools encounter compatibility issues, giving you more control over your serverless infrastructure.

## Acknowledgments

Special thanks to [Ricardo Zanini](https://github.com/ricardozanini) for creating the [original repository example](https://github.com/ricardozanini/poc-kafka-logic-operator) that was used throughout this tutorial. His work provides an excellent foundation for exploring serverless workflow implementation in development mode and with the preview profile.
