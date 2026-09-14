---
name: consolidate-design-docs
description: Merges the already-generated HLD and LLD documents from each component repo of a project (e.g. frontend and backend) into a single unified HLD and LLD for an Enterprise Architect audience. Discovers component repos at runtime and is runner-agnostic. Does not re-analyze source code.
---

# Consolidate Design Documents

You are merging documentation that has **already been generated** (by the
generate-design-docs skill or an equivalent) — you are not analyzing source code
and you are not regenerating anything from scratch.

This skill is reused across many projects, so it never assumes specific repo
folder names, and it runs in different agents. It discovers the component repos at
runtime. Its output uses the same section taxonomy as the generate-design-docs
skill, so a consolidated doc has a section for every section that skill produces.

## Execution Protocol (runner-agnostic — read this FIRST)

This skill runs in different agents (e.g. the GitHub Copilot coding agent on
github.com, or Copilot in VS Code). Do not assume any specific tool name — use
whatever file-read, file-create, file-edit, directory-list, branch, and
pull-request capabilities the current runner provides.

**Write each document as a whole, in a single file operation — one write for the
consolidated HLD, one write for the consolidated LLD.** Compose the complete
document (all sections, in the order given below) and then write the entire file at
once. Do NOT split a document into many per-section writes.

**Run straight through — do not wait for the user.** Produce the complete
consolidated HLD and write it, then produce the complete consolidated LLD and write
it, then continue to the change check and PR — all in one run. Do not stop, hand
back to the user, or ask them to say "next"/"continue"/"yes" between the two
documents or at any other point. If the runner shows a file-write approval prompt,
that is the runner's own behavior (outside your control); as soon as it is
approved, continue to the next step without waiting to be told. If your runner can
stage both file writes in a single confirmation, you may write both together.

**Resume safely.** If re-invoked after an interruption, do not restart blindly:
read current state (do the consolidated files already exist and look complete? do
the branch/PR exist?) and continue from the first incomplete artifact. Do not
create a second branch or a duplicate PR if one already exists.

## Step 0 — Locate the component docs (do this FIRST, exactly once)

1. **If the invocation explicitly names the component repos** (e.g. "consolidate
   `web/` and `api/`"), use exactly those paths and skip discovery.
2. **Otherwise, auto-discover.** List the current workspace's root folder(s) and
   their immediate subdirectories (one level deep only), and collect every folder
   containing BOTH `docs/HLD.md` and `docs/LLD.md`. Match the exact filenames
   `HLD.md` and `LLD.md` — a consolidated output file does NOT count as an input.
3. Do this enumeration **exactly once**. Do not repeatedly rescan or wait for
   files to appear.

**Outcome — exactly one of these, never a loop:**
- **Fewer than two** components found: STOP and report how many you found, their
  paths, and where you scanned. Do not read source code or guess contents.
- **Two or more** found: proceed. Each discovered folder is a "component."

## Component roles

For each component, determine its role (typically frontend/client vs
backend/service) by reading ONLY that component's own `docs/HLD.md` overview —
never its source code. If the role can't be determined, label it by folder name
and proceed. Best-effort, one read per component, never loop.

## Inputs (read-only)

For each component, its `docs/HLD.md` and `docs/LLD.md`. Read each once, in full.
These are your ONLY inputs — never read source code.

## Output target

Write the consolidated docs into the `docs/` folder of the **consolidation target
repository**:
- If the invocation names a target repo/path, use it.
- Otherwise use the repository you are operating in (VS Code: the open workspace
  repo; Copilot coding agent: the repository the session is attached to).
Open the PR (if there are changes) in that same repository. Do not write
consolidated output into the individual component repos unless one of them is
explicitly the target.

Output files:
- `docs/HLD_consolidated.md`
- `docs/LLD_consolidated.md`

(Adjust the filenames if the target repo has an existing `docs/` naming
convention — keep whatever distinguishes the consolidated doc from per-component
docs.)

## Hard Rules

1. **Do not re-analyze code.** Your only inputs are the per-component HLD/LLD
   markdown files. Note gaps rather than inferring from anything else.
2. **Do not simply concatenate.** The result should read as one coherent system
   description, not stacked per-component docs.
3. **Preserve accuracy.** Only reorganize, merge, and de-duplicate — do not alter
   or reinterpret factual claims.
4. **Regenerate fully each run — do not attempt incremental diffing of source.**
   Produce both consolidated documents in full from the current inputs each run.
   Do NOT try to compute "what changed since last consolidation" — that comparison
   is a common cause of spinning. The runner's diff/status detects whether the
   regenerated output differs from what's committed (see Process).
5. **Validate merged Mermaid diagrams** using the rules below — a diagram valid in
   each source doc can still break after merging.

## Mermaid Rules for Merged Diagrams

Self-contained — do not look them up in another skill.

- **Always wrap every node label and every edge label in double quotes**, no
  exceptions, even single words. `A["Web UI"]`, not `A[Web UI]`. `X -->|"HTTPS"| Y`,
  not `X -->|HTTPS| Y`. Punctuation in an unquoted label is the most common parse
  error — e.g. `-->|DB queries (Devices/ApiKeys/Users)|` breaks;
  `-->|"DB queries (Devices/ApiKeys/Users)"|` is safe.
- **Rename colliding IDs before merging** — identical IDs silently overwrite each
  other in Mermaid.
- **Rename colliding sequence-diagram aliases** the same way.
- **Never use a reserved word as an ID or alias** (`end`, `graph`, `subgraph`,
  `class`, `style`, `loop`, `alt`, `opt`, `note`, etc.).
- **One statement per line; declare direction once; balance every `activate` with
  a `deactivate`.**
- After merging, re-read once and confirm every label is quoted, no IDs collide,
  no reserved words as IDs. One pass — do not re-parse repeatedly.

## Merge Logic

The consolidated docs use the SAME section taxonomy as the generate-design-docs
skill — one consolidated section per section that skill produces. For each
section, combine every component's content into one coherent whole (do not stack
per-component copies), and keep any section that exists in only one component,
clearly scoped to it. Group content by component role: in the common
two-component case the groups are **Frontend** and **Backend**; with more
components, use each component's role or folder-name label. Read
"frontend"/"backend" below as "client-side component(s)"/"server-side
component(s)."

