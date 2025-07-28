# Upgrading from Orchestrator 1.6 to RHDH 1.7

Starting with Red Hat Developer Hub (RHDH) v1.7, Orchestrator will be shipped as an integrated component of RHDH, available through both the RHDH operator and RHDH Helm Chart. Users currently running Orchestrator v1.6 cannot directly upgrade to "Orchestrator 1.7". Instead, they must install the new RHDH release with Orchestrator enabled and migrate their existing data and resources.

## Background: RHDH and Orchestrator Integration

From RHDH v1.7 onwards, Orchestrator will be shipped alongside RHDH as a plugin with all current capabilities. The components will be installed together, and Orchestrator will no longer have its own dedicated operator. For detailed information about this transition, see our [blog post about installing Orchestrator via RHDH Chart](https://www.rhdhorchestrator.io/blog/installing-orchestrator-via-rhdh-chart/).

### Current Orchestrator Architecture

Orchestrator v1.6 currently interacts with several operators and components:

- RHDH operator installation
- OpenShift Serverless and OpenShift Serverless Logic operators (including PostgreSQL database)
- Orchestrator dynamic plugins
- Software templates and GitOps integration

These components remain available in RHDH v1.7. However, instead of the Orchestrator Operator serving as a meta-operator managing RHDH and OSL installations, all deployment logic is integrated into the RHDH installation process.

### Database Management

Orchestrator v1.6 supports two database management options, both of which remain available in RHDH v1.7:

1. **Local PostgreSQL instance**: Installed via Helm chart, serving the SonataFlow Platform with schemas for Orchestrator workflows
2. **External database**: Configured in the SonataFlow Platform as the designated database instance

Understanding these options is crucial for the upgrade process, as you'll need to configure an external database connection to preserve your existing data.

## Upgrade Process Overview

To upgrade from Orchestrator v1.6 to RHDH v1.7:

1. **Install RHDH v1.7** via the RHDH Operator or RHDH Helm Chart
2. **Migrate SonataFlow resources** from the `sonataflow-infra` namespace (or your custom namespace) to the RHDH installation namespace
3. **Move workflow deployments** (SonataFlow CRs) to the new namespace
4. **Configure external database connection** to preserve existing data

The key strategy is to treat your existing SonataFlow Platform database as an external database for the new RHDH installation.

## Upgrading with RHDH Helm Chart

**Prerequisites**: Familiarize yourself with [deploying RHDH via Helm Chart](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.1/html/administration_guide_for_red_hat_developer_hub/assembly-install-rhdh-ocp#proc-install-rhdh-ocp-helm_admin-rhdh).

### Steps

1. **Prepare for installation**:

   - Delete the Orchestrator CR and the Backstage CR that was previously used
   - Disable or delete your existing SonataFlow Platform before installation
   - The Helm installation will create a new SonataFlow Platform in the same namespace as RHDH
   - The installation will reuse your existing 'sonataflow' database if found
   
1. **Configure external database connection** following the [RHDH external database documentation](https://github.com/redhat-developer/rhdh-chart/blob/main/docs/external-db.md). Create a secret containing PostgreSQL connection properties. This guide will also have additional values for the Helm Chart, that will be used in the next step.

1. **Install RHDH with Orchestrator** using the [RHDH Helm Chart](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/backstage), following the external database configuration guide. Make sure to include the extra values issued by the external-db installation doc from the former point.


1. **Migrate workflows**: After installation, migrate any existing workflow deployments (SonataFlow CRs) to the RHDH namespace.

## Upgrading with RHDH Operator

**Prerequisites**: Familiarize yourself with [deploying RHDH via Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.1/html/administration_guide_for_red_hat_developer_hub/assembly-install-rhdh-ocp#proc-install-rhdh-ocp-operator_admin-rhdh).

// TBA
