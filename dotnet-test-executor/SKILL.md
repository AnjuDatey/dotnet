<skill name="dotnet-test-executor">

## PURPOSE

Identify, execute, and record an authoritative automated test
that validates the reported bug scenario. Produce a structured
size-capped test-validation.md artifact containing actual
execution evidence.

This skill never substitutes static analysis, source-code
reading, or build output for actual test execution.

---

## PROHIBITED SOURCES

The following are never permitted as input, fallback, or
verification at any point:

- Memory or cached results from previous runs
- Previous workflow or agent output
- Source-code analysis as a substitute for execution
- Static reasoning about what a test would return
- Build output alone
- Test discovery output alone
- Previous test-validation.md content
- Another agent's claim about test results

---

## STALE ARTIFACT CHECK

Before any other step, verify Work Item ID consistency.

Read WORK_ITEM_ID from bug-details.md.
Read WORK_ITEM_ID from bug-analysis.md.

Both must exactly match the Work Item ID provided in the
current task.

If either does not match:
- Do not proceed with test identification or execution.
- Write test-validation.md with:
  TEST_VALIDATION_STATUS: TEST_EXECUTION_FAILED
  TEST_FAILURE_REASON: STALE_BUG_CONTEXT
  All other fields: Not Available
- Return exactly:
  TEST_VALIDATION_STATUS=TEST_EXECUTION_FAILED

---

## INPUT RULES

After the stale artifact check passes, read:

From bug-details.md:
- WORK_ITEM_ID
- AFFECTED_AREA
- REPRO_STEPS_SUMMARY
- EXPECTED_BEHAVIOUR
- ACTUAL_BEHAVIOUR

From bug-analysis.md:
- AFFECTED_FILES
- AFFECTED_CLASSES
- AFFECTED_METHODS
- REGRESSION_TESTS
- FIX_APPROACH (for understanding scope only)

---

## SDK RESOLUTION

Execute the dotnet-sdk-resolver skill procedure before any
dotnet command. Record the resolved DOTNET_PATH and
SDK_VERSION before proceeding.

If SDK resolution fails, write test-validation.md with:
TEST_VALIDATION_STATUS: TEST_EXECUTION_FAILED
TEST_FAILURE_REASON: SDK_RESOLUTION_FAILED
Return: TEST_VALIDATION_STATUS=TEST_EXECUTION_FAILED

---

## TEST SELECTION RULES

1. Prefer an existing test that directly asserts the reported
   bug scenario from an end-user or API perspective.
2. Create a focused regression test only when no suitable
   existing test exists.
3. The authoritative test MUST assert the CORRECT intended
   behaviour — not the defective behaviour.
4. The test must fail against the pre-fix implementation
   because the defect is present.
5. The test must pass against the post-fix implementation
   because the defect is resolved.

Correct test design example:
Bug — PageSize=0 causes DivideByZeroException.
Authoritative test — asserts that PageSize=0 returns a
default page size response without throwing an exception.
Pre-fix result — FAIL (exception thrown).
Post-fix result — PASS (correct behaviour).

Wrong test design:
Asserting that DivideByZeroException IS thrown.
This passes before the fix and passes after — proving nothing.

---

## TEST EXECUTION PROCEDURE

STEP 1 — Resolve SDK using dotnet-sdk-resolver procedure.

STEP 2 — Restore dependencies:
"$DOTNET_PATH" restore <SolutionOrProjectPath>

STEP 3 — Build the test project:
"$DOTNET_PATH" build <TestProjectPath> --no-restore

STEP 4 — Execute the authoritative test:
"$DOTNET_PATH" test <TestProjectPath> \
  --filter "FullyQualifiedName~<TestMethod>" \
  --no-build \
  --verbosity normal

STEP 5 — Capture full stdout and stderr from STEP 4.

STEP 6 — Record counts from test runner output:
- Tests discovered
- Tests executed
- Tests passed
- Tests failed

---

## BUG_REPRODUCED EVIDENCE RULES

BUG_REPRODUCED may only be returned when ALL of the
following are true:

1. The authoritative test actually executed.
2. The test failed.
3. The failure was caused by the reported application defect.
4. The observed behaviour matches the reported ACTUAL_BEHAVIOUR.
5. The SDK was successfully invoked via "$DOTNET_PATH".
6. Restore and build completed successfully.
7. The failure is a test assertion or application failure,
   not an infrastructure or environment failure.

