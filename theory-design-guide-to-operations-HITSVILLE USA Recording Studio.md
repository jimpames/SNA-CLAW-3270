# HITSVILLE USA Recording Studio  
**SNA-CLAW-3270 Session Recorder + Smart INI Builder**

**Version:** 1.0 (June 2026)  
**Codename:** Hitsville USA ("Where the hits are recorded")

---

## THEORY OF DESIGN

### Core Philosophy
The Hitsville USA Recording Studio was designed to solve one of the biggest barriers in mainframe automation: **making powerful 3270 automation accessible to real users who know their workflows but do not want to write code or complex INI files**.

**Key Design Principles:**

1. **Field/Action Level Recording** (not keystroke level)
   - Optimized for block-mode 3270 environments
   - Captures meaningful business actions (`EnterField`, `Press(PF3)`, etc.)
   - Produces much more maintainable and reliable skills

2. **Full Interactive Terminal Experience**
   - Uses `tnz.zti` (IBM’s interactive console emulator) for recording mode
   - Gives the user the exact same experience as their normal terminal
   - Does **not** interfere with existing lightweight `tnz` playback engine

3. **Intelligent Screen Aliasing**
   - Automatically detects new screens
   - Prompts user for friendly names (e.g. `ADMIN_MAIN_MENU`)
   - Stores original screen signature in comments for future-proofing

4. **Modern Keyboard Mapping**
   - Designed for standard 101-key keyboards (no physical PA keys)
   - Full support for PF1–PF24, PA1–PA3, ATTN, CLEAR, ENTER

5. **Forgiving & Production Ready**
   - Supports pause/resume and partial saves on session interrupts
   - Generates heavily commented, clean `.ini` files
   - Creates visual HTML recording logs for documentation and auditing

---

## GUIDE TO OPERATIONS

### 1. Starting a Recording Session

```bash
# Basic usage
sna-claw3270 --learn MyOrderEntrySkill

# With screenshots enabled
sna-claw3270 --learn MyOrderEntrySkill --screenshot

2. During Recording
You will see a full interactive 3270 terminal. Work completely naturally.
Special Hitsville Commands (type in terminal):

//STEP StepName → Manually name current logical step
//PAUSE → Pause recording
//RESUME → Resume recording
//SAVE → Finish and generate skill files
//SAVE PARTIAL → Save current progress (for interrupted sessions)
//HELP → Show command list

Keyboard Mapping (Modern 101-key)


3270 FunctionPhysical Keys
ENTER Enter
PF1 – PF12
F1 – F12
PF13 – PF24 Shift + F1 – F12
PA1 Ctrl + Shift + F1
PA2 Ctrl + Shift + F2
PA3 Ctrl + Shift + F3
ATTN Ctrl + Shift + A
CLEAR Esc
3. Output Files Created
When you save, the following files are generated:

MySkill.ini — Production-ready skill file
MySkill_recording.html — Visual timestamped report
MySkill_recording.log — Raw JSON recording log
MySkill_Partial.ini — (if partial save used)

4. Resuming Partial Recordings
Bash# Load and continue from a partial recording
sna-claw3270 --load MySkill_Partial

ARCHITECTURE OVERVIEW

 → Core recording engine
hitsville_studio.py → Launches full tnz.zti interactive session
hitsville_cli.py → Command handling (//STEP, //SAVE, etc.)
hitsville_utils.py → Helper functions (step suggestions, previews)
HALL-OF-MIRRORS-TEST-HARNESS.py / SIX-MILLION-DOLLAR-TEST-HARNESS.py → CI/CD protection

All recorder components are completely isolated from the main playback engine.

Status: Production Ready
Test Coverage: Full (Six Million Dollar Test Harness)

