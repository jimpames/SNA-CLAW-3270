# SNA-CLAW-3270
## Operations and Design Guide

---

**Document version:** 1.0
**Tool version:** at time of writing, the system described here corresponds
to the codebase as of late May 2026.
**Audience:** technical evaluators, integrators, and operators considering
SNA-CLAW-3270 for use in their environment.

---

## Table of contents

1. Introduction
2. What SNA-CLAW-3270 is, in one paragraph each for three readers
3. Architecture
4. The Telnet renegotiation problem and how this tool solves it
5. Prerequisites
6. Installation
7. Configuration: `backends.ini`
8. Configuration: `skills.ini` (complete file shown)
9. Operations: running and using the tool
10. The skill execution model
11. Skill authoring overview
12. Logging and tracing
13. Troubleshooting reference
14. Use cases
15. What is not in scope
16. Roadmap
17. Glossary

---

## 1. Introduction

SNA-CLAW-3270 is a Python program that drives an IBM 3270 mainframe session
through a sequence of named, gated actions called "skills." It is intended
for operators who need scripted or semi-scripted control of mainframe
applications and for engineers building higher-level automation on top of
3270 hosts.

It is not a terminal emulator. A user does not watch a window and type into
it. Instead, the user invokes a skill — by name, or eventually by natural
language — and the tool drives the host on their behalf, verifying at every
step that the screen is what it expects to be.

The tool was developed against the public mainframe service at
`tso.tso3270.com`, which fronts a Hercules-hosted MUSIC/SP system (among
others) using the `proxy3270` gateway. Driving that stack reliably required
solving a Telnet-layer protocol problem that breaks naive Python 3270
clients. The solution — a custom Telnet renegotiation proxy embedded in
the tool — is one of the things that distinguishes SNA-CLAW-3270 from
other Python automation attempts against this kind of gateway.

This document covers the architecture, installation, configuration,
operation, and known limitations of the tool. It does not include source
code walkthroughs. The skills system is documented in detail in a separate
"Skills Authoring Guide."

---

## 2. What SNA-CLAW-3270 is, in one paragraph each for three readers

**For the buyer or evaluator:** SNA-CLAW-3270 is a Python-only mainframe
automation tool that turns common 3270 workflows into reusable, gated,
declaratively-described skills. It does not require any vendor 3270
emulator binary to be installed. It handles the Telnet renegotiation that
gateway-fronted Hercules systems require. Skills are added as text-file
entries, not Python code. The current build runs interactively; an
LLM-driven planning layer is on the immediate roadmap.

**For the operator:** Once configured, you start the tool and type the name
of a skill, or a recognized natural-language prompt. The tool establishes a
session if one does not exist, verifies you are on the expected screen,
performs the action, and reports success or a specific failure. Every
action is logged. You can add new skills by writing one INI section, often
with no Python at all.

**For the integrator or engineer:** SNA-CLAW-3270 is a four-layer Python
program — REPL, worker, connector, shim — with a configuration-driven
skill registry on top. It uses IBM's `tnz` library as its 3270 protocol
engine. It enforces preconditions and postconditions on every skill at the
registry level, so any caller (human, script, or future LLM dispatcher)
cannot drive the host into an undefined state by invoking a skill from the
wrong screen. The shim that handles the proxy3270 renegotiation is
independent of `tnz`'s internal state and can be reused or replaced.

---

## 3. Architecture

SNA-CLAW-3270 is structured as four layers, plus a configuration layer.

```
   ┌─────────────────────────────────────────────────────────────┐
   │  REPL (claw_interface.py)                                   │
   │  - reads user input                                         │
   │  - calls the worker's execute_intent()                      │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  Worker (ai_3270_claw_worker.py)                            │
   │  - loads backends.ini and skills.ini                        │
   │  - matches prompt to a skill or composed plan               │
   │  - holds the session registry                               │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  Skill Registry (subsystems/skill_registry.py)              │
   │  - validates parameters                                     │
   │  - enforces preconditions                                   │
   │  - executes DSL steps or Python implementation              │
   │  - enforces postconditions                                  │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  Connector (subsystems/tn3270_connector.py)                 │
   │  - wraps tnz.ati                                            │
   │  - exposes send / wait_for / scrhas / get_screen            │
   │  - drives the keyboard-unlock state machine                 │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  Shim (subsystems/proxy3270_shim.py)                        │
   │  - local TCP listener                                       │
   │  - intercepts Telnet renegotiation on backend handoff       │
   │  - forwards 3270 data verbatim once stable                  │
   └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                       remote host (proxy3270)
```

