---
date: 2025-04-21
title: 'Building and Deploying Serverless Workflows'
---

The Orchestrator provides a way to run workflows directly from Backstage/RHDH. But how do these workflows actually become available in Backstage?  
This blog will take you through the journey from running a workflow locally on your machine to having it deployed on a cluster and available in the Orchestrator plugin.

> **Note:** This blog focuses on the build and deploy process, not workflow development itself. If you're looking for workflow development resources, there are guides available online.

We’ll use the [orchestrator-demo](https://github.com/rhdhorchestrator/orchestrator-demo) repository for all examples, tools, directory structures, and references throughout this blog.
The main goal of the `orchestrator-demo` repo is to showcase interesting use cases of the Orchestrator. Using tools from that repo, we'll show how to:

- Build workflow images
- Generate workflow manifests
- Deploy workflows to a cluster

And the best part? You can do it all from a single [script](https://github.com/rhdhorchestrator/orchestrator-demo/blob/main/scripts/build.sh).

## 🚀 Why Build Workflow Images?

You might wonder: why bother building images when the OpenShift Serverless Logic Operator can build workflows on the fly?

There are a couple of reasons:

- **Production readiness**: Prebuilt images can be scanned, secured, and tested before going live.
- **GitOps compatibility**: The Orchestrator relies on a central OpenShift Serverless Logic Operator instance to track workflows and their states. To use this central tracking service, workflows need to be deployed with the `gitops` profile, which expects a prebuilt image.
- **Testing & quality**: Just like any other software, workflows should be tested before deployment. Building an image gives us more control over that process.

On-the-fly builds (`preview` profile) are great for experimentation—but for production, prebuilding is the way to go.
Find more about building workflows on the fly [here](https://sonataflow.org/serverlessworkflow/latest/cloud/operator/build-and-deploy-workflows.html#configure-workflow-build-system).

Now that we understand *why* we build workflow images, let's look at *how* it's done.

## 🧱 Project Structure

We'll focus on a [Quarkus project](https://quarkus.io/guides/getting-started) layout (Maven project structure), specifically the `01_basic` workflow example:

```
01_basic
├── pom.xml
├── README.md
└── src
    └── main
        ├── docker
        │   ├── Dockerfile.jvm
        │   ├── Dockerfile.legacy-jar
        │   ├── Dockerfile.native
        │   └── Dockerfile.native-micro
        └── resources
            ├── application.properties
            ├── basic.svg
            ├── basic.sw.yaml
            ├── schemas
            │   ├── basic__main-schema.json
            │   └── workflow-output-schema.json
            └── secret.properties
```

This structure was generated using the [kn-workflow](https://mirror.openshift.com/pub/cgw/serverless-logic/1.35.0/) CLI.  
You can try it yourself by following the [Getting Started](https://github.com/rhdhorchestrator/orchestrator-demo/blob/main/README.md#getting-started) guide.

The main workflow resources are located under `src/main/resources/`.

## 🛠️ Building Locally

We use the [build script](https://github.com/rhdhorchestrator/orchestrator-demo/blob/main/scripts/build.sh) to handle everything.

You can run it either **locally** or **inside a container** (see the [documentation](https://github.com/rhdhorchestrator/orchestrator-demo/blob/main/BUILD_WORKFLOWS.md) for containerized execution).

Let's explore running it locally.

First, clone the project:

```bash
git clone git@github.com:rhdhorchestrator/orchestrator-demo.git
cd orchestrator-demo
```

Check out the script's help menu:

```bash
./scripts/build.sh --help
```

### 🧾 What the script does:

This script does the following (in order):

- Generates workflow manifests using the `kn-workflow` CLI (requires version **1.35.0**)
- Builds the workflow image with `podman` or `docker`
- Optionally pushes the image to an image registry and deploys the workflow using `kubectl`

### 🔧 Script usage:

```bash
./scripts/build.sh [flags]
```

**Important Flags:**

| Flag | Description |
|------|-------------|
| `-i, --image` (required) | Full image path, e.g. `quay.io/orchestrator/demo:latest` |
| `-w, --workflow-directory` | Workflow source directory (default: current directory) |
| `-m, --manifests-directory` | Where to save generated manifests |
| `--push` | Push the image to the registry |
| `--deploy` | Deploy the workflow |
| `-h, --help` | Show help message |

> 💡 The script also supports builder/runtime image overrides, namespace targeting, and persistence flags.

#### Environment Variables Supported by the Build Script

The build script supports environment variables to customize the workflow build process without modifying the script itself.

##### `QUARKUS_EXTENSIONS`

Specifies additional [Quarkus](https://quarkus.io/extensions/) extensions required by the workflow.

- **Format:** Comma-separated list of fully qualified extension IDs.
- **Example:**
  ```bash
  export QUARKUS_EXTENSIONS="io.quarkus:quarkus-smallrye-reactive-messaging-kafka"
  ```
- **Use Case:** Add Kafka messaging support or other integrations at build time.

##### `MAVEN_ARGS_APPEND`

Appends additional arguments to the Maven build command.

- **Format:** String of Maven CLI arguments.
- **Example:**
  ```bash
  export MAVEN_ARGS_APPEND="-DmaxYamlCodePoints=35000000"
  ```
- **Use Case:** Control build behavior, e.g., set `maxYamlCodePoints` parameter that contols the maximum input size for YAML input files to 35000000 characters (~33MB in UTF-8).

---

### 🧰 Required tools:

| Tool | Purpose |
|------|---------|
| podman or docker | Container runtime for building images |
| kubectl | Kubernetes CLI |
| yq | YAML processor |
| jq | JSON processor |
| curl, git, find, which | Shell utilities |
| kn-workflow | CLI for generating workflow manifests |

---

## ✅ Example: Building the `01_basic` Workflow

Now, let's build the `01_basic` workflow image:

```bash
./scripts/build.sh --image=quay.io/orchestrator/demo-basic:test -w 01_basic/ -m 01_basic/manifests
```

You **must** specify the target image (with a tag).

If you run the script from the repo's root directory, use the `-w` flag to point to the workflow directory, and specify the output directory with `-m`.

The command produces two artifacts:

- A workflow image: `quay.io/orchestrator/demo-basic:test` (and tagged as `latest`).
- Kubernetes manifests under: `01_basic/manifests/`

If you want, you can add the `--push` flag to automatically push the image after building.  
(Otherwise, pushing manually is mandatory before deploying.)

### Generated Workflow Manifests

Let's look at what was generated under `01_basic/manifests`:

```
01_basic/manifests
├── 00-secret_basic-secrets.yaml
├── 01-configmap_basic-props.yaml
├── 02-configmap_01-basic-resources-schemas.yaml
└── 03-sonataflow_basic.yaml
```

**A quick overview:**

- **00-secret_basic-secrets.yaml**  
  Contains secrets from `01_basic/src/main/resources/secret.properties`.  
  Values are not required at this stage—you can set them later (after applying CRs or when using GitOps).  
  *Important:* In OpenShift Serverless Logic v1.35, after updating a secret, you must manually restart the workflow Pod for changes to apply. (v1.36 is not released yet.)

- **01-configmap_basic-props.yaml**  
  Holds application properties from `application.properties`.  
  Any change to this ConfigMap triggers an automatic Pod restart.

- **02-configmap_01-basic-resources-schemas.yaml**  
  Contains JSON schemas from `src/main/resources/schemas`.  
  **Note:** When using the GitOps profile, this ConfigMap (and similar ones like `specs`) does **not** need to be deployed.

- **03-sonataflow_basic.yaml**  
  The SonataFlow custom resource (CR) that defines the workflow.  
  A few important parts:

  **Image and Secrets Mounting:**

  ```yaml
  podTemplate:
    container:
      image: quay.io/orchestrator/demo-basic
      resources: {}
      envFrom:
        - secretRef:
            name: basic-secrets
  ```

  **Persistence Configuration:**

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
        databaseSchema: basic
  ```

  If you need to connect to an external database, you must replace `serviceRef` with a `jdbcUrl`. See [here](https://sonataflow.org/serverlessworkflow/latest/cloud/operator/enabling-jobs-service.html#_using_the_persistence_field_defined_in_the_sonataflowplatform_cr).

By default, all the manifests are generated without a namespace. There is an option to specify a namespace to the script by the `--namespace` flag if we know the target namespace in advance. Otherwise, the namespace needs to be provided when applying the manifests to the cluster.

Find more about configuring workflows [here](https://sonataflow.org/serverlessworkflow/latest/cloud/operator/configuring-workflows.html).

---

### 🚀📦 Deploy Workflows on a Cluster

With the image pushed to the image registry and the manifests available, we can now deploy the workflow on a cluster.

#### Prerequisites

You need an OpenShift Container Platform (OCP) cluster with:

- Red Hat Developer Hub (RHDH) v1.5
- Orchestrator plugins v1.4 or v1.5
- OpenShift Serverless v1.35
- OpenShift Serverless Logic v1.35

Make sure you follow the installation instructions provided in the [Orchestrator Documentation](https://github.com/rhdhorchestrator/orchestrator-go-operator/tree/main/docs/release-1.5).

By default, OpenShift Serverless Logic services are installed in the `sonataflow-infra` namespace.  
We must apply the workflow's manifests in a namespace that contains a `SonataflowPlatform` custom resource, which manages the [supporting services](https://sonataflow.org/serverlessworkflow/latest/cloud/operator/supporting-services.html).

#### Deploy Workflow

```bash
kubectl create -n sonataflow-infra -f ./01_basic/manifests/.
```

#### Monitor Deployment

Check the status of the pods:

```bash
kubectl get pods -n sonataflow-infra -l app=basic
```

Initially, the pod may appear in an `Error` state because of missing or incomplete configuration in the Secret or ConfigMap.

Inspect the pod logs:

```bash
oc logs -n sonataflow-infra basic-f7c6ff455-vwl56
```

Example output:

```
SRCFG00040: The config property quarkus.openapi-generator.notifications.auth.BearerToken.bearer-token is defined as the empty String ("") which the following Converter considered to be null: io.smallrye.config.Converters$BuiltInConverter
java.lang.RuntimeException: Failed to start quarkus
...
Caused by: io.quarkus.runtime.configuration.ConfigurationException: Failed to read configuration properties
```

The error indicates a missing property: `quarkus.openapi-generator.notifications.auth.BearerToken.bearer-token`.

#### Inspect and Fix the ConfigMap

Retrieve the ConfigMap:

```bash
oc get -n sonataflow-infra configmaps basic-props -o yaml
```

Sample output:

```yaml
apiVersion: v1
data:
  application.properties: |
    # Backstage Notifications service
    quarkus.rest-client.notifications.url=${BACKSTAGE_NOTIFICATIONS_URL}
    quarkus.openapi-generator.notifications.auth.BearerToken.bearer-token=${NOTIFICATIONS_BEARER_TOKEN}
...
```

The placeholders must be resolved using values provided via a Secret.

#### Edit the Secret

Edit the Secret and provide appropriate (even dummy) values:

```bash
kubectl edit secrets -n sonataflow-infra basic-secrets
```

Use base64-encoded dummy values if needed for this simple example.
A restart of the workflow pod is required for the changes to take effect.

#### Verify Deployment Status

Check the pods again:

```bash
oc get pods -n sonataflow-infra -l app=basic
```

Expected output:

```
NAME                    READY   STATUS    RESTARTS   AGE
basic-f7c6ff455-grkxd   1/1     Running   0          47s
```

The pod should now be in the `Running` state.

With that, the workflow should now appear in the Orchestrator plugin inside Red Hat Developer Hub.

---

## 🛠️➡️ Next Steps

Now that the process of moving from a local development environment to the cluster is clear, the next logical step is designing a CI pipeline.

You can inspect the provided build script to extract the actual steps and implement them in your preferred CI/CD tool (e.g., GitHub Actions, GitLab CI, Jenkins, Tekton).
