### **2. CLI-INTEGRATION-GUIDE.md**

```markdown
# CLI Integration Guide  
**Adding --learn Support to SNA-CLAW-3270**

---

## Goal

Integrate the Hitsville USA Recording Studio into the main `sna-claw3270` command line without touching existing production code.

---

## Recommended Integration (Clean & Safe)

### 1. Create a new main launcher (or modify existing CLI handler)

Add this code in your main CLI entry point (e.g. `sna-claw3270.cmd` or main Python launcher):

```python
# ====================== HITSVILLE INTEGRATION ======================

import sys
from hitsville_studio import HitsvilleStudio
from hitsville_cli import start_recording_session

if __name__ == "__main__":
    args = sys.argv[1:]
    
    # Handle --learn mode
    if "--learn" in args:
        try:
            idx = args.index("--learn")
            skill_name = args[idx + 1]
            
            # Optional screenshot flag
            screenshot = "--screenshot" in args
            
            print(f"🎤 Starting Hitsville USA Recording Studio...")
            
            # Option A: Full Studio (Recommended - Rich Terminal)
            studio = HitsvilleStudio(skill_name)
            studio.launch()
            
            # Option B: Lightweight integration (if you want more control)
            # recorder = start_recording_session(skill_name, screenshot_mode=screenshot)
            # Then pass recorder into your existing session loop
            
        except IndexError:
            print("Usage: --learn <SkillName>")
            sys.exit(1)
    
    # ... rest of your existing main code ...

Integration Points Needed
You will need to wire these hooks in your main session loop:

on_screen_change(screen_text) — Call when a new screen appears
on_field_action(field_name, value) — When user enters data in a field
on_aid_key(key) — When user presses PF, PA, ATTN, etc.
handle_recorder_command(recorder, user_input) — Check for // commands



Example Hook Placement
Python# Inside your main TN3270 processing loop:

if is_recording_mode:
    recorder.on_screen_change(current_screen_text)
    
    if user_entered_data:
        recorder.on_field_action(field_name, value)
    
    if key_pressed in ["PF3", "ENTER", "PA1", ...]:
        recorder.on_aid_key(key_pressed)
    
    # Check for Hitsville commands
    if user_input.startswith("//"):
        if handle_recorder_command(recorder, user_input):
            # End session if //SAVE was used
            break
Testing Integration
Run the full test harness before merging:
Bashcd hitsville_hall_of_mirrors
python SIX-MILLION-DOLLAR-TEST-HARNESS.py

Integration Status: ✅ Clean & Isolated
Risk Level: Very Low (All new code is in separate files)


