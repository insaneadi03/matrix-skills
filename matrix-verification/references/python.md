# Python

`pip install matrix-verify` (Python 3.9+, no dependencies). The package name on
PyPI is `matrix-verify`; the import is `matrix_verify`.

Same surface as the TypeScript SDK in snake_case, and the same traces on the
wire — the verifier cannot tell which language sent them.

## LangChain and LangGraph

The callback handler reads the tool calls the agent already makes. Nothing is
wrapped by hand.

```python
import os
from matrix_verify import verify
from matrix_verify.langchain import verify_callback_handler

verify.init(api_key=os.environ["MATRIX_API_KEY"])

handler = verify_callback_handler(
    from_resolver=lambda tool_name: os.environ.get("AGENT_EMAIL"),
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": instruction}]},
    config={"callbacks": [handler]},
)

verify.shutdown()
```

`callbacks` goes in the **run's** `config`, not only on the model, or tool calls
made by the agent loop are not seen. In Python the keyword is `config=`, unlike
the TypeScript second argument.

The handler needs `langchain-core` present, which any LangChain agent already
has. It is an optional extra of the package (`pip install
"matrix-verify[langchain]"`), not a dependency.

### What the handler records

| Source | Becomes |
| --- | --- |
| First human message of the outermost chain | the instruction |
| Each `on_tool_start` / `on_tool_end` | a tool span with the parsed arguments |
| Final output of the outermost chain | the claim |

Inner chains are ignored on purpose: they carry derived state, and recording
them would produce several conflicting claims for one run.

LangChain hands `on_tool_end` a `ToolMessage` rather than the tool's return
value. The SDK unwraps it and parses the body, so a result like
`{"ok": true, "status": 403}` is recorded as the tool actually returned it. Do
not pre-serialise tool results to work around this.

### Options

| Option | Use when |
| --- | --- |
| `from_resolver(tool_name)` | Always, for email. Returns the acting account, or `None`. |
| `tool_span_name(tool_name)` | The tool's real name is not one the verifier recognises: `tool_span_name=lambda n: "email.send" if n == "dispatch_message" else n` |
| `claim_from_output(outputs)` | The final answer is not a string or a `messages` list. |
| `instruction_from_input(inputs)` | The request is under an unusual key. |

Return `None` from the last two rather than guessing — a wrong instruction or
claim writes fiction into the evidence trail.

## Without a framework

Use this when the agent has no callback system to hook. `with_span` is both a
context manager and a decorator; pick whichever fits the call site.

```python
import os
from matrix_verify import verify

verify.init(api_key=os.environ["MATRIX_API_KEY"])

verify.instruction("Email the August invoice to dana@northwind.example")

with verify.with_span(
    "gmail.send_email",
    span_type="tool",
    input={"to": to, "subject": subject, "from": os.environ["AGENT_EMAIL"]},
) as span:
    span.set_output(send_email(to=to, subject=subject))

verify.claim(f"Email sent successfully to {to}")

verify.shutdown()
```

As a decorator, which records the return value for you:

```python
@verify.with_span("gmail.send_email", span_type="tool")
def send_email(to, subject):
    ...
```

**Do not use the decorator for a send.** It records the call's arguments as
`{"args": [...], "kwargs": {...}}`, and the verifier reads the acting identity
from a top-level `from` on the span input — so nested under `kwargs` it is not
found, and positional arguments arrive unlabelled besides. Either shape yields
`inconclusive / account_unverified` on every email claim. Checked against the
verifier: `{"to": ..., "from": ...}` resolves the account, `{"args": [],
"kwargs": {"from": ...}}` resolves nothing.

Use the decorator for tools whose arguments are not what gets verified, and the
context manager with an explicit `input=` for anything that sends.

### Rules

- The body of a `with_span` block runs whether or not tracing is configured. It
  is safe to wrap the real call.
- Record `from` as a top-level key of the span input, beside `to` and
  `subject`. That is the acting identity; nested anywhere else, or absent, an
  email claim is `inconclusive / account_unverified`.
- Record the arguments the tool actually received, after any defaults are
  applied — not the arguments the model proposed.
- `verify.claim` takes the agent's own words. If the agent produces a final
  message, pass that message. Do not compose a claim the agent never made.
- If the wrapped code raises, the span records the error and the exception
  propagates. Let it; a failed tool call is evidence.

## If the agent is a long-running server

`init` once at startup; do not call `shutdown()` per request. Flush on exit:

```python
import atexit

atexit.register(verify.shutdown)
```

`atexit` does not run on `SIGKILL` or `os._exit`. For a container that is
stopped with `SIGTERM`, register a signal handler that calls
`verify.shutdown()` before exiting.

The export thread is a daemon, so a process that exits without `shutdown()`
loses whatever is still queued — silently, like every other misconfiguration in
this SDK.

## Script shape

For a script, put the flush in the path that always runs:

```python
try:
    result = agent.invoke(...)
finally:
    verify.shutdown()
```

## Version notes

`matrix-verify` 0.1.0 is the first release and has none of the TypeScript
SDK's early traps: `endpoint` already defaults to
`https://matrixverify.dev/api/traces`, and the instruction and tool results are
recorded correctly. Leave `endpoint` unset unless self-hosting.
