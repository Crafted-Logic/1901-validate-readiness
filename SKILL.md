---
name: 1901-validate-readiness
description: Validates whether a 1901 design may enter production; read-only, fails closed.
---

# 1901 Validate Readiness

Decides whether one 1901 Main Street design is allowed to enter production and
returns a structured verdict. It only reads and reports. It never approves,
repairs, renders, publishes, delegates, or changes any record or external system.

Core rule: **VERIFY, DON'T ASSUME.** Every check passes only on evidence you
actually read. Anything missing, unreadable, contradictory, or open to more than
one interpretation fails the check. Ambiguity is a failure, never a pass.

## When to Use

Trigger on requests such as:

- "Is design 1901-042 ready for production?"
- "Validate readiness for <design_id>"
- "Can this go to Printify / to the next production step?"
- "Run the readiness gate" / "pre-production check"
- Any time Walter is about to hand a design to a production step. Run this first.

Do not use it to fix a design, chase approvals, or decide artwork. It answers one
question: may this design enter production right now, yes or no, and why.

## Inputs

One design record, identified by `design_id`, as it exists in the current
governing records. Read the fields below with read-only tools. Do not accept
values recited from memory or from an earlier conversation as evidence.

| Field | Expected |
|---|---|
| `design_id` | Exactly one design identifier |
| `status` | Must equal `Approved` (exact, case-sensitive) |
| `human_decision` | Must equal `APPROVE`. Human-only field. Never write it. |
| `render_source_path` | Path to the exact human-approved master required by the current render-stage rule |
| Open Items | Unresolved items linked to the design, with their bearing on the next production step |
| Soft-IP concerns | Ame's active soft-IP concerns for the design |
| Budget rules | Current budget rules and the next production step's cost |
| Governing docs | The documents `00 Foundations (CURRENT)` identifies as governing |

If the record cannot be read, or a required field is absent or has more than one
value, the verdict is `INVALID_RECORD`. Never fill a gap with a guess.

## Canonical Documentation

**`00 Foundations (CURRENT)`** is the canonical documentation entry point, the
front door. It holds summaries, decisions, open items, and links to the
governing documents. It is where you start reading, not itself the rulebook.

Authority hierarchy:

- Where a document is explicitly identified as governing, that governing
  document controls the rule it covers.
- Proposed, draft, unverified, stale, or superseded material never overrides a
  current governing document.
- If two sources conflict and the governing authority is clear, apply the
  governing rule and note the conflict in that check's `detail`. The conflict
  alone does not fail the check.
- Return `DOCUMENTATION_CONFLICT` only when you cannot confidently determine
  which rule governs. Not knowing is a failure, never a pass.

## Procedure

Run every check you are able to run. Record each in `checks`. The verdict's
`reason_code` is the **highest-precedence failure** in the order below. If every
check passes on evidence, the code is `READY`.

1. **Record validity.** Exactly one `design_id`, record readable, required
   fields present and single-valued. Fail: `INVALID_RECORD`.
2. **Status.** `status` equals `Approved` exactly. Fail: `STATUS_NOT_APPROVED`.
3. **Human decision.** `human_decision` equals `APPROVE`. Blank or absent:
   `MISSING_HUMAN_APPROVAL`. `REVISE`: `HUMAN_REVISE`. `REJECT`: `HUMAN_REJECT`.
   Any other value: `INVALID_RECORD`. Read this field only. Never set, clear,
   normalise, or "correct" it, and never infer it from status, chat, or intent.
4. **Source master.** `render_source_path` must name one exact file, and that
   file must be the exact human-approved master that the current governing
   render-stage rule requires. Read that rule first. Dimensions are supporting
   evidence only: never enforce a minimum resolution unless the current
   governing documentation explicitly defines one.
   - Absent or blank: `MISSING_SOURCE`.
   - A folder, a wildcard, a list, several candidate files, or a path that could
     resolve to more than one file: `AMBIGUOUS_SOURCE`.
   - A thumbnail, preview, proof, contact sheet, mockup, web export, or any file
     whose name, folder, or designation marks it as derived rather than the
     approved master: `SOURCE_NOT_MASTER`.
   - Path exists but you cannot confirm it is the approved master the rule
     requires (no master designation, no approval link, unreadable file):
     `SOURCE_UNVERIFIED`.
   - The governing render-stage source rule is itself unresolved or you cannot
     confidently identify it: `DOCUMENTATION_CONFLICT`. Do not guess which
     master the step needs.
5. **Soft IP.** Any active soft-IP concern raised by Ame on this design:
   `SOFT_IP_BLOCK`. Only Ame or Jody can close a concern. A concern with no
   recorded resolution is active.
