# SNA-CLAW-3270

**A natural-language operator for IBM 3270 mainframe systems, in pure Python.**

SNA-CLAW-3270 drives a real 3270 terminal session against a real mainframe
through a configurable skill catalog. Each skill is a small, gated, replayable
unit of work — sign on, navigate a menu, run a command, sign off. The tool
participates in the 3270 protocol natively (not screen scraping) and includes
a custom Telnet renegotiation shim that allows it to drive sessions through
the `proxy3270` gateway used by hobbyist and educational mainframe
installations.

This is not a 3270 emulator. It is an automation substrate.

<img width="1832" height="898" alt="sna-claw-3270" src="https://github.com/user-attachments/assets/7932d598-91ba-4b4b-a01a-0d54025cc22a" />



intro

https://www.linkedin.com/pulse/introducing-sna-claw-3270-ai-native-3270-controller-real-jim-ames-6xuie


video demo


https://youtu.be/JYI631-Rb1A?si=KHIFC1icFitV-4w-


## What it does today

- Connects to a remote 3270 host through a configurable gateway
- Renders full screens, including extended attributes, via the `tnz` library
- Executes named skills declared in `config/skills.ini`
- Enforces preconditions and postconditions on every skill — a skill cannot
  fire from the wrong screen, and cannot claim success without proof
- Handles the `proxy3270` mid-session option renegotiation that breaks naive
  Python 3270 clients
- Accepts both direct skill invocation (`skill:music-signon userid=$000 password=music`)
  and prompt-shaped invocation (`logon to music/sp as userid $000 password music`)
- Logs everything: state transitions, screen contents, and (optionally) raw
  Telnet bytes for debugging

## What it does not do today

- It does not yet take free-form natural language and decide which skills to
  invoke. The current dispatcher recognizes a small set of prompt shapes and
  routes to fixed plans. The LLM-driven planner is documented in the guide
  as a near-term addition.
- It does not include skills for every workflow. Five skills ship by default,
  covering CERBERUS gate navigation, MUSIC/SP sign-on, sign-off, menu
  selection, and file listing. Adding skills is a config change, not a code
  change.
- It does not work against every mainframe stack out of the box. It has been
  tested against `proxy3270` → Hercules → MUSIC/SP. Other gateways and
  backends are likely to work but are not yet verified.

## Quick start

```
git clone <repository>
cd sna-claw-3270
python -m venv venv
venv\Scripts\activate          # or source venv/bin/activate on Linux/macOS
pip install tnz
sna-claw3270.cmd start         # or python claw_interface.py
```

At the REPL:

```
👤 You: skill:nav-cerberus-gate
👤 You: skill:music-signon userid=$000 password=music
👤 You: skill:music-signoff
```

For installation details, configuration, the architecture, the skill
authoring guide, and the troubleshooting reference, see the
[full operations and design guide](SNA-CLAW-3270-GUIDE.md).

To add new skills, see the [skills authoring guide](SKILLS-AUTHORING.md).

## Project layout

```
sna-claw-3270/
├── claw_interface.py              REPL frontend
├── ai_3270_claw_worker.py         intent dispatcher + skill orchestrator
├── subsystems/
│   ├── tn3270_connector.py        ATI/tnz session wrapper
│   ├── proxy3270_shim.py          Telnet renegotiation proxy
│   └── skill_registry.py          INI loader + DSL runner
├── config/
│   ├── backends.ini               backend definitions
│   └── skills.ini                 skill catalog
├── skills/
│   └── <name>.py                  optional Python skill implementations
└── sna-claw3270.cmd               Windows launcher
```

## Status

Working against the public reference installation at `tso.tso3270.com:26640`,
which is a `proxy3270` gateway in front of Hercules-hosted MUSIC/SP. The
sign-on and sign-off path has been verified end-to-end. Other backends behind
the same gateway (MVS TK5, VM/370 Sixpack) are reachable but their skill
catalogs have not yet been written.

## License

To be determined by the project owner.

## Acknowledgements

- IBM's `tnz` library provides the underlying 3270 data-stream parser
- `proxy3270` by Matthew R. Wilson (racingmars) is the gateway this tool
  was developed against
- The Hercules emulator and the MUSIC/SP, MVS 3.8j, and VM/370 community
  distributions make this kind of work possible in the first place
