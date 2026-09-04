# Material Handler kiosk — project kickoff

**For: the new Material-Handler session. Written 2026-09-04 by the
Assembler Board - Supervisor session (Chuck's instruction). This repo was
empty except git plumbing when this file was written.**

## What Chuck asked for, in his words

> "We need a kiosk made for 'Material Handle' to be able to complete the
> routing."

## Why (the case that triggered it)

WO-2026-534410 (work_order_id 3368, SO 597323 line 5, YS82BC0200, qty 16,
due 06/08 — long past) was flagged by the assembler board's new orphan
audit: Assigned to Sean Davis, no start day — and invisible on every
board. Root cause: its routing step 10 **Material Handling has sat at
Ready since 8/3** (wo_step_id 73, department_id 4), so step 20 Assemble
(wo_step_id 74) is Pending and the supervisor board hides the WO. Nobody
on the floor has a screen for completing Material Handling steps — the
only path today is the PC app's routing form, which the MH crew doesn't
use. This kiosk closes that gap. It's live test data for this build.

## Decisions Chuck has already made (2026-09-04, this session)

1. **Shape: shared department queue + name picker.** One screen showing
   every WO whose Material Handling step is workable; the operator taps
   their NAME first so Complete events record who did it. NOT a clone of
   the assembler/fab kiosk's my_day pattern — Material Handling has no
   supervisor assignment board, so there is no "my" work; the queue is
   shared.
2. **Look and feel: match the `pick_worker` form** in
   `C:\Users\cbolton\Claude\Development\team_anvil_code\Hub Transfers v2.1\client_code\pick_worker`.
   This session could NOT read that app (folder access not granted here) —
   **your first move: request access to that folder and study the form**
   (layout, name-picker flow, how it presents rows and quantities, its
   confirm pattern). Chuck pointed at it when asked how Complete should
   handle quantities, so take the qty UX from there too.
3. **Repo: this folder** (`Projects\Material-Handler`, git already
   initialized). New Anvil app, created fresh from this repo when it's
   ready (never push over an app that has its own master commits — the
   standing git↔Anvil lesson).

## The mechanics — already built, just drive them

- **The queue**: `dbo.vw_pc_department_queue` filtered
  `department_name = 'Material Handling'` (department_id 4;
  `department_name` is UNIQUE on `dbo.pc_departments`, so name-based
  predicates are the house style). Columns available: wo_step_id,
  work_order_id, work_order_number, sales_order_number, sales_order_rel,
  sales_order_line, part_number, wo_qty, wo_due_date, due_date, wo_status,
  sequence_number, department_id, department_name, step_status,
  qty_required, qty_completed, qty_remaining, step_notes, step_created_at.
  Workable = `step_status IN ('Ready', 'In Progress')`.
- **Completing a step**: `EXEC dbo.sp_pc_complete_step @wo_step_id=%s,
  @qty=%s, @actor=%s, @notes=%s` — returns a row with `status` /
  `message`. This proc (migration 007) is THE single write path for ALL
  apps: it owns qty accumulation (partial completes leave the step
  In Progress until qty_required is met), out-of-sequence flagging, the
  Ready cascade to the next department, WO auto-complete, and
  notification queueing. Do NOT update `pc_wo_routing_steps` directly.
  See the PC app's `server_code/routing_dashboard.py` (~line 610) for the
  reference call.
- **The payoff cascade**: when the MH step completes, the next
  department's step (usually Assemble) flips to Ready and the WO appears
  on that supervisor board automatically. Nothing else to wire.

## Roster — a small companion change in ANOTHER app

The name picker needs Material Handling operators in the shared roster
(`dbo.assembly_operators` + `department_id`, migration 045, APPLIED).
The roster app (`Projects\assembler-roster`, Anvil app Assembler_Roster)
is the ONLY writer to that table. It needs:

- A Material Handling entry in `DEPT_PREFIXES` (roster_service.py).
  **Prefixes must be exactly 3 chars** (the code-minting SUBSTRING starts
  at position 4) — Assemble = `ASM<n>`, Fab Shop = `FAB<n>`; suggest
  `MAT<n>` (Chuck decides).
- The Add dropdown currently offers Assemble + Fab Shop only — extend it.

That's a handoff to the roster app's session (or a small enough change to
ask Chuck to route there); this kiosk should only READ the roster:
`SELECT operator_code, display_name FROM dbo.assembly_operators WHERE
is_active = 1 AND department_id = (SELECT department_id FROM
dbo.pc_departments WHERE department_name = 'Material Handling') ORDER BY
sort_order, display_name` — the same department-filtered pattern as the
fab apps.

Event actor: pass the operator's code (or `display_name (kiosk)`) as
`@actor` so the step events say WHO — that's the whole point of the name
picker.

## Reference apps (Chuck can connect the folders)

- `Projects\Assembler-Board` — the kiosk PATTERN (kiosk_home name picker,
  large-touch layout, NavSidebar, db_config): steal the skeleton, drop
  my_day/so_detail/pull_parts (assignment-driven; doesn't apply here).
- `Projects\Fab-Board` — how a department clone was done cleanly
  (fab-kiosk requirements doc in the team project lists the exact diffs).
- `Projects\Production-Control` — `server_code/routing_dashboard.py` is
  the reference for calling sp_pc_complete_step correctly.
- `Hub Transfers v2.1` (path above) — the LOOK, per Chuck.

## House rules (all Titan Anvil apps — follow them from commit one)

- Anvil classic runtime v3, bootstrap3 (`legacy_features`), stock
  Material Design roles, NO custom CSS; colors set in code and never
  carrying meaning alone (the word is always present).
- Component naming: fp_/cp_/lk_/grid_/spc_/lp_/lbl_/btn_/dd_/tb_/rp_
  prefixes; theme-slot names (content_panel, navbar_links, nav) exempt.
- Skulpt (client Python) landmines: a dict/list/set comprehension
  containing a generator expression CRASHES compile — use explicit
  loops; no `datetime.strptime` client-side — parse dates manually;
  `spacing_above/spacing_below` (default 'small') are the real vertical
  gap knobs.
- Server: pymssql via a `db_config.py` module reading App Secrets
  `NEXUS_SERVER / NEXUS_DATABASE / NEXUS_USERNAME / NEXUS_PASSWORD`
  (copy from any sibling app; no secrets in anvil.yaml). Callables
  return `{'status': 'success'/'error', ...}`.
- Dates display MM/DD/YYYY (MM/DD/YY where space is tight); never
  reference migration 029; new .sql files also copied to
  `C:\Users\cbolton\Claude\Development\migrations` (none expected here —
  everything this app needs is applied).
- **Chuck runs ALL git.** Claude sessions must not run git commands over
  the desktop bridge — even `git status` leaves an undeletable stale
  `.git\index.lock` (move strays to `.git\_stale_locks\` if it happens).
  Deliver files via the conversation + write them into this folder, then
  Chuck commits/pushes.
- NO scheduled task in this app (the single nightly rollover task lives
  in Assembler Board - Supervisor and, post-045, serves every
  department — and this kiosk doesn't schedule anything anyway).
- Anvil package name: pick one and verify it matches every form-template
  reference (suggest `Material_Handler`); the parent apps' clone lessons
  live in `claude/fab-kiosk-requirements.md` in the team project.

## Suggested first steps for your session

1. Request folder access: `Hub Transfers v2.1` (path above) — study
   `client_code/pick_worker` (form_template.html + __init__.py) and note
   the flow to mirror.
2. Read `claude/fab-kiosk-requirements.md` and
   `claude/fab-board-requirements.md` in the team project — the clone
   conventions, 045 background, and roster model live there.
3. Discovery pass on Nexus (read-only): MH queue depth today
   (`vw_pc_department_queue WHERE department_name = 'Material Handling'`),
   confirm department_id 4, check whether ANY MH operators exist yet
   (`assembly_operators WHERE department_id = 4` — expected: none).
4. Ask Chuck the open questions below, then build.

## Open questions for Chuck (not yet decided)

- Operator codes: `MAT<n>` prefix OK? Who's on the MH crew (names to
  seed via the roster app once it supports the department)?
- Does the kiosk need anything besides the queue + Complete — e.g. a
  step-notes viewer, a "problem / can't complete" flag, or a Done Today
  list like the boards have?
- Sort order for the queue: due date? Daily Plan flag score (the shared
  `vw_pc_order_flag_scores` view is available if Sales' priorities should
  drive MH order too)?
- App title on the floor: "Material Handling Board"? (pc_departments
  says 'Material Handling'; Chuck said "Material Handle" — confirm the
  display name.)
