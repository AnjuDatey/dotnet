## PURPOSE

Execute targeted .NET tests to validate a bug fix and produce
a structured size-capped test-validation.md artifact.

This skill governs test discovery, execution, evidence
collection, and status classification only.

---

## PROHIBITED SOURCES

The following are never permitted as input, fallback, or
justification at any point:

- Memory or cached results from previous runs
- Previous workflow or agent output
- Git history or branch names
- Stale artifact content from another Work Item
- Independent root cause analysis
- Static code inspection as a substitute for execution

---

## STALE ARTIFACT CHECK

Before executing any test, verify Work Item ID consistency.

Read WORK_ITEM_ID from bug-details.md.
Read WORK_ITEM_ID from bug-analysis.md.
Read WORK_ITEM_ID from test-validation.md.

All three must exactly match the Work Item ID provided in
the current task.

If any does not match:
- Do not execute any test.
- Update test-validation.md POST-FIX section with:
  VALIDATION_TYPE: POST-FIX
  TEST_VALIDATION_STATUS: NOT_EXECUTED
  TEST_FAILURE_REASON: STALE_BUG_CONTEXT
  All other fields: Not Available
- Return exactly:
  TEST_VALIDATION_STATUS=NOT_EXECUTED
  TEST_FAILURE_REASON=STALE_BUG_CONTEXT

---

## AUTHORITATIVE TEST ANCHOR

Read the PRE-FIX VALIDATION section from test-validation.md.

Identify and record:
- AUTHORITATIVE_TEST_PROJECT
- AUTHORITATIVE_TEST_FILE
- AUTHORITATIVE_TEST_METHOD
- TEST_SCENARIO
- EXPECTED_BEHAVIOUR
- PRE-FIX result and evidence

The authoritative test established by the Pre-Fix Test
Sub-Agent is fixed. You must reuse it exactly.

Do NOT:
- Replace it with a different test
- Modify it to make it pass
- Treat a different test as equivalent
- Skip it because it is inconvenient to execute

---

## TEST EXECUTION PROCEDURE

STEP 1 — Restore:
"$DOTNET_PATH" restore <SolutionOrProjectPath>

STEP 2 — Build:
"$DOTNET_PATH" build <SolutionOrProjectPath> \
  --no-restore \
  --configuration Release

STEP 3 — Execute authoritative test:
"$DOTNET_PATH" test <TestProjectPath> \
  --no-build \
  --filter "<AuthoritativeTestFilter>" \
  --configuration Release \
  --logger "console;verbosity=detailed"

STEP 4 — Only if authoritative test passes, execute
targeted regression tests:
"$DOTNET_PATH" test <TestProjectPath> \
  --no-build \
  --filter "<RegressionTestFilter>" \
  --configuration Release \
  --logger "console;verbosity=detailed"

You may create or update targeted regression test code only.
Do not create or modify the authoritative test.

---

## STATUS CLASSIFICATION

TEST_VALIDATION_STATUS=PASSED only when ALL are true:
- Authoritative test actually executed
- Authoritative test passed
- Required targeted regression tests actually executed
- Required targeted regression tests passed
- SDK successfully invoked
- Actual execution evidence exists for current run

TEST_VALIDATION_STATUS=FAILED when:
- Authoritative test executed but failed, OR
- A required regression test executed but failed

TEST_VALIDATION_STATUS=NOT_EXECUTED when:
- SDK failure
- Restore failure
- Build failure
- Environment failure
- Test discovery failure

A zero exit code alone is NOT sufficient if the intended
tests did not actually execute.

---

## OUTPUT FORMAT

Update working-repo/.agent/test-validation.md.

PRESERVE the complete existing PRE-FIX VALIDATION section.
Do not overwrite, remove, or alter it.

Append a POST-FIX VALIDATION section with exactly these
fields in this exact order.

No headings. No markdown. No prose. No extra fields.
Respect all word limits.

POST-FIX VALIDATION

WORK_ITEM_ID: <integer>
VALIDATION_TYPE: POST-FIX
SDK_RESOLUTION: <RESOLVED|FAILED>
DOTNET_PATH: <absolute path>
SDK_VERSION: <e.g. 10.0.1>
TEST_PROJECT: <relative path>
TEST_FILE: <relative path>
TEST_CLASS: <class name>
TEST_METHOD: <method name>
TEST_FILTER: <exact filter string used>
TEST_SCENARIO: <max 40 words>
EXPECTED_BEHAVIOUR: <max 40 words>
AUTHORITATIVE_TEST_EXECUTED: <YES|NO>
AUTHORITATIVE_TEST_RESULT: <PASSED|FAILED|NOT_EXECUTED>
FAILURE_EVIDENCE: <max 80 words of actual stdout/stderr>
BUG_CORRESPONDENCE: <max 40 words — how result confirms fix>
REGRESSION_TESTS_EXECUTED: <integer>
REGRESSION_TESTS_PASSED: <integer>
REGRESSION_TESTS_FAILED: <integer>
REGRESSION_FAILURE_EVIDENCE: <max 80 words, blank if none>
TEST_VALIDATION_STATUS: <PASSED|FAILED|NOT_EXECUTED>
TEST_FAILURE_REASON: <specific reason, blank if PASSED>

---

## COMPLETION

After updating test-validation.md return exactly these
lines and nothing else:

TEST_VALIDATION_STATUS=PASSED

or on failure:

TEST_VALIDATION_STATUS=FAILED
TEST_FAILURE_REASON=<AUTHORITATIVE_TEST_FAILED|REGRESSION_TEST_FAILED>

or if tests could not execute:

TEST_VALIDATION_STATUS=NOT_EXECUTED
TEST_FAILURE_REASON=<specific reason>

After returning the status lines:
- Take no further action.
- Do not modify production code.
- Do not commit, push, or create a branch.
- Do not call any other tool.
- Do not continue reasoning.
- Return control to the orchestrator immediately.
