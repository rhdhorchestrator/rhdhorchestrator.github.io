---
title: "Workflow Types"
date: 2024-05-07
---

The Orchestrator features two primary workflow categories:
- *Infrastructure workflows*: focus on automating infrastructure-related tasks
- *Assessment workflows*: focus on evaluating and analyzing data to suggest suitable infrastructure workflow options for subsequent execution

### Infrastructure workflow
In the Orchestrator, an infrastructure refers to a workflow that executes a sequence of operations based on user input (optional) and generates output (optional) without requiring further action.

To define this type, developers need to include the following annotation in the workflow definition file:

```yaml
annotations:
  - "workflow-type/infrastructure"
```

The Orchestrator plugin utilizes this metadata to facilitate the processing and visualization of infrastructure workflow inputs and outputs within the user interface.

##### Examples:
- [Greeting](https://github.com/rhdhorchestrator/serverless-workflows/blob/main/workflows/greeting/greeting.sw.yaml)
- [Ticket Escalation](https://github.com/rhdhorchestrator/serverless-workflows/blob/main/workflows/escalation/ticketEscalation.sw.yaml)
- [Move2Kube](https://github.com/rhdhorchestrator/serverless-workflows/blob/main/workflows/move2kube/m2k.sw.yml)


### Assessment workflow
In the Orchestrator, an assessment is akin to an infrastructure workflow that concludes with a recommended course of action.
Upon completion, the assessment yields a *workflowOptions* object, which presents a list of infrastructure workflows suitable from the user's inputs evaluation.

To define this type, developers must include the following annotation in the workflow definition file:

```yaml
annotations:
  - "workflow-type/assessment"
```

The Orchestrator plugin utilizes this metadata to facilitate the processing and visualization of assessment workflow inputs and outputs within the user interface.
This includes generating links to initiate infrastructure workflows from the list of recommended options, enabling seamless execution and integration.

The *workflowOptions* object must possess six essential attributes with specific types, including lists that can be empty or contain objects with `id` and `name` properties, similar to the `currentVersion` attribute. See an example in the below code snippet.

*It is the assessment workflow developer's responsibility to ensure that the provided workflow **id** in each workflowOptions attribute exists and is available in the environment.*

```json
{
    "workflowOptions": {
      "currentVersion": {
        "id": "_AN_INFRASTRUCTURE_WORKFLOW_ID_",
        "name": "_AN_INFRASTRUCTURE_WORKFLOW_NAME_"
      },
      "newOptions": [],
      "otherOptions": [],
      "upgradeOptions": [],
      "migrationOptions": [
        {
            "id": "_ANOTHER_INFRASTRUCTURE_WORKFLOW_ID_",
            "name": "_ANOTHER_INFRASTRUCTURE_WORKFLOW_NAME_"
        }
      ],
      "continuationOptions": []
    }
}
```

##### Examples:
- [MTA](https://github.com/rhdhorchestrator/serverless-workflows/blob/main/workflows/mta-v7.x/mta.sw.yaml)
- [Dummy Assessment](https://github.com/rhdhorchestrator/serverless-workflow-examples/tree/main/assessment)

#### Note
If the aforementioned annotation is missing in the workflow definition file, the Orchestrator plugin will default to treating the workflow as an infrastructure workflow, without considering its output.

To avoid unexpected behavior and ensure clarity, it is strongly advised to always include the annotation to explicitly specify the workflow type, preventing any surprises or misinterpretations.

