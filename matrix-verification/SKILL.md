---
name: matrix-verification
description: >-
  Add Matrix claim verification to an agent with @matrixverify/verify. Use when
  adding Matrix, verifying what an agent claims it did, wiring
  verifyCallbackHandler into LangChain or LangGraph, instrumenting tool calls
  with withSpan, or debugging traces that never arrive, claims that are never
  extracted, or verdicts that come back account_unverified or
  setup_unrecognised_tool_name.
---

# Matrix verification

Matrix answers one question: the agent said it did something — did it?

It records what the agent was asked, which tools it called with which
arguments, and what it claimed afterwards, then checks the claim against the
real system and returns `confirmed`, `contradicted`, or `inconclusive` with the
evidence attached.

## Operating mode

Wire the SDK into the existing agent. Do not restructure the agent, do not add
a framework, and do not wrap tools by hand when the framework has callbacks.

Work in this order:

1. Detect the language, framework, agent entry point, tool call sites, and
   whether any Matrix wiring already exists.
2. Choose the integration path from the table below.
3. Make the edits.
4. Verify with `MATRIX_DEBUG=1` and a real run. An integration is not done
   until a trace has been accepted by the server.

Node.js 18+. TypeScript and JavaScript only today — if the agent is Python,
say so plainly and stop rather than improvising.

## The contract

Every integration must produce, for one agent run:

- **The instruction** — what the user asked, verbatim.
- **A tool span per tool call**, carrying the arguments the tool was called
  with, named so the verifier recognises the action.
- **The claim** — the agent's own final statement about what it did.

Miss the instruction and the finding cannot show intent next to action. Miss
the tool span and a real send looks like a claim with nothing behind it. Miss
the claim and there is nothing to verify.

## Choose the integration path

| Situation | Path |
| --- | --- |
| LangChain or LangGraph | `verifyCallbackHandler` passed in `callbacks`. See [references/langchain.md](references/langchain.md). |
| Any other TypeScript agent | `verify.instruction`, `verify.withSpan`, `verify.claim`. See [references/direct-sdk.md](references/direct-sdk.md). |
| Vercel AI SDK, CrewAI, n8n | Not supported yet. Say so; do not fake it with the direct SDK unless the user asks for manual instrumentation. |
| Agent is Python | Not supported yet. Stop and say so. |

## The four calls

```ts
import { verify } from "@matrixverify/verify";
import { verifyCallbackHandler } from "@matrixverify/verify/langchain";

verify.init({ apiKey: process.env.MATRIX_API_KEY });

const handler = verifyCallbackHandler({
  fromResolver: () => process.env.AGENT_EMAIL ?? null,
});

await agent.invoke(input, { callbacks: [handler] });

await verify.shutdown();
```

Two of those four are the ones people forget, and both fail silently:

- **`fromResolver`** supplies the acting identity — which mailbox sent the
  mail. The model's arguments never contain it. Without it every email claim
  comes back `inconclusive / account_unverified`.
- **`shutdown()`** flushes the batch. There is no exit hook in the SDK, so a
  script that exits without it loses every span it recorded.

## Detection checklist

| Signal | How to detect |
| --- | --- |
| Matrix already wired | `@matrixverify/verify`, `verify.init`, `verifyCallbackHandler` |
| LangChain | `langchain`, `@langchain/*`, `createAgent`, `AgentExecutor`, `.invoke(` |
| LangGraph | `@langchain/langgraph`, `StateGraph`, compiled graph `.invoke(` |
| Agent boundary | request handler, CLI entry, job processor, cron task |
| Tool call sites | `tool(...)`, `new DynamicStructuredTool`, functions passed as `tools:` |
| The claim | the agent's final message, the string returned to the user |
| Identity | which account the tool acts as — SMTP user, OAuth account, `from` address |

Ask one focused question only when the agent boundary or the acting identity is
genuinely ambiguous. Prefer reading the tool's own credentials over asking.

## Implementation rules

- Call `verify.init` once, at startup, on the server side.
- `MATRIX_API_KEY` is a server secret. Never put it in `NEXT_PUBLIC_*`, client
  bundles, or committed files. Add it to `.env` and confirm `.env` is ignored.
- Leave `endpoint` unset. It defaults to `https://matrixverify.dev/api/traces`
  on 0.1.1 and later. Set it only for self-hosting — and on 0.1.0, where the
  default pointed at localhost and silently sent traces nowhere.
- Pin `@matrixverify/verify` to `^0.1.1` or later for the same reason.
- Name tool spans so the verifier recognises the action: `gmail.send_email`,
  `send_email`, and `email.send` all match a send. A span named
  `dispatch_message` is reported as a setup problem, not as the agent lying —
  use `toolSpanName` to map an existing name rather than renaming the tool.
- Put `await verify.shutdown()` in the path that always runs at the end of a
  run: a `finally` block for a script, a shutdown hook for a server.
- Never change what the agent claims in order to make verification pass. The
  claim is evidence; editing it defeats the point.
- Do not log or record secrets in span inputs. Record the arguments the tool
  received, not credentials it used.

## Verification

An integration is done when a trace has arrived, not when the code compiles.

1. Run the agent once with `MATRIX_DEBUG=1`.
2. Expect these lines, in this order:

```text
[matrix-sdk] init endpoint=https://matrixverify.dev/api/traces ...
[matrix-sdk] exporting N span(s) -> https://matrixverify.dev/api/traces: ...
[matrix-sdk] export OK (N span(s) accepted)
```

3. Confirm the span list contains the instruction, the tool call, and the
   claim — not just the tool call.
4. The finding appears at [matrixverify.dev](https://matrixverify.dev) within
   about ten minutes. Verification runs on a schedule; nothing to trigger.

If `export OK` never appears, read
[references/troubleshooting.md](references/troubleshooting.md) before changing
anything else.

## Reading the verdict

| Verdict | Meaning |
| --- | --- |
| `confirmed` | The authoritative system has the record. Evidence attached. |
| `contradicted` | The claim is not supported. `no_tool_call` means the agent never attempted it; `no_matching_sent_message` means the tool was called and nothing arrived. |
| `inconclusive` | The check could not settle it — `account_unverified` (no `fromResolver`), `account_mismatch` (a mailbox Matrix does not query), `adapter_error`, or `setup_unrecognised_tool_name`. Not an accusation. |

During early access, email claims are checked against a mailbox connected by
hand. Until the user's own mailbox is connected, a claim backed by a real send
comes back `inconclusive`, not `confirmed`. Say this plainly rather than
letting the user think the integration is broken.