6. **Open Items.** Any unresolved Open Item that materially affects the design
   or the specific next production step (artwork, text, placement, product,
   sizing, colour, legal, approval scope, the step's inputs): `OPEN_ITEM_BLOCK`.
   Unrelated Open Items do not block readiness; name them in `detail` anyway.
   If you cannot determine whether an item bears on the next step, treat it as
   material and fail closed.
7. **Documentation.** Every rule used in checks 2 to 8 must come from a
   document that `00 Foundations (CURRENT)` identifies as governing. If a
   proposed, draft, unverified, stale, or superseded document disagrees with
   the governing one, apply the governing rule, pass this check, and record
   the conflict in `detail`. Fail with `DOCUMENTATION_CONFLICT` only when you
   cannot confidently determine which document governs a rule this verdict
   depends on.
8. **Budget.** The current budget rules must permit the next production step.
   Cap reached, step cost unknown, or rules unreadable: `BUDGET_BLOCK`.
9. **Anything else.** A blocker that fits no code above: `UNKNOWN_BLOCKER`.
   Never map an unfamiliar problem onto `READY`.

Stop evaluating a check as soon as it fails, but keep evaluating the remaining
checks where the evidence is available, so the human sees the full picture.
Mark a check `not_evaluated` when an earlier failure makes it meaningless.

## Output

Return exactly this shape, as JSON, and nothing that contradicts it in prose:

```json
{
  "design_id": "string",
  "ready": true,
  "state": "Ready | Not Ready | Blocked",
  "reason_code": "one code from the list below",
  "message": "one or two plain sentences a human can act on",
  "human_action_required": "what a human must do, or null",
  "checks": [
    { "check": "record_validity", "result": "pass | fail | not_evaluated", "detail": "evidence read" },
    { "check": "status", "result": "…", "detail": "…" },
    { "check": "human_decision", "result": "…", "detail": "…" },
    { "check": "source_master", "result": "…", "detail": "…" },
    { "check": "soft_ip", "result": "…", "detail": "…" },
    { "check": "open_items", "result": "…", "detail": "…" },
    { "check": "documentation", "result": "…", "detail": "…" },
    { "check": "budget", "result": "…", "detail": "…" }
  ]
}
```

`ready` is `true` only when `reason_code` is `READY`. State mapping:

| state | reason codes |
|---|---|
| `Ready` | `READY` |
| `Not Ready` | `STATUS_NOT_APPROVED`, `MISSING_HUMAN_APPROVAL`, `HUMAN_REVISE`, `MISSING_SOURCE`, `AMBIGUOUS_SOURCE`, `SOURCE_NOT_MASTER`, `SOURCE_UNVERIFIED`, `INVALID_RECORD` |
| `Blocked` | `HUMAN_REJECT`, `SOFT_IP_BLOCK`, `OPEN_ITEM_BLOCK`, `DOCUMENTATION_CONFLICT`, `BUDGET_BLOCK`, `UNKNOWN_BLOCKER` |

`Not Ready` means a routine human step will fix it. `Blocked` means a human
decision is needed before anything else happens. The `detail` of every check
names what was read (field, file, doc) so the verdict can be audited.

## Never Do

This skill is validation-only. While running it, never:

- write to Google Sheets, or change any cell, row, status, or decision
- write to Google Drive: no uploads, moves, renames, copies, or deletes
- call Printify or Etsy, for any purpose, including reads that create orders,
  drafts, or listings
- render, export, resize, or convert any artwork
- delegate the check or any part of it to another agent
- modify `human_decision`, `status`, or any field of the design record
- mutate any external system at all, or retry an action that would
- treat a failed or unverifiable check as a pass because "it is probably fine"

If completing the check would require any of the above, stop and return
`UNKNOWN_BLOCKER` with the reason in `message`.

## Pitfalls

- **Status says Approved, decision is blank.** Status is not approval. Only
  `human_decision = APPROVE` is. Return `MISSING_HUMAN_APPROVAL`.
- **The source folder holds one obvious master.** A folder is still ambiguous.
  The record must name the file. Return `AMBIGUOUS_SOURCE`.
- **A file looks small, so it must be wrong.** Width alone rejects nothing.
  The test is whether the file is the exact approved master the current
  governing render-stage rule requires. 1901 has not settled whether the
  render stage takes the approved master or the final print file, so read the
  current rule and apply it. No rule you can identify: `DOCUMENTATION_CONFLICT`.
- **A "final" file with no approval link.** Name and size are not evidence
  of approval. If you cannot tie the file to the human-approved master,
  `SOURCE_UNVERIFIED`.
- **A newer proposed rulebook loosens the budget cap.** Proposed does not
  govern. Apply the current governing rule, pass the documentation check with
  the conflict noted, and let the budget check return `BUDGET_BLOCK`.
- **Open Item marked "minor" by an agent.** Materiality is judged against the
  design and the next production step, not the label. If it changes what gets
  printed or what the next step consumes, it is material. An item about a
  different product or a later step does not block this one.
- **Human asks "just pass it this once".** Return the true verdict. Humans can
  change the record; this skill cannot.

## Examples

### 1. Missing human approval

Input:

```json
{ "design_id": "1901-017", "status": "Approved", "human_decision": "",
  "render_source_path": "Masters/1901-017_master.png",
  "open_items": [], "soft_ip_concerns": [], "budget": { "next_step_permitted": true } }
```

Output:

```json
{ "design_id": "1901-017", "ready": false, "state": "Not Ready",
  "reason_code": "MISSING_HUMAN_APPROVAL",
  "message": "Status is Approved but human_decision is blank. A human must record APPROVE before production.",
  "human_action_required": "Jody or Ame: set human_decision to APPROVE, REVISE, or REJECT for 1901-017.",
  "checks": [
    { "check": "record_validity", "result": "pass", "detail": "single record, required fields present" },
    { "check": "status", "result": "pass", "detail": "status = Approved" },
    { "check": "human_decision", "result": "fail", "detail": "human_decision is empty" },
    { "check": "source_master", "result": "pass", "detail": "exact approved master; dimensions verified against current governing source rule" },
    { "check": "soft_ip", "result": "pass", "detail": "no active concerns" },
    { "check": "open_items", "result": "pass", "detail": "none unresolved" },
    { "check": "documentation", "result": "pass", "detail": "governing docs identified via 00 Foundations (CURRENT); no conflicts" },
    { "check": "budget", "result": "pass", "detail": "next step within cap" } ] }
```

### 2. Thumbnail source

Input:

```json
{ "design_id": "1901-021", "status": "Approved", "human_decision": "APPROVE",
  "render_source_path": "Previews/1901-021_thumb_300x360.png",
  "open_items": [], "soft_ip_concerns": [], "budget": { "next_step_permitted": true } }
```

Output:

```json
{ "design_id": "1901-021", "ready": false, "state": "Not Ready",
  "reason_code": "SOURCE_NOT_MASTER",
  "message": "render_source_path points to a thumbnail in Previews, not the exact approved master the current render-stage rule requires.",
  "human_action_required": "Set render_source_path to the exact production master file for 1901-021.",
  "checks": [
    { "check": "record_validity", "result": "pass", "detail": "single record, required fields present" },
    { "check": "status", "result": "pass", "detail": "status = Approved" },
    { "check": "human_decision", "result": "pass", "detail": "human_decision = APPROVE" },
    { "check": "source_master", "result": "fail", "detail": "Previews folder, named thumb; not the approved master the current render-stage rule requires (300x360 supports this)" },
    { "check": "soft_ip", "result": "pass", "detail": "no active concerns" },
    { "check": "open_items", "result": "pass", "detail": "none unresolved" },
    { "check": "documentation", "result": "pass", "detail": "governing docs identified via 00 Foundations (CURRENT); no conflicts" },
    { "check": "budget", "result": "pass", "detail": "next step within cap" } ] }
```

### 3. Fully ready design

Input:

```json
{ "design_id": "1901-033", "status": "Approved", "human_decision": "APPROVE",
  "render_source_path": "Masters/1901-033_master.png",
  "open_items": [ { "id": "OI-88", "resolved": true } ], "soft_ip_concerns": [],
  "budget": { "next_step_permitted": true } }
```

Output:

```json
{ "design_id": "1901-033", "ready": true, "state": "Ready", "reason_code": "READY",
  "message": "All readiness checks passed on read evidence. 1901-033 may enter the next production step.",
  "human_action_required": null,
  "checks": [
    { "check": "record_validity", "result": "pass", "detail": "single record, required fields present" },
    { "check": "status", "result": "pass", "detail": "status = Approved" },
    { "check": "human_decision", "result": "pass", "detail": "human_decision = APPROVE" },
    { "check": "source_master", "result": "pass", "detail": "exact approved master; dimensions verified against current governing source rule" },
    { "check": "soft_ip", "result": "pass", "detail": "no active concerns" },
    { "check": "open_items", "result": "pass", "detail": "OI-88 resolved; none open" },
    { "check": "documentation", "result": "pass", "detail": "governing docs identified via 00 Foundations (CURRENT); no conflicts" },
    { "check": "budget", "result": "pass", "detail": "next step within cap" } ] }
```

### 4. Soft-IP block

Input:

```json
{ "design_id": "1901-040", "status": "Approved", "human_decision": "APPROVE",
  "render_source_path": "Masters/1901-040_master.png",
  "open_items": [], "soft_ip_concerns": [ { "raised_by": "Ame", "note": "slogan resembles a registered mark", "resolved": false } ],
  "budget": { "next_step_permitted": true } }
```

Output:

```json
{ "design_id": "1901-040", "ready": false, "state": "Blocked", "reason_code": "SOFT_IP_BLOCK",
  "message": "Ame has an active soft-IP concern on the slogan. Production is blocked until she resolves it.",
  "human_action_required": "Ame: resolve or withdraw the soft-IP concern on 1901-040.",
  "checks": [
    { "check": "record_validity", "result": "pass", "detail": "single record, required fields present" },
    { "check": "status", "result": "pass", "detail": "status = Approved" },
    { "check": "human_decision", "result": "pass", "detail": "human_decision = APPROVE" },
    { "check": "source_master", "result": "pass", "detail": "exact approved master; dimensions verified against current governing source rule" },
    { "check": "soft_ip", "result": "fail", "detail": "1 active concern raised by Ame, unresolved" },
    { "check": "open_items", "result": "pass", "detail": "none unresolved" },
    { "check": "documentation", "result": "pass", "detail": "governing docs identified via 00 Foundations (CURRENT); no conflicts" },
    { "check": "budget", "result": "pass", "detail": "next step within cap" } ] }
```

### 5. Unresolved Open Item

Input:

```json
{ "design_id": "1901-045", "status": "Approved", "human_decision": "APPROVE",
  "render_source_path": "Masters/1901-045_master.png",
  "open_items": [ { "id": "OI-102", "note": "back print placement not confirmed", "resolved": false } ],
  "soft_ip_concerns": [], "budget": { "next_step_permitted": true } }
```

Output:

```json
{ "design_id": "1901-045", "ready": false, "state": "Blocked", "reason_code": "OPEN_ITEM_BLOCK",
  "message": "Open Item OI-102 (back print placement) is unresolved and bears on the next production step.",
  "human_action_required": "Resolve OI-102 for 1901-045 and record the decision, then re-run readiness.",
  "checks": [
    { "check": "record_validity", "result": "pass", "detail": "single record, required fields present" },
    { "check": "status", "result": "pass", "detail": "status = Approved" },
    { "check": "human_decision", "result": "pass", "detail": "human_decision = APPROVE" },
    { "check": "source_master", "result": "pass", "detail": "exact approved master; dimensions verified against current governing source rule" },
    { "check": "soft_ip", "result": "pass", "detail": "no active concerns" },
    { "check": "open_items", "result": "fail", "detail": "OI-102 unresolved, material to the next step (placement)" },
    { "check": "documentation", "result": "pass", "detail": "governing docs identified via 00 Foundations (CURRENT); no conflicts" },
    { "check": "budget", "result": "pass", "detail": "next step within cap" } ] }
```

### 6. Proposed doc conflicts with the governing budget rule

Input:

```json
{ "design_id": "1901-051", "status": "Approved", "human_decision": "APPROVE",
  "render_source_path": "Masters/1901-051_master.png",
  "open_items": [], "soft_ip_concerns": [],
  "budget": { "next_step_cost": 48, "cap_remaining_per_current_docs": 40, "cap_remaining_per_proposed_docs": 60 } }
```

Output:

```json
{ "design_id": "1901-051", "ready": false, "state": "Blocked", "reason_code": "BUDGET_BLOCK",
  "message": "The current governing budget rule does not permit the next step (48 against 40 remaining). A proposed document would permit it, but proposed material does not override the current governing rule.",
  "human_action_required": "Jody: raise the cap under the governing rule, adopt the proposed rule as governing, or hold 1901-051.",
  "checks": [
    { "check": "record_validity", "result": "pass", "detail": "single record, required fields present" },
    { "check": "status", "result": "pass", "detail": "status = Approved" },
    { "check": "human_decision", "result": "pass", "detail": "human_decision = APPROVE" },
    { "check": "source_master", "result": "pass", "detail": "exact approved master; dimensions verified against current governing source rule" },
    { "check": "soft_ip", "result": "pass", "detail": "no active concerns" },
    { "check": "open_items", "result": "pass", "detail": "none unresolved" },
    { "check": "documentation", "result": "pass", "detail": "governing budget rule identified via 00 Foundations (CURRENT) and applied; a proposed doc conflicts on the cap but does not govern" },
    { "check": "budget", "result": "fail", "detail": "next step 48 exceeds 40 remaining under the governing rule" } ] }
```

## Verification

The skill worked if the reply is one JSON object in the shape above, every
`checks[].detail` names the evidence that was read, `ready` is `true` only with
`reason_code = READY`, and no record, sheet, file, or external system changed
during the run.
