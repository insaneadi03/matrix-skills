---
name: matrix-verification
description: >-
  Add Matrix claim verification to an agent, in TypeScript with
  @matrixverify/verify or in Python with matrix-verify. Use when adding Matrix,
  verifying what an agent claims it did, wiring verifyCallbackHandler or
  verify_callback_handler into LangChain or LangGraph, instrumenting tool calls
  with withSpan or with_span, or debugging traces that never arrive, claims
  that are never extracted, or verdicts that come back account_unverified,
  no_mailbox_connected or setup_unrecognised_tool_name.
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

1. Confirm `MATRIX_API_KEY` exists. If it does not, stop and ask — see
   [Before writing any code](#before-writing-any-code-the-key).
2. Detect the language first — see [Detect the language](#detect-the-language)
   — then the framework, agent entry point, tool call sites, and whether any
   Matrix wiring already exists.
3. Choose the integration path from the table below.
4. Make the edits.
5. Verify with `MATRIX_DEBUG=1` and a real run. An integration is not done
   until a trace has been accepted by the server.

Both languages are supported: TypeScript and JavaScript on Node.js 18+ with
`@matrixverify/verify`, and Python 3.9+ with `matrix-verify`. The two produce
identical traces; the verifier cannot tell which sent them. Everything below
that is not in a code block applies to both.

## Before writing any code: the key

Check for `MATRIX_API_KEY` first — in the environment, in `.env`, in
`.env.local`, in the deployment's variables. It is the one thing that cannot be
derived from the codebase.

**If it is not there, stop and ask the user for it.** Say exactly this much:

> Matrix needs an API key. Sign up at https://matrixverify.dev — the key is
> shown once, at signup — then add it to `.env`:
>
> ```
> MATRIX_API_KEY=sk_...
> ```
>
> Tell me when it is there and I will continue.

Do not continue past this point without it. Specifically, never:

- invent a placeholder such as `sk_your_key_here`, `sk_xxx`, or `<your-key>` —
  it looks configured, `init` silently does nothing, and the user discovers
  weeks later that nothing was ever verified;
- hardcode a key you found anywhere, or copy one out of another project;
- write the key into source, a committed file, or a `NEXT_PUBLIC_*` variable;
- wire the integration anyway and "leave the key for later" — the run that
  proves it works cannot happen, so the work cannot be checked.

This is not caution for its own sake. The SDK is fail-silent by design: with no
key, `init` is a no-op, the traced call still runs, `claim` does nothing, and
the agent behaves exactly as before. Nothing errors. An integration finished
without a key is indistinguishable from one that works until someone looks at
an empty findings page.

In Python, `.env` is not read by the interpreter. If the project has no
`python-dotenv` (or equivalent) loading it before `init` runs, a key sitting in
`.env` reaches the process as nothing at all — the same silent failure as no
key. Either load it explicitly or export the variable in the environment that
runs the agent, and confirm which one the project already does rather than
adding a second mechanism.

The same applies to the acting identity for email claims, in both languages. If
you cannot tell which account the agent sends as — from the mail tool's own
credentials, an existing `from` address, or a variable already in the project —
ask the user rather than guessing. A guessed address produces
`inconclusive / account_mismatch` on every claim; no address at all produces
`account_unverified`. Both are silent: the agent runs, the trace arrives, and
only the verdict says anything is wrong.

## The contract

Every integration must produce, for one agent run:

- **The instruction** — what the user asked, verbatim.
- **A tool span per tool call**, carrying the arguments the tool was called
  with, named so the verifier recognises the action.
- **The claim** — the agent's own final statement about what it did.

Miss the instruction and the finding cannot show intent next to action. Miss
the tool span and a real send looks like a claim with nothing behind it. Miss
the claim and there is nothing to verify.

## Detect the language

Decide this from the agent's own entry point, not from whatever the repository
has most of.

| Signal | Language |
| --- | --- |
| `pyproject.toml`, `requirements.txt`, `Pipfile`, `setup.py`, a `.venv/` | Python |
| `package.json`, `tsconfig.json`, `.ts` / `.mjs` entry point | TypeScript / JavaScript |
| `langchain`, `langchain-core`, `langgraph` in `requirements.txt` or `pyproject.toml` | Python LangChain |
| `langchain`, `@langchain/*` in `package.json` | TypeScript LangChain |

A repository containing both is common — a Python agent behind a TypeScript
web app, or the reverse. What decides it is the process that runs the agent
and makes the tool calls, because that is the process the SDK must live in.
Instrumenting the wrong one produces no traces at all and nothing says why.

**If both are plausible and nothing settles it, ask one question:** which
process runs the agent. Do not instrument both, and do not guess — a wrong
guess costs the user a full debugging cycle against a silent SDK.

Then follow that language's path and only that one. The two SDKs have
different names for the same things (`withSpan` and `with_span`,
`fromResolver` and `from_resolver`); mixing them produces code that does not
run.

## Choose the integration path

| Situation | Path |
| --- | --- |
| TypeScript, LangChain or LangGraph | `verifyCallbackHandler` passed in `callbacks`. See [references/langchain.md](references/langchain.md). |
| Any other TypeScript agent | `verify.instruction`, `verify.withSpan`, `verify.claim`. See [references/direct-sdk.md](references/direct-sdk.md). |
| Python, LangChain or LangGraph | `verify_callback_handler` passed in `config={"callbacks": [...]}`. See [references/python.md](references/python.md). |
| Any other Python agent | `verify.instruction`, `verify.with_span`, `verify.claim`. See [references/python.md](references/python.md). |
| Vercel AI SDK, CrewAI, n8n | Not supported yet. Say so; do not fake it with the direct SDK unless the user asks for manual instrumentation. |

## The four calls

TypeScript — `npm install @matrixverify/verify`:

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

Python — `pip install matrix-verify`:

```python
import os
from matrix_verify import verify
from matrix_verify.langchain import verify_callback_handler

verify.init(api_key=os.environ["MATRIX_API_KEY"])

handler = verify_callback_handler(
    from_resolver=lambda tool_name: os.environ.get("AGENT_EMAIL"),
)

agent.invoke(input, config={"callbacks": [handler]})

verify.shutdown()
```

Two of those four are the ones people forget, and both fail silently:

- **`fromResolver` / `from_resolver`** supplies the acting identity — which
  mailbox sent the mail. The model's arguments never contain it. Without it
  every email claim comes back `inconclusive / account_unverified`.
- **`shutdown()`** flushes the batch. There is no exit hook in either SDK, so a
  script that exits without it loses every span it recorded. In Python the
  export thread is a daemon, so the process exits cleanly and says nothing.

## Detection checklist

| Signal | How to detect |
| --- | --- |
| Matrix already wired | `@matrixverify/verify`, `matrix_verify`, `verify.init`, `verifyCallbackHandler`, `verify_callback_handler` |
| LangChain (TS) | `langchain`, `@langchain/*`, `createAgent`, `AgentExecutor`, `.invoke(` |
| LangChain (Python) | `langchain`, `langchain-core`, `from langchain.agents import create_agent`, `@tool`, `.invoke(` |
| LangGraph | `@langchain/langgraph`, `langgraph`, `StateGraph`, `create_react_agent`, compiled graph `.invoke(` |
| Agent boundary | request handler, CLI entry, job processor, cron task |
| Tool call sites (TS) | `tool(...)`, `new DynamicStructuredTool`, functions passed as `tools:` |
| Tool call sites (Python) | `@tool`, `StructuredTool.from_function`, functions passed as `tools=` |
| The claim | the agent's final message, the string returned to the user |
| Identity | which account the tool acts as — SMTP user, OAuth account, `from` address |

Ask one focused question only when the agent boundary or the acting identity is
genuinely ambiguous. Prefer reading the tool's own credentials over asking.

## Implementation rules

- Call `verify.init` once, at startup, on the server side.
- Never invent, guess, or hardcode `MATRIX_API_KEY`. No placeholders.
- `MATRIX_API_KEY` is a server secret. Never put it in `NEXT_PUBLIC_*`, client
  bundles, or committed files. Add it to `.env` and confirm `.env` is ignored.
- Leave `endpoint` unset; it defaults to `https://matrixverify.dev/api/traces`.
  Set it only for self-hosting. The exception is `@matrixverify/verify` 0.1.0,
  where the default pointed at localhost and silently sent traces nowhere — so
  pin `^0.1.1` or later. `matrix-verify` has always defaulted to the apex;
  `matrix-verify>=0.1.0` is enough.
- Name tool spans so the verifier recognises the action: `gmail.send_email`,
  `send_email`, and `email.send` all match a send. A span named
  `dispatch_message` is reported as a setup problem, not as the agent lying —
  use `toolSpanName` to map an existing name rather than renaming the tool.
- Put `shutdown()` in the path that always runs at the end of a run: a
  `finally` block for a script, a shutdown hook for a server. In TypeScript it
  is awaited; in Python it is a plain call, and `atexit.register(verify.shutdown)`
  covers a long-running process.
- Never change what the agent claims in order to make verification pass. The
  claim is evidence; editing it defeats the point.
- Do not log or record secrets in span inputs. Record the arguments the tool
  received, not credentials it used.

## Verification

An integration is done when a trace has arrived, not when the code compiles.
If no key is configured, this step cannot run — and the integration is not
done. Stop and ask for the key rather than reporting success.

1. Run the agent once with `MATRIX_DEBUG=1`. Both SDKs honour it and print the
   same lines under the same `[matrix-sdk]` prefix.
2. Expect these lines, in this order:

```text
[matrix-sdk] init endpoint=https://matrixverify.dev/api/traces ...
[matrix-sdk] exporting N span(s) -> https://matrixverify.dev/api/traces: ...
[matrix-sdk] export OK (N span(s) accepted)
```

3. Confirm the span list contains the instruction, the tool call, and the
   claim — not just the tool call.
4. The finding appears at [matrixverify.dev](https://matrixverify.dev) within
   about half an hour. Verification runs on a schedule; nothing to trigger.

If `export OK` never appears, read
[references/troubleshooting.md](references/troubleshooting.md) before changing
anything else.

## Reading the verdict

| Verdict | Meaning |
| --- | --- |
| `confirmed` | The authoritative system has the record. Evidence attached. |
| `contradicted` | The claim is not supported. `no_tool_call` means the agent never attempted it; `no_matching_sent_message` means the tool was called and nothing arrived. |
| `inconclusive` | The check could not settle it — `account_unverified` (no `fromResolver` / `from_resolver`), `no_mailbox_connected` (no mailbox connected for this project), `account_mismatch` (the send came from a mailbox other than the connected one), `adapter_error`, or `setup_unrecognised_tool_name`. Not an accusation. |

Email claims are checked against the Gmail account the project connects under
Settings at [matrixverify.dev](https://matrixverify.dev). Until one is
connected, a claim backed by a real send comes back `inconclusive /
no_mailbox_connected` — the integration is working and there is nothing to
check against yet. Say that plainly rather than letting the user think the
wiring is broken.
