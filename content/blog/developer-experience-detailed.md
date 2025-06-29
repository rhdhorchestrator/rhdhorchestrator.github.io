---
date: 2024-03-15
title: 'Serverless Workflows: an Automated Developer Experience Step-by-Step'
tags: [Orchestrator-1.4]
---
In this blog, we'll guide you through the journey from a software template to bootstrapping the workflow development, building, packaging, releasing, and deploying it on a cluster. If you need a high-level explanation or want to dive into the architecture of the solution, check out our previous [blog](../developer-experience). You can also watch a detailed demonstration of the content covered in this post in this [recording](https://www.youtube.com/watch?v=G6wnRHjvhv0).

## Prerequisites and Assumptions
This blog assumes familiarity with specific tools, technologies, and methodologies. We'll start with RHDH (Backstage) by launching a basic workflow template, working with GitHub for source control, pushing the workflow image to [Quay](quay.io), and using Kustomize to deploy the ArgoCD application for GitOps.
* The target [Quay](quay.io) repository for the workflow's image should exist.
* The target namespace for both the pipeline and the workflow is set to `sonataflow-infra` and not configurable.

### Creating a Workflow Repository
Let's begin by creating a workflow named `demo` under the Quay organization `orchestrator-testing`. We'll use the repository `orchestrator-testing/serverless-workflow-demo` to store the workflow image.

![Creating a new workflow repository in Quay](/blog/images/new-quay-workflow-repo.png)

### Setting Robot Account Permissions
Next, add robot account permissions to the created repository.

![Setting permissions](/blog/images/add-robot-accout-perm-to-workflow.png)

### Creating a Secret for GitOps Cluster
Refer to the instructions [here](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/gitops/README.md#installing-docker-credentials) for creating and configuring the secret for the target cluster.

## Creating the Software Template
The Orchestrator plugin provides templates to kickstart your workflow project. By selecting a template tagged with `orchestrator`, you gain access to the following benefits:
* A fully operational software project in a new Git repository under your chosen organization.
* A configuration repository with `kustomize` configurations for deploying the workflow on RHDH.
* Automated CI tool deployment using OpenShift Pipelines.
* Automated CD deployment for applications using OpenShift GitOps.

### Selecting and Launching the Template
Navigate to the Catalog and select the *Basic workflow bootstrap* project template. Click "Launch Template" to start filling in the input parameters for creating the workflow and its GitOps projects.

![Selecting the software template](/blog/images/software-template-catalog.png)

### Input Parameters Overview
Review the parameters required for workflow creation, including organization name, repository name, workflow ID, workflow type, CI/CD method, namespaces, Quay details, persistence option, and database properties.

![Input parameters](/blog/images/template-input-parameters-1.png)

This section provides an overview of the parameters required for workflow creation:
- **Organization Name** - The GitHub organization where workflow repositories will be created. Ensure that the GitHub token provided during Orchestrator chart installation includes repository creation permissions in this organization.
- **Repository Name** - The name of the repository containing the workflow definition, spec and schema files, and application properties. Workflow development occurs in this repository. For example, if this repository is named *onboarding*, a second repository named *onboarding-gitops* is created for CD automated deployment of the workflow.
- **Description** - This description will be added to the README.md file of the generated project and the workflow definition shown in the Orchestrator plugin.
- **Workflow ID** - A unique identifier for the workflow. This ID is used to generate project resources (appearing in file names) and acts as the name of the Sonataflow CR for that workflow. After deploying the CR to the cluster, the ID identifies the workflow in Sonataflow.

On the second screen, you'll need to select the workflow type. You can learn more about different workflow types [here](/docs/core-concepts/workflow-types/).
![Input parameters](/blog/images/template-input-parameters-2.png)
- **Workflow Type** - There are two supported types: infrastructure for operations returning output, and assessment for evaluation/assessment leading to potential infrastructure workflows.

> **🚨 Deprecation Notice: 🚨**  
> In the next Orchestrator release, Workflow Types will be retired. All workflows will act as infrastructure workflows, and no workflow will act as an assessment workflow. <br>
> The following document will be relevant up to Orchestrator version 1.6.

On the final screen, you'll be prompted to input the CI/CD parameters and persistence-related parameters.
- **Select a CI/CD method** - Choosing *None* means no GitOps resources are created in target repositories, only the workflow source repository. Selecting *Tekton with ArgoCD* creates two repositories: one for the workflow and another for GitOps resources for deploying the built workflow on a cluster.
- **Workflow Namespace** - The namespace for deploying the workflow in the target cluster, currently supporting *sonataflow-infra* where Sonataflow infrastructure is deployed.
- **GitOps Namespace** - Namespace for GitOps secrets and ArgoCD application creation. The default *orchestrator-gitops* complies with the default installation steps of the Orchestrator deployment.
- **Quay Organization Name** - Organization name in Quay for the published workflow. The Tekton pipeline pushes the workflow to this organization.
- **Quay Repository Name** - Repository name in Quay for the published workflow, which must exist before deploying GitOps. The secret created in the GitOps Namespace needs permission to push to this repository.
- **Enable Persistence** - Check this option to enable persistence for the workflow. It ensures each workflow persists its instances in a configured database schema, with the schema name matching the workflow ID. Persistence is recommended for long-running workflows and to support the Abort operation.
- **Database properties** - Self-explanatory list of database properties.

After providing all parameters, click Review, ensure correctness, and then click Create. Successful creation leads to:

![Template created](/blog/images/template-created.png)

This includes links to three resources:
- **Bootstrap the GitOps Resources** - Directs to the workflow GitOps repository, enabling GitOps for ArgoCD deployment on the target cluster.
- **Open the Source Code Repository** - Opens the Git repository for workflow development.
- **Open the Catalog Info Component** - The RHDH Catalog Components view which should include the newly created components: the workflow source repository and the workflow GitOps repository.

## Bootstrap the GitOps Resources
Navigate to the first link to enable GitOps automation on the cluster. Follow the steps provided, including setting up CI pipelines and viewing ArgoCD resources.

## Exploring the Repositories
The source code repository is where the workflow development happens. Each commit triggers the CI workflow.

The GitOps resources repository contains deployment configurations for the workflow on the OCP cluster.

## Viewing the Catalog Info Components
Both repositories are represented as components in RHDH:
![Catalog Items](/blog/images/workflow-catalog-items.png)

### View the Source Code Repository Component
This component represents the Git repository where workflow development occurs. Navigating to the CI tab reveals the pipeline-run diagram:
![workflow Ci](/blog/images/workflow-ci-pipeline.png)

Once the pipeline-run is completed, the CD step starts, and the workflow is deployed on the cluster.

### View the GitOps Resources Repository Component
This component represents the deployment of the workflow on the OCP cluster. Navigating to the CD tab shows the K8s resources representing the deployed workflow. When the items in this view are ready, the workflow should be ready to be executed from the Orchestrator plugin.

### Running the workflow
After completing the CI/CD pipelines, navigate to the Orchestrator plugin, choose the workflow, and run it.

## Conclusion
Streamlining workflow development and deployment empowers developers to focus on creating impactful workflows tailored to their needs.



