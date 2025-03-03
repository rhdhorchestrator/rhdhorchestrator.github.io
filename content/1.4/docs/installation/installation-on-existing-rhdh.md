---
date: 2024-03-13
title: "Orchestrator on existing RHDH instance"
---

When [RHDH](https://developers.redhat.com/rhdh) is already installed and in use, reinstalling it is unnecessary. Instead, integrating the Orchestrator into such an environment involves a few key steps:

1. Utilize the Orchestrator operator to install the requisite components, such as the OpenShift Serverless Logic Operator and the OpenShift Serverless Operator, while ensuring the RHDH installation is disabled.
2. Manually update the existing RHDH ConfigMap resources with the necessary configuration for the Orchestrator plugin.
3. Import the Orchestrator software templates into the Backstage catalog.

## Prerequisites
- RHDH is already deployed with a running Backstage instance.
  - Software templates for workflows requires GitHub provider to be configured.
- Ensure that a [PostgreSQL](https://www.postgresql.org/) database is available and that you have credentials to manage the tablespace (optional).
  - For your convenience, a [reference implementation](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/postgresql/README.md) is provided.
  - If you already have a PostgreSQL database installed, please refer to this [note](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/postgresql/README.md#note-the-default-settings-provided-in-postgresql-values-match-the-defaults-provided-in-the-orchestrator-values) regarding default settings.

In this approach, since the RHDH instance is not managed by the Orchestrator operator, its configuration is handled through the Backstage CR along with the associated resources, such as ConfigMaps and Secrets.

The installation steps are detailed [here](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/main/existing-rhdh.md).