### HLD merge (`docs/HLD_consolidated.md`)
1. Title & Metadata — project name, last updated date, doc owner; list the
   component repos merged and their last-updated dates.
2. Executive Overview — one overview of the whole system (all components).
3. Objective — combined objective for the system.
4. Architecture Description — combine the components' diagrams into a single
   `flowchart TB` showing them as connected layers/systems, each connection
   labeled with its protocol (HTTPS/REST, gRPC, WebSocket). Merge — do not place
   diagrams side by side.
5. Core Workflows — unified end-to-end workflows spanning client → server where
   applicable.
6. Data Flow — unified end-to-end data flow across components.
7. Key Features — combined; scope a feature to its component where it belongs to
   only one.
8. Infrastructure & Deployment Overview — combined across all components.
9. Deployment Strategy — combined across all components.
10. Data Protection — full data lifecycle across all components.
11. Security Requirements — one section; if components differ, describe each and
    how they relate (e.g. client obtains a token the server validates).
12. Integrations — all integrations from every component in one table; note which
    component owns each.
13. Environment Variables & Secrets Inventory — one inventory, labeled by
    component (names and purposes only, never values).
14. Change Log — entry noting this consolidation run and which component docs
    (with dates if present) were used.

### LLD merge (`docs/LLD_consolidated.md`)
1. Title & Metadata.
2. Module/Component Breakdown — one top-level group per component, with a
   "Cross-Cutting" note where one component's module depends on another's.
3. Key Classes / Functions — merged, keeping per-component attribution; only
   architecturally significant ones.
4. Data Models / Schemas — one section; flag where types/schemas in different
   components represent the same entity.
5. Sequence Diagrams — prefer merged end-to-end diagrams for cross-component
   workflows over separate per-component ones.
6. Error Handling & Retry Behavior — merged, per-component attribution where
   useful.
7. Configuration & Environment-Specific Behavior — merged.
8. Known Limitations / Technical Debt — merged.
9. Change Log.

## Process

Run these steps straight through, without pausing for user input between them.

1. **Discover** the component repos (Step 0), once. Fewer than two → STOP and
   report. Otherwise continue.
2. **Read** each component's `docs/HLD.md` and `docs/LLD.md` in full, once, and
   determine each component's role.
3. **Compose the complete consolidated HLD** (all sections in order, validating
   Mermaid) and write `docs/HLD_consolidated.md` in one operation.
4. **Compose the complete consolidated LLD** the same way and write
   `docs/LLD_consolidated.md` in one operation.
5. **Detect changes** using the runner's diff/status capability. If neither output
   file differs from what's committed, STOP — no branch or PR.
6. **Branch**: `docs/consolidated-update-<YYYYMMDD-HHMM>` off the default branch.
7. **Commit** both files: `docs: automated consolidated HLD/LLD update <date>`.
8. **Open the PR** against the default branch, titled
   `docs(hld): automated consolidated update <date>`, labeled `automerge`, body
   noting which component docs (with dates) were used. Never push directly to the
   default branch.

If re-invoked after an interruption, read current state and resume from the first
incomplete step — do not redo a completed file, and do not create a second branch
or a duplicate PR.
