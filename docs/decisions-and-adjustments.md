# Architecture Decisions and Adjustments - PendriveAgent

This document records design decisions identifiable directly in the PendriveAgent
source code, along with the declared reasoning for each. The goal is not to list
every implementation choice, but the decisions that shape the architecture and
are worth carrying into the diagrams and to whoever will maintain or extend the
system in the future.

## Decision 1 - Deterministic collection in Python; the language model only interprets and summarizes

**Context.** The first conceptual version of the agent would follow a common
pattern of LLM-based agents: the model would get access to a set of tools (the
same three collection functions now exposed in `tools.py` as `TOOL_SCHEMAS`) and
would decide, at runtime, which to call and in what order, through the native
tool-calling/function-calling of the chat completions API.

**Decision.** That approach was abandoned. In the flow actually implemented in
`agent.py`, Python directly and always runs the same fixed set of three
collections - processes, network connections and startup items - without asking
the model what to collect. The language model comes into play only afterwards:
it receives the already collected data (and the diff relative to the previous
scan, when one exists) formatted as text in a single prompt, and its only
responsibility is to interpret that data and write the final summary. The
`TOOL_SCHEMAS`/`TOOL_REGISTRY` mechanism is still present in `tools.py`, but is
not exercised by the agent's main execution path - it is a remnant of the
previous approach.

**Reason recorded in the code.** `agent.py` itself documents the reason directly
in its module docstring: small models (7B-8B), running via llama.cpp, show
parsing bugs in tool-calling when they try to call several tools in the same
response - described in the code as a known native function-call formatting bug
in `llama-server`. Since the scan's tool set is always the same and always
executed in full, there was no real need to delegate to the model the decision
of "which tools to call" - a decision it would, in practice, carry out
unreliably.

**Consequences.** Predictability is gained: data collection never fails because
of a model formatting error, since the model never takes part in it. In return,
the agent loses flexibility - it is not possible, for example, to ask the agent
in natural language to run only one of the three collections, or to add a new
collection without changing `agent.py` directly. This is an architecture
decision consistent with the constraint of running with small, local models (see
`docs/description.md`, Constraints section), but it ties the system's
extensibility to the Python code and no longer to the model's capability.

---

No other architecture decision is explicitly documented in the source code (for
example, in comments or docstrings) besides the one above. Other implementation
choices exist - such as the whitelist criteria or the fixed limit and timeout
values - but, since no justification is recorded for them in the code, they are
listed as gaps in `docs/description.md`, and not as documented decisions here.

## Adjustments made to the diagrams (v1 -> final version)

The initial diagrams (`structural-v1.mmd` and `sequence-v1.mmd`) were produced
simulating an architect without access to the source code, only to the prose of
`docs/description.md`. After reading the real code and recording the
divergences in `docs/v1-comparison.md`, the final diagrams (`structural.mmd`,
`sequence.mmd`) fix what proved wrong, and a new third diagram
(`sequence-failure.mmd`) covers a scenario that v1 never tried to model. The v1
files were kept in the folder to allow the before/after comparison. Below, each
adjustment, what motivated the change and why.

**Snapshot save order.** The v1 sequence diagram placed
`Agent->>History: save collected snapshot` as the last step of the whole
journey, after the LLM call and after printing the summary - a reasonable
assumption for someone reading the task as "the journey ends with the report
saved in the history", but one that did not match the code. In `agent.py`, the
snapshot is written to disk right after computing the diff and *before* building
the prompt or calling the inference server. This was fixed in `sequence.mmd`:
the snapshot-saving block now appears before the prompt-building step and the
LLM call, and an explicit note records that the snapshot is already on disk at
that point, regardless of what happens afterwards. The structural diagram also
gained numbering on the main arrows (1 to 7) to make this order visible even in
a container diagram, which by nature is not sequential.

**Moment of the whitelist lookup.** v1 showed `tools.py` "talking" to
`whitelist.json` as a runtime message exchange, repeated on every collection
(once for processes, once for startup items) - a plausible model for someone who
does not know the loading is done by a module-level variable. The code shows
that the file read happens only once, at the moment `tools.py` is imported, and
that the "is it known?" checks made afterwards are just lookups in a set already
loaded into memory, with no new message exchanged with the file. In
`sequence.mmd`, this was fixed: there is now a single interaction with
`whitelist.json`, right after the module import and before any collection,
followed by a note explaining that the names stay in memory; the two checks that
previously appeared as messages to `Whitelist` became self-messages from `Tools`
to itself, representing the in-memory membership test.

**New diagram: inference server failure scenario.** v1 had no equivalent,
because the original instruction asked only for the main flow.
`sequence-failure.mmd` is entirely new and covers the two requested scenarios,
modeled exactly as the real code handles (or does not handle) them: when the
server is down or responds with an HTTP error, the code catches the exception,
prints an error message and terminates the process in a controlled way
(`sys.exit(1)`) - this is represented in an `alt` branch. When the server
responds with HTTP 200 but with an empty body, invalid JSON, or a structure
without `choices`/`message`/`content`, the code handles none of it: the
corresponding exception propagates uncaught and the process ends with a raw
traceback. This second branch of the diagram records explicitly, in a note, that
it is a gap in the real code, and not an omission of the diagram - it is the
system itself that lacks this handling.

**Structural diagram: whitelist label and order numbering.** Besides the step
numbering already mentioned, the label on the edge between `tools.py` and
`whitelist.json` was rewritten from "reads known names" (which suggested a
recurring read) to "loaded once, at module import" - incorporating directly into
the container diagram a fact that had only been recorded in prose in
`docs/description.md`.

No container or module was removed or added in the final diagrams relative to
v1: the corrections were all about order, about the granularity of interactions
(message exchanged vs. in-memory lookup) and about coverage (adding the failure
scenario) - not about the existence of components.
