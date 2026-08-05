---
name: oqtopus
description: Submit and manage quantum jobs on OQTOPUS, an open-source quantum computing cloud, through the oqtopus-client Python SDK. Use when the user wants to run a quantum circuit or QASM program on a QPU or simulator, choose a device, check job status, wait for a job, fetch counts or other job results, or review job history.
---

# OQTOPUS jobs

OQTOPUS is driven from Python via the `oqtopus-client` SDK. There is no CLI: write a short
script and run it. Job submission is a multi-step protocol (register, upload, submit) that the
SDK already handles — never reimplement it against the REST API.

Requires `oqtopus-client` (see "Install into the right environment"). Circuits are plain
OpenQASM 3 strings, so nothing else is needed; `qiskit` is optional and only used to generate
QASM.

## Non-negotiable rules

- **Never read the config file** (`~/.config/oqtopus/config.ini` or
  `$XDG_CONFIG_HOME/oqtopus/config.ini`). It stores the API token in plaintext. The SDK reads
  it for you; you only ever need the section name.
- **Never print, log, or repr an `OqtopusConfig`.** It is a plain dataclass, so `api_token`
  appears in full in its default `repr` — including in any uncaught traceback that has the
  config in a local variable. Do not put a config object in an f-string or an error message.
- **Never call** `create_api_token`, `delete_api_token`, or `delete_current_user`. Token and
  account management is the user's job, in the web console.
- **A returned result does not mean the job succeeded.** See "Read the result".

If the user has no config yet, tell them to create it themselves and stop — you cannot type a
secret for them:

```ini
# ~/.config/oqtopus/config.ini   (chmod 600)
[default]
base_url = https://demo-api.oqtopus.io
api_token = <token from the web console, Settings > Security>
```

`OqtopusClient()` raises immediately with a clear message when the section is missing. Show
that message to the user rather than guessing.

## Install into the right environment

Follow the environment the user already has rather than imposing one. Check for these
signals, in order, and use the first that matches:

1. **A project venv** — `ls .venv/bin/python` (file-listing tools often skip hidden
   directories, so check explicitly). Install into it:
   `uv pip install --python .venv/bin/python oqtopus-client`.
2. **A manifest or lockfile that names the manager** — `uv.lock` → `uv add oqtopus-client`;
   `poetry.lock` → `poetry add oqtopus-client`; `requirements.txt` → append it and install
   into the project's venv; an active conda env → install inside that env.
3. **Nothing found and no stated preference** — default to `uv`:
   `uv run --with oqtopus-client python script.py` needs no setup and leaves nothing behind.

Never install into the system Python — `pip` may not even exist there.

Verify with `python -c "import oqtopus_client"`. The package has no `__version__` attribute;
use `importlib.metadata.version("oqtopus-client")` if you need the version.

## Sandboxed execution

Harnesses that sandbox commands by default (Codex; Claude Code in sandboxed modes) restrict
filesystem writes and outbound network. Two symptoms to recognize:

- **Read-only cache**: `uv` fails with `Read-only file system` on `~/.cache/uv`. Point the
  cache at a writable directory: `UV_CACHE_DIR=/tmp/uv-cache uv ...`.
- **Silent network hangs**: with network blocked, the first API call (usually
  `list_devices`) **hangs instead of failing** — no exception, no output. Don't misread
  this as a slow API: the demo endpoint answers in seconds. Print a marker (`flush=True`)
  *before* the first client call; if nothing appears within ~15 s, kill the process and
  rerun with network access approved (escalated/unsandboxed) instead of polling a process
  that will never return. Request that approval for the final `python script.py` invocation
  itself — one approval, one run — after the script is already written and syntax-checked
  offline.

## Client

```python
from oqtopus_client import OqtopusClient, OqtopusConfig

client = OqtopusClient()                                      # reads the [default] section
client = OqtopusClient(OqtopusConfig.from_file("prod"))       # another section
```

Environment variables (`OQTOPUS_BASE_URL`, `OQTOPUS_API_TOKEN`) work via
`OqtopusConfig.from_env()`, but prefer the config file: exporting a token puts it in shell
history and `ps`.

## Choose a device

Always confirm the device before submitting — a wrong `device_id` fails the submit call, and
too many qubits fails the job after it has queued.

```python
for d in client.list_devices():
    print(d.device_id, d.device_type, d.status, d.n_qubits, d.n_pending_jobs)

dev = client.get_device("qulacs")
```

Check `dev.status == "available"` and `dev.n_qubits >= <qubits your circuit uses>`.

