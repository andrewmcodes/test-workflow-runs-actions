# test-workflow-runs-actions

This repository contains GitHub Actions workflows to test the `workflow_run` trigger functionality.

## Workflows

### CI Workflow (`.github/workflows/ci.yml`)
- **Trigger**: Runs on push to `main` branch
- **Purpose**: Simulates a failing CI workflow for testing
- **Behavior**: Prints "Running test CI" and exits with failure (`exit 1`)

### Notify on Failure Workflow (`.github/workflows/notify_on_failure.yml`)
- **Trigger**: Runs when the "CI" workflow completes on `main` branch
- **Purpose**: Demonstrates `workflow_run` trigger with failure condition
- **Behavior**: Only executes when CI fails, prints "CI failed on main → triggering secondary workflow"