Each layer is independent. The REPL can be replaced with a web service or
an LLM agent without touching anything below. The shim can be removed for
direct connections to backends that do not need renegotiation. The
connector can be swapped for a different 3270 stack (e.g. `p3270` +
`wc3270`) without affecting skills. The skill registry has no knowledge of
the network at all; it only knows about the connector's API.

---

## 4. The Telnet renegotiation problem and how this tool solves it

This section exists because the renegotiation problem is the single hardest
technical issue the tool addresses, and any serious evaluator needs to
understand it to judge whether the tool's solution is acceptable for their
environment.

### The problem

When a 3270 client connects to a `proxy3270` gateway, the gateway and the
client first negotiate Telnet options normally: BINARY, EOR, TERMINAL-TYPE,
and so on. The client then sees a menu screen and selects a backend system.

At that point, `proxy3270` (in some configurations, particularly when
fronting Hercules emulators) tears down the original Telnet option
negotiation and forwards the client to the backend, which begins its own
fresh negotiation. The byte sequence from `proxy3270` looks like:

```
IAC WONT EOR
IAC WONT BINARY
IAC DONT BINARY
IAC DONT EOR
IAC DONT TERMINAL-TYPE
```

A well-behaved 3270 client must respond to every one of those with a
matching reply (DONT or WONT respectively), then handle the backend's
fresh DO/WILL sequence and complete a new terminal-type negotiation cycle.

The IBM `tnz` library, in our testing, did not handle this correctly. It
responded to only the last of the five WONT/DONT commands, and it answered
every subsequent SEND for terminal-type with the same `IBM-DYNAMIC` value
rather than cycling through types as RFC 1091 prescribes. The backend
Hercules instance, seeing an incomplete negotiation, dropped the session.

Commercial 3270 emulators (`QWS3270`, `Vista3270`, `wc3270`) handle this
correctly. Hence the asymmetry: a user's terminal worked, but Python
automation against the same host did not.

### The solution

SNA-CLAW-3270 includes `proxy3270_shim.py`, a local TCP listener that sits
between the connector and the remote host. The shim has two operational
modes:

1. **Passthrough.** During the initial connection and menu navigation, the
   shim forwards bytes verbatim in both directions. `tnz` handles this
   first negotiation correctly.

2. **Intercept.** When the shim sees an unsolicited `IAC WONT` or `IAC
   DONT` from the server, it enters intercept mode. In this mode:
   - It answers every Telnet command on `tnz`'s behalf, using RFC-correct
     replies
   - It cycles through a list of terminal types
     (`IBM-DYNAMIC`, `IBM-3279-2-E`, `IBM-3278-2-E`, `IBM-3278-2`) on
     successive `SEND` subnegotiations
   - It does not forward any Telnet bytes to `tnz` during this window
   - When the first non-Telnet byte arrives (the start of the backend's
     first 3270 data stream), the shim exits intercept mode

The result, from `tnz`'s perspective, is a normal session: it sees the
initial proxy3270 negotiation, then nothing for ~500 milliseconds, then
3270 data. It never sees the renegotiation. The session "Just Works."

### Why this is acceptable

The shim is small, well-bounded (only Telnet bytes are interpreted; all
3270 data passes through untouched), and runs on `127.0.0.1`. It does not
introduce a security boundary because it does not terminate or originate
the application protocol. It can be turned off via `backends.ini` for
backends that do not need it.

### Limitations

- The shim assumes the renegotiation pattern matches the one
  `proxy3270` produces. Gateways with different renegotiation patterns
  may require a different shim or extensions to this one
- The terminal-type cycle is hardcoded. If a backend insists on a type
  not in the list, the cycle will repeat the last entry, which the
  RFC permits but some backends may not like
