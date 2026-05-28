# SNA-CLAW-3270
## Skills Authoring Guide

---

This document is a hands-on reference for adding new skills to
SNA-CLAW-3270. It assumes you have read the main operations and design
guide and have a working installation.

The goal of this guide is to make adding a skill take five minutes for
simple cases and an hour for complicated ones. Both numbers are achievable
because the system is designed that way: the simple cases stay in INI;
the complicated ones drop to a small Python file.

---

## Table of contents

1. What a skill is, conceptually
2. The minimum viable skill
3. The skill INI section in full
4. The DSL action reference
5. Parameter handling
6. Preconditions and postconditions — picking good markers
7. When to use a Python override
8. The Python override contract
9. Patterns: common skill shapes
10. Anti-patterns: skills that look reasonable but cause trouble
11. Naming conventions
12. Testing a new skill
13. Debugging when a skill misbehaves
14. Reloading and packaging

---

## 1. What a skill is, conceptually

A skill is one named, gated, deterministic action that drives a 3270
host from a known starting screen to a known ending screen. "Known" is
strict: the registry verifies the starting screen before any input is
sent, and verifies the ending screen before reporting success.

Skills are designed to be:

- **Atomic.** A skill does one thing. "Sign on" is one skill. "Sign on
  and check mail and reply to the first message" is three skills, not
  one
- **Composable.** Skills can be chained because each one's
  postcondition can be another's precondition
- **Idempotent where possible.** Re-running a skill from the same
  starting screen should produce the same result. Where idempotency is
  not possible (a skill that creates a file, for example), the skill
  should fail clearly rather than silently produce a different outcome

A skill is **not**:

- A script. It does not branch on input data in INI form. If you need
  branching, drop to a Python override
- A workflow. A workflow is a sequence of skill invocations, not a
  single skill
- A test. Skills do not assert business outcomes; they assert screen
  states. "User mail count is greater than zero" is not a precondition;
  "screen contains the word `Mail`" is

---

## 2. The minimum viable skill

```ini
[skill.music-show-options]
description    = From ADMIN menu, type ? to show command options
backend        = music_sp
precondition   = SELECT OPTION
steps          =
    send:?[enter]
```

That is a complete, valid skill. It has no postcondition (it is a
diagnostic skill, not a workflow step). It has no parameters. It will
fire only when the ADMIN menu is on screen.

Save it in `config/skills.ini`. Restart the REPL. The skill is now
available:

```
👤 You: skill:music-show-options
```

---

## 3. The skill INI section in full

```ini
[skill.<name>]
description    = <one-line summary, used in REPL listings and LLM prompts>
backend        = <backend id from backends.ini>
params         = <param1, param2, param3=default>
precondition   = <text the screen must contain before the skill runs>
postcondition  = <text the screen must contain after the skill runs>
timeout        = <per-step timeout in seconds, default 20>
steps          =
    <action>:<argument>
    <action>:<argument>
    ...
```

Required keys: `description`, `backend`, and either `steps` (DSL) or a
Python override file.

Optional keys: `params`, `precondition`, `postcondition`, `timeout`.

If neither `steps` nor a Python override exists, the skill loads but
cannot be called.

---

## 4. The DSL action reference

The DSL is intentionally small. Five actions cover the vast majority of
3270 navigation. If you need something else, drop to a Python override.

### `send:<mnemonic_string>`

Send a `tnz` mnemonic string, then wait for the keyboard to unlock. This
is the workhorse. Examples:

```
send:[enter]
send:[pf3]
send:[pf6]
send:[clear]
send:6[enter]
send:$000[tab]music[enter]
send:[tab][tab]MYDATA[enter]
```

`[enter]`, `[tab]`, `[pf1]` through `[pf24]`, `[clear]`, `[pa1]`,
`[pa2]`, and a handful of others are recognized. Literal text outside
brackets is typed verbatim into the current field.

### `type:<text>`

Type literal text without an AID. Rarely needed — most input ends with
Enter or a PF key, in which case `send:` is correct.

### `wait:<text>`

Wait for the screen to contain the text, up to `timeout` seconds. If
the text does not appear, the skill fails at this step.

```
wait:MUSIC Userid:
wait:Press ENTER to continue
wait:File Names
```

### `fail_if:<text>`

Fail the skill immediately if the screen contains the text. The check
is performed right at this point in the step sequence, not on every
step. Use it after the screen has settled, typically after a `wait:`.

```
fail_if:not authorized
fail_if:Invalid command
fail_if:LOGON DENIED
```

### `peek:<label>`

Log a screen snapshot to the diagnostic log. Has no effect on success
or failure. Useful when developing or debugging a skill.

```
peek:before-signon
peek:after-menu-select
```

### Parameter substitution

Any `{name}` token inside the argument of a `send:`, `type:`, `wait:`,
or `fail_if:` line is replaced with the value of the parameter at run
time. References to undeclared parameters cause the skill to fail
before any action is taken — including before the precondition check.

---

## 5. Parameter handling

### Declaration

```ini
params = userid, password, retries=3, log_level=INFO
```

`userid` and `password` are required. `retries` and `log_level` are
optional, with the defaults shown. Required parameters that are not
passed cause an immediate failure.

### Passing parameters at invocation

```
skill:music-signon userid=$000 password=music
```

Parameters are space-separated `key=value` pairs after the skill name.
Values may not contain spaces in the current parser; this is a known
limitation. For values with spaces, use the Python override path.

### Substitution in steps

```ini
steps =
    send:{userid}[tab]{password}[enter]
    wait:Press ENTER to continue
    send:[enter]
```

### Validation

All required parameters must be present at call time. Default values
fill in for absent optional parameters. Unknown parameter names passed
at call time are silently ignored — this is by design, so callers can
pass a superset and skills can ignore what they do not need.

---

## 6. Preconditions and postconditions — picking good markers

This is the single most common place new skill authors get it wrong. A
marker that is "almost unique" to the target screen will match on other
screens too, and your skill will appear to succeed when it has actually
landed somewhere wrong.

### Pick the most specific phrase

Good markers are phrases that **only appear on the target screen**.

| Screen | Bad marker | Good marker |
|---|---|---|
| CERBERUS menu | `Select` | `CERBERUS GATE` |
| MUSIC/SP banner | `MUSIC` | `Multi-User System` |
| MUSIC/SP sign-on | `Userid` | `MUSIC Userid:` |
| MUSIC/SP ADMIN menu | `MAIN` | `SELECT OPTION ====>` |
| MUSIC/SP file list | `Files` | `File Names` |

`MUSIC` alone matches the banner, the sign-on screen, the ADMIN menu,
and almost every other MUSIC/SP screen. `Userid` matches the sign-on
screen and any error screen that references a user. The good markers
above are unique to the target screen as far as we have observed.

### Markers are substring matches

The check is a substring match. It is case-sensitive. It does not
support regular expressions. If you need pattern matching, override
the skill in Python.

### Multi-line markers

The current build matches against a flattened screen — newlines are
preserved, so a marker that spans two lines can be matched if you
include the right whitespace. In practice, single-line markers are
easier.

### When to omit a postcondition

A skill without a postcondition is "fire and forget." This is fine for
diagnostic skills (`peek` only), or for skills whose effect is to leave
the host in a state that the *next* skill's precondition will verify.
For chained skills, you often do not need both — skill A's postcondition
duplicates skill B's precondition.

### When to add a `fail_if`

`fail_if` catches the negative case. After typing credentials and
pressing Enter, the screen will either advance to the continue page
(success) or repaint with an error like "Userid not authorized"
(failure). The skill's `wait:Press ENTER to continue` will time out on
failure — but it will take the full timeout to do so. A `fail_if:not
authorized` step between the credential send and the wait gives an
immediate, specific failure reason.

---

## 7. When to use a Python override

The DSL is sufficient for any skill that is a linear sequence of:
"send something, wait for something, send something, wait for
something." That is the majority of 3270 operator workflows.

You need a Python override if the skill must:

- **Branch on screen content.** "If we see X, send A; otherwise send B"
- **Loop until a condition is met.** "Press Enter repeatedly until we
  reach the end of the banner"
- **Retry on failure.** "Try the credential up to three times before
  giving up"
- **Coerce or validate parameters.** "If userid is missing a leading
  `$`, add one"
- **Compute values at run time.** "Compute today's date in MUSIC/SP
  date format"
- **Return structured data.** "Parse the file list screen into a Python
  list of filenames"

For everything else, DSL is faster to write and easier to read.

---

## 8. The Python override contract

A skill `music-signon` is overridden by a file
`skills/music_signon.py`. Hyphens in the skill name become underscores
in the filename. The file must define a function:

```python
def run(session, **params):
    ...
```

The `session` object exposes the same methods the DSL drives:

- `session.send(mnemonic, wait_for=None, timeout=20.0)` → screen text or error sentinel
- `session.scrhas(text)` → bool
- `session.wait_for(text, timeout=20.0)` → bool
- `session.is_alive()` → bool
- `session.get_screen()` → str
- `session.connector.type_text(text)` → screen text
- `session.connector._screen_peek(label)` → None (logs the screen)

The function may return any of:

- A `SkillResult` instance (most explicit)
- A `dict` with at minimum `{"ok": bool, "message": str}` (most common)
- `True` for success / `False` for failure (briefest)
- `None` (treated as success)

The registry still enforces the precondition before calling the Python
function, and the postcondition after. The Python function does not
need to check either — it can assume the precondition is true on
entry, and the registry will verify the postcondition on its return.

### Minimal Python override

```python
def run(session, **params):
    if not session.wait_for("Continue", timeout=10):
        return {"ok": False, "message": "Continue prompt did not appear"}
    session.send("[enter]")
    return {"ok": True}
```

### Override pattern: branching on screen content

```python
def run(session, **params):
    if session.scrhas("Press the ENTER key"):
        session.send("[enter]")
    if session.scrhas("MUSIC Userid:"):
        session.send(f"{params['userid']}[tab]{params['password']}[enter]")
    else:
        return {"ok": False, "message": "expected sign-on screen, got something else"}
    if session.scrhas("not authorized"):
        return {"ok": False, "message": "credentials rejected"}
    return {"ok": True}
```

---

## 9. Patterns: common skill shapes

### Pattern A: Linear navigation

```ini
[skill.music-list-files]
description    = From ADMIN menu, open file-name display
backend        = music_sp
precondition   = SELECT OPTION
steps          =
    send:6[enter]
    wait:File Names
postcondition  = File Names
```

The cleanest, most common shape. Use DSL.

### Pattern B: Linear with negative check

```ini
[skill.music-signon]
description    = Sign on to MUSIC/SP, reach the ADMIN menu
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
```

Add `fail_if` for the most common failure mode. DSL still suffices.

### Pattern C: Parameterized menu choice

```ini
[skill.music-admin-select]
description    = From ADMIN menu, choose a numbered option
backend        = music_sp
params         = opt
precondition   = SELECT OPTION
steps          =
    send:{opt}[enter]
```

The parameter is substituted into the send. No postcondition — what
the menu choice does depends on which option was chosen, so the
postcondition belongs to the *next* skill in the chain.

### Pattern D: Conditional dismissal (Python required)

```python
# skills/music_dismiss_banner.py
def run(session, **params):
    pages = 0
    while session.scrhas("Press the ENTER key") and pages < 5:
        session.send("[enter]")
        pages += 1
    return {"ok": True, "message": f"dismissed {pages} banner page(s)"}
```

The DSL has no loop. Python is required for this pattern.

### Pattern E: Verified write

```python
# skills/music_create_file.py
def run(session, filename, contents):
    # Navigate to the file editor
    session.send(f"create {filename}[enter]")
    if not session.wait_for("EDIT", timeout=10):
        return {"ok": False, "message": "editor did not open"}
    # Insert content
    session.send(f"{contents}[enter]")
    # Save and exit
    session.send("save[enter]")
    if session.scrhas("error"):
        return {"ok": False, "message": "save reported an error"}
    session.send("file[enter]")
    return {"ok": True, "message": f"created {filename}"}
```

For write operations that have multiple failure modes, Python is
clearer than chained DSL.

---

## 10. Anti-patterns: skills that look reasonable but cause trouble

### Anti-pattern 1: Too-generic markers

```ini
precondition   = MUSIC    # WRONG - matches almost every MUSIC/SP screen
precondition   = Userid   # WRONG - matches sign-on AND error screens
```

Symptom: skill appears to succeed from the wrong screen, then later
skills fail mysteriously because they were called from the wrong place.