The following are NOT valid evidence for BUG_REPRODUCED:
- Source-code analysis
- Static reasoning
- Creating or compiling the test
- Test discovery alone
- Predicting that the test will fail
- Previous test results or workflow output
- Another agent's claim
- Build success alone
- Code review or Git status

---

## STATUS DECISION RULES

TEST_VALIDATION_STATUS=BUG_REPRODUCED
All seven BUG_REPRODUCED evidence rules are satisfied.

TEST_VALIDATION_STATUS=BUG_NOT_REPRODUCED
The authoritative test executed and passed before the fix.
The reported defect is not present in the current code.

TEST_VALIDATION_STATUS=TEST_EXECUTION_FAILED
The test could not execute due to SDK, restore, build,
dependency, environment, or infrastructure failure.

TEST_VALIDATION_STATUS=TEST_UNAVAILABLE
No appropriate test can be identified or created after
genuine investigation of the repository.

---

## OUTPUT FORMAT

Write working-repo/.agent/test-validation.md with exactly
these fields in this exact order.

No headings. No markdown. No prose. No extra fields.
Respect all word limits.

WORK_ITEM_ID: <integer>
VALIDATION_TYPE: PRE-FIX
TEST_VALIDATION_STATUS: <BUG_REPRODUCED|BUG_NOT_REPRODUCED|
                         TEST_EXECUTION_FAILED|TEST_UNAVAILABLE>
TEST_FAILURE_REASON: <one line, blank if BUG_REPRODUCED>
SDK_RESOLUTION: <PASSED|FAILED>
SDK_VERSION: <e.g. 10.0.1>
DOTNET_PATH: <resolved absolute path>
TEST_PROJECT: <relative path to .csproj>
TEST_FILE: <relative path to .cs file>
TEST_CLASS: <class name>
TEST_METHOD: <method name>
TEST_SCENARIO: <max 40 words describing the scenario>
TEST_FILTER: <filter string used>
TEST_COMMAND: <full command executed>
TESTS_DISCOVERED: <integer>
TESTS_EXECUTED: <integer>
TESTS_PASSED: <integer>
TESTS_FAILED: <integer>
AUTHORITATIVE_TEST_EXECUTED: <YES|NO>
TEST_RESULT: <PASSED|FAILED|ERROR|NOT_RUN>
EXPECTED_BEHAVIOUR: <max 40 words>
ACTUAL_BEHAVIOUR: <max 40 words>
FAILURE_EVIDENCE: <max 80 words of actual stdout failure output>
BUG_CORRESPONDENCE: <max 40 words explaining how the failure
                     maps to the reported defect>

---

## FAILURE HANDLING

If TEST_EXECUTION_FAILED or TEST_UNAVAILABLE:

WORK_ITEM_ID: <integer>
VALIDATION_TYPE: PRE-FIX
TEST_VALIDATION_STATUS: <TEST_EXECUTION_FAILED|TEST_UNAVAILABLE>
TEST_FAILURE_REASON: <max 40 words — specific reason>
SDK_RESOLUTION: <PASSED|FAILED>
SDK_VERSION: <version if resolved, else Not Available>
DOTNET_PATH: <path if resolved, else Not Available>
TEST_PROJECT: Not Available
TEST_FILE: Not Available
TEST_CLASS: Not Available
TEST_METHOD: Not Available
TEST_SCENARIO: Not Available
TEST_FILTER: Not Available
TEST_COMMAND: Not Available
TESTS_DISCOVERED: 0
TESTS_EXECUTED: 0
TESTS_PASSED: 0
TESTS_FAILED: 0
AUTHORITATIVE_TEST_EXECUTED: NO
TEST_RESULT: NOT_RUN
EXPECTED_BEHAVIOUR: Not Available
ACTUAL_BEHAVIOUR: Not Available
FAILURE_EVIDENCE: <max 80 words of actual error output>
BUG_CORRESPONDENCE: Not Available

---

## COMPLETION

After writing working-repo/.agent/test-validation.md return
exactly one of the following lines and nothing else:

TEST_VALIDATION_STATUS=BUG_REPRODUCED
TEST_VALIDATION_STATUS=BUG_NOT_REPRODUCED
TEST_VALIDATION_STATUS=TEST_EXECUTION_FAILED
TEST_VALIDATION_STATUS=TEST_UNAVAILABLE

After returning the status line:
- Take no further action.
- Do not read any other file.
- Do not modify any file.
- Do not call any other tool.
- Do not continue reasoning.
- Return control to the orchestrator immediately.

</skill>
