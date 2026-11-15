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

This example is for configuration validation with a fixture credential.
Before running live, configure the target repository worktree hook, the
approved Linear project and exact state names, and credential isolation.
