name: dotnet-coding-standards
description: Enforce .NET coding standards and stale artifact checks when implementing a production-code bug fix in a cloned repository. Use when a workflow needs to apply a validated fix to .NET application source code and verify the build passes. Do NOT use for root cause analysis, test creation or execution, code review, committing changes, or creating pull requests.

## PURPOSE

Implement a targeted production-code bug fix in a .NET
codebase, verify the build passes, and produce a structured
size-capped fix-summary.md artifact.

This skill handles production implementation and build
validation only. Testing is owned by separate sub-agents.

---

## PROHIBITED SOURCES

The following are never permitted as input, fallback, or
justification at any point:

- Memory or cached results from previous runs
- Previous workflow or agent output
- Git history or branch names
- Stale artifact content from another Work Item
- Independent root cause analysis
- Assumptions not present in bug-analysis.md

---

## STALE ARTIFACT CHECK

Before modifying any file, verify Work Item ID consistency.

Read WORK_ITEM_ID from bug-details.md.
Read WORK_ITEM_ID from bug-analysis.md.

Both must exactly match the Work Item ID provided in the
current task.

If either does not match:
- Do not modify any file.
- Write fix-summary.md with:
  FIX_STATUS: FAILED
  FIX_FAILURE_REASON: STALE_BUG_CONTEXT
  BUILD_STATUS: NOT_EXECUTED
  All other fields: Not Available
- Return exactly:
  FIX_STATUS=FAILED
  BUILD_STATUS=NOT_EXECUTED

---

## INPUT RULES

After the stale artifact check passes read:

From bug-analysis.md:
- WORK_ITEM_ID
- ROOT_CAUSE_SUMMARY
- AFFECTED_FILES
- AFFECTED_CLASSES
- AFFECTED_METHODS
- FIX_APPROACH

From bug-details.md:
- EXPECTED_BEHAVIOUR — defines correct behaviour to restore
- ACCEPTANCE_CRITERIA — defines the success condition

Do not read test-validation.md. Test context is not an
input to the implementation stage.

---

## SDK RESOLUTION

Execute the dotnet-sdk-resolver skill procedure before any
dotnet command. Record the resolved DOTNET_PATH and
SDK_VERSION before proceeding.

If SDK resolution fails:
- Do not modify any file.
- Write fix-summary.md with:
  FIX_STATUS: FAILED
  FIX_FAILURE_REASON: SDK_RESOLUTION_FAILED
  BUILD_STATUS: NOT_EXECUTED
- Return:
  FIX_STATUS=FAILED
  BUILD_STATUS=NOT_EXECUTED

---

## IMPLEMENTATION RULES

1. Implement only the fix defined in bug-analysis.md
   FIX_APPROACH.
2. Modify only files listed in bug-analysis.md
   AFFECTED_FILES.
3. If a file not in AFFECTED_FILES requires modification,
   record the reason in fix-summary.md ADDITIONAL_FILES
   before modifying it.
4. Do not perform independent root cause analysis.
5. Do not expand the scope of the bug.
6. Do not refactor unrelated code.
7. Do not change public API contracts unless the approved
   fix explicitly requires it.
8. Do not upgrade unrelated packages or change framework
   versions.

## SKILL GUIDANCE BOUNDARY

This skill may provide .NET and ASP.NET Core implementation
patterns. Apply skill recommendations only when required by
the approved fix and compatible with the existing
architecture.

Do not use skill recommendations as justification for:
- Sealed records where not required by the fix
- New service abstractions not required by the fix
- New middleware, DI patterns, or OpenAPI conventions
- Serialization setting changes not required by the fix
- Any modernization not directly required by the fix

The existing codebase and approved bug scope take precedence
over optional modernization.

---

## BUILD PROCEDURE

After implementing the fix, build using the resolved SDK.

STEP 1 — Restore dependencies:
"$DOTNET_PATH" restore <SolutionOrProjectPath>

STEP 2 — Build:
"$DOTNET_PATH" build <SolutionOrProjectPath> \
  --no-restore \
  --configuration Release

BUILD_STATUS=PASSED only when:
- Exit code is 0
- stdout contains: Build succeeded

BUILD_STATUS=FAILED for any other exit code or error output.

## BUILD CORRECTION ATTEMPTS

You may attempt up to 3 build corrections for compilation
errors caused by your production-code changes only.

Do not correct pre-existing unrelated build errors.

After 3 failed build attempts:
- Set FIX_STATUS=FAILED
- Set BUILD_STATUS=FAILED
- Set FIX_FAILURE_REASON=BUILD_FAILED_AFTER_3_ATTEMPTS
- Write fix-summary.md immediately
- Return status lines
- Do not attempt further corrections

---

## OUTPUT FORMAT

Write working-repo/.agent/fix-summary.md with exactly these
fields in this exact order.

No headings. No markdown. No prose. No extra fields.
Respect all word limits.

WORK_ITEM_ID: <integer>
FIX_STATUS: <COMPLETED|FAILED>
FIX_FAILURE_REASON: <one line, blank if COMPLETED>
BUILD_STATUS: <PASSED|FAILED|NOT_EXECUTED>
SDK_VERSION: <e.g. 10.0.1>
DOTNET_PATH: <resolved absolute path>
FILES_MODIFIED: <comma-separated relative paths>
ADDITIONAL_FILES: <comma-separated paths not in AFFECTED_FILES,
                   blank if none>
FIX_DESCRIPTION: <max 60 words — what was changed and why>
BUILD_ATTEMPTS: <integer 1-3>
BUILD_OUTPUT_SUMMARY: <max 50 words of actual build stdout>

---

## FAILURE HANDLING

If FIX_STATUS=FAILED for any reason:

WORK_ITEM_ID: <integer>
FIX_STATUS: FAILED
FIX_FAILURE_REASON: <specific reason>
BUILD_STATUS: <PASSED|FAILED|NOT_EXECUTED>
SDK_VERSION: <version if resolved, else Not Available>
DOTNET_PATH: <path if resolved, else Not Available>
FILES_MODIFIED: <files modified before failure, or None>
ADDITIONAL_FILES: None
FIX_DESCRIPTION: Not Available
BUILD_ATTEMPTS: <integer or 0>
BUILD_OUTPUT_SUMMARY: <actual error output, max 50 words>

---

## COMPLETION

After writing working-repo/.agent/fix-summary.md return
exactly these lines and nothing else:

FIX_STATUS=COMPLETED
BUILD_STATUS=PASSED

or on failure:

FIX_STATUS=FAILED
FIX_FAILURE_REASON=<specific reason>
BUILD_STATUS=<FAILED|NOT_EXECUTED>

After returning the status lines:
- Take no further action.
- Do not execute tests.
- Do not read any other file.
- Do not commit, push, or create a branch.
- Do not call any other tool.
- Do not continue reasoning.
- Return control to the orchestrator immediately.
