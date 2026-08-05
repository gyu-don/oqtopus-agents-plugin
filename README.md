# OQTOPUS Agent Skills

An [Agent Skill](https://code.claude.com/docs/en/skills) that teaches coding agents to submit
and manage quantum jobs on [OQTOPUS](https://github.com/oqtopus-team) through the
[oqtopus-client](https://github.com/oqtopus-team/oqtopus-client) Python SDK.

With this installed, you can ask your agent things like:

- "Run a Bell state on the qulacs simulator with 1000 shots"
- "Which devices are available, and how many qubits do they have?"
- "Is job `<id>` done yet? Show me the counts"
- "What did I run yesterday?"

## Install

```bash
npx skills add oqtopus-team/oqtopus-agents-plugin
```

This works across [many coding agents](https://github.com/vercel-labs/skills) (Claude Code,
Codex, Cursor, Cline, OpenCode, ...). To install by hand instead, copy `skills/oqtopus/` into
your agent's skills directory.

## Prerequisites

```bash
pip install oqtopus-client   # qiskit is optional, for generating QASM3 from code
```

You need an OQTOPUS account and an API token. The demo environment at
<https://demo.oqtopus.io> is open to anyone; issue a token under **Settings > Security**.

Write it to `~/.config/oqtopus/config.ini` yourself — do not paste the token into a chat with
an agent, and do not pass it on a command line:

```bash
mkdir -p -m 700 ~/.config/oqtopus
: > ~/.config/oqtopus/config.ini && chmod 600 ~/.config/oqtopus/config.ini
read -r -s -p "OQTOPUS API token: " TOKEN; echo
printf '[default]\nbase_url = %s\napi_token = %s\n' "https://demo-api.oqtopus.io" "$TOKEN" \
  > ~/.config/oqtopus/config.ini
unset TOKEN
```

The skill instructs agents never to read this file and never to print a config object, since
the token appears in plaintext in both.

## Scope

Covered: choosing a device, submitting OpenQASM 3 programs, waiting, reading results, and
browsing job history.

Not covered yet: server-side execution (SSE), batch/parallel submission, estimation operators
in detail, and circuit design guidance. Token and account management is deliberately excluded.

## Development

Design notes and the decision log live in [`docs/`](docs/) and
[`oqtopus-skills-design.md`](oqtopus-skills-design.md).
