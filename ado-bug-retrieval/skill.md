---
name: ado-bug-retrieval
description: Retrieve and validate an Azure DevOps Work Item for bug-fix eligibility, producing a structured bug-details.md artifact. Use when a workflow needs to fetch and validate a Bug work item before any analysis or fix work begins. Do NOT use for root-cause analysis, source code inspection, code modification, running tests, or creating branches/commits.
---
## PURPOSE

Retrieve and validate the Azure DevOps Work Item identified by
the current task and produce a structured, size-capped
bug-details.md artifact.

This skill performs Work Item retrieval and eligibility
validation only.

It must not perform root-cause analysis, inspect source code,
modify code, run tests, create branches, commit changes, or
implement fixes.

---

## PROHIBITED SOURCES

The following are never permitted as input, fallback, or
verification source at any point during this skill's execution:

- Git history
- Repository files or source code
- Existing working-repo/.agent/bug-details.md
- Existing working-repo/.agent/bug-analysis.md
- Memory or cached Work Item information
- Previous workflow output or agent output
- Assumptions based on similar bugs or prior executions

This prohibition applies to all sections of this skill without
exception.

---

## INPUT

The task provides the current Azure DevOps Work Item ID.

The Work Item ID provided in the current task is the only
authoritative identifier for this execution.

Do not obtain or infer the Work Item ID from any other source.

---

## AUTHORITATIVE SOURCE

The Azure DevOps Work Item retrieved through the Azure DevOps
MCP tool is the only authoritative source for this skill.

Use only the current Work Item retrieved for the current task.

---

## RETRIEVAL RULES

1. Retrieve the Work Item using the Azure DevOps MCP tool.
2. Use the exact Work Item ID provided in the current task.
3. Verify that the retrieved Work Item ID exactly matches the
   requested ID.
4. Do not continue processing if the retrieved ID does not
   match the requested ID.
5. If the retrieved ID does not match, set:
   BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE
   INELIGIBILITY_REASON=WORK_ITEM_MISMATCH
6. If the Work Item cannot be retrieved or no valid Work Item
   is returned, set:
   BUG_ELIGIBILITY_STATUS=RETRIEVAL_FAILED
7. Never retry retrieval using a different Work Item ID.
8. See PROHIBITED SOURCES for all forbidden fallback sources.

---

## ELIGIBILITY CRITERIA

A Work Item is ELIGIBLE only when all four criteria below pass.

### Criterion 1 — Work Item Type

The Work Item type must be exactly: Bug

If the type is anything else:
BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE
Reason: WORK_ITEM_TYPE_NOT_BUG

### Criterion 2 — Work Item State

The Work Item state must be exactly one of:
- Active
- In Progress

If the state is anything else:
BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE
Reason: WORK_ITEM_STATE_NOT_ACTIVE_OR_IN_PROGRESS

### Criterion 3 — Reproduction Information

At least one of these fields must contain non-empty content:
- Microsoft.VSTS.TCM.ReproSteps
- System.Description

Empty means the field is null, missing, whitespace-only, or
contains no meaningful text after stripping HTML markup.

If both fields are empty:
BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE
Reason: REPRO_STEPS_AND_DESCRIPTION_EMPTY

### Criterion 4 — Duplicate Status

Check the System.Reason field of the retrieved Work Item.

If System.Reason equals Duplicate:
BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE
Reason: WORK_ITEM_MARKED_DUPLICATE

Do not infer duplicate status from title text, comments,
links, relationship metadata, or any source other than the
System.Reason field.

---

## ELIGIBILITY DECISION

Set BUG_ELIGIBILITY_STATUS=ELIGIBLE only when all four
eligibility criteria pass.

Set BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE when one or more
eligibility criteria fail.

If multiple criteria fail, record all applicable reasons on
the same line separated by semicolons.

Example:
WORK_ITEM_TYPE_NOT_BUG;WORK_ITEM_STATE_NOT_ACTIVE_OR_IN_PROGRESS

For an eligible Work Item, INELIGIBILITY_REASON must be blank.

For a retrieval failure, set BUG_ELIGIBILITY_STATUS=RETRIEVAL_FAILED
and record a concise retrieval failure reason in
INELIGIBILITY_REASON.

---

## FIELD EXTRACTION RULES

Populate bug-details.md using only data retrieved from the
current Azure DevOps Work Item.

Do not infer missing values from repository code or external
context. See PROHIBITED SOURCES.

Strip HTML markup from retrieved fields before extracting
field values. Apply the specified word limit after stripping.

### WORK_ITEM_ID
Use the retrieved Work Item ID. Must exactly match the
requested Work Item ID.

### WORK_ITEM_TYPE
Use the retrieved Work Item type. Expected value: Bug

### BUG_ELIGIBILITY_STATUS
Use exactly one of: ELIGIBLE, NOT_ELIGIBLE, RETRIEVAL_FAILED

### INELIGIBILITY_REASON
Blank when eligible. One line stating the failed criterion
or criteria when not eligible. Concise retrieval failure
reason when retrieval fails.