### Anti-pattern 2: Missing postcondition on a writing skill

A skill that creates, modifies, or deletes something without verifying
the result is a silent-failure waiting to happen. Always add a
postcondition for skills that change state on the host.

### Anti-pattern 3: Hardcoded credentials in the INI

```ini
steps          =
    send:$000[tab]music[enter]    # WRONG - credentials in plaintext
```

Use parameters with defaults — or, better, no defaults and require
the caller to pass them.

### Anti-pattern 4: One skill that does everything

```ini
[skill.do-all-the-things]
steps          =
    send:[tab]3[enter]
    wait:Multi-User System
    send:[enter]
    send:$000[tab]music[enter]
    ... (twenty more lines)
```

Symptom: when this fails, you cannot tell where it failed. Split it
into atomic skills and compose them.

### Anti-pattern 5: Substring marker that overlaps the input

```ini
precondition   = MUSIC Userid:
steps          =
    send:something
    wait:MUSIC Userid:   # may match the precondition's screen if nothing changed
```

If the wait marker is identical to or a substring of the precondition,
the wait will succeed immediately whether or not the screen actually
advanced. Pick a marker that only appears on the *destination* screen.

---

## 11. Naming conventions

Recommended naming:

- All lowercase, hyphen-separated: `music-list-files`, not `MusicListFiles`
- Prefix with the backend or subsystem: `music-`, `mvs-`, `vm-`
- Verb-noun order: `music-list-files`, not `music-files-list`
- Use `nav-` for skills that navigate without doing work:
  `nav-cerberus-gate`, `nav-mvs-tso-logon`
- Use `signon` and `signoff` (not `login`, not `logout`) to match
  mainframe terminology

The corresponding Python file replaces hyphens with underscores:
`music-list-files` → `skills/music_list_files.py`.

---

## 12. Testing a new skill

1. **Write the skill in `config/skills.ini`**

2. **Restart the REPL.** The startup banner should list the new skill.
   If it does not, check the INI syntax — `configparser` is forgiving
   but a malformed section header will silently drop the section

3. **Verify the gate.** Without driving the host to the precondition
   screen, call the skill. It should refuse with "precondition not met"
   and log the actual screen content. This proves the precondition is
   not matching by accident

4. **Drive the host to the precondition screen** (using whatever skill
   gets you there), then call your new skill. It should succeed

5. **Test the negative path.** If the skill takes parameters, try
   invalid ones. If it has a `fail_if`, try to trigger it. Confirm the
   failure message is useful

6. **Test idempotency where applicable.** Run the skill twice from the
   same starting screen. The first should succeed; the second should
   either succeed identically or fail with a clear "wrong screen"
   message

---

## 13. Debugging when a skill misbehaves

### "Precondition not met"

The diagnostic log will include a screen peek labeled
`precondition[<skill-name>]`. Compare what is on the screen to what
the skill expects. Common causes:

- The marker text is wrong (typo)
- The marker is case-sensitive and the host uses different case
- The screen genuinely is not where you expected — go back one skill

### A step fails

The result includes which step failed:

```
! failed at: wait:File Names
```

That tells you the step before this one ran, but the screen never
showed `File Names`. Cause is usually:

- The previous send was wrong (typo, wrong key)
- The wait marker is wrong
- The host actually did something other than what you expected

Run the same skill with `SHIM_TRACE=1` enabled. The byte-level log
will show exactly what bytes were exchanged.

### Postcondition not met

All steps ran; the final screen does not match the postcondition. This
is the case where a skill's `send:` reached the host and the host
responded, but it responded with something unexpected. Inspect the
final screen content in the log.

### Skill hangs

A `wait:` step with a default 20-second timeout will hang for 20
seconds before failing. If you see hangs longer than that, the
underlying connector's keyboard-unlock is stuck. Check the connector
state dump for `pwait: True` and `slock: True` — this is the host
holding the keyboard locked, which usually means it is processing or
waiting for further input.

---

## 14. Reloading and packaging

There is currently no hot-reload. After editing `config/skills.ini`
or any file in `skills/`, restart the REPL.

Skill packaging — distributing skills as a separate package — is not
currently supported. Skills live in the project's `config/` and
`skills/` directories. A future version may support skill packs as
installable bundles; for now, share skills as text.

---

*End of document.*
