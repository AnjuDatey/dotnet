## PURPOSE

Investigate the reported bug against the repository source code,
confirm the technical root cause, and produce a structured
size-capped bug-analysis.md artifact.

This skill performs root cause analysis and fix planning only.

It must not modify source code, modify tests, implement the fix,
commit, push, create branches, or create pull requests.

---

## PROHIBITED SOURCES

The following are never permitted as input, fallback, or
verification source at any point during this skill's execution:

- Memory or cached analysis from previous runs
- Previous workflow output or agent output
- Existing working-repo/.agent/bug-analysis.md content
- Assumptions based on similar bugs or prior executions
- Any Work Item other than the one specified in the current task

This prohibition applies to all sections of this skill
without exception.

---

## INPUT RULES

Read working-repo/.agent/bug-details.md before beginning
analysis.

Use only these fields from bug-details.md:

- WORK_ITEM_ID — carry into bug-analysis.md exactly as-is
- AFFECTED_AREA — starting point for code investigation
- REPRO_STEPS_SUMMARY — primary input for tracing application flow
- EXPECTED_BEHAVIOUR — defines the correct outcome
- ACTUAL_BEHAVIOUR — defines the observed defect
- ACCEPTANCE_CRITERIA — defines the success condition

Do not use BUG_ELIGIBILITY_STATUS or INELIGIBILITY_REASON
as analysis inputs.

If bug-details.md does not exist or cannot be read:
- Set ANALYSIS_STATUS=INSUFFICIENT_INFORMATION
- Set ANALYSIS_FAILURE_REASON=BUG_DETAILS_NOT_FOUND
- Write bug-analysis.md with all other fields set to
  Not Available
- Return control immediately

---

## ANALYSIS STEPS

Execute these steps in order:

STEP 1 — Read bug-details.md using the fields listed in
INPUT RULES.

STEP 2 — Locate the affected area in working-repo/ using
the AFFECTED_AREA field as the starting point.

STEP 3 — Trace the application flow related to the reported
issue using REPRO_STEPS_SUMMARY as the guide.

STEP 4 — Identify the specific code location where the
defect originates.

STEP 5 — Distinguish clearly between:
- Confirmed root cause: supported by direct code evidence
  (specific file, class, method, line range)
- Unconfirmed hypothesis: plausible but lacking direct
  code evidence

STEP 6 — Identify all files that require modification.
List only files with a confirmed reason for change.
Do not list speculative files.

STEP 7 — Define the implementation approach at a conceptual
level. Do not write code. Do not modify any file.

STEP 8 — Identify required regression tests.
Prefer existing tests that can be adapted.
Note new tests only when no existing test covers the scenario.

STEP 9 — Identify relevant edge cases introduced or exposed
by the fix.

STEP 10 — Write bug-analysis.md using the OUTPUT FORMAT
defined below.

---

## ROOT CAUSE CONFIRMATION RULES

Set ANALYSIS_STATUS=READY_FOR_FIX only when ALL of the
following are true:

1. A specific file, class, and method has been identified
   as the origin of the defect.
2. The code evidence directly supports the reported
   ACTUAL_BEHAVIOUR.
3. A concrete implementation approach is definable without
   further investigation.
4. The required change is within the repository scope.

Set ANALYSIS_STATUS=ROOT_CAUSE_NOT_CONFIRMED when:
- The investigation does not produce direct code evidence
  linking a specific code location to the reported defect.

Set ANALYSIS_STATUS=INSUFFICIENT_INFORMATION when:
- bug-details.md does not contain enough information to
  begin or complete a meaningful investigation.

Set ANALYSIS_STATUS=UNSUPPORTED_CHANGE when:
- The required change is outside the repository scope.
- The change violates repository constraints.
- The change requires external system modifications beyond
  this repository.

---

## OUTPUT FORMAT

Write working-repo/.agent/bug-analysis.md with exactly these
fields in this exact order.

No headings. No markdown formatting. No prose paragraphs.
No comments. No additional fields. Respect all word limits.

WORK_ITEM_ID: <integer matching bug-details.md WORK_ITEM_ID>
ANALYSIS_STATUS: <READY_FOR_FIX|ROOT_CAUSE_NOT_CONFIRMED|
                  INSUFFICIENT_INFORMATION|UNSUPPORTED_CHANGE>
ANALYSIS_FAILURE_REASON: <one line, blank if READY_FOR_FIX>
ROOT_CAUSE_CONFIRMED: <YES|NO>
ROOT_CAUSE_SUMMARY: <max 60 words — confirmed cause only>
ROOT_CAUSE_EVIDENCE: <max 80 words — file, class, method,
                      line range supporting the conclusion>
AFFECTED_FILES: <comma-separated relative paths, max 10 files>
AFFECTED_CLASSES: <comma-separated class names>
AFFECTED_METHODS: <comma-separated method names>
FIX_APPROACH: <max 80 words — conceptual implementation
               approach, no code>
REGRESSION_TESTS: <max 60 words — existing tests to adapt
                   or new tests required>
EDGE_CASES: <max 40 words — relevant edge cases>
RISK_ASSESSMENT: <max 40 words — regression, compatibility,
                  security, or integration risks>
OUT_OF_SCOPE: <max 40 words — unrelated issues found,
               or None>

---

## FAILURE HANDLING

If the root cause cannot be confirmed after full investigation:

WORK_ITEM_ID: <integer>
ANALYSIS_STATUS: ROOT_CAUSE_NOT_CONFIRMED
ANALYSIS_FAILURE_REASON: <max 40 words explaining why>
ROOT_CAUSE_CONFIRMED: NO
ROOT_CAUSE_SUMMARY: Not Confirmed
ROOT_CAUSE_EVIDENCE: Not Available
AFFECTED_FILES: Not Available
AFFECTED_CLASSES: Not Available
AFFECTED_METHODS: Not Available
FIX_APPROACH: Not Available
REGRESSION_TESTS: Not Available
EDGE_CASES: Not Available
RISK_ASSESSMENT: Not Available
OUT_OF_SCOPE: Not Available

Apply the same pattern for INSUFFICIENT_INFORMATION and
UNSUPPORTED_CHANGE, setting ANALYSIS_STATUS and
ANALYSIS_FAILURE_REASON appropriately.

---

## COMPLETION

After writing working-repo/.agent/bug-analysis.md return
exactly one of the following lines and nothing else:

ANALYSIS_STATUS=READY_FOR_FIX
ANALYSIS_STATUS=ROOT_CAUSE_NOT_CONFIRMED
ANALYSIS_STATUS=INSUFFICIENT_INFORMATION
ANALYSIS_STATUS=UNSUPPORTED_CHANGE

After returning the status line:
- Take no further action.
- Do not read any other file.
- Do not call any other tool.
- Do not modify any file.
- Do not continue reasoning.
- Return control to the orchestrator immediately.
