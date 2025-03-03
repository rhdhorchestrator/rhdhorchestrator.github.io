---
title: "Architecture"
weight: 1
---
The Orchestrator architecture comprises several integral components, each contributing to the seamless execution and management of workflows. Illustrated below is a breakdown of these components:

- [**Red Hat Developer Hub**](https://developers.redhat.com/rhdh/overview): Serving as the primary interface, Backstage fulfills multiple roles:
  - [Orchestrator Plugins](https://github.com/redhat-developer/rhdh-plugins/blob/main/workspaces/orchestrator/README.md): Both frontend and backend plugins are instrumental in presenting deployed workflows for execution and monitoring.
  - [Notifications Plugin](https://backstage.io/docs/notifications): Employs notifications to inform users or groups about workflow events.
- [**OpenShift Serverless Logic Operator**](https://sonataflow.org/serverlessworkflow/main/cloud/operator/install-serverless-operator.html): This controller manages the Sonataflow custom resource (CR), where each CR denotes a deployed workflow.
- [**Sonataflow Runtime/Workflow Application**](https://github.com/apache/incubator-kie-kogito-runtimes): As a deployed workflow, Sonataflow Runtime is currently managed as a Kubernetes (K8s) deployment by the operator. It operates as an HTTP server, catering to requests for executing workflow instances. Within the Orchestrator deployment, each Sonataflow CR corresponds to a singular workflow. However, outside this scope, Sonataflow Runtime can handle multiple workflows. Interaction with Sonataflow Runtime for workflow execution is facilitated by the Orchestrator backend plugin.
- [**Data Index Service**](https://sonataflow.org/serverlessworkflow/latest/data-index/data-index-core-concepts.html): This serves as a repository for workflow definitions, instances, and their associated jobs. It exposes a GraphQL API, utilized by the Orchestrator backend plugin to retrieve workflow definitions and instances.
- [**Job Service**](https://sonataflow.org/serverlessworkflow/latest/job-services/core-concepts.html): Dedicated to orchestrating scheduled tasks for workflows.
- [**OpenShift Serverless**](https://docs.openshift.com/serverless/1.33/about/about-serverless.html): This operator furnishes serverless capabilities essential for workflow communication. It employs Knative eventing to interface with the Data Index service and leverages Knative functions to introduce more intricate logic to workflows.
- [**OpenShift AMQ Streams**](https://access.redhat.com/documentation/en-us/red_hat_amq_streams/2.6/html/amq_streams_on_openshift_overview/index) (Strimzi/Kafka): While not presently integrated into the deployment's current iteration, this operator is crucial for ensuring the reliability of the eventing system.
- [**KeyCloak**](https://www.keycloak.org/): Responsible for authentication and security services within applications. While not installed by the Orchestrator operator, it is essential for enhancing security measures.
- [**PostgreSQL Server**](https://www.postgresql.org/) - Utilized for storing both Sonataflow information and Backstage data, PostgreSQL Server provides a robust and reliable database solution essential for data persistence within the Orchestrator ecosystem.


<a href="./architecture-context-diagram.jpg" target="_blank">
  <img src="./architecture-context-diagram.jpg" alt="Architecture Context Diagram" title="Architecture Context Diagram" width="400" />
</a>

<a href="./architecture-container-diagram.jpg" target="_blank">
  <img src="./architecture-container-diagram.jpg" alt="Architecture Container Diagram" title="Architecture Container Diagram" width="800" />
</a>

<a href="./architecture-diagram.png" target="_blank">
  <img src="./architecture-diagram.png" alt="Architecture Diagram" title="Architecture Diagram"/>
</a>


