---
title: "Quick Start"
date: 2024-04-05
weight: 1
---

## Quickstart Guide

This quickstart guide will help you install Orchestrator via Red Hat Developer Hub (RHDH) and execute a sample workflow through the Orchestrator plugin on the RHDH UI.

1. **Install Orchestrator via RHDH**:
   Choose one of the following installation methods:

   - [Installation via RHDH Operator](../installation/orchestrator-on-rhdh-operator/) - Recommended for most users
   - [Installation via RHDH Chart](../installation/orchestrator-on-rhdh-chart/) - For environments where the RHDH Operator is not available

2. **Install a sample workflow**:
   Follow the [installation instructions for the greetings workflow](https://github.com/rhdhorchestrator/serverless-workflows/blob/main/deploy/docs/release-1.7/greeting/README.md).

3. **Access Red Hat Developer Hub**:
   Open your web browser and navigate to the Red Hat Developer Hub application. Retrieve the URL using the following OpenShift CLI command.

   ```bash
   oc get route backstage-backstage -n rhdh-operator -o jsonpath='{.spec.host}'
   ```

   Make sure the route is accessible to you locally.

4. **Login to Backstage**
   Login to Backstage with the Guest account.

5. **Navigate to Orchestrator**:
   Navigate to the Orchestrator page by clicking on the Orchestrator icon in the left navigation menu.
   ![orchestratorIcon](./orchestratorIcon.png)

6. **Execute Greeting Workflow**:
   Click on the 'Execute' button in the ACTIONS column of the Greeting workflow.
   ![workflowsPage](./workflowsPage.png)
   The 'Run workflow' page will open. Click 'Next step' and then 'Run'
   ![executePageNext](./executePageNext.png)
   ![executePageRun](./executePageRun.png)
7. **Monitor Workflow Status**:
   Wait for the status of the Greeting workflow execution to become _Completed_. This may take a moment.
   ![workflowCompleted](./workflowCompleted.png)
