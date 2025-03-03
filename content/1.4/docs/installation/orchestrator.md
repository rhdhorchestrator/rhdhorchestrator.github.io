---
title: "Orchestrator on OpenShift"
date: 2024-09-10
---

Installing the Orchestrator is facilitated through an operator available in the Red Hat Catalog in the OLM package. This operator is responsible for installing all of the Orchestrator components.
The Orchestrator is based on the [SonataFlow](https://sonataflow.org/serverlessworkflow/latest/index.html) and the [Serverless Workflow](https://serverlessworkflow.io/) technologies to design and manage the workflows.
The Orchestrator plugins are deployed on a [Red Hat Developer Hub
](https://developers.redhat.com/rhdh/overview) instance, which serves as the frontend.

When installing a Red Hat Developer Hub (RHDH) instance using the Orchestrator operator, the RHDH configuration is managed through the Orchestrator resource.

To utilize *Backstage* capabilities, the Orchestrator imports software templates designed to ease the development of new workflows and offers an opinionated method for managing their lifecycle by including CI/CD resources as part of the template.

{{< remoteMD "https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/main/README.md?raw=true" >}}