- The shim is per-session: each new TN3270 session gets its own shim
  instance on a local port

---

## 5. Prerequisites

### Host operating system

- **Windows 10 or 11**, 64-bit, with Python 3.10 or newer. This is the
  primary tested platform
- **Linux** (Ubuntu, Debian, Fedora) with Python 3.10+. The shim's
  per-socket keepalive uses `TCP_KEEPIDLE` on Linux; basic operation
  works on macOS too but is less tested

### Python packages

- `tnz` 0.6.6 or newer. Installed via `pip install tnz`
- Standard library only otherwise. No native extensions, no DLLs

### Network requirements

- Outbound TCP to the target 3270 gateway. For the public test stack,
  this is `tso.tso3270.com:26640`
- The shim binds to a random ephemeral port on `127.0.0.1`. No external
  firewall openings are required

### Mainframe-side requirements

- A reachable `proxy3270` (or similar) gateway, or a direct TN3270
  endpoint. The tool does not require any agent or program to be
  installed on the mainframe itself
- For the bundled skills, valid credentials for the target MUSIC/SP
  system. The public test credentials (`$000` / `music`) are widely
  documented for the MUSIC/SP educational distribution

### What is not required

- No 3270 emulator binary. The tool does not invoke `wc3270`, `x3270`,
  `c3270`, or any vendor product
- No SNA stack or VTAM gateway. The tool is pure TN3270 over TCP
- No mainframe-side agent or REXX exec

---

## 6. Installation

```
git clone <repository-url> sna-claw-3270
cd sna-claw-3270
python -m venv venv

# Windows
venv\Scripts\activate

# Linux / macOS
source venv/bin/activate

pip install --upgrade pip
pip install tnz
```

Then verify the configuration files are present:

```
config\backends.ini
config\skills.ini
```

Examples of both files are shipped with the project. Edit them to point
at your target host.

Run the test harness:

```
python test_harness.py
```

This loads configurations without opening a session and reports what it
found. A successful run prints the list of loaded backends and skills.

Start the interactive REPL:

```
sna-claw3270.cmd start
```

(or `python claw_interface.py` directly)

---

## 7. Configuration: `backends.ini`

Backends are the remote 3270 systems the tool can connect to. Each backend
is one INI section. A typical configuration with all four bundled backends
looks like this:

```ini
[backends.music_sp]
name                = MUSIC_SP
host                = tso.tso3270.com
port                = 26640
protocol            = TN3270
menu_option         = 3
use_proxy3270_shim  = true
secure              = false
description         = McGill MUSIC/SP (via proxy3270 option 3)

[backends.mvs_tk5]
name                = MVS_TK5
host                = tso.tso3270.com
port                = 26640
protocol            = TN3270
menu_option         = 1
use_proxy3270_shim  = true
secure              = false
description         = Turnkey MVS 3.8j with TK5 updates

[backends.vm_sixpack]
name                = VM_SIXPACK
host                = tso.tso3270.com
port                = 26640
protocol            = TN3270
menu_option         = 2
use_proxy3270_shim  = true
secure              = false
description         = VM/370 Community Sixpack

[backends.zlinux_390x]
name                = ZLINUX
host                = zlinux.example.local
port                = 23
protocol            = TN3270
menu_option         =
use_proxy3270_shim  = false
secure              = false
description         = Direct connection, no proxy3270 in front
```

**Key-by-key reference:**

| Key | Meaning |
|---|---|
| `name` | Display name used in the REPL and logs |
| `host` | DNS name or IP address of the gateway |
| `port` | TCP port. `26640` is the public `tso.tso3270.com` test port. `992` is the conventional secure port. `23` is plain Telnet |
| `protocol` | Reserved. Always `TN3270` in this build |
| `menu_option` | Numeric choice on the `proxy3270` menu, if any. Leave empty for direct connections that do not show a menu |
| `use_proxy3270_shim` | `true` to route through the renegotiation shim. Required for backends behind `proxy3270`. Set `false` for direct TN3270 endpoints |
| `secure` | `true` if the endpoint expects implicit TLS. Defaults to `false`. The connector sets `SESSION_SSL=0` on `tnz` when `secure` is false, which is required to avoid a 60-second SSL handshake timeout against plaintext endpoints |
| `description` | Free-text description for documentation |

