# Azure DevOps – Concepts to Learn (Step by Step)

A structured learning path for Azure DevOps, organized from fundamentals to advanced topics. Follow the steps in order — each builds on the previous one.

---
```
Organization
 └── Project(s)
      ├── Boards        (work tracking)
      ├── Repos         (source control)
      ├── Pipelines     (CI/CD)
      ├── Test Plans    (manual + exploratory testing)
      └── Artifacts     (package feeds)
```
## Step 1: Core Fundamentals

Before touching any feature, understand the hierarchy:

- **Organization** – the top-level container (e.g., `dev.azure.com/yourcompany`). Holds billing, policies, and users.
- **Project** – lives inside an organization. Each project has its own Boards, Repos, Pipelines, Test Plans, and Artifacts.
- **Process/Methodology** – when creating a project, you pick a process template: **Basic**, **Agile**, **Scrum**, or **CMMI**. This determines the work item types available.

**Goal:** Know the difference between an Organization and a Project, and pick the right process template.

---
### What a Process Template Controls

A process template decides:
- What **work item types** exist (Epic, Feature, Bug, User Story, PBI, etc.)
- What **fields** appear on each work item (e.g., "Story Points" vs "Effort" vs "Size")
- What the **default Kanban board columns/states** look like
- The overall **workflow states** a work item moves through (New → Active → Resolved → Closed, etc.)

It does **not** change Repos, Pipelines, or Artifacts — those work the same regardless of process.

### The Four Templates

**1. Basic**
- Work item types: Issue, Task, Epic
- Best for: Small teams, simple projects, or people new to Azure DevOps who don't want ceremony.
- A simple to-do/tracking system, no formal "sprint" backlog terminology.

**2. Agile**
- Work item types: Epic → Feature → User Story → Task, plus Bug
- Best for: Teams following general Agile practices (not strict Scrum).
- Field naming: "User Story," "Story Points," "Effort."
- The most commonly used template for typical software teams.

**3. Scrum**
- Work item types: Epic → Feature → Product Backlog Item (PBI) → Task, plus Bug
- Best for: Teams strictly following Scrum ceremonies (Sprint Planning, Sprint Review, etc.)
- Field naming: "Product Backlog Item" instead of "User Story."
- Agile template's stricter, Scrum-terminology cousin.

**4. CMMI (Capability Maturity Model Integration)**
- Work item types: Epic → Feature → Requirement → Task, plus Bug, Change Request, Risk, Review
- Best for: Larger, formal, compliance-heavy organizations (government, regulated industries) needing detailed traceability and change control.
- The heaviest, most process-rigorous template — more overhead, more auditability.

### Quick Comparison Table

| Template | Work Items | Ceremony Level | Typical Use |
|---|---|---|---|
| Basic | Issue, Task, Epic | Minimal | Small teams, quick projects |
| Agile | Epic, Feature, User Story, Task, Bug | Medium | Most general software teams |
| Scrum | Epic, Feature, PBI, Task, Bug | Medium-High | Teams strictly doing Scrum |
| CMMI | Epic, Feature, Requirement, Task, Bug, Risk, Review, Change Request | High | Enterprise/regulated environments |

### Practical Advice

- **Learning or starting a new small-to-mid project:** pick **Agile** — most widely used, most tutorials/community support.
- **Team runs formal Scrum sprints with a Scrum Master:** pick **Scrum** for terminology alignment.
- **Regulated industry needing audit trails:** pick **CMMI**.
- **Want the simplest possible tracking:** pick **Basic**.

⚠️ **Important catch:** once a project is created, you generally **can't change its process template** (migrating work items manually is possible but painful). Decide deliberately rather than defaulting to whatever's pre-selected.

---

## Step 2: Azure Boards (Work Tracking)

This is where planning and tracking happens.

- **Work Item Types**: Epic → Feature → User Story/Backlog Item → Task (hierarchy depends on process template). Also: Bug, Issue.
- **Backlogs**: Prioritized lists of work items (Product Backlog, Sprint Backlog).
- **Boards**: Kanban-style visual board with customizable columns (To Do, Doing, Done) and swimlanes.
- **Sprints/Iterations**: Time-boxed periods for Scrum teams; configured under Project Settings.
- **Queries**: Custom filters (WIQL-based) to find/report on work items.
- **Area Paths & Iteration Paths**: Used to organize work by team/component and by time period.
- **Dashboards & Widgets**: Visual summaries of team progress.

**Goal:** Create a work item, move it through a board, assign it to a sprint.

---

