---
title: "Installation"
date: 2024-09-10
weight: 3
---

On previous Orchestrator versions (<1.6), an RHDH operator installation was triggered by the Orchestrator operator, or a pre-existing RHDH installation was connected. On RHDH/Orchestrator 1.7 - that is no longer the case. RHDH operator is responsible for installing the Orchestrator resources, and Orchestrator will cease to exist as a standalone operator.

## Installation Methods

### RHDH Operator

- [Installation via RHDH Operator](./orchestrator-on-rhdh-operator/) - Complete setup using the RHDH Operator

### RHDH Helm Chart

- [Installation via RHDH Chart](./orchestrator-on-rhdh-chart/) - Use Helm charts to install Orchestrator

### Workflows

In addition to the Orchestrator deployment, we offer several [workflows](workflows/) that can be deployed using their respective installation methods.
