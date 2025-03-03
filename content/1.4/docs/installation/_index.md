---
title: "Installation"
date: 2024-09-10
weight: 3
---

The deployment of the orchestrator involves multiple independent components, each with its unique installation process. In an OpenShift Cluster, the Red Hat Catalog provides an operator that can handle the installation for you. This installation process is modular, as the CRD exposes various flags that allow you to control which components to install. For a vanilla Kubernetes, there is a helm chart that installs the orchestrator compoments.

The Orchestrator deployment encompasses the installation of the engine for serving serverless workflows and Backstage, integrated with orchestrator plugins for workflow invocation, monitoring, and control.

In addition to the Orchestrator deployment, we offer several *workflows* (linked below) that can be deployed using their respective installation methods.