**Important note on `secure`.** The `tnz` library defaults to attempting an
SSL/TLS handshake on connect. For plaintext 3270 endpoints, this causes a
60-second hang followed by a `ConnectionAbortedError`. SNA-CLAW-3270's
connector reads the `secure` flag and explicitly sets `SESSION_SSL=0` when
the flag is false, which suppresses the SSL attempt. If you see SSL
handshake errors, this is the knob.

---

## 8. Configuration: `skills.ini` (complete file shown)

Skills are the unit of work in SNA-CLAW-3270. Each skill is one INI
section. The complete current skill catalog ships as the following file:

```ini
# SNA-CLAW-3270 Skill Catalog
# ---------------------------
# Each [skill.NAME] section defines one callable atomic action.

[skill.nav-cerberus-gate]
description    = From CERBERUS GATE, select MUSIC/SP and arrive at its banner
backend        = music_sp
precondition   = CERBERUS GATE
steps          =
    send:[tab]3[enter]
    wait:Multi-User System
postcondition  = Multi-User System


[skill.music-signon]
description    = From MUSIC/SP banner, sign on and reach the ADMIN main menu
backend        = music_sp
params         = userid, password
precondition   = Multi-User System
steps          =
    send:[enter]
    wait:MUSIC Userid:
    send:{userid}[tab]{password}[enter]
    fail_if:not authorized
    wait:Press ENTER to continue
    send:[enter]
    wait:SELECT OPTION
postcondition  = SELECT OPTION


[skill.music-signoff]
description    = From any MUSIC/SP screen, return to command line and sign off
backend        = music_sp
precondition   = MUSIC
steps          =
    send:[pf3]
    send:off[enter]


[skill.music-admin-select]
description    = From ADMIN main menu, choose a numbered option
backend        = music_sp
params         = opt
precondition   = SELECT OPTION
steps          =
    send:{opt}[enter]


[skill.music-list-files]
description    = From ADMIN main menu, open file-name display (option 6)
backend        = music_sp
precondition   = SELECT OPTION
steps          =
    send:6[enter]
    wait:File Names
postcondition  = File Names
```

**Key-by-key reference:**

| Key | Meaning |
|---|---|
| `description` | One-line human-readable summary. Used in the REPL listing and (eventually) provided to LLM planners |
| `backend` | Which entry in `backends.ini` this skill targets |
| `params` | Comma-separated parameter names. Use `name=default` for defaults. Required parameters that are not provided cause the skill to fail before any action is taken |
| `precondition` | Text that must appear on the current screen before the skill runs. If not present, the skill refuses to fire and reports the screen content. This is the "no traffic ticket" guarantee |
| `postcondition` | Text that must appear on the screen after all steps complete. If not present, the skill reports failure even if all steps individually succeeded |
| `timeout` | Per-step timeout in seconds. Defaults to 20 |
| `steps` | DSL lines, one action per line. See below |

**DSL actions:**

| Action | Effect |
|---|---|
| `send:<mnemonic>` | Sends a `tnz` mnemonic string. Waits for keyboard unlock afterward. Examples: `send:[enter]`, `send:[pf3]`, `send:$000[tab]music[enter]` |
| `type:<text>` | Types literal text without an AID (no Enter). Used rarely |
| `wait:<text>` | Waits for the screen to contain text. Fails on timeout |
| `fail_if:<text>` | Fails the skill immediately if the screen contains text. Useful for catching "Userid not authorized" or similar |
| `peek:<label>` | Logs a snapshot of the screen. Diagnostic only; never affects success |

**Parameter substitution.** Anywhere inside a `send:`, `type:`, `wait:`,
or `fail_if:` line, `{name}` is replaced with the resolved parameter value.
References to undefined parameters cause the skill to fail before any
action is taken.

