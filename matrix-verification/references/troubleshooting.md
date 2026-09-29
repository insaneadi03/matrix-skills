# When it does not work

Run with `MATRIX_DEBUG=1` first. Every diagnosis below is based on what that
prints. Both SDKs print the same `[matrix-sdk]` lines, so unless a heading says
otherwise these apply to TypeScript and Python alike.

## No `[matrix-sdk]` lines at all

`verify.init` never ran, or it ran without a key. The SDK is fail-silent: a
missing key makes `init` a no-op, the traced call still runs, and `claim` does
nothing. Check that `MATRIX_API_KEY` is set in the process that actually runs
the agent.

In a repository with both a Python and a TypeScript side, check *which* process
that is. Instrumenting the one that does not run the agent produces exactly
these symptoms and nothing says why.

If there is no key at all, stop and ask the user to sign up at
https://matrixverify.dev and put it in `.env`. Do not substitute a placeholder
to get past this — a placeholder produces exactly these symptoms, permanently.

## `init` logs, but no `exporting`

Nothing was recorded. With LangChain, `callbacks` was not passed to the run
that makes the tool calls. Without a framework, no `withSpan` or `claim` call
was reached.

## `exporting` but no `export OK`

The spans were built and the POST failed.

| Symptom | Cause |
| --- | --- |
| `code=401` | The key is wrong, or belongs to a different project. |
| `code=308` | The endpoint is a redirecting host. Use the exact apex: `https://matrixverify.dev/api/traces`. The SDK does not follow redirects on POST. |
| Nothing after `exporting` | The process exited before the flush. Call `verify.shutdown()` — awaited in TypeScript, plain in Python, where the export thread is a daemon and the process exits without waiting for it. |

## `export OK`, but no finding appears

Verification runs on a schedule, roughly every half hour. Wait a full cycle
before investigating. If the trace is there and the finding is not, the claim
was not extracted: the agent's final message did not state what it did. "All
done!" gives the verifier nothing; "Emailed the invoice to dana@example.com"
does.

## `ModuleNotFoundError: No module named 'matrix_verify'` (Python)

The package name and the import differ: install `matrix-verify`, import
`matrix_verify`. If both look right, the agent is running in a different
interpreter or virtualenv from the one that was installed into.

`No module named 'langchain_core'` from `matrix_verify.langchain` means the
handler was imported into a project without LangChain. Use the framework-free
path from [python.md](python.md) instead.

## Verdict is `inconclusive / account_unverified`

No `fromResolver` / `from_resolver`, or it returned `None`. Matrix could not
tell which mailbox to search, so absence of a message proves nothing.

## Verdict is `inconclusive / no_mailbox_connected`

The project has no Gmail account connected. Connect one under Settings at
https://matrixverify.dev. The integration is working — there is nothing to
check the claim against yet.

## Verdict is `inconclusive / account_mismatch`

The agent sent from an account other than the one connected for the project.
Either the resolver reports the wrong address, or the mail tool sends as
someone else. Check the tool's own credentials before changing the resolver.

## Verdict is `inconclusive / setup_unrecognised_tool_name`

The span name is not one the verifier recognises as a send. Map it with
`toolSpanName`, or name the span `email.send`.

## Verdict is `contradicted / no_tool_call`

The claim says an action happened and no matching tool span is in the trace.
Either the agent really did claim without acting — the case Matrix exists to
catch — or the tool call happened outside the traced path. Check the span list
in the debug output before assuming the agent lied.
