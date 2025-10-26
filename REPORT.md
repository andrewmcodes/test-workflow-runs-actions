# Triggering GitHub Actions Workflows on Failure

## Overview

This document explains how to trigger a GitHub Actions workflow when another workflow fails. This pattern is useful for implementing automated notifications, rollback procedures, incident response, or other failure-handling mechanisms.

The key to this functionality is GitHub Actions' `workflow_run` trigger, which allows workflows to be triggered based on the completion status of other workflows.

## How It Works

### The `workflow_run` Trigger

The `workflow_run` event triggers a workflow based on the execution and completion of another workflow. This trigger runs in the context of the default branch (typically `main`), not the branch where the original workflow ran.

**Key Features:**
- Triggers when specified workflows complete, regardless of success or failure
- Can filter by workflow conclusion (success, failure, cancelled, etc.)
- Runs with the permissions of the default branch
- Has access to the triggering workflow's metadata via `github.event.workflow_run`

### Technical Flow

1. **Primary Workflow Executes**: A workflow (e.g., CI tests) runs and completes with a specific conclusion (success, failure, etc.)
2. **Event Generated**: GitHub generates a `workflow_run` event
3. **Secondary Workflow Triggered**: Any workflows configured to listen for this event are triggered
4. **Conditional Execution**: The secondary workflow can use conditionals to run only on specific conclusions (e.g., failure)

## Implementation Guide

### Step 1: Create the Primary Workflow

First, create your primary workflow that may fail. This could be a CI pipeline, deployment workflow, or any other automated task.

**Example: `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  push:
    branches:
      - main

permissions: {}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run test CI
        run: |
          echo "Running test CI"
          exit 1  # Simulates a failure
```

### Step 2: Create the Failure-Triggered Workflow

Create a secondary workflow that triggers when the primary workflow completes.

**Example: `.github/workflows/notify_on_failure.yml`**

```yaml
name: Notify on Failure

on:
  workflow_run:
    workflows: ["CI"]  # Name of the workflow to monitor
    types:
      - completed      # Trigger when the workflow completes
    branches:
      - main          # Only for runs on main branch

permissions: {}

jobs:
  notify:
    runs-on: ubuntu-latest
    # Only run if the workflow failed
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    steps:
      - name: Notify failure
        run: echo "CI failed on main → triggering secondary workflow"
```

### Step 3: Customize the Failure Response

In the secondary workflow, you can implement various responses to the failure:

```yaml
steps:
  - name: Send Slack notification
    uses: slackapi/slack-github-action@v1
    with:
      webhook-url: ${{ secrets.SLACK_WEBHOOK }}
      payload: |
        {
          "text": "🚨 CI failed on ${{ github.event.workflow_run.head_branch }}"
        }
  
  - name: Create incident issue
    uses: actions/github-script@v7
    with:
      script: |
        await github.rest.issues.create({
          owner: context.repo.owner,
          repo: context.repo.repo,
          title: `CI Failure on ${context.payload.workflow_run.head_branch}`,
          body: `Workflow run failed: ${context.payload.workflow_run.html_url}`,
          labels: ['incident', 'automated']
        })
```

## Example Use Cases

### 1. **Automated Incident Notifications**

Notify the team immediately when critical workflows fail:
- Send Slack/Teams/Discord messages
- Create PagerDuty incidents
- Send email alerts to on-call engineers

### 2. **Automated Rollback**

Trigger a rollback workflow when a deployment fails:
```yaml
name: Auto Rollback

on:
  workflow_run:
    workflows: ["Production Deploy"]
    types: [completed]
    branches: [main]

jobs:
  rollback:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    steps:
      - name: Rollback to previous version
        run: ./scripts/rollback.sh
```

### 3. **Failure Investigation**

Automatically gather debugging information when tests fail:
```yaml
name: Collect Debug Info

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]

jobs:
  debug:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
      - name: Analyze logs
        run: ./scripts/analyze-failure.sh
      - name: Post analysis to PR
        run: ./scripts/comment-on-pr.sh
```

### 4. **Metrics and Monitoring**