**Python override.** If a file named `skills/<skill-name-with-underscores>.py`
exists and defines a `run(session, **params)` function, the registry calls
that function instead of interpreting the DSL `steps` block. This is for
skills that need branching, retries, or richer logic. The shipped
`music-signon` skill includes both a DSL definition and a Python
implementation; the Python one is preferred. See the Skills Authoring
Guide for details.

---

## 9. Operations: running and using the tool

Start the REPL:

```
sna-claw3270.cmd start
```

The startup banner reports:

```
Loaded N backends
  (loaded python impl for music-signon)
SkillRegistry: loaded 5 skill(s)
  - nav-cerberus-gate [dsl] (music_sp): ...
  - music-signon [py] (music_sp): ...
  - music-signoff [dsl] (music_sp): ...
  - music-admin-select [dsl] (music_sp): ...
  - music-list-files [dsl] (music_sp): ...
```

`[dsl]` means the skill runs from its INI `steps` block. `[py]` means a
Python override was found in `skills/` and will be used instead.

### Invoking skills

Three ways:

**Direct invocation.** Type `skill:` followed by the skill name and any
parameters as `key=value` pairs:

```
skill:music-signon userid=$000 password=music
skill:music-admin-select opt=6
skill:music-signoff
```

The session is established automatically on first use of a backend.

**Composed plan.** A small number of natural-language prompts are
recognized and routed to fixed plans. The current build recognizes
"logon to music..." style prompts and composes
`nav-cerberus-gate` + `music-signon`:

```
logon to music/sp as userid $000 password music
```

The credentials are parsed from the prompt; if they are not present,
defaults from the skill definition apply.

**Status and listing.** Any prompt that does not match a skill name or a
recognized composition is treated as a status query. The tool prints the
current screen and the list of available skills:

```
list backends
help
status
```

### Output

Every skill invocation produces a structured result:

```
[ok] music-signon: signed on as $000
        . send:[enter]
        . wait:MUSIC Userid:
        . send:$000[tab]music[enter]
        . fail_if:not authorized
        . wait:Press ENTER to continue
        . send:[enter]
        . wait:SELECT OPTION
```

or, on failure:

```
[FAIL] music-signon: credentials rejected by MUSIC/SP
        . send:[enter]
        . wait:MUSIC Userid:
        . send:WRONG[tab]WRONG[enter]
        ! failed at: fail_if:not authorized
```

The final screen content is logged whenever a skill reports failure, so
the cause is always visible.

### Session lifecycle

The tool maintains one session per backend, lazily created on first use
and held for the lifetime of the REPL. Closing the REPL drops the
sessions. There is no current command for closing an individual session
without exiting the REPL; this is on the roadmap.

---

## 10. The skill execution model

A skill invocation goes through five phases in strict order:

1. **Parameter resolution.** The registry collects passed parameters,
   fills in defaults, and verifies all required parameters are present.
   Missing required parameters cause an immediate failure with no
   action taken
2. **Session liveness check.** If the session is not alive (e.g. dropped
   by the host), the skill fails immediately
3. **Precondition gate.** The current screen is checked for the
   precondition text. The check includes a short `wait_for` window
   (3 seconds by default) to allow for screens still painting. If the
   precondition is not satisfied, the skill fails and the actual screen
   content is logged for diagnosis. No `send` is performed
4. **Step execution.** Each DSL step (or the Python implementation) runs
   in order. Any single step failure halts the skill at that step. The
   step that failed is recorded in the result
5. **Postcondition gate.** If all steps succeeded and a postcondition is
   defined, the current screen is checked for the postcondition text.
   Failure here is reported even if all individual steps reported
   success — this catches "we did everything we were told but the host
   ended up somewhere unexpected" cases

A skill cannot partially succeed. Either all gates and steps pass, or the
skill is reported as failed with a specific reason.

This model exists so that any caller — a human, a script, an LLM
planner — can compose skills into longer workflows without each caller
having to verify state at every step.

---

## 11. Skill authoring overview

A new skill is, at minimum, one new section in `config/skills.ini`. The
tool reloads the catalog at startup. There is no compilation step, no
registration call, no decorator. The skill is available as soon as the
REPL restarts.

A trivial skill:

```ini
[skill.music-show-mail]
description    = From ADMIN menu, open the mail facility
backend        = music_sp
precondition   = SELECT OPTION
steps          =
    send:6[enter]
    wait:Mail
postcondition  = Mail
```

For skills that need parameters, branching, retries, or richer logic, a
Python file at `skills/<name-with-underscores>.py` overrides the DSL.

The full authoring guide — including parameter handling, common patterns,
debugging tips, and naming conventions — is in `SKILLS-AUTHORING.md`.

---

## 12. Logging and tracing

The tool logs at INFO level by default. Every state transition produces
a log line. The most important categories are:

- **Worker dispatch**: `Intent received: ...`
- **Skill execution**: `SKILL CALL: ...`, `SKILL OK: ...`, `SKILL FAIL: ...`
- **Screen peeks**: every `peek:` action and every gate check produces
  a screen snapshot in the log
- **Connector state**: `STATE [before send(...)]` lines showing the tnz
  state machine
- **Shim activity**: `Shim accepted`, `Shim connected upstream`,
  `Shim: ENTERING INTERCEPT MODE`, `Shim: EXIT INTERCEPT`

### Byte-level Telnet tracing

For deeper debugging, set the environment variable `SHIM_TRACE=1` before
starting the tool:

```
set SHIM_TRACE=1
sna-claw3270.cmd start
```

This adds hex dumps of every chunk of bytes the shim sees in either
direction. Each line is labeled `s->raw` (server to client raw),
`s->c (fwd)` (server to client after filtering), `shim->s` (shim to
server), or `c->s` (client to server). This is the level used for
diagnosing renegotiation issues. It is verbose enough to be impractical
for normal operation but invaluable when investigating new backends.

---

## 13. Troubleshooting reference

| Symptom | Most likely cause | Action |
|---|---|---|
| `ConnectionAbortedError: SSL handshake is taking longer than 60.0 seconds` in `tnz.log` | `secure=true` on a plaintext endpoint, or `secure` unset and `tnz` defaulting to SSL | Set `secure=false` in `backends.ini` for the target backend |
| Connect succeeds, screen is blank | Connector exited the connect path before the screen finished painting | Verify the connector includes the post-handoff content wait (`_wait_for(self._has_content)` after `menu_option`). This is in the shipped build |
| Skill fails with "precondition not met" but the screen looks correct | The marker text may not be unique to the target screen, or the screen is still buffering | The registry already does a 3s `wait_for` on preconditions; if that is insufficient, the marker is wrong or the screen genuinely is not where expected |
| Skill says it succeeded but the host did something unexpected | Postcondition is missing or too loose | Tighten the postcondition. A skill without a postcondition is "fire and forget" by design |
| `Userid not authorized` from MUSIC/SP | Wrong credentials, or the cursor was not on the userid field when input started | The shipped `music-signon` skill does not type a leading TAB. If your MUSIC/SP variant lands the cursor elsewhere, override the skill in `skills/music_signon.py` |
| Session drops during sign-on with `seslost` | The renegotiation shim's intercept did not complete cleanly | Enable `SHIM_TRACE=1` and inspect the post-menu byte trace. Compare to the reference trace in section 4. The most common cause is the shim's terminal-type cycle not satisfying the backend |
| `Shim: server->client error: timed out` | Older builds had aggressive TCP keepalive tuning that killed the socket | The shipped shim uses OS-default keepalive and explicitly resets the upstream socket to blocking mode. If you see this message, you are on an older build |
| Worker reports SUCCESS but the screen is clearly wrong | Markers in the demo function are too generic | Use unique-phrase markers. `Multi-User System` is unique to the MUSIC banner; `MUSIC` alone matches almost every MUSIC/SP screen |

---

## 14. Use cases

The current tool, without the LLM planning layer, is suitable for:

- **Scripted operator workflows.** Sign on, run an admin task, sign off,
  log the result. A scheduled task can invoke skills via direct
  invocation
- **3270 application regression testing.** Verify that a host-side
  change does not break a known navigation sequence. The postcondition
  gates give a clear pass/fail signal
