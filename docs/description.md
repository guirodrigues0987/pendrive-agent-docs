# Architecture Description - PendriveAgent

## Scope

PendriveAgent is a security triage agent that runs entirely on a local machine,
from a USB drive, with no dependency on internet connectivity or any cloud
service. Its purpose is to collect a picture of a computer's current state -
running processes, active network connections and items configured to start
automatically with the system - and, with the help of a local language model,
produce a summary aimed at a non-technical person, pointing out what looks out
of the ordinary. The system explicitly declares itself a reading and triage
layer, not an antivirus: it does not recognize known malware signatures and has
no reputation database; it only compares the current state with the state of
previous runs and with a list of items considered expected in that environment.
The scope of this documentation covers the agent's architecture as implemented -
the Python orchestrator, the collection and history modules, the whitelist
configuration file and the integration with the local inference server -
without going into line-by-line source code, since the goal here is the
container view and the boundaries between containers.

## View level adopted (C4 - Level 2, Containers)

This description and the diagrams derived from it adopt level 2 of the C4
model, that is, the **container** level: units of execution or storage that make
up the system and how they communicate with each other and with the outside
world. The internal structure of classes or functions of each module is not
detailed here (that would be level 3, components), nor are systems external to
PendriveAgent described beyond what is strictly needed to understand its
boundaries - the host operating system itself and the local inference server.
The containers identified in the system are: the Python agent process (which
concentrates orchestration, collection and prompt building), the whitelist
configuration file, the scan history directory on disk, and the local language
model server, treated as an external container with which the agent exchanges
HTTP requests.

## Module boundaries and responsibilities

The system is split into three Python modules with well-defined
responsibilities, plus a configuration file that works as external data read at
runtime.

**`agent.py`** is the entry point and the orchestrator. It reads the
command-line arguments (LLM server address and model name), triggers the three
system data collections, asks the history module for the comparison with the
previous scan, **saves the newly collected snapshot**, builds the text prompt
that will be sent to the language model, only then makes the HTTP call to the
inference server and prints the returned summary. The order matters: the
snapshot is written to disk *before* any attempt to contact the language model,
so even if the LLM call fails or the server is down, the scan history already
reflects the collection made in that run. Its responsibility ends at
orchestration and communication with the LLM - it does not know *how* system
data is collected nor *how* snapshots are persisted, it only consumes the
interfaces of `tools.py` and `history.py`. The payload sent to the inference
server carries two messages - a fixed system prompt with the rules for how the
model should behave, and a user prompt with the collected data - inside a single
JSON body, with temperature fixed at 0.2.

**`tools.py`** is the module responsible for all collection of operating system
data, and is the only place in the code that has to deal with the differences
between Windows and Linux. It exposes three collection functions - running
processes, active network connections and automatic startup items - each
returning a list of dictionaries in a simple and consistent format, designed to
be easy to interpret both by code and by the language model. This module is also
responsible for loading the whitelist file and flagging, on each collected item,
whether it is recognized as expected in that environment - but this loading
happens **only once**, at the moment the module is imported (that is, before the
first collection even starts), and is retained in memory for the rest of the
run: each "is it whitelisted?" check made during a collection is just a lookup
in an in-memory set, not a new read of the file. The module also keeps, for
compatibility with a function-calling style, a registry (`TOOL_REGISTRY`) and
the schemas (`TOOL_SCHEMAS`) of these three functions - a remnant of an
architecture in which the model itself would decide which tools to invoke, no
longer exercised by the agent's main flow (see
`docs/decisions-and-adjustments.md`).

**`history.py`** is responsible exclusively for persisting and comparing scan
snapshots. It writes each run as a timestamped JSON file inside a scans
directory, loads the most recent previous snapshot when one exists, and computes
the difference between the current state and that previous snapshot - which
processes, remote connections and startup items are new relative to the last
run. This module does not collect data on its own nor decide what to do with the
comparison result; it only hands the caller (`agent.py`) a structure with what
changed, leaving the interpretation and the writing of the summary entirely to
the language model.

**`whitelist.json`** is not code, but works as a configuration container
external to the agent process. It is a JSON file with two lists of names
(lowercase) - one of process names and one of terms associated with startup
items - considered normal or expected in that specific environment. The format
is intentionally simple: name strings, with no additional metadata, freely
editable by whoever uses the agent to adapt it to their own environment. This
documentation describes only that format; no real content of the file is
reproduced here, as it is machine-specific data.

## Integrations

