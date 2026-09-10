<skill name="dotnet-coding-standards">

## PURPOSE
Implement a targeted bug fix in a .NET codebase, verify the
build passes, and produce a structured fix-summary.md artifact.

## PRE-IMPLEMENTATION REQUIREMENTS
- SDK must be resolved using the dotnet-sdk-resolver skill
  before any dotnet command.
- Read working-repo/.agent/bug-analysis.md AFFECTED_FILES
  field before touching any file.
- Read working-repo/.agent/test-validation.md TEST_METHOD
  field to confirm the authoritative test scenario.

## FIX SCOPE RULES
ALLOWED modifications:
- Production and application source files listed in
  bug-analysis.md AFFECTED_FILES.
- Supporting configuration files directly required by the fix.

NOT ALLOWED:
- Modifying test files (except adding a regression test
  explicitly approved by bug-analysis.md).
- Modifying files not listed in AFFECTED_FILES without
  recording the reason in fix-summary.md.
- Committing, pushing, or creating branches.
- Modifying Git configuration.
- Cloning the repository again.

## BUILD VERIFICATION PROCEDURE
After implementing the fix run:
  "$DOTNET_BIN" build <SolutionOrProjectPath> \
    --configuration Release \
    --no-restore

Build PASSES only when exit code is 0 and stdout contains
  Build succeeded.
Build FAILS for any other exit code or error output.

Do not claim BUILD_STATUS=PASSED based on:
- Source-code review alone.
- Successful test run alone.
- Previous build output.

## OUTPUT FORMAT
Write working-repo/.agent/fix-summary.md with EXACTLY these
fields. Enforce word limits strictly.

WORK_ITEM_ID: <integer>
FIX_STATUS: <COMPLETED|FAILED>
BUILD_STATUS: <PASSED|FAILED|NOT_RUN>
SDK_VERSION_USED: <e.g. 10.0.1>
FILES_MODIFIED: <comma-separated list of relative paths>
FIX_DESCRIPTION: <max 60 words>
BUILD_OUTPUT_SUMMARY: <max 50 words of actual build stdout>
FAILURE_REASON: <max 40 words, blank if COMPLETED>

## STATUS RETURN
After writing the file return exactly one of:
FIX_STATUS=COMPLETED
FIX_STATUS=FAILED

And separately:
BUILD_STATUS=PASSED
BUILD_STATUS=FAILED

</skill>