- **Education and training.** Demonstrate 3270 protocol behavior, menu
  navigation, and EBCDIC screen rendering to students or new operators
  without exposing them to a full emulator
- **Inventory and reporting.** Skills can drive a host to a listing
  screen, capture the rendered content, and feed it to downstream
  tooling
- **Documentation by example.** A skill in `skills.ini` is a literal,
  unambiguous description of how to perform a task. The file itself is
  the documentation

With the planned LLM dispatcher (see Roadmap), the tool extends to:

- **Conversational operator assistance.** A user types or says what
  they want done; the LLM picks and composes skills, executes them, and
  reports back
- **Adaptive workflows.** When a skill fails, the LLM can see the
  failure and pick a recovery skill instead of halting

The tool is **not** intended for:

- High-volume transactional workloads. It is operator-facing
- Replacing CICS or IMS application interfaces. It drives sessions
  intended for human operators
- Real-time monitoring. It pulls; it does not push

---

## 15. What is not in scope

The following are explicit non-goals of the current build. They may be
addressed in future versions; they are not silently missing.

- **Terminal emulator UI.** The tool has no screen window. Output is via
  log and skill result. If you need a visible terminal, use a real
  emulator alongside the tool
- **SNA / VTAM support.** TN3270 only
- **File transfer (IND$FILE).** A skill could be written to drive a file
  transfer, but no transfer protocol is implemented as a primitive
- **Multi-user concurrency.** The REPL is single-session per backend.
  Multiple concurrent users require multiple instances
- **Authentication management.** Credentials are passed as parameters
  or hardcoded in plans. A credential store is on the roadmap
- **Recording and replay.** Sessions are driven from skills, not
  recorded from interactive sessions. A future "record a skill" mode
  is possible but not implemented

---

## 16. Roadmap

The following items are documented as planned, not promised:

- **LLM-driven dispatcher.** A natural-language layer that takes free-form
  intent, inspects the skill catalog, and composes plans. The registry
  already exposes the metadata this requires (`list()`, `describe()`).
  The dispatcher is approximately 150 lines of Python wrapping an LLM
  API call. Target: next development iteration

- **Credential vault.** Replace prompt-time credential passing with a
  configurable credential store (Windows Credential Manager, macOS
  Keychain, or an encrypted file)

- **Additional backend skills.** The MVS TK5 and VM/370 Sixpack
  backends behind the same gateway have no skills written yet. Adding
  them is configuration work, not engineering

- **File transfer primitives.** IND$FILE-style transfers as a primitive
  the skill DSL can invoke

- **Session recording.** A mode where the user drives the host
  interactively and the tool produces a skill definition automatically

---

## 17. Glossary

| Term | Meaning |
|---|---|
| ADMIN | A MUSIC/SP main menu and the userid used to reach it on the test system |
| AID | Attention Identifier — a 3270 byte sequence signaling a user action (Enter, PF key, Clear, etc.) |
| CERBERUS GATE | The branded menu screen presented by the public test gateway. Functionally a `proxy3270` selection menu |
| EBCDIC | The character encoding used by 3270 data streams. Not ASCII |
| EOR | End-of-Record — a Telnet option commonly used with 3270 |
| Hercules | Open-source IBM mainframe hardware emulator |
| IAC | Interpret-As-Command — the Telnet escape byte (0xFF) |
| MUSIC/SP | McGill University System for Interactive Computing — a mainframe operating system originally developed at McGill, available as a free distribution for Hercules |
| MVS TK5 | Turnkey MVS 3.8j — a packaged Hercules-based MVS distribution |
| proxy3270 | Open-source 3270 menu/proxy gateway, by Matthew R. Wilson |
| Renegotiation | The Telnet option-negotiation pattern that `proxy3270` performs when handing off from its menu to a backend |
| Skill | A named, gated, replayable unit of mainframe interaction defined in `skills.ini` and/or `skills/*.py` |
| TN3270 | The Telnet-based protocol for 3270 terminals |
| tnz | IBM-published Python library implementing TN3270 and the ATI scripting interface |
| VM/370 Sixpack | A packaged Hercules-based VM/370 distribution |

---

*End of document.*
