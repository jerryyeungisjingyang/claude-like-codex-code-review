# Fix mode

Use this mode only when `--fix` is present or the user otherwise explicitly requests fixes.

## Scope

Apply only retained findings from the completed review. Preserve unrelated user changes and do not reformat, simplify, or refactor code beyond what each fix requires. Do not create commits, push branches, open PRs, install dependencies, or update generated artifacts unless the user separately requests it or the repository's normal focused verification requires regeneration.

Before each edit, confirm that the recorded content and line still match the reviewed version. If the file changed, revalidate the finding against the current content before editing.

## Execution

Fix findings in dependency order, one root cause at a time. Use the smallest complete change that removes the trigger without weakening tests, validation, authorization, error handling, or documented behavior.

Run focused existing tests, type checks, or linters that directly cover the changed code when available and reasonably bounded. Do not perform dependency installation or broad environment setup without separate authorization. If verification cannot run, report the exact reason and retain the static evidence.

After edits, inspect the final diff, confirm that unrelated changes were preserved, and map each retained finding to fixed, partially fixed, or not fixed. Do not describe a partial mitigation as a completed fix.
