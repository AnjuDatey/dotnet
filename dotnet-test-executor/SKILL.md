<skill name="dotnet-test-executor">

## PURPOSE
Execute a targeted dotnet test and produce structured,
size-capped execution evidence. Never substitute static
analysis, source-code reading, or build output for actual
test execution.

## PRE-EXECUTION REQUIREMENTS
- SDK must be resolved using the dotnet-sdk-resolver skill
  before any test command is run.
- Test project and test method must be identified from
  working-repo/.agent/bug-analysis.md before execution.

## TEST SELECTION RULES
1. Prefer an existing test that directly asserts the reported
   bug scenario.
2. Create a focused test only when no suitable existing test
   exists.
3. The authoritative test MUST assert the CORRECT/INTENDED
   behaviour, not the defective behaviour.
4. The test should FAIL against the pre-fix implementation
   because the defect is present.
5. The test should PASS against the post-fix implementation
   because the defect is resolved.

## TEST EXECUTION COMMAND FORMAT
"$DOTNET_BIN" test <TestProjectPath> \
  --filter "FullyQualifiedName~<TestMethod>" \
  --no-build \
  --verbosity normal

Run dotnet build before dotnet test if --no-build fails.

## VALID EXECUTION EVIDENCE
Only the following constitutes valid test execution evidence:
- Actual dotnet test stdout showing test runner output.
- Lines containing: Passed, Failed, Skipped counts.
- Exit code of the dotnet test process.

The following are NOT valid evidence:
- Source-code analysis.
- Static reasoning about what the test would return.
- Build output alone.
- Test discovery output alone.
- Previous run logs.
- Agent claims without stdout evidence.

## OUTPUT FORMAT
Write working-repo/.agent/test-validation.md with EXACTLY
these fields. Enforce word limits strictly.

WORK_ITEM_ID: <integer>
TEST_VALIDATION_STATUS: <BUG_REPRODUCED|BUG_NOT_REPRODUCED|
                         TEST_EXECUTION_FAILED|TEST_UNAVAILABLE>
TEST_PROJECT: <relative path to .csproj>
TEST_FILE: <relative path to .cs file>
TEST_CLASS: <class name>
TEST_METHOD: <method name>
TEST_FILTER: <filter string used>
TEST_COMMAND: <full command executed>
SDK_VERSION_USED: <e.g. 10.0.1>
TESTS_DISCOVERED: <integer>
TESTS_EXECUTED: <integer>
TESTS_PASSED: <integer>
TESTS_FAILED: <integer>
AUTHORITATIVE_TEST_EXECUTED: <YES|NO>
TEST_RESULT: <PASSED|FAILED|ERROR>
FAILURE_EVIDENCE: <max 60 words of actual stdout failure output>
BUG_CORRESPONDENCE: <max 40 words explaining how failure maps
                     to the reported defect>

## STATUS RETURN
After writing the file return exactly one of:
TEST_VALIDATION_STATUS=BUG_REPRODUCED
TEST_VALIDATION_STATUS=BUG_NOT_REPRODUCED
TEST_VALIDATION_STATUS=TEST_EXECUTION_FAILED
TEST_VALIDATION_STATUS=TEST_UNAVAILABLE

</skill>
