---
name: consolidate-design-docs
description: Merges the already-generated HLD and LLD documents from the frontend repo and backend repo into a single unified HLD and LLD, written for an Enterprise Architect audience. Does not re-analyze source code.
---

# Consolidate Design Documents

You are merging documentation that has **already been generated** — you are not
analyzing source code and you are not regenerating anything from scratch.

## Step 0 — Fail fast if inputs are missing (do this FIRST)

Before doing anything else, check that all four input files below exist. Look
for them exactly once — do not repeatedly search the tree or wait for them to
appear.

- `frontend-repo/docs/HLD.md`
- `frontend-repo/docs/LLD.md`
- `backend-repo/docs/HLD.md`
- `backend-repo/docs/LLD.md`

If any of the four is missing, **STOP immediately** and report which file(s)
are missing and the paths you checked. Do not guess their contents, do not read
source code, and do not keep searching. This check has exactly one outcome:
either all four exist and you proceed, or you stop and report. Never loop here.

## Inputs (read-only)

The four files listed above. Read each one exactly once, in full. These are
your ONLY inputs — never read source code.

## Outputs

- `docs/HLD_consolidated.md`
- `docs/LLD_consolidated.md`

(Adjust these output paths if this repo has an existing `docs/` convention —
keep whatever naming distinguishes the consolidated doc from the individual
per-repo docs.)

## Hard Rules

1. **Do not re-analyze code.** Your only inputs are the four markdown files
   above. If something seems missing or unclear, note it as a gap rather than
   inferring it from anything else.
2. **Do not simply concatenate.** A unified document should read as one
   coherent system description, not "frontend doc" + "backend doc" stapled
   together.
3. **Preserve accuracy.** Do not alter or reinterpret factual claims from the
   source documents — only reorganize, merge, and de-duplicate them.
4. **Regenerate fully each run — do not attempt incremental diffing.** On every
   run, produce the two consolidated documents in full from the four current
   inputs. Do NOT try to figure out "what changed since the last consolidation"
   or touch only some sections — that comparison is the main cause of the
   process spinning. Git will detect whether the regenerated output actually
   differs from what's committed (see Process step 4); rely on that, not on your
   own section-by-section diff.
5. **Validate merged Mermaid diagrams** using the rules in the Mermaid section
   below — a diagram that was valid in each source doc can still break after
   merging.

## Mermaid Rules for Merged Diagrams

These are self-contained — do not go look them up in another skill.

- **Always wrap every node label and every edge label in double quotes**, with
  no exceptions, even single plain words. `A["Web UI"]`, not `A[Web UI]`.
  `X -->|"HTTPS"| Y`, not `X -->|HTTPS| Y`. Punctuation in an unquoted label
  (commas, slashes, parentheses, colons) is the most common parse error — e.g.
  `-->|DB queries (Devices/ApiKeys/Users)|` breaks;
  `-->|"DB queries (Devices/ApiKeys/Users)"|` is safe.
- **Rename colliding IDs before merging.** If the frontend and backend diagrams
  each define a node with the same ID but different meaning, rename one — same
  IDs silently overwrite each other in Mermaid.
- **Rename colliding sequence-diagram aliases** the same way if both source
  diagrams use the same participant alias for different actors.
- **Never use a reserved word as an ID or alias** (`end`, `graph`, `subgraph`,
  `class`, `style`, `loop`, `alt`, `opt`, `note`, etc.).
- **One statement per line; declare direction once; balance every `activate`
  with a `deactivate`.**
- After merging a diagram, re-read it once and confirm every label is quoted,
  no IDs collide, and no reserved words are used as IDs. One pass — do not
  re-parse repeatedly.

## Merge Logic

### HLD merge (`docs/HLD_consolidated.md`)
- **Executive Overview / Objective**: one combined overview describing the
  system as a whole (frontend + backend together), not two separate ones.
- **Architecture Description**: combine both Mermaid diagrams into a single
  `flowchart TB` showing frontend and backend as connected layers/systems, with
  the connection clearly labeled (protocol, e.g. HTTPS/REST, gRPC, WebSocket).
  Merge them — do not place two diagrams side by side.
- **Core Workflows / Data Flow**: merge into unified end-to-end workflows that
  span frontend → backend where applicable.
- **Data Protection**: one section covering the full data lifecycle across both
  systems — where data enters (frontend), how it travels, where it's stored
  (backend).
- **Security Requirements**: one section; if frontend and backend have
  different auth mechanisms, describe both and how they relate (e.g. frontend
  obtains a token the backend validates).
- **Integrations**: all integrations from both docs in one table; note which
  side owns each.
- **Environment Variables & Secrets Inventory**: one inventory, labeled by
  which repo/service each variable belongs to.
- **Change Log**: append an entry noting this is a consolidation run and which
  source docs it pulled from (include their last-updated dates if present).
- Sections unique to one side should be kept, clearly scoped.

### LLD merge (`docs/LLD_consolidated.md`)
- **Module/Component Breakdown**: organize under two top-level groups, Frontend
  and Backend, with a short "Cross-Cutting" note wherever a frontend module
  directly depends on a backend module or endpoint.
- **Sequence Diagrams**: prefer merged, end-to-end sequence diagrams for
  cross-system workflows over separate frontend-only and backend-only diagrams
  of the same workflow.
- **Data Models / Schemas**: one section; flag where a frontend-side type and a
  backend-side schema represent the same entity.
- Everything else (Error Handling, Configuration, Known Limitations, Change
  Log): merge straightforwardly, keeping frontend/backend attribution where it
  aids clarity.

## Process

1. Do Step 0 (fail fast on missing inputs). If all four exist, continue.
2. Read all four source files in full, once.
3. Produce BOTH consolidated documents in full per the Merge Logic above, and
   write them to this repo's `docs/` folder (create it if needed). Do not stop
   after the HLD — the LLD is part of the same run.
4. Compare the freshly written files against what's committed (e.g. `git
   status` / `git diff`). If neither consolidated file differs from what's
   already committed, STOP — do not create a branch or PR.
5. If either file changed, open a single pull request containing both:
   - Create a new branch named `docs/consolidated-update-<YYYYMMDD-HHMM>` off
     the default branch.
   - Commit both files with a message like
     `docs: automated consolidated HLD/LLD update <date>`.
   - Push and open a PR against the default branch, titled
     `docs(hld): automated consolidated update <date>`, labeled `automerge`,
     with a body noting which source docs (and their last-updated dates) were
     used.
   - Never push directly to the default branch — always go through a PR.