`device_type` is `"simulator"` or a real QPU. Which devices exist depends entirely on the
`base_url` the user is pointed at — never hardcode a device id, always list first. Do not dump
`dev.device_info`, `dev.raw`, or calibration data wholesale into the conversation; they can be
large. Read the specific field you need.

## Build the program

The program is an OpenQASM 3 string. Write it directly:

```python
program = """OPENQASM 3.0;
include "stdgates.inc";
qubit[2] q;
bit[2] c;
h q[0];
cx q[0], q[1];
c[0] = measure q[0];
c[1] = measure q[1];
"""
```

Or generate it with Qiskit (`pip install qiskit`) when the circuit is easier to express in
code — parameterised, many qubits, or built in a loop:

```python
from qiskit import QuantumCircuit, qasm3

qc = QuantumCircuit(2, 2)
qc.h(0)
qc.cx(0, 1)
qc.measure([0, 1], [0, 1])
program = qasm3.dumps(qc)
```

Do not transpile locally. The server transpiles against the device's real basis gates and
calibration; a local transpile only produces a worse mapping. Use any gate you like —
`qasm3.dumps` emits `gate ... { ... }` definition blocks for non-`stdgates` gates and OQTOPUS
accepts them.

## Submit

Build a spec with the factory matching the job type, then submit.

```python
from oqtopus_client import OqtopusJobSpec

spec = OqtopusJobSpec.sampling(
    device_id="qulacs",     # keyword-only
    program=program,
    shots=1000,
    name="bell-state",      # optional, but makes job history readable
)
```

Job types: `sampling` (measure and count bitstrings — the default choice), `estimation`
(expectation value of an `operator`), `multi_manual`, `sse` (server-side execution of a Python
file near the QPU). `program` also accepts a sequence of strings for multi-program jobs.

**Non-blocking — prefer this.** Submit, report the job id, and check back on a later turn:

```python
job_id = client.submit_job(spec).job_id
```

**Blocking** — only for fast simulator jobs. It polls in-process and will sit there for up to
`timeout` seconds:

```python
result = client.run_sampling(spec, timeout=120)   # returns a finished job
```

Use `run_sampling` / `run_estimation` rather than the generic `run_job` so the result is the
typed subclass. A bad `device_id` raises `UserApiError` here, synchronously.

## Wait

```python
client.status(job_id)      # 'registered'|'submitted'|'ready'|'running'|'succeeded'|'failed'|'cancelled'
client.is_finished(job_id)
```

For a real QPU, poll with `client.status(job_id)` once per turn instead of blocking; queues
can be far longer than any sensible in-process timeout.

To block anyway:

```python
result = client.wait(job_id, timeout=300, interval=2.0, interval_backoff=1.5)
```

`wait` raises `TimeoutError` when the deadline passes — the job keeps running, so catch it and
poll later rather than treating it as a failure. It returns normally for **any** terminal
status, including `failed` and `cancelled`.

Also available: `client.cancel_job(job_id)`.

## Read the result

```python
result = client.result(job_id)      # also: client.get_job_result(job_id)
```

**Check the status first.** No exception was raised is not evidence of success: an invalid
program is accepted at submit time and only fails later, asynchronously.

```python
from oqtopus_client.rest import models

if result.status != models.JobsJobStatus.SUCCEEDED:
    print(result.status, result.message)   # do not read counts
```

`result.message` is the raw server-side error and is often unhelpful (a QASM syntax error can
surface as a JSON parse error). When the status is `failed`, say so plainly and suggest
checking the QASM, rather than passing the raw message along as the explanation.

On success, for a sampling job:

```python
counts = result.get_counts()          # {'00': 501, '11': 499}, keyed by bitstring
result.execution_time                 # seconds
result.transpile_result.transpiled_program   # server-side transpiled QASM3, for debugging
```

`counts` has up to `2**n_qubits` entries. Beyond a few qubits, show the top entries and the
total shots, and write the full dictionary to a file — never paste a large counts dict into the
conversation.

## Job history

```python
for j in client.list_jobs(size=20):
    print(j.job_id, j.status, j.device_id, j.name, j.submitted_at)
```

Filters: `status`, `start_time` / `end_time` (datetimes), `q` (search), `page` / `size`,
`order` (by creation time), `fields`. Use this to pick up a job submitted in an earlier session — a `job_id` is
all you need to resume.

## When this file is not enough

`OqtopusClient` has ~35 methods (batch submission, async variants, SSE, announcements). Check
the real signature at runtime rather than guessing:

```python
python -c "from oqtopus_client import OqtopusClient; help(OqtopusClient.run_jobs_batch)"
```

Full docs: https://oqtopus-client.readthedocs.io/en/latest/
