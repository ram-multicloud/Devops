# GitHub Actions

GitHub Actions is GitHub's built-in automation and CI/CD platform. It lets you automatically build, test, and deploy code — or run any custom task — in response to events that happen in your repository.

## Core Concept

You define automation as **workflows**: YAML files stored in `.github/workflows/` inside your repo. GitHub watches for events (like a push or pull request) and runs the matching workflow automatically.

## Key Components

### 1. Workflow
The top-level automation process, defined in a `.yml` file. A repo can have multiple workflows (e.g., `ci.yml`, `deploy.yml`).

### 2. Event (Trigger)
What starts the workflow — `push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual), issue creation, release publishing, etc.

### 3. Job
A workflow contains one or more jobs. Each job runs on a fresh virtual machine (**runner**). Jobs run in parallel by default, unless you set dependencies with `needs:`.

### 4. Runner
The machine that executes the job — GitHub-hosted (Ubuntu, Windows, macOS) or self-hosted (your own server).

### 5. Steps
Each job has a sequence of steps that run in order on the same runner. A step can:
- Run a shell command (`run:`)
- Use a prebuilt **Action** (`uses:`) — a reusable unit of code from the GitHub Marketplace or your own repo

### 6. Actions
Reusable building blocks, e.g., `actions/checkout` (pulls your repo code), `actions/setup-node` (installs Node.js). You can also write custom actions.

## Basic Example

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

```
