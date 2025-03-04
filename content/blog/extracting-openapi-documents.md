---
date: 2024-09-05
title: Creating Extracted OpenAPI Documents for Integrating Systems on Serverless Workflows
---

# Creating Extracted OpenAPI Documents for Integrating Systems on Serverless Workflows
The blog post will guide developers on how to extract openAPI documents to a new file of manageable size. The need for this procedure has risen in account of restrictions that Quarkus imposes with accepting large YAML files as input [(see appendix)](#appendix). This restriction directs us to be mindful and plan ahead which resources services we would need in our workflow.

Please note that there is a way to work around the input file size restriction, as will be demonstrated in the [(appendix.)](#appendix)


In this guide, we will explain what is an OpenAPI Document, how to read and use the openAPI Specification, and eventually we will cover the steps to extract an openAPI document in a valid manner.

## What is an OpenAPI Document?
>A self-contained or composite resource which defines or describes an API or elements of an API.

OpenAPI documents are a standardized way to view a system’s exposed paths, resources and webhooks, and act as an alternative to using restAPI calls for interacting with other systems.
An OpenAPI document uses and conforms to the OpenAPI Specification, and in itself is a JSON object (which may be represented either in JSON or YAML format).

For the OpenAPI Spec Documentation, see: https://swagger.io/specification/

## How to make sense of an OpenAPI Document:
Let’s take a look at this [openAPI document](../../assets/examples/github-openapi.yaml) for some reference. This is an extracted version of a [larger openAPI document](https://github.com/github/rest-api-description/blob/main/descriptions/api.github.com/api.github.com.yaml)

<div style="height: 200px; overflow-x: auto; border=10px">
<pre>
<code>
  openapi: 3.0.3
  info:
    version: 1.1.4
    title: GitHub v3 REST API
    description: |-
      |
      GitHub's v3 REST API.
    license:
      name: MIT
      url: https://spdx.org/licenses/MIT
    termsOfService: https://docs.github.com/articles/github-terms-of-service
    contact:
      name: Support
      url: https://support.github.com/contact?tags=dotcom-rest-api
    x-github-plan: api.github.com
  servers:
    - url: https://api.github.com/
  security:
    - BearerToken: []
  paths:
    /repos/{owner}/{repo}/actions/workflows:
      get:
        summary: List repository workflows
        description: |
          Lists the workflows in a repository.
          Anyone with read access to the repository can use this endpoint.
          OAuth app tokens and personal access tokens (classic) need the `repo` scope
          to use this endpoint with a private repository.
        tags:
          - actions
        operationId: actions/list-repo-workflows
        externalDocs:
          description: API method documentation
          url: https://docs.github.com/rest/actions/workflows#list-repository-workflows
        parameters:
          - $ref: "#/components/parameters/owner"
          - $ref: "#/components/parameters/repo"
          - $ref: "#/components/parameters/per-page"
          - $ref: "#/components/parameters/page"
        responses:
          "200":
            description: Response
            content:
              application/json:
                schema:
                  type: object
                  required:
                    - total_count
                    - workflows
                  properties:
                    total_count:
                      type: integer
                    workflows:
                      type: array
                      items:
                        $ref: "#/components/schemas/workflow"
            headers:
              Link:
                $ref: "#/components/headers/link"
        x-github:
          githubCloudOnly: false
          enabledForGitHubApps: true
          category: actions
          subcategory: workflows
    /repos/{owner}/{repo}/actions/workflows/{workflow_id}/runs:
      get:
        summary: List workflow runs for a workflow
        description: |
          "List all workflow runs for a workflow. You can replace `workflow_id`
          with the workflow file name. For example, you could use `main.yaml`. You can
          use parameters to narrow the list of results. For more information about using
          parameters, see [Parameters](https://docs.github.com/rest/guides/getting-started-with-the-rest-api#parameters).
          Anyone with read access to the repository can use this endpoint
          OAuth app tokens and personal access tokens (classic) need the `repo` scope
          to use this endpoint with a private repository."
        tags:
          - actions
        operationId: actions/list-workflow-runs
        externalDocs:
          description: API method documentation
          url: https://docs.github.com/rest/actions/workflow-runs#list-workflow-runs-for-a-workflow
        parameters:
          - $ref: "#/components/parameters/owner"
          - $ref: "#/components/parameters/repo"
          - $ref: "#/components/parameters/workflow-id"
          - $ref: "#/components/parameters/actor"
          - $ref: "#/components/parameters/workflow-run-branch"
          - $ref: "#/components/parameters/event"
          - $ref: "#/components/parameters/workflow-run-status"
          - $ref: "#/components/parameters/per-page"
          - $ref: "#/components/parameters/page"
          - $ref: "#/components/parameters/created"
          - $ref: "#/components/parameters/exclude-pull-requests"
          - $ref: "#/components/parameters/workflow-run-check-suite-id"
          - $ref: "#/components/parameters/workflow-run-head-sha"
        responses:
          "200":
            description: Response
            content:
              application/json:
                schema:
                  type: object
                  required:
                    - total_count
                    - workflow_runs
                  properties:
                    total_count:
                      type: integer
                    workflow_runs:
                      type: array
                      items:
                        $ref: "#/components/schemas/workflow-run"
            headers:
              Link:
                $ref: "#/components/headers/link"
        x-github:
          githubCloudOnly: false
          enabledForGitHubApps: true
          category: actions
          subcategory: workflow-runs
  components:
    schemas:
      pull-request-minimal:
        title: Pull Request Minimal
        type: object
        properties:
          id:
            type: integer
            format: int64
          number:
            type: integer
          url:
            type: string
          head:
            type: object
            properties:
              ref:
                type: string
              sha:
                type: string
              repo:
                type: object
                properties:
                  id:
                    type: integer
                    format: int64
                  url:
                    type: string
                  name:
                    type: string
                required:
                  - id
                  - url
                  - name
            required:
              - ref
              - sha
              - repo
          base:
            type: object
            properties:
              ref:
                type: string
              sha:
                type: string
              repo:
                type: object
                properties:
                  id:
                    type: integer
                    format: int64
                  url:
                    type: string
                  name:
                    type: string
                required:
                  - id
                  - url
                  - name
            required:
              - ref
              - sha
              - repo
        required:
          - id
          - number
          - url
          - head
          - base
      nullable-simple-commit:
        title: Simple Commit
        description: A commit.
        type: object
        properties:
          id:
            type: string
            description: SHA for the commit
            example: 7638417db6d59f3c431d3e1f261cc637155684cd
          tree_id:
            type: string
            description: SHA for the commit's tree
          message:
            description: Message describing the purpose of the commit
            example: "Fix #42"
            type: string
          timestamp:
            description: Timestamp of the commit
            example: "2014-08-09T08:02:04+12:00"
            format: date-time
            type: string
          author:
            type: object
            description: Information about the Git author
            properties:
              name:
                description: Name of the commit's author
                example: Monalisa Octocat
                type: string
              email:
                description: Git email address of the commit's author
                example: monalisa.octocat@example.com
                type: string
                format: email
            required:
              - name
              - email
            nullable: true
          committer:
            type: object
            description: Information about the Git committer
            properties:
              name:
                description: Name of the commit's committer
                example: Monalisa Octocat
                type: string
              email:
                description: Git email address of the commit's committer
                example: monalisa.octocat@example.com
                type: string
                format: email
            required:
              - name
              - email
            nullable: true
        required:
          - id
          - tree_id
          - message
          - timestamp
          - author
          - committer
        nullable: true
      workflow:
        title: Workflow
        description: A GitHub Actions workflow
        type: object
        properties:
          id:
            type: integer
            example: 5
          node_id:
            type: string
            example: MDg6V29ya2Zsb3cxMg==
          name:
            type: string
            example: CI
          path:
            type: string
            example: ruby.yaml
          state:
            type: string
            example: active
            enum:
              - active
              - deleted
              - disabled_fork
              - disabled_inactivity
              - disabled_manually
          created_at:
            type: string
            format: date-time
            example: "2019-12-06T14:20:20.000Z"
          updated_at:
            type: string
            format: date-time
            example: "2019-12-06T14:20:20.000Z"
          url:
            type: string
            example: https://api.github.com/repos/actions/setup-ruby/workflows/5
          html_url:
            type: string
            example: https://github.com/actions/setup-ruby/blob/master/.github/workflows/ruby.yaml
          badge_url:
            type: string
            example: https://github.com/actions/setup-ruby/workflows/CI/badge.svg
          deleted_at:
            type: string
            format: date-time
            example: "2019-12-06T14:20:20.000Z"
        required:
          - id
          - node_id
          - name
          - path
          - state
          - url
          - html_url
          - badge_url
          - created_at
          - updated_at
      simple-user:
        title: Simple User
        description: A GitHub user.
        type: object
        properties:
          name:
            nullable: true
            type: string
          email:
            nullable: true
            type: string
          login:
            type: string
            example: octocat
          id:
            type: integer
            format: int64
            example: 1
          node_id:
            type: string
            example: MDQ6VXNlcjE=
          avatar_url:
            type: string
            format: uri
            example: https://github.com/images/error/octocat_happy.gif
          gravatar_id:
            type: string
            example: 41d064eb2195891e12d0413f63227ea7
            nullable: true
          url:
            type: string
            format: uri
            example: https://api.github.com/users/octocat
          html_url:
            type: string
            format: uri
            example: https://github.com/octocat
          followers_url:
            type: string
            format: uri
            example: https://api.github.com/users/octocat/followers
          following_url:
            type: string
            example: https://api.github.com/users/octocat/following{/other_user}
          gists_url:
            type: string
            example: https://api.github.com/users/octocat/gists{/gist_id}
          starred_url:
            type: string
            example: https://api.github.com/users/octocat/starred{/owner}{/repo}
          subscriptions_url:
            type: string
            format: uri
            example: https://api.github.com/users/octocat/subscriptions
          organizations_url:
            type: string
            format: uri
            example: https://api.github.com/users/octocat/orgs
          repos_url:
            type: string
            format: uri
            example: https://api.github.com/users/octocat/repos
          events_url:
            type: string
            example: https://api.github.com/users/octocat/events{/privacy}
          received_events_url:
            type: string
            format: uri
            example: https://api.github.com/users/octocat/received_events
          type:
            type: string
            example: User
          site_admin:
            type: boolean
          starred_at:
            type: string
            example: '"2020-07-09T00:17:55Z"'
        required:
          - avatar_url
          - events_url
          - followers_url
          - following_url
          - gists_url
          - gravatar_id
          - html_url
          - id
          - node_id
          - login
          - organizations_url
          - received_events_url
          - repos_url
          - site_admin
          - starred_url
          - subscriptions_url
          - type
          - url
      workflow-run:
        title: Workflow Run
        description: An invocation of a workflow
        type: object
        properties:
          id:
            type: integer
            format: int64
            description: The ID of the workflow run.
            example: 5
          name:
            type: string
            description: The name of the workflow run.
            nullable: true
            example: Build
          node_id:
            type: string
            example: MDEwOkNoZWNrU3VpdGU1
          check_suite_id:
            type: integer
            format: int64
            description: The ID of the associated check suite.
            example: 42
          check_suite_node_id:
            type: string
            description: The node ID of the associated check suite.
            example: MDEwOkNoZWNrU3VpdGU0Mg==
          head_branch:
            type: string
            nullable: true
            example: master
          head_sha:
            description: |
              The SHA of the head commit that points to the version of the
              workflow being run.
            example: "009b8a3a9ccbb128af87f9b1c0f4c62e8a304f6d"
            type: string
          path:
            description: The full path of the workflow
            example: octocat/octo-repo/.github/workflows/ci.yml@main
            type: string
          run_number:
            type: integer
            description: The auto incrementing run number for the workflow run.
            example: 106
          run_attempt:
            type: integer
            description: |
              Attempt number of the run, 1 for first attempt and higher if
              the workflow was re-run.
            example: 1
          referenced_workflows:
            type: array
            nullable: true
            items:
              "$ref": "#/components/schemas/referenced-workflow"
          event:
            type: string
            example: push
          status:
            type: string
            nullable: true
            example: completed
          conclusion:
            type: string
            nullable: true
            example: neutral
          workflow_id:
            type: integer
            description: The ID of the parent workflow.
            example: 5
          url:
            type: string
            description: The URL to the workflow run.
            example: https://api.github.com/repos/github/hello-world/actions/runs/5
          html_url:
            type: string
            example: https://github.com/github/hello-world/suites/4
          pull_requests:
            description: Pull requests that are open with a `head_sha` or `head_branch` that matches the workflow run. The returned pull requests do not necessarily indicate pull requests that triggered the run.
            type: array
            nullable: true
            items:
              "$ref": "#/components/schemas/pull-request-minimal"
          created_at:
            type: string
            format: date-time
          updated_at:
            type: string
            format: date-time
          actor:
            "$ref": "#/components/schemas/simple-user"
          triggering_actor:
            "$ref": "#/components/schemas/simple-user"
          run_started_at:
            type: string
            format: date-time
            description: The start time of the latest run. Resets on re-run.
          jobs_url:
            description: The URL to the jobs for the workflow run.
            type: string
            example: https://api.github.com/repos/github/hello-world/actions/runs/5/jobs
          logs_url:
            description: The URL to download the logs for the workflow run.
            type: string
            example: https://api.github.com/repos/github/hello-world/actions/runs/5/logs
          check_suite_url:
            description: The URL to the associated check suite.
            type: string
            example: https://api.github.com/repos/github/hello-world/check-suites/12
          artifacts_url:
            description: The URL to the artifacts for the workflow run.
            type: string
            example: https://api.github.com/repos/github/hello-world/actions/runs/5/rerun/artifacts
          cancel_url:
            description: The URL to cancel the workflow run.
            type: string
            example: https://api.github.com/repos/github/hello-world/actions/runs/5/cancel
          rerun_url:
            description: The URL to rerun the workflow run.
            type: string
            example: https://api.github.com/repos/github/hello-world/actions/runs/5/rerun
          previous_attempt_url:
            nullable: true
            description: The URL to the previous attempted run of this workflow, if one exists.
            type: string
            example: https://api.github.com/repos/github/hello-world/actions/runs/5/attempts/3
          workflow_url:
            description: The URL to the workflow.
            type: string
            example: https://api.github.com/repos/github/hello-world/actions/workflows/main.yaml
          head_commit:
            "$ref": "#/components/schemas/nullable-simple-commit"
          repository:
            "$ref": "#/components/schemas/minimal-repository"
          head_repository:
            "$ref": "#/components/schemas/minimal-repository"
          head_repository_id:
            type: integer
            example: 5
          display_title:
            type: string
            example: Simple Workflow
            description: |
              The event-specific title associated with the run or the run-name
              if set, or the value of `run-name` if it is set in the workflow.
        required:
          - id
          - node_id
          - head_branch
          - run_number
          - display_title
          - event
          - status
          - conclusion
          - head_sha
          - path
          - workflow_id
          - url
          - html_url
          - created_at
          - updated_at
          - head_commit
          - head_repository
          - repository
          - jobs_url
          - logs_url
          - check_suite_url
          - cancel_url
          - rerun_url
          - artifacts_url
          - workflow_url
          - pull_requests
      referenced-workflow:
        title: Referenced workflow
        description: A workflow referenced/reused by the initial caller workflow
        type: object
        properties:
          path:
            type: string
          sha:
            type: string
          ref:
            type: string
        required:
          - path
          - sha
      minimal-repository:
        title: Minimal Repository
        description: Minimal Repository
        type: object
        properties:
          id:
            type: integer
            format: int64
            example: 1296269
          node_id:
            type: string
            example: MDEwOlJlcG9zaXRvcnkxMjk2MjY5
          name:
            type: string
            example: Hello-World
          full_name:
            type: string
            example: octocat/Hello-World
          owner:
            "$ref": "#/components/schemas/simple-user"
          private:
            type: boolean
          html_url:
            type: string
            format: uri
            example: https://github.com/octocat/Hello-World
          description:
            type: string
            example: This your first repo!
            nullable: true
          fork:
            type: boolean
          url:
            type: string
            format: uri
            example: https://api.github.com/repos/octocat/Hello-World
          archive_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/{archive_format}{/ref}
          assignees_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/assignees{/user}
          blobs_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/git/blobs{/sha}
          branches_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/branches{/branch}
          collaborators_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/collaborators{/collaborator}
          comments_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/comments{/number}
          commits_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/commits{/sha}
          compare_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/compare/{base}...{head}
          contents_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/contents/{+path}
          contributors_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/contributors
          deployments_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/deployments
          downloads_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/downloads
          events_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/events
          forks_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/forks
          git_commits_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/git/commits{/sha}
          git_refs_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/git/refs{/sha}
          git_tags_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/git/tags{/sha}
          git_url:
            type: string
          issue_comment_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/issues/comments{/number}
          issue_events_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/issues/events{/number}
          issues_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/issues{/number}
          keys_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/keys{/key_id}
          labels_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/labels{/name}
          languages_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/languages
          merges_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/merges
          milestones_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/milestones{/number}
          notifications_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/notifications{?since,all,participating}
          pulls_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/pulls{/number}
          releases_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/releases{/id}
          ssh_url:
            type: string
          stargazers_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/stargazers
          statuses_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/statuses/{sha}
          subscribers_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/subscribers
          subscription_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/subscription
          tags_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/tags
          teams_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/teams
          trees_url:
            type: string
            example: http://api.github.com/repos/octocat/Hello-World/git/trees{/sha}
          clone_url:
            type: string
          mirror_url:
            type: string
            nullable: true
          hooks_url:
            type: string
            format: uri
            example: http://api.github.com/repos/octocat/Hello-World/hooks
          svn_url:
            type: string
          homepage:
            type: string
            nullable: true
          language:
            type: string
            nullable: true
          forks_count:
            type: integer
          stargazers_count:
            type: integer
          watchers_count:
            type: integer
          size:
            description: The size of the repository, in kilobytes. Size is calculated hourly. When a repository is initially created, the size is 0.
            type: integer
          default_branch:
            type: string
          open_issues_count:
            type: integer
          is_template:
            type: boolean
          topics:
            type: array
            items:
              type: string
          has_issues:
            type: boolean
          has_projects:
            type: boolean
          has_wiki:
            type: boolean
          has_pages:
            type: boolean
          has_downloads:
            type: boolean
          has_discussions:
            type: boolean
          archived:
            type: boolean
          disabled:
            type: boolean
          visibility:
            type: string
          pushed_at:
            type: string
            format: date-time
            example: "2011-01-26T19:06:43Z"
            nullable: true
          created_at:
            type: string
            format: date-time
            example: "2011-01-26T19:01:12Z"
            nullable: true
          updated_at:
            type: string
            format: date-time
            example: "2011-01-26T19:14:43Z"
            nullable: true
          permissions:
            type: object
            properties:
              admin:
                type: boolean
              maintain:
                type: boolean
              push:
                type: boolean
              triage:
                type: boolean
              pull:
                type: boolean
          role_name:
            type: string
            example: admin
          temp_clone_token:
            type: string
          delete_branch_on_merge:
            type: boolean
          subscribers_count:
            type: integer
          network_count:
            type: integer
          code_of_conduct:
            "$ref": "#/components/schemas/code-of-conduct"
          license:
            type: object
            properties:
              key:
                type: string
              name:
                type: string
              spdx_id:
                type: string
              url:
                type: string
              node_id:
                type: string
            nullable: true
          forks:
            type: integer
            example: 0
          open_issues:
            type: integer
            example: 0
          watchers:
            type: integer
            example: 0
          allow_forking:
            type: boolean
          web_commit_signoff_required:
            type: boolean
            example: false
          security_and_analysis:
            "$ref": "#/components/schemas/security-and-analysis"
        required:
          - archive_url
          - assignees_url
          - blobs_url
          - branches_url
          - collaborators_url
          - comments_url
          - commits_url
          - compare_url
          - contents_url
          - contributors_url
          - deployments_url
          - description
          - downloads_url
          - events_url
          - fork
          - forks_url
          - full_name
          - git_commits_url
          - git_refs_url
          - git_tags_url
          - hooks_url
          - html_url
          - id
          - node_id
          - issue_comment_url
          - issue_events_url
          - issues_url
          - keys_url
          - labels_url
          - languages_url
          - merges_url
          - milestones_url
          - name
          - notifications_url
          - owner
          - private
          - pulls_url
          - releases_url
          - stargazers_url
          - statuses_url
          - subscribers_url
          - subscription_url
          - tags_url
          - teams_url
          - trees_url
          - url
      security-and-analysis:
        nullable: true
        type: object
        properties:
          advanced_security:
            type: object
            properties:
              status:
                type: string
                enum:
                  - enabled
                  - disabled
          dependabot_security_updates:
            description: Enable or disable Dependabot security updates for the repository.
            type: object
            properties:
              status:
                description: The enablement status of Dependabot security updates for the repository.
                type: string
                enum:
                  - enabled
                  - disabled
          secret_scanning:
            type: object
            properties:
              status:
                type: string
                enum:
                  - enabled
                  - disabled
          secret_scanning_push_protection:
            type: object
            properties:
              status:
                type: string
                enum:
                  - enabled
                  - disabled
          secret_scanning_non_provider_patterns:
            type: object
            properties:
              status:
                type: string
                enum:
                  - enabled
                  - disabled
      code-of-conduct:
        title: Code Of Conduct
        description: Code Of Conduct
        type: object
        properties:
          key:
            type: string
            example: contributor_covenant
          name:
            type: string
            example: Contributor Covenant
          url:
            type: string
            format: uri
            example: https://api.github.com/codes_of_conduct/contributor_covenant
          body:
            type: string
            example: |
              # Contributor Covenant Code of Conduct
              ## Our Pledge
              In the interest of fostering an open and welcoming environment, we as contributors and maintainers pledge to making participation in our project and our community a harassment-free experience for everyone, regardless of age, body size, disability, ethnicity, gender identity and expression, level of experience, nationality, personal appearance, race, religion, or sexual identity and orientation.
              ## Our Standards
              Examples of behavior that contributes to creating a positive environment include:
              * Using welcoming and inclusive language
              * Being respectful of differing viewpoints and experiences
              * Gracefully accepting constructive criticism
              * Focusing on what is best for the community
              * Showing empathy towards other community members
              Examples of unacceptable behavior by participants include:
              * The use of sexualized language or imagery and unwelcome sexual attention or advances
              * Trolling, insulting/derogatory comments, and personal or political attacks
              * Public or private harassment
              * Publishing others' private information, such as a physical or electronic address, without explicit permission
              * Other conduct which could reasonably be considered inappropriate in a professional setting
              ## Our Responsibilities
              Project maintainers are responsible for clarifying the standards of acceptable behavior and are expected to take appropriate and fair corrective action in response
                                to any instances of unacceptable behavior.
              Project maintainers have the right and responsibility to remove, edit, or reject comments, commits, code, wiki edits, issues, and other contributions that are not aligned to this Code of Conduct, or to ban temporarily or permanently any contributor for other behaviors that they deem inappropriate, threatening, offensive, or harmful.
              ## Scope
              This Code of Conduct applies both within project spaces and in public spaces when an individual is representing the project or its community. Examples of representing a project or community include using an official project e-mail address,
                                posting via an official social media account, or acting as an appointed representative at an online or offline event. Representation of a project may be further defined and clarified by project maintainers.
              ## Enforcement
              Instances of abusive, harassing, or otherwise unacceptable behavior may be reported by contacting the project team at [EMAIL]. The project team will review and investigate all complaints, and will respond in a way that it deems appropriate to the circumstances. The project team is obligated to maintain confidentiality with regard to the reporter of an incident. Further details of specific enforcement policies may be posted separately.
              Project maintainers who do not follow or enforce the Code of Conduct in good faith may face temporary or permanent repercussions as determined by other members of the project's leadership.
              ## Attribution
              This Code of Conduct is adapted from the [Contributor Covenant](http://contributor-covenant.org), version 1.4, available at [http://contributor-covenant.org/version/1/4](http://contributor-covenant.org/version/1/4/).
          html_url:
            type: string
            format: uri
            nullable: true
        required:
          - url
          - html_url
          - key
          - name
    parameters:
      owner:
        name: owner
        description: The account owner of the repository. The name is not case sensitive.
        in: path
        required: true
        schema:
          type: string
      repo:
        name: repo
        description: The name of the repository without the `.git` extension. The name is not case sensitive.
        in: path
        required: true
        schema:
          type: string
      per-page:
        name: per_page
        description: The number of results per page (max 100). For more information, see "[Using pagination in the REST API](https://docs.github.com/rest/using-the-rest-api/using-pagination-in-the-rest-api)."
        in: query
        schema:
          type: integer
          default: 30
      page:
        name: page
        description: The page number of the results to fetch. For more information, see "[Using pagination in the REST API](https://docs.github.com/rest/using-the-rest-api/using-pagination-in-the-rest-api)."
        in: query
        schema:
          type: integer
          default: 1
      workflow-id:
        name: workflow_id
        in: path
        description: The ID of the workflow.
        required: true
        schema:
          type: integer
      actor:
        name: actor
        description: Returns someone's workflow runs. Use the login for the user who created the `push` associated with the check suite or workflow run.
        in: query
        required: false
        schema:
          type: string
      workflow-run-branch:
        name: branch
        description: Returns workflow runs associated with a branch. Use the name of the branch of the `push`.
        in: query
        required: false
        schema:
          type: string
      event:
        name: event
        description: Returns workflow run triggered by the event you specify. For example, `push`, `pull_request` or `issue`. For more information, see "[Events that trigger workflows](https://docs.github.com/actions/automating-your-workflow-with-github-actions/events-that-trigger-workflows)."
        in: query
        required: false
        schema:
          type: string
      workflow-run-status:
        name: status
        description: Returns workflow runs with the check run `status` or `conclusion` that you specify. For example, a conclusion can be `success` or a status can be `in_progress`. Only GitHub Actions can set a status of `waiting`, `pending`, or `requested`.
        in: query
        required: false
        schema:
          type: string
          enum:
            - completed
            - action_required
            - cancelled
            - failure
            - neutral
            - skipped
            - stale
            - success
            - timed_out
            - in_progress
            - queued
            - requested
            - waiting
            - pending
      created:
        name: created
        description: Returns workflow runs created within the given date-time range. For more information on the syntax, see "[Understanding the search syntax](https://docs.github.com/search-github/getting-started-with-searching-on-github/understanding-the-search-syntax#query-for-dates)."
        in: query
        required: false
        schema:
          type: string
          format: date-time
      exclude-pull-requests:
        name: exclude_pull_requests
        description: If `true` pull requests are omitted from the response (empty array).
        in: query
        required: false
        schema:
          type: boolean
          default: false
      workflow-run-check-suite-id:
        name: check_suite_id
        description: Returns workflow runs with the `check_suite_id` that you specify.
        in: query
        schema:
          type: integer
          format: int64
      workflow-run-head-sha:
        name: head_sha
        description: Only returns workflow runs that are associated with the specified `head_sha`.
        in: query
        required: false
        schema:
          type: string
    headers:
      link:
        example: <https://api.github.com/resource?page=2>; rel="next", <https://api.github.com/resource?page=5>; rel="last"
        schema:
          type: string
      content-type:
        example: text/html
        schema:
          type: string
      x-common-marker-version:
        example: 0.17.4
        schema:
          type: string
      x-rate-limit-limit:
        example: 5000
        schema:
          type: integer
      x-rate-limit-remaining:
        example: 4999
        schema:
          type: integer
      x-rate-limit-reset:
        example: 1590701888
        schema:
          type: integer
          format: timestamp
      location:
        example: https://pipelines.actions.githubusercontent.com/OhgS4QRKqmgx7bKC27GKU83jnQjyeqG8oIMTge8eqtheppcmw8/_apis/pipelines/1/runs/176/signedlogcontent?urlExpires=2020-01-24T18%3A10%3A31.5729946Z&urlSigningMethod=HMACV1&urlSignature=agG73JakPYkHrh06seAkvmH7rBR4Ji4c2%2B6a2ejYh3E%3D
        schema:
          type: string
    securitySchemes:
      BearerToken:
        type: http
        scheme: bearer
</code>
</pre>
</div>

The JSON object hierarchy can be described in the following diagram:
```
openapi
info
├── title
└── version
servers
security
paths
├── path_1
└── path_2
components
├── schemas
│   ├── shema_1
│   └── schema_2
├── securitySchemas
├── parameters
└── headers
```

The document begins by specifying the openAPI version, an info object, and specifying the servers.
```
openapi: 3.0.3
info:
  version: 1.1.4
  title: GitHub v3 REST API
  description: |-
    |
    GitHub's v3 REST API.
  license:
    name: MIT
    url: https://spdx.org/licenses/MIT
  termsOfService: https://docs.github.com/articles/github-terms-of-service
  contact:
    name: Support
    url: https://support.github.com/contact?tags=dotcom-rest-api
  x-github-plan: api.github.com
servers:
  - url: https://api.github.com/
security:
  - BearerToken: []
 ```

The document will then display the system's paths, and must include at least one path.
Under each path, the document specifies the path URI, an HTTP method, parameters (if needed), the requestBody, and defines the responses.
Note that both the request and the responses can reference a component from a different part of the document. This will be important when we try to extract the document to our needed purposes.

Moreover, it's good to notice now that references to components from some paths may trigger more refernces, as some components reference other components within themselves.

An example for a path structure can be seen here:

<div style="height: 200px; overflow-x: auto;">
<pre>
<code>
paths:
  /repos/{owner}/{repo}/actions/workflows/{workflow_id}/runs:
      get:
        summary: List workflow runs for a workflow
        description: |
          "List all workflow runs for a workflow. You can replace `workflow_id`
          with the workflow file name. For example, you could use `main.yaml`. You can
          use parameters to narrow the list of results. For more information about using
          parameters, see [Parameters](https://docs.github.com/rest/guides/getting-started-with-the-rest-api#parameters).
          Anyone with read access to the repository can use this endpoint
          OAuth app tokens and personal access tokens (classic) need the `repo` scope
          to use this endpoint with a private repository."
        tags:
          - actions
        operationId: actions/list-workflow-runs
        externalDocs:
          description: API method documentation
          url: https://docs.github.com/rest/actions/workflow-runs#list-workflow-runs-for-a-workflow
        parameters:
          - $ref: "#/components/parameters/owner"
          - $ref: "#/components/parameters/repo"
          - $ref: "#/components/parameters/workflow-id"
          - $ref: "#/components/parameters/actor"
          - $ref: "#/components/parameters/workflow-run-branch"
          - $ref: "#/components/parameters/event"
          - $ref: "#/components/parameters/workflow-run-status"
          - $ref: "#/components/parameters/per-page"
          - $ref: "#/components/parameters/page"
          - $ref: "#/components/parameters/created"
          - $ref: "#/components/parameters/exclude-pull-requests"
          - $ref: "#/components/parameters/workflow-run-check-suite-id"
          - $ref: "#/components/parameters/workflow-run-head-sha"
        responses:
          "200":
            description: Response
            content:
              application/json:
                schema:
                  type: object
                  required:
                    - total_count
                    - workflow_runs
                  properties:
                    total_count:
                      type: integer
                    workflow_runs:
                      type: array
                      items:
                        $ref: "#/components/schemas/workflow-run"
            headers:
              Link:
                $ref: "#/components/headers/link"
        x-github:
          githubCloudOnly: false
          enabledForGitHubApps: true
          category: actions
          subcategory: workflow-runs
</code>
</pre>
</div>

Lastly, The document specifies the components it uses in its paths.
A component can be a schema for any resource, some parameter specification object, HTTP headers and more. To get a better understanding, [see some openAPI official examples.](https://github.com/OAI/OpenAPI-Specification/tree/main/examples/v3.0)
As mentioned before, some components can also reference other components. Let's take a look in the 'workflow-run' component.
<div style="height: 200px; overflow-x: auto;">
<pre>
<code>
  workflow-run:
    title: Workflow Run
    description: An invocation of a workflow
    type: object
    properties:
      id:
        type: integer
        format: int64
        description: The ID of the workflow run.
        example: 5
      name:
        type: string
        description: The name of the workflow run.
        nullable: true
        example: Build
      node_id:
        type: string
        example: MDEwOkNoZWNrU3VpdGU1
      check_suite_id:
        type: integer
        format: int64
        description: The ID of the associated check suite.
        example: 42
      check_suite_node_id:
        type: string
        description: The node ID of the associated check suite.
        example: MDEwOkNoZWNrU3VpdGU0Mg==
      head_branch:
        type: string
        nullable: true
        example: master
      head_sha:
        description: |
          The SHA of the head commit that points to the version of the
          workflow being run.
        example: "009b8a3a9ccbb128af87f9b1c0f4c62e8a304f6d"
        type: string
      path:
        description: The full path of the workflow
        example: octocat/octo-repo/.github/workflows/ci.yml@main
        type: string
      run_number:
        type: integer
        description: The auto incrementing run number for the workflow run.
        example: 106
      run_attempt:
        type: integer
        description: |
          Attempt number of the run, 1 for first attempt and higher if
          the workflow was re-run.
        example: 1
      referenced_workflows:
        type: array
        nullable: true
        items:
          "$ref": "#/components/schemas/referenced-workflow"
      event:
        type: string
        example: push
      status:
        type: string
        nullable: true
        example: completed
      conclusion:
        type: string
        nullable: true
        example: neutral
      workflow_id:
        type: integer
        description: The ID of the parent workflow.
        example: 5
      url:
        type: string
        description: The URL to the workflow run.
        example: https://api.github.com/repos/github/hello-world/actions/runs/5
      html_url:
        type: string
        example: https://github.com/github/hello-world/suites/4
      pull_requests:
        description: Pull requests that are open with a `head_sha` or `head_branch` that matches the workflow run. The returned pull requests do not necessarily indicate pull requests that triggered the run.
        type: array
        nullable: true
        items:
          "$ref": "#/components/schemas/pull-request-minimal"
      created_at:
        type: string
        format: date-time
      updated_at:
        type: string
        format: date-time
      actor:
        "$ref": "#/components/schemas/simple-user"
      triggering_actor:
        "$ref": "#/components/schemas/simple-user"
      run_started_at:
        type: string
        format: date-time
        description: The start time of the latest run. Resets on re-run.
      jobs_url:
        description: The URL to the jobs for the workflow run.
        type: string
        example: https://api.github.com/repos/github/hello-world/actions/runs/5/jobs
      logs_url:
        description: The URL to download the logs for the workflow run.
        type: string
        example: https://api.github.com/repos/github/hello-world/actions/runs/5/logs
      check_suite_url:
        description: The URL to the associated check suite.
        type: string
        example: https://api.github.com/repos/github/hello-world/check-suites/12
      artifacts_url:
        description: The URL to the artifacts for the workflow run.
        type: string
        example: https://api.github.com/repos/github/hello-world/actions/runs/5/rerun/artifacts
      cancel_url:
        description: The URL to cancel the workflow run.
        type: string
        example: https://api.github.com/repos/github/hello-world/actions/runs/5/cancel
      rerun_url:
        description: The URL to rerun the workflow run.
        type: string
        example: https://api.github.com/repos/github/hello-world/actions/runs/5/rerun
      previous_attempt_url:
        nullable: true
        description: The URL to the previous attempted run of this workflow, if one exists.
        type: string
        example: https://api.github.com/repos/github/hello-world/actions/runs/5/attempts/3
      workflow_url:
        description: The URL to the workflow.
        type: string
        example: https://api.github.com/repos/github/hello-world/actions/workflows/main.yaml
      head_commit:
        "$ref": "#/components/schemas/nullable-simple-commit"
      repository:
        "$ref": "#/components/schemas/minimal-repository"
      head_repository:
        "$ref": "#/components/schemas/minimal-repository"
      head_repository_id:
        type: integer
        example: 5
      display_title:
        type: string
        example: Simple Workflow
        description: |
          The event-specific title associated with the run or the run-name
          if set, or the value of `run-name` if it is set in the workflow.
    required:
      - id
      - node_id
      - head_branch
      - run_number
      - display_title
      - event
      - status
      - conclusion
      - head_sha
      - path
      - workflow_id
      - url
      - html_url
      - created_at
      - updated_at
      - head_commit
      - head_repository
      - repository
      - jobs_url
      - logs_url
      - check_suite_url
      - cancel_url
      - rerun_url
      - artifacts_url
      - workflow_url
      - pull_requests
</code>
</pre>
</div>

As we can see, the 'workflow-run' component is referencing other schemas in the document. This becomes important if you wish to extract an openAPI document to a smaller size, as you would need to track all recursive references and include all components that got mentioned in the way.

Note the securitySchemas object, which specifies the authentication method used:
```
securitySchemes:
   BearerToken:
     type: http
     scheme: bearer
```

## How to extract an OpenAPI Document to workable size?
### Manual and logical approach
Let's say we have an outline of a serverless workflow, and a very large openAPI document that exposes many system resources, much more than we intend on using.

The procedure for extracting the openAPI document can be logically described as the following:

1. Copy any ‘openapi’, ‘info’, ‘security’, ‘servers’ objects to the new document.
2. Identify all system resources that are needed for the workflow.
3. For each resource, locate the path object  that returns the needed resource.
4. For each path, create a dependency tree of all components referenced by the path, and of all components referenced by those components, recursively.
4. Include the securitySchema of the original ‘components’ object.
5. Include any webhooks the application may need.

This procedure can be quite tedious when done manually, so some efforts were made to automate the process.

### Python script to extract documents

The [following script](https://github.com/rhdhorchestrator/serverless-workflows/blob/v1.4.x/hack/filter-openapi-specs.py) can be used to extract reduced openAPI documents from larger ones.
The script takes an input file, and output file, and a list of tuples (path objects, http method).
```
python extract.py openapi_spec.json filtered_openapi_spec.json \
"/apis/apps/v1/namespaces/{namespace}/deployments post" \
"/apis/apps/v1/namespaces/{namespace}/deployments/{name} get" \
"/api/v1/namespaces/{namespace}/services post" \
"/apis/route.openshift.io/v1/namespaces/{namespace}/routes post" \
"/apis/route.openshift.io/v1/namespaces/{namespace}/routes/{name} get" \
"/apis/route.openshift.io/v1/namespaces/{namespace}/routes/{name}/status get"
```
For each path, it extracts all dependencies (refs) that come with it in a recursive way.
* Note that currently some references may be missed by the script.
* The script accepts a JSON file type as input.

Some upcoming efforts will look into integrating this logic natively in the kn-workflow CLI.

### Use kn-workflow version greater than 1.35
`kn-workflow` introduces a new feature to minify the OpenAPI spec file to the used paths and components:
```
kn-workflow specs minify --help

	Minification of OpenAPI specs:
	Minification allows us to reduce the size of an OpenAPI spec file, which is essential given the maximum YAML
	size supported by Kubernetes is limited to 3,145,728 bytes.

	Note: right now only OpenAPI specs are supported for minification, see examples below.
	

Usage:
  kn workflow specs minify [command]

Examples:

	#Minify the workflow project's OpenAPI spec file located in the current project.
	kn workflow specs minify openapi
		

Available Commands:
  openapi     Minify the openAPI spec files to trim operations only used by the workflows

Flags:
  -h, --help   help for minify

Use "kn workflow specs minify [command] --help" for more information about a command.
```

## Best Practices and Warnings:
**Document size:** Even though the Quarkus engine has a YAML input file size limit of 3MB, the `kn-workflow` CLI generates k8s resources (configmaps) for spec/schema files.
K8s has a default size limit of ~1MB per resource, and by default they get applied to all as part of the helm charts created.
Therefore, it is best practice to keep the extracted openAPI documents to under 1MB, until this practice is changed.

<a name="appendixA"></a>

### Appendix:

#### Input file size restriction errors:

Quarkus uses [snakeYAML](https://bitbucket.org/snakeyaml/snakeyaml/src/master/) to process the input YAML files in a serverless workflow. A default setting of a maximum size limit has been put in place, which inhibits accepting large files.
```
Caused by: org.yaml.snakeyaml.error.YAMLException: The incoming YAML document exceeds the limit: 3145728 code points.
[...]
[ERROR] Error parsing content
com.fasterxml.jackson.dataformat.yaml.JacksonYAMLParseException: The incoming YAML document exceeds the limit: 3145728 code points.
 at [Source: (StringReader); line: 82506, column: 15]
 ```

 For reference, a [reproducer](https://github.com/ElaiShalevRH/reproducers) was creater for this issue.

<a name="appendixB"></a>

#### Size restriction workaround:
By passing the following parameter to the Java build environment, it is possible to set the maximum YAML input file size limit.
```
-DmaxYamlCodePoints=99999999
```
The CodePoints parameter refers to Unicode points, which are essentially the number of characters in the input file (for UTF-8 encoding)
The maximum code point number is the integer max, 2^31-1.


