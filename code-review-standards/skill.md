## PURPOSE

Independently review a production-code bug fix against the
approved RCA, verify scope compliance, and produce a
structured size-capped code-review.md artifact.

This skill governs review criteria, findings classification,
artifact format, and status values only.

---

## PROHIBITED SOURCES

The following are never permitted as input, fallback, or
justification at any point:

- Memory or cached results from previous runs
- Previous workflow or agent output
- Independent root cause analysis
- Assumptions not present in bug-details.md or
  bug-analysis.md
- Approval based on the fact that changes were made

---

## STALE ARTIFACT CHECK

Before inspecting any code, verify Work Item ID consistency.

Read WORK_ITEM_ID from bug-details.md.
Read WORK_ITEM_ID from bug-analysis.md.
Read WORK_ITEM_ID from fix-summary.md.

All three must exactly match the Work Item ID provided in
the current task.

If any does not match:
- Do not inspect any code.
- Write code-review.md with:
  REVIEW_STATUS: FAILED
  REVIEW_FAILURE_REASON: STALE_BUG_CONTEXT
  All other fields: Not Available
- Return exactly:
  REVIEW_STATUS=FAILED
  REVIEW_FAILURE_REASON=STALE_BUG_CONTEXT

---

## INPUT RULES

After the stale artifact check passes read:

From bug-details.md:
- WORK_ITEM_ID
- TITLE
- EXPECTED_BEHAVIOUR
- ACCEPTANCE_CRITERIA

From bug-analysis.md:
- ROOT_CAUSE_SUMMARY
- AFFECTED_FILES
- AFFECTED_CLASSES
- AFFECTED_METHODS
- FIX_APPROACH

From fix-summary.md:
- FILES_MODIFIED
- ADDITIONAL_FILES
- FIX_DESCRIPTION
- BUILD_STATUS
- BUILD_ATTEMPTS

Use FILES_MODIFIED and ADDITIONAL_FILES as the declared
scope boundary for the review.

---

## GIT DIFF PROCEDURE

Execute exactly this command to obtain the diff:

git -C working-repo diff main...HEAD \
  --unified=5 \
  --no-color

If that fails, fall back to:

git -C working-repo diff HEAD~1 HEAD \
  --unified=5 \
  --no-color

Record which command produced the diff.
If neither produces a usable diff, set REVIEW_STATUS=FAILED
and REVIEW_FAILURE_REASON=GIT_DIFF_UNAVAILABLE.

---

## REVIEW CRITERIA

Apply all seven criteria. Each must be explicitly assessed.

CRITERION 1 — ROOT CAUSE ADDRESSED
The implementation must address the root cause identified
in bug-analysis.md ROOT_CAUSE_SUMMARY and FIX_APPROACH.
Static reasoning must not substitute for evidence in the
diff.

CRITERION 2 — SCOPE COMPLIANCE
Files present in the diff must match FILES_MODIFIED and
ADDITIONAL_FILES from fix-summary.md.
Any file in the diff not declared in fix-summary.md is a
scope violation.

CRITERION 3 — ARCHITECTURE AND PATTERNS
The implementation must follow the existing architecture
and coding patterns visible in the surrounding code.
Do not require modernization not mandated by the fix.

CRITERION 4 — CONTRACT PRESERVATION
Existing API, response, and interface contracts must be
preserved unless the approved fix explicitly requires a
change.

CRITERION 5 — ERROR HANDLING AND SECURITY
Appropriate error handling must be present.
No new security concerns must be introduced.

CRITERION 6 — TEST ADEQUACY
Relevant unit or regression tests must be present.
Tests must cover the reported defect scenario and
important edge cases.
Test coverage is assessed against bug-analysis.md
REGRESSION_TESTS and EDGE_CASES fields.

CRITERION 7 — NO REGRESSIONS
The implementation must not introduce obvious regressions
in behavior not related to the approved fix.

---

## FINDINGS CLASSIFICATION

BLOCKING — must be resolved before approval:
- Root cause not addressed
- Scope violation — undeclared files modified
- Contract broken without justification
- Security concern introduced
- Required tests absent
- Authoritative test modified

NON-BLOCKING — should be noted but do not prevent approval:
- Style inconsistencies
- Optional improvements
- Minor naming issues

APPROVED — no blocking findings.

CHANGES_REQUIRED — one or more blocking findings present.

FAILED — review cannot be completed:
- Stale artifact context
- Git diff unavailable
- Required artifacts missing or unreadable

---

## REVIEW-FIX ITERATION RULES

When this is a review-fix iteration:
- Re-read bug-details.md, bug-analysis.md, and
  fix-summary.md fresh.
- Re-obtain the git diff fresh.
- Verify each previously reported blocking finding is
  resolved.
- Do not approve solely because changes were made.
- Re-apply all seven criteria to the updated diff.
- Report any remaining or newly introduced blocking issues.

---

## OUTPUT FORMAT

Write working-repo/.agent/code-review.md with exactly these
fields in this exact order.

No headings. No markdown. No prose. No extra fields.
Respect all word limits.

WORK_ITEM_ID: <integer>
REVIEW_STATUS: <APPROVED|CHANGES_REQUIRED|FAILED>
REVIEW_FAILURE_REASON: <one line, blank if not FAILED>
GIT_DIFF_COMMAND: <exact command used>
FILES_IN_DIFF: <comma-separated list>
UNDECLARED_FILES: <files in diff not in fix-summary.md,
                   blank if none>
CRITERION_1_ROOT_CAUSE: <PASS|FAIL> — <max 30 words>
CRITERION_2_SCOPE: <PASS|FAIL> — <max 30 words>
CRITERION_3_ARCHITECTURE: <PASS|FAIL> — <max 30 words>
CRITERION_4_CONTRACTS: <PASS|FAIL> — <max 30 words>
CRITERION_5_ERROR_SECURITY: <PASS|FAIL> — <max 30 words>
CRITERION_6_TESTS: <PASS|FAIL> — <max 30 words>
CRITERION_7_REGRESSIONS: <PASS|FAIL> — <max 30 words>
BLOCKING_FINDINGS: <count>
REJECTION_REASONS: <comma-separated blocking finding
                    labels, blank if APPROVED>
REVIEW_NOTES: <max 120 words — actionable findings for
               the Coding Sub-Agent, blank if APPROVED>

---

## COMPLETION

After writing working-repo/.agent/code-review.md return
exactly these lines and nothing else:

REVIEW_STATUS=APPROVED

or:

REVIEW_STATUS=CHANGES_REQUIRED
BLOCKING_FINDINGS=<count>

or:

REVIEW_STATUS=FAILED
REVIEW_FAILURE_REASON=<specific reason>

After returning the status lines:
- Take no further action.
- Do not modify any source or test file.
- Do not commit, push, or create a branch.
- Do not call any other tool.
- Do not continue reasoning.
- Return control to the orchestrator immediately.
