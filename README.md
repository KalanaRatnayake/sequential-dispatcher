# Sequential Dispatcher (GitHub Action)

Dispatch another workflow and optionally wait for it to complete. Use this composite action to orchestrate multiple workflows in sequence from a single "orchestrator" workflow.

- Supports waiting for completion with timeout and polling interval
- Emits the dispatched run_id and final conclusion
- Works with GITHUB_TOKEN or a passed-in PAT

## Usage

Reference the composite action via the path in this repository:

```yaml
- name: Dispatch workflow
  uses: ./.github/actions/sequential-dispatcher
  with:
    workflow_file: .github/workflows/build.yml
    ref: main
    wait: 'true'
    timeout-seconds: '7200'
    poll-interval-seconds: '10'
    label: Build
```

If you publish this repository and tag a release, others can consume it like:

```yaml
- name: Dispatch workflow
  uses: <owner>/<repo>/.github/actions/sequential-dispatcher@v1
  with:
    workflow_file: .github/workflows/build.yml
    ref: main
```

### Required permissions

Grant your orchestrator workflow minimal permissions:

```yaml
permissions:
  actions: write
  contents: read
```

### Example: Orchestrate multiple workflows in sequence

Below is a real-world example that dispatches four workflows one after another. Each step waits for the previous to finish.

```yaml
name: 01. Orchestrate r35.2.1 pipeline

on:
  workflow_dispatch:

permissions:
  actions: write
  contents: read

concurrency:
  group: orchestrate-r3521-${{ github.ref }}
  cancel-in-progress: false

jobs:
  dispatch-sequence:
    runs-on: ubuntu-24.04-arm
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build PyTorch r35.2.1
        uses: ./.github/actions/sequential-dispatcher
        with:
          workflow_file: .github/workflows/r3521-pytorch.yml
          ref: main
          label: Build PyTorch r35.2.1

      - name: Build ROS Humble Core r35.2.1
        uses: ./.github/actions/sequential-dispatcher
        with:
          workflow_file: .github/workflows/r3521-humblecore.yml
          ref: main
          label: Build ROS Humble Core r35.2.1

      - name: Build ROS Humble Base r35.2.1
        uses: ./.github/actions/sequential-dispatcher
        with:
          workflow_file: .github/workflows/r3521-humblebase.yml
          ref: main
          label: Build ROS Humble Base r35.2.1

      - name: Build ROS + PyTorch Humble Core r35.2.1
        uses: ./.github/actions/sequential-dispatcher
        with:
          workflow_file: .github/workflows/r3521-pytorch-humblecore.yml
          ref: main
          label: Build ROS + PyTorch Humble Core r35.2.1
```

## Inputs

- workflow_file (required): Path to the workflow file inside .github/workflows (e.g., .github/workflows/build.yml)
- ref (default: main): Git ref to run (branch or tag)
- wait (default: 'true'): Whether to wait for completion
- timeout-seconds (default: '7200'): Max seconds to wait when waiting
- poll-interval-seconds (default: '10'): Polling interval while waiting
 - label (default: 'Sequential Dispatcher'): Friendly label for logs
- token (optional): GitHub token to call the API. If omitted, uses GITHUB_TOKEN

## Outputs

- run_id: Dispatched workflow run id (only when waiting)
- conclusion: Final conclusion of the completed workflow (only when waiting)

Note: If `wait` is set to `'false'`, these outputs will be empty as the action does not wait for the run to complete.

## Test locally (two tiny workflows)

This repo includes a minimal pair of workflows to validate the action:

- `.github/workflows/test-target.yml`: a tiny workflow invoked via `workflow_dispatch`
- `.github/workflows/test-dispatch.yml`: an orchestrator that dispatches `test-target.yml` and waits for completion

Trigger the orchestrator manually from the Actions tab to see the end-to-end behavior.

## Security notes

- The action uses the GitHub REST API and requires `actions: write` and `contents: read` permissions.
- For cross-repo dispatch or private repos, supply a PAT via the `token` input with sufficient `repo` scope.

## License

MIT License. See `LICENSE`.