### TITLE
Use the Work Item title. Maximum 20 words. Preserve meaning
without adding information when shortening is necessary.

### STATE
Use the retrieved Work Item state exactly as returned.

### ASSIGNED_TO
Use the retrieved assigned person's display name.
If no assignee: Not Available
Do not infer from comments, commits, or repository data.

### AFFECTED_AREA
Use an affected area explicitly identified in the Work Item.
Prefer in order:
1. Explicit component, service, or controller named in the
   Work Item.
2. Azure DevOps Area Path when it clearly represents the
   affected area.
If not determinable from the Work Item: Not Specified
Do not inspect repository code to determine this value.

### REPRO_STEPS_SUMMARY
Summarize the retrieved Repro Steps and/or Description.
Maximum 80 words. Preserve the essential reproduction
sequence, inputs, and relevant conditions.
Do not add technical steps not present in the Work Item.

### EXPECTED_BEHAVIOUR
Summarize the expected behavior stated in the Work Item.
Maximum 40 words.
If not specified: Not Specified
Do not infer from application code or general assumptions.

### ACTUAL_BEHAVIOUR
Summarize the actual or observed behavior stated in the
Work Item. Maximum 40 words.
If not specified: Not Specified
Do not infer from source code or previous analysis.

### ACCEPTANCE_CRITERIA
Summarize the Work Item's Acceptance Criteria.
Maximum 40 words.
If absent or empty: Not Specified
Do not create or infer acceptance criteria.

---

## OUTPUT FILE

Write: working-repo/.agent/bug-details.md

Create the .agent directory if it does not exist.

The file must contain exactly these 12 fields in this exact
order. No headings. No markdown formatting. No prose
paragraphs. No comments. No additional fields.

WORK_ITEM_ID: <integer>
WORK_ITEM_TYPE: <Bug|other>
BUG_ELIGIBILITY_STATUS: <ELIGIBLE|NOT_ELIGIBLE|RETRIEVAL_FAILED>
INELIGIBILITY_REASON: <one line, blank if ELIGIBLE>
TITLE: <max 20 words>
STATE: <Active|In Progress|other>
ASSIGNED_TO: <name or Unassigned>
AFFECTED_AREA: <one line>
REPRO_STEPS_SUMMARY: <max 80 words>
EXPECTED_BEHAVIOUR: <max 40 words>
ACTUAL_BEHAVIOUR: <max 40 words>
ACCEPTANCE_CRITERIA: <max 40 words or Not Specified>

---

## FAILURE HANDLING — ID MISMATCH

If the retrieved Work Item ID does not exactly match the
requested Work Item ID:

1. Stop eligibility evaluation immediately.
2. Do not treat the retrieved Work Item as the requested bug.
3. Do not retrieve another Work Item.
4. Write bug-details.md with exactly these values:

WORK_ITEM_ID: <the requested ID passed to this agent>
WORK_ITEM_TYPE: Unknown
BUG_ELIGIBILITY_STATUS: NOT_ELIGIBLE
INELIGIBILITY_REASON: WORK_ITEM_MISMATCH
TITLE: Not Available
STATE: Not Available
ASSIGNED_TO: Not Available
AFFECTED_AREA: Not Available
REPRO_STEPS_SUMMARY: Not Available
EXPECTED_BEHAVIOUR: Not Available
ACTUAL_BEHAVIOUR: Not Available
ACCEPTANCE_CRITERIA: Not Available

5. Return exactly:
BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE

---

## FAILURE HANDLING — RETRIEVAL FAILURE

If the Azure DevOps MCP retrieval fails or no valid Work Item
is returned:

1. Do not use any source listed in PROHIBITED SOURCES.
2. Write bug-details.md with exactly these values:

WORK_ITEM_ID: <the requested ID passed to this agent>
WORK_ITEM_TYPE: Unknown
BUG_ELIGIBILITY_STATUS: RETRIEVAL_FAILED
INELIGIBILITY_REASON: <concise retrieval failure reason>
TITLE: Not Available
STATE: Not Available
ASSIGNED_TO: Not Available
AFFECTED_AREA: Not Available
REPRO_STEPS_SUMMARY: Not Available
EXPECTED_BEHAVIOUR: Not Available
ACTUAL_BEHAVIOUR: Not Available
ACCEPTANCE_CRITERIA: Not Available

3. Return exactly:
BUG_ELIGIBILITY_STATUS=RETRIEVAL_FAILED
---
## COMPLETION

After writing working-repo/.agent/bug-details.md, return
exactly one of the following lines and nothing else:

BUG_ELIGIBILITY_STATUS=ELIGIBLE
BUG_ELIGIBILITY_STATUS=NOT_ELIGIBLE
BUG_ELIGIBILITY_STATUS=RETRIEVAL_FAILED

After returning the status line:
- Take no further action.
- Do not read any other file.
- Do not call any other tool.
- Do not continue reasoning.
- Return control to the orchestrator immediately.