**llama.cpp / llama-server (or Ollama).** The agent talks to a local inference
server through an HTTP API compatible with the OpenAI chat completions format,
on the `/v1/chat/completions` endpoint. This protocol compatibility is what lets
the same `agent.py` code work with either llama.cpp's own `llama-server` (the
binary is shipped with the USB drive) or a local Ollama instance - just by
changing the host and model name through a command-line parameter. The agent
does not manage the lifecycle of that server: it assumes the server is already
up before being run, and handles connection failures or HTTP errors by
terminating the run with a diagnostic message.

**Host operating system.** Collection of processes and network connections is
done via `psutil`, in a cross-platform way. Collection of automatic startup
items, on the other hand, is operating-system specific: on Windows, the agent
reads the `Run` registry keys under `HKEY_CURRENT_USER` and `HKEY_LOCAL_MACHINE`
via `winreg`; on Linux, it queries enabled systemd units, the current user's
crontab and the application autostart `.desktop` files. In both cases, full
visibility of other users' processes and connections depends on the agent being
run with administrative privileges.

**File system.** Besides the whitelist file, the agent depends on the local file
system for two purposes: reading the language model binary and the inference
server executable (organized in folders inside the USB drive itself) and
writing/reading the scan snapshot history in a dedicated directory. This scans
directory is treated as local data of the machine where the agent runs, not
something to be versioned.

## Constraints

The system was designed under a clear set of constraints. Everything must run
**100% locally**: there are no calls to cloud services, neither for the language
model nor for any other functionality - the only network the agent uses is local
HTTP communication (loopback) with the inference server running on the same
machine. The system is meant to **run from a USB drive**, carrying with it the
Python interpreter (optional, in case the target machine has none), the
inference engine binaries for Windows and Linux, and the model file itself, so
that plugging the USB drive into a computer is enough to run the agent without
prior installation. Because it runs on varied hardware and, in most cases,
without a dedicated GPU, the system is meant for **small, quantized models**
(on the order of 7B to 13B parameters) - which, as recorded in
`docs/decisions-and-adjustments.md`, has a direct impact on an important
architecture decision of the agent.

## Gaps

Not everything is decided or defined in the source code that was read. The gaps
below are genuinely open points, not inferences or suggestions:

The code has no automated tests - no test file was found alongside the three
modules. There is also no structured logging mechanism; all of the agent's
output is done via `print`, which means there is no persistent log format for
auditing beyond the snapshots saved in `scans/` themselves.

Growth of the scans directory is not managed: each run writes a new JSON file
and there is no purge or rotation routine for old snapshots. Likewise, the
format of those JSON files has no version number or declared schema - a future
change in the structure of the collected data would silently break the
comparison with snapshots saved by an earlier version of the agent.

Coverage of automatic startup items is asymmetric across the supported
operating systems: on Windows, only the user and machine `Run` registry keys are
checked (there is no check of Task Scheduler, `RunOnce` or Windows services); on
Linux, coverage includes systemd, crontab and application autostart, but there
is no macOS support anywhere in the code.

The whitelist logic uses two different comparison criteria, without the code
explaining the difference as intentional: for processes, the check is an exact
match of the (lowercase) name; for startup items, the check is a substring
match. Whether this asymmetry is deliberate or an unreviewed implementation
detail is not stated in any available comment or commit.

Error handling of the call to the inference server is partial, and the boundary
between what is handled and what is not is only visible by reading the code
carefully (see `docs/diagrams/sequence-failure.mmd`). Connection failure (server
down) and HTTP error responses are explicitly caught and result in a friendly
error message followed by a controlled termination of the process
(`sys.exit(1)`) - but even then, with no retry, backoff or degraded mode. An
HTTP 200 response with an empty body, invalid JSON, or valid JSON without the
expected structure (without the `choices`/`message`/`content` keys) **has no
handling at all**: the code lets the corresponding exception (JSON decoding
error, or missing key/index error) propagate uncaught, ending the process with a
raw traceback instead of an understandable message. There is also, in the code,
no form of authentication or verification that the server answering at `--host`
is indeed the expected inference server - the integration implicitly assumes the
local environment is trusted.

Finally, parameters such as the limit of displayed processes (30), the maximum
size of each prompt section in characters (6000) and the model call timeout
(1800 seconds) are fixed in the code as constants, with no command-line
exposure - it is not possible to know, from reading the code alone, whether this
rigidity is a deliberate simplicity decision or a point not yet developed.
