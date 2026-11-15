---
tracker:
  kind: linear
  project_slug: $LINEAR_PROJECT_SLUG
  api_key: $LINEAR_API_KEY
  active_states: [Ready, "In Progress"]
  terminal_states: ["Human Review", Done, Blocked, Canceled, Cancelled]
  archive_after_days: 0
polling:
  interval_ms: 30000
wiki:
  sweep_every_n: 0
workspace:
  root: ./.symphony/linear-pilot-workspaces
  reuse_policy: preserve
hooks:
  after_create: |
    set -euo pipefail
    repo="${LINEAR_PILOT_REPO_DIR:?Set LINEAR_PILOT_REPO_DIR to the pilot checkout}"
    base="${LINEAR_PILOT_BASE_BRANCH:-main}"
    branch="symphony/${PWD##*/}"
    git -C "$repo" check-ref-format "refs/heads/$branch"
    base_sha="$(git -C "$repo" rev-parse --verify "refs/heads/$base^{commit}")"
    if git -C "$repo" show-ref --verify --quiet "refs/heads/$branch"; then
      git -C "$repo" worktree add "$PWD" "$branch"
    else
      git -C "$repo" worktree add -b "$branch" "$PWD" "$base_sha"
    fi
  before_run: |
    set -euo pipefail
    repo="${LINEAR_PILOT_REPO_DIR:?Set LINEAR_PILOT_REPO_DIR to the pilot checkout}"
    git_common_dir() {
      local common
      common="$(git rev-parse --git-common-dir)" || return
      cd "$common" && pwd -P
    }
    expected="$(cd "$repo" && git_common_dir)"
    actual="$(git_common_dir)"
    root="$(git rev-parse --show-toplevel)"
    root="$(cd "$root" && pwd -P)"
    if [[ "$actual" != "$expected" || "$root" != "$(pwd -P)" || "$(git symbolic-ref --quiet HEAD)" != "refs/heads/symphony/${PWD##*/}" ]]; then
      echo "Pilot workspace does not match the configured repository and ticket branch." >&2
      exit 1
    fi
agent:
  kind: codex
  max_concurrent_agents: 1
  max_attempts: 2
  auto_commit_on_done: false
  auto_merge_on_done: false
  auto_merge_push_target: false
codex:
  command: codex app-server
  approval_policy: never
  thread_sandbox: workspace-write
  turn_sandbox_policy: workspace-write
---

# Linear factory pilot

Work only on {{ issue.identifier }}: {{ issue.title }}.

{{ issue.description }}

The issue is the approved scope. If it is unclear or the workspace hasn't
been provisioned for the target repository, stop and report the blocker.
Use the repository's normal validation commands. Return the commit, checks
run, their results, and any remaining uncertainty for human review. Do not
merge, publish, or mark the project complete.

Set `LINEAR_PILOT_REPO_DIR` to the absolute path of a dedicated checkout of
the target repo. The hook branches from its local `main` by default. Set
`LINEAR_PILOT_BASE_BRANCH` for another branch and update that local branch
before starting a pilot. Existing ticket branches keep their progress.
For a live run, configure the approved Linear project, credentials and
exact state names, and use a trusted execution host.
The runner strips the Linear key from worker subprocess environments. Run
only trusted lifecycle hooks, which execute on the controller host.
