# test-workflow-runs-actions

This repository contains GitHub Actions workflows to test the `workflow_run` trigger functionality.

## Workflows

### CI Workflow (`.github/workflows/ci.yml`)
- **Triggers**: 
  - Runs on push to `main` branch
  - Can be manually triggered via `workflow_dispatch`
- **Purpose**: Simulates CI workflow for testing with configurable success/failure status
- **Behavior**: 
  - By default, prints "Running test CI" and succeeds (`exit 0`)
  - When triggered manually via workflow_dispatch, accepts a `should_fail` input parameter:
    - `false` (default): Workflow succeeds
    - `true`: Workflow fails (`exit 1`)

### Notify on Failure Workflow (`.github/workflows/notify_on_failure.yml`)
- **Trigger**: Runs when the "CI" workflow completes on `main` branch
- **Purpose**: Demonstrates `workflow_run` trigger with failure condition
- **Behavior**: Only executes when CI fails, prints "CI failed on main → triggering secondary workflow"