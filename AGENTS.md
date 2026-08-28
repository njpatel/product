# Orchestration repo

This repo holds no product code. It is the home base for **orchestrator agents**: each one plans an epic with the user here, then executes it across the real project repos via Herdr worktrees, coordinating one worker agent per slice. The deliverable is the *integration* — usually cross-project e2e working — not the individual slices.

## Multi-tenancy

You are not alone. The user works in this session, and other orchestrators may run alongside you on different epics. Rules:

- Operate only on workspaces, worktrees, panes, and agents **you created**. Everything else belongs to someone.
- A `working`/`blocked` agent you didn't start is someone else's — leave it.
- Always pass `--no-focus` on creation commands; never steal the user's focus.
- Target panes/agents by IDs parsed from JSON responses or by unique names you assigned.

## Your folder

Create one directory at repo root named after your epic (same name as your worktree/workspace, e.g. `beam-prism-atom/`). Everything you produce lives there:

- `plan.md` — the agreed plan; write it *before* creating any worktree, keep it current as slices land.
- Per-slice briefs, cross-repo contracts (API shapes, ports, schemas — fixed up front, before dispatch), and integration/e2e status.
- Evidence and presentation artifacts: e2e results, screenshots, findings, write-ups.

Never write into another agent's folder.

## Audit trail

This repo *is* the audit trail. Commit as you go on your epic branch (same name as your folder) and push to `origin` — plan revisions, briefs, evidence — so the trail survives the session. Small frequent commits with plain messages; don't batch a whole epic into one commit.

## Executing an epic

1. **Plan with the user first.** No worktrees until the plan in your folder is agreed.
2. **Discover** repo roots and live state: `herdr workspace list`, `herdr worktree list`. Root workspaces (e.g. `beam`, `prism`, `atom`) point at the canonical checkouts under `~/src/`.
3. **Create worktrees** per slice:
   ```
   herdr worktree create --cwd <repo-root> --branch njpatel/<epic>-<slice> --label <epic>/<slice> --no-focus
   ```
   Consistent `<epic>` prefix in branch and label so all worktrees of one epic read as a set in the sidebar.
4. **Start workers** with the `omp-rust` harness unless told otherwise. `omp-rust` is a shell alias, not an agent kind — expand it yourself:
   ```
   herdr agent start <epic>-<slice> --kind omp --pane <pane-id> -- --config ~/.omp/model-combos/rust.yml
   ```
5. **Drive and supervise**: prompt with `--wait`, watch `idle/working/blocked/done`, unblock stalled workers, verify each worker's claims against its checkout — a worker saying "done" is not evidence.
6. **Integrate**: once slices land, wire the projects together and run the e2e scenario that is the epic's acceptance criterion. Report to the user with evidence, not summaries.
7. **Clean up** worktrees/workspaces you created once the user confirms the epic is done — never before.

## Herdr CLI

If the Herdr skill is not in your context, run `herdr --skill` and follow it. The installed binary's `--help` output is the authority for syntax; do not guess flags.