Track failure rates and patterns:
```yaml
name: Track Failures

on:
  workflow_run:
    workflows: ["CI", "Deploy", "Tests"]
    types: [completed]

jobs:
  metrics:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    steps:
      - name: Send metrics to monitoring system
        run: |
          curl -X POST ${{ secrets.METRICS_ENDPOINT }} \
            -d "workflow=${{ github.event.workflow_run.name }}" \
            -d "status=failure" \
            -d "timestamp=$(date -u +%s)"
```

### 5. **Issue Triage Automation**

Create and label issues for failed workflows:
```yaml
name: Auto-Create Issues

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]

jobs:
  create-issue:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const issue = await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `CI Failed: ${context.payload.workflow_run.display_title}`,
              body: `
                ## Workflow Failure Report
                
                - **Workflow**: ${context.payload.workflow_run.name}
                - **Branch**: ${context.payload.workflow_run.head_branch}
                - **Commit**: ${context.payload.workflow_run.head_sha}
                - **Run URL**: ${context.payload.workflow_run.html_url}
                - **Triggered by**: @${context.payload.workflow_run.triggering_actor.login}
                
                Please investigate this failure.
              `,
              labels: ['ci-failure', 'needs-triage']
            })
```

## Available Workflow Conclusions

The `github.event.workflow_run.conclusion` can have the following values:
- `success` - Workflow completed successfully
- `failure` - Workflow failed
- `cancelled` - Workflow was cancelled
- `skipped` - Workflow was skipped
- `timed_out` - Workflow timed out
- `action_required` - Workflow requires action
- `neutral` - Workflow completed with neutral status

## Important Considerations

### Security

1. **Permissions**: `workflow_run` workflows run in the context of the default branch, which means they have access to secrets even if triggered by a pull request from a fork.
2. **Trust**: Be cautious about what actions you take in response to workflow failures, especially if they involve external systems.
3. **Branch Protection**: Consider using branch protection rules to prevent unauthorized changes to these critical workflows.

### Best Practices

1. **Naming**: Use clear, descriptive workflow names since these are referenced by name in the `workflows` field.
2. **Idempotency**: Ensure your failure-handling actions are idempotent (can be run multiple times safely).
3. **Rate Limiting**: Be mindful of rate limits when sending notifications to external services.
4. **Testing**: Test your failure workflows by intentionally failing the primary workflow in a safe environment.
5. **Logging**: Include detailed logging in failure-handling workflows for troubleshooting.

### Limitations

1. **Workflow Name Dependency**: The trigger is based on the workflow name, not the file name. If you rename the workflow, update all references.
2. **Default Branch Context**: The workflow runs in the context of the default branch, not the branch where the failure occurred.
3. **Delay**: There may be a slight delay between the primary workflow completing and the secondary workflow triggering.
4. **No Direct Data Transfer**: The secondary workflow doesn't have direct access to outputs from the primary workflow, though it can download artifacts.

## Accessing Workflow Run Information

The `github.event.workflow_run` object provides useful information about the failed workflow:

```yaml
- name: Log workflow info
  run: |
    echo "Workflow: ${{ github.event.workflow_run.name }}"
    echo "Conclusion: ${{ github.event.workflow_run.conclusion }}"
    echo "Branch: ${{ github.event.workflow_run.head_branch }}"
    echo "SHA: ${{ github.event.workflow_run.head_sha }}"
    echo "Run ID: ${{ github.event.workflow_run.id }}"
    echo "Run URL: ${{ github.event.workflow_run.html_url }}"
    echo "Actor: ${{ github.event.workflow_run.triggering_actor.login }}"
```

## Testing Your Implementation

1. **Push to Trigger**: Make a commit to the main branch to trigger the primary workflow
2. **Verify Failure**: Ensure the primary workflow fails as expected
3. **Check Secondary Workflow**: Verify that the secondary workflow triggers and executes correctly
4. **Review Logs**: Check the logs to ensure the failure was detected and handled properly

## Additional Resources

- [GitHub Actions: workflow_run event](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run)
- [GitHub Actions: Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [GitHub Actions: Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context)

## Conclusion

The `workflow_run` trigger is a powerful tool for implementing automated failure handling in GitHub Actions. By following the patterns and examples in this document, engineering teams can build robust CI/CD pipelines that automatically respond to failures, improving incident response times and reducing manual intervention.

The workflows in this repository serve as a minimal, working example that can be adapted and extended for your team's specific needs.
