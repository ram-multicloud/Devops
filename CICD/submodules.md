# Git Submodules

## What is a Submodule?

A Git submodule is a repository embedded inside another repository as a subdirectory. The parent (superproject) repo does not store the submodule's files directly — it stores a pointer to a specific commit in the submodule's repository. This lets you keep a dependency's source code in its own repository while referencing an exact version of it from another project.

## Why Use Submodules

- Share common code (libraries, shared configs, tooling) across multiple projects without duplicating it.
- Pin a dependency to an exact commit so the parent project's build is reproducible.
- Keep a third-party or vendored repository's history separate from your own.
- Split a large monolithic repo into independently versioned components while still assembling them together.

## How It Works

- The superproject stores a `.gitmodules` file mapping each submodule's path to its remote URL and branch (optional).
- The superproject also tracks a **gitlink** — a special entry in the tree that records the exact commit SHA the submodule should be checked out at.
- Cloning the superproject does **not** automatically clone submodule contents; the submodule directories are created empty until explicitly initialized and updated.

## Adding a Submodule

```bash
git submodule add <repository-url> <path>
```

Example:

```bash
git submodule add https://github.com/example/shared-lib.git libs/shared-lib
```

This:
1. Clones the submodule repo into `libs/shared-lib`.
2. Adds an entry to `.gitmodules`.
3. Stages the gitlink and `.gitmodules` for commit.

Commit the change:

```bash
git commit -m "Add shared-lib submodule"
```

### `.gitmodules` File Format

```ini
[submodule "libs/shared-lib"]
    path = libs/shared-lib
    url = https://github.com/example/shared-lib.git
    branch = main
```

## Cloning a Repository That Has Submodules

Submodules are not populated by a plain clone. Use one of the following:

**Clone and initialize in one step:**

```bash
git clone --recurse-submodules <repository-url>
```

**Or, after a normal clone:**

```bash
git clone <repository-url>
cd <repository>
git submodule init
git submodule update
```

**Shortcut for init + update:**

```bash
git submodule update --init --recursive
```

The `--recursive` flag is needed if a submodule itself contains nested submodules.

## Updating Submodules

### Pull the Latest Commit Pinned by the Superproject

If someone else updated which commit the submodule points to:

```bash
git submodule update --remote
```

Wait — `update` alone (no `--remote`) checks out the commit currently recorded in the superproject:

```bash
git pull                # updates superproject, including gitlink pointers
git submodule update    # checks out submodules to the recorded commits
```

### Move a Submodule to the Latest Commit on Its Tracked Branch

```bash
git submodule update --remote
```

This fetches the submodule's remote and checks out the latest commit on the branch configured in `.gitmodules` (or `main`/`master` by default).

To make this the new default behavior for a submodule, add `-b` when adding it:

```bash
git submodule add -b main <repository-url> <path>
```

### Update All Submodules Recursively

```bash
git submodule update --init --recursive --remote
```

## Making Changes Inside a Submodule

A submodule is a full Git repository. To change it:

```bash
cd libs/shared-lib
git checkout main
# make changes
git add .
git commit -m "Fix bug in shared-lib"
git push
```

Then update the superproject to point at the new commit:

```bash
cd ../..
git add libs/shared-lib
git commit -m "Bump shared-lib to latest commit"
git push
```

Without that final step, the superproject still points to the old commit — other developers pulling the superproject won't get your submodule change until the gitlink is updated and committed.

## Checking Submodule Status

```bash
git submodule status
```

Output format: `<commit-sha> <path> (<describe-output>)`

- A `-` prefix before the SHA means the submodule is not initialized.
- A `+` prefix means the checked-out commit differs from the one recorded in the superproject.

## Removing a Submodule

Git has no single command for this; it's a multi-step process:

```bash
git submodule deinit -f <path>
git rm -f <path>
rm -rf .git/modules/<path>
git commit -m "Remove submodule <path>"
```

## Common Pitfalls

| Issue | Cause | Fix |
|---|---|---|
| Submodule directory is empty after clone | Submodules aren't cloned by default | `git submodule update --init --recursive` |
| Submodule shows as "modified" with no visible diff | Local checkout is at a different commit than the recorded gitlink | `git submodule update` to reset, or commit the new pointer if the change is intentional |
| Changes made inside submodule don't appear for teammates | Forgot to commit the updated gitlink in the superproject | `git add <submodule-path> && git commit` in the parent repo |
| Detached HEAD inside submodule | `git submodule update` checks out a specific commit, not a branch | `cd` into the submodule and `git checkout <branch>` before making changes |
| Cloning is slow | `--recurse-submodules` recursively fetches full history of every submodule | Use `--shallow-submodules` alongside it for shallow submodule clones |

## Useful Configuration

**Always recurse into submodules on pull:**

```bash
git config submodule.recurse true
```

**Show submodule summary in `git status`:**

```bash
git config status.submodulesummary 1
```

**Show submodule diffs in `git diff`:**

```bash
git config diff.submodule log
```

## Use Cases

1. **Shared internal libraries** — Multiple services depend on a common utility or config library maintained in its own repo; each service pins the exact version it's tested against.
2. **Vendoring third-party code** — Including an external open-source project's source in your build without merging its history into yours.
3. **Monorepo alternative** — Splitting a large system into independently owned repositories (e.g., frontend, backend, infra-as-code) while still being able to check them out together for local development or CI.
4. **Documentation or theme repos** — A static site repo pulling in a shared theme or documentation set maintained separately.
5. **Environment/config separation** — Keeping environment-specific configuration (e.g., Kubernetes manifests, Ansible inventories) in a separate repo referenced by an application repo, so config changes don't trigger app CI pipelines and vice versa.

## Submodules vs. Alternatives

| Approach | Best For | Tradeoff |
|---|---|---|
| Submodules | Pinning exact versions of separate, independently versioned repos | Extra commands to init/update; easy to forget to commit pointer updates |
| Subtree merge (`git subtree`) | Embedding another repo's code with full history merged in | Repo size grows; harder to push changes back upstream |
| Package manager (npm, pip, Maven, etc.) | Versioned, publishable dependencies | Requires publishing artifacts; not ideal for internal-only, frequently-changing code |
| Monorepo | Tightly coupled components that change together often | Larger repo, requires tooling for selective builds/tests |