## Step 3: Azure Repos (Source Control)

- **Git repos** (default) vs **TFVC** (centralized, legacy) — learn Git first, it's what's used today.
- **Branching strategy**: `main`/`master`, feature branches, release branches. Learn Git Flow or trunk-based development.
- **Pull Requests (PRs)**: Code review workflow — create PR, add reviewers, resolve comments, complete/merge.
- **Branch Policies**: Require PR reviews, build validation, linked work items before merging.
- **Commits & Tags**: Standard Git operations, linking commits to work items (`#123` in commit message).

**Goal:** Clone a repo, create a branch, push a commit, open a PR, merge it with a policy in place.

---

## Step 4: Azure Pipelines (CI/CD)

This is usually the most important skill for DevOps roles.

- **Continuous Integration (CI)**: Automatically build/test code on every push.
- **Continuous Deployment (CD)**: Automatically deploy to environments (Dev, QA, Prod) after a successful build.
- **Pipeline types**:
  - **YAML pipelines** (modern, code-as-config, stored in repo as `azure-pipelines.yml`) — learn this primarily.
  - **Classic pipelines** (UI-based, older approach) — good to know but less used now.
- **Key YAML concepts**:
  - `trigger` – what starts the pipeline (branch push, PR, schedule)
  - `pool` – the agent/VM that runs the job
  - `stages`, `jobs`, `steps` – the structure of a pipeline
  - `tasks` – built-in or marketplace actions (e.g., `AzureCLI@2`, `DotNetCoreCLI@2`)
  - `variables` and `variable groups` – reusable/config values
  - `templates` – reusable YAML snippets across pipelines
- **Environments**: Target deployment destinations with approval gates.
- **Service Connections**: Secure credentials to connect to Azure, AWS, Docker Hub, etc.
- **Artifacts (build output)**: Files published from a build stage, consumed by release/deploy stages.
- **Self-hosted vs Microsoft-hosted agents**: Where pipeline jobs actually run.

**Goal:** Write a simple YAML pipeline that builds an app, runs tests, and deploys it to one environment with an approval gate.

---

## Step 5: Azure Artifacts (Package Management)

- **Feeds**: Private package repositories for NuGet, npm, Maven, Python (pip), etc.
- **Upstream sources**: Pull public packages (e.g., from nuget.org) through your private feed.
- **Publishing/consuming packages** in pipelines.

**Goal:** Publish a package to a feed and consume it in a build.

---

## Step 6: Azure Test Plans

- **Test Plans & Test Suites**: Organize manual and automated test cases.
- **Test Cases**: Steps + expected results.
- **Exploratory Testing**: Ad-hoc testing with the browser extension.
- **Integrating automated tests** into pipelines (test results reporting).

**Goal:** Create a test plan with a few manual test cases and link them to a user story.

---

## Step 7: Security & Administration

- **Permissions**: Organization-level, project-level, and object-level (repo, pipeline) security.
- **Groups**: Built-in groups (Contributors, Readers, Project Administrators) vs custom groups.
- **Policies**: Branch policies, pipeline permissions, approval/check gates.
- **Service Connections & PATs (Personal Access Tokens)**: Secure ways to authenticate external tools/scripts.

**Goal:** Understand who can do what, and how to lock down a production environment.

---

## Step 8: Integrations & Extensions

- **Azure DevOps CLI (`az devops`)**: Automate tasks from the command line.
- **REST API**: Programmatic access to boards, repos, pipelines.
- **Marketplace extensions**: Slack integration, SonarQube, third-party tasks.
- **Webhooks / Service Hooks**: Trigger external actions on Azure DevOps events.

**Goal:** Know these exist and where to find them when you need automation beyond the UI.

---

## Suggested Learning Order (Summary)

1. Organization → Project → Process template
2. Boards: work items, backlogs, sprints
3. Repos: Git basics, branching, PRs, policies
4. Pipelines: YAML CI, then CD with environments/approvals
5. Artifacts: package feeds
6. Test Plans
7. Security/permissions
8. CLI, REST API, extensions (as needed)

---

## Hands-On Practice Idea

Build one small project end-to-end:
1. Create a Project → add a Backlog item → move it on the Board.
2. Create a feature branch → commit code → open a PR (with a branch policy requiring 1 reviewer).
3. Write a YAML pipeline that builds the app and runs tests on every PR.
4. Add a deployment stage to a "Dev" environment with a manual approval.
5. Publish a build artifact and consume it in the deploy stage.

Doing this once will teach you 80% of what's needed in real projects.
