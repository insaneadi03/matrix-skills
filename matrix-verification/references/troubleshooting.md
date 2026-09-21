# When it does not work

Run with `MATRIX_DEBUG=1` first. Every diagnosis below is based on what that
prints.

## No `[matrix-sdk]` lines at all

`verify.init` never ran, or it ran without an `apiKey`. The SDK is fail-silent:
a missing key makes `init` a no-op, `withSpan` still runs the function, and
`claim` does nothing. Check that `MATRIX_API_KEY` is set in the process that
actually runs the agent.

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
| Nothing after `exporting` | The process exited before the flush. Await `verify.shutdown()`. |

## `export OK`, but no finding appears

Verification runs on a schedule, roughly every ten minutes. Wait a full cycle
before investigating. If the trace is there and the finding is not, the claim
was not extracted: the agent's final message did not state what it did. "All
done!" gives the verifier nothing; "Emailed the invoice to dana@example.com"
does.

## Verdict is `inconclusive / account_unverified`

No `fromResolver`, or it returned null. Matrix could not tell which mailbox to
search, so absence of a message proves nothing.

## Verdict is `inconclusive / account_mismatch`

The agent sent from an account Matrix does not query. During early access the
mailbox is connected by hand — this is expected until yours is connected, and
is not a bug in the integration.

## Verdict is `inconclusive / setup_unrecognised_tool_name`

The span name is not one the verifier recognises as a send. Map it with
`toolSpanName`, or name the span `email.send`.

## Verdict is `contradicted / no_tool_call`

The claim says an action happened and no matching tool span is in the trace.
Either the agent really did claim without acting — the case Matrix exists to
catch — or the tool call happened outside the traced path. Check the span list
in the debug output before assuming the agent lied.
