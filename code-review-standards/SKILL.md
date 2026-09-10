<skill name="code-review-standards">

## PURPOSE
Review the implemented fix against defined quality and safety
criteria. Produce a structured, size-capped code-review.md
artifact with a clear approval or rejection decision.

## PRE-REVIEW REQUIREMENTS
- Read working-repo/.agent/bug-analysis.md for fix scope.
- Read working-repo/.agent/fix-summary.md for FILES_MODIFIED.
- Read working-repo/.agent/test-validation.md for
  authoritative test scenario.
- Review only files listed in fix-summary.md FILES_MODIFIED.

## REVIEW CHECKLIST
Evaluate each criterion. Record PASS, FAIL, or N/A.

1. SCOPE_COMPLIANCE
   Fix touches only files in AFFECTED_FILES. No unrelated changes.

2. BUG_CORRESPONDENCE
   The fix directly addresses the root cause in bug-analysis.md.
   Not a workaround. Not a symptom fix.

3. NO_REGRESSIONS_INTRODUCED
   No existing behaviour broken. No new null references, exceptions,
   or edge cases introduced.

4. BUILD_VERIFIED
   fix-summary.md BUILD_STATUS=PASSED with actual build evidence.

5. TEST_COVERAGE
   Authoritative test from test-validation.md covers the fix.
   Test asserts correct behaviour, not defective behaviour.

6. CODE_QUALITY
   No hardcoded values without justification.
   No commented-out dead code left behind.
   Naming and structure consistent with surrounding code.

7. SECURITY
   No secrets, tokens, or credentials introduced.
   No SQL injection or input validation regressions.

## APPROVAL RULES
REVIEW_STATUS=APPROVED only when ALL of the following are true:
- SCOPE_COMPLIANCE=PASS
- BUG_CORRESPONDENCE=PASS
- BUILD_VERIFIED=PASS
- TEST_COVERAGE=PASS
- SECURITY=PASS

REVIEW_STATUS=REJECTED if any mandatory criterion is FAIL.
NO_REGRESSIONS_INTRODUCED and CODE_QUALITY may be FAIL with
a recorded reason without blocking approval, at reviewer
discretion.

## OUTPUT FORMAT
Write working-repo/.agent/code-review.md with EXACTLY these
fields.

WORK_ITEM_ID: <integer>
REVIEW_STATUS: <APPROVED|REJECTED>
SCOPE_COMPLIANCE: <PASS|FAIL|N/A>
BUG_CORRESPONDENCE: <PASS|FAIL|N/A>
NO_REGRESSIONS_INTRODUCED: <PASS|FAIL|N/A>
BUILD_VERIFIED: <PASS|FAIL|N/A>
TEST_COVERAGE: <PASS|FAIL|N/A>
CODE_QUALITY: <PASS|FAIL|N/A>
SECURITY: <PASS|FAIL|N/A>
REJECTION_REASONS: <max 60 words, blank if APPROVED>
REVIEW_NOTES: <max 60 words of overall observations>

## STATUS RETURN
After writing the file return exactly one of:
REVIEW_STATUS=APPROVED
REVIEW_STATUS=REJECTED

</skill>
