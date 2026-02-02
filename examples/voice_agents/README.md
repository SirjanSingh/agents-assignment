# Intelligent Interruption Handling for LiveKit Voice Agent

## Overview

This document explains the modifications made to `basic_agent.py` to implement intelligent interruption handling that distinguishes between **filler words** (acknowledgments like "yeah", "okay") and **command words** (interruptions like "stop", "wait").

---

## The Challenge

In a natural voice conversation, users often say acknowledgment words like "yeah", "okay", or "hmm" while the agent is speaking. These are **backchannel responses** that mean "I'm listening, continue" — not "stop talking."

However, LiveKit's default Voice Activity Detection (VAD) treats ALL user speech as potential interruptions, causing the agent to stop mid-sentence when hearing these fillers.

**Requirements:**
1. **When agent is speaking + user says filler** → Agent continues uninterrupted
2. **When agent is speaking + user says command** → Agent stops immediately  
3. **When agent is silent** → All user speech is valid input
4. **Mixed input** → Commands always take priority over fillers

---

## The Core Problem: Timing

The fundamental challenge is **VAD interrupts BEFORE transcripts arrive**:

```
Time 0.0s: User starts saying "yeah"
Time 0.3s: VAD detects speech → Interrupts agent
Time 0.5s: User finishes saying "yeah"  
Time 0.8s: Transcript arrives → "Yeah."
```

By the time we know it was a filler word, the agent has already stopped!

---

## The Solution: Hybrid Approach

We use a **three-layer defense system**:

### Layer 1: Medium VAD Thresholds
```python
min_interruption_duration=0.6,  # Requires 0.6 seconds of speech
min_interruption_words=2,        # Requires at least 2 words
```

**Purpose:** Filters out very quick, single-word fillers ("yeah!", "okay!")

**Tradeoff:** Longer fillers (1.5s "okaaaay") can still slip through

---

### Layer 2: Automatic Resume on False Interruptions
```python
resume_false_interruption=True,
false_interruption_timeout=1.0,
```

**Purpose:** If VAD interrupts the agent, LiveKit waits 1 second for more user speech. If nothing substantial comes, it automatically resumes the agent's speech.

**How it helps:** When a slow filler ("okaaaay") interrupts the agent, this mechanism resumes automatically within 1 second.

---

### Layer 3: Transcript-Based Manual Control
The most important layer — our custom logic that analyzes transcripts:

```python
@session.on("user_input_transcribed")
def on_user_input_transcribed(ev):
    # Analyze what the user actually said
    if contains_command(text):
        session.interrupt()  # Force stop
    elif is_filler_input(text):
        return  # Ignore completely
    else:
        # Real input - allow processing
```

This handles three cases:

#### Case 1: Agent Was Just Interrupted by VAD
```python
if kelly.was_interrupted_by_vad:
    if contains_command(text):
        # Real command - stay stopped
    elif is_filler_input(text):
        # False alarm - resume_false_interruption handles it
    else:
        # Real input - process normally
```

#### Case 2: Agent Is Currently Speaking (VAD Hasn't Triggered Yet)
```python
if kelly.is_speaking:
    if contains_command(text):
        session.interrupt()  # Force interrupt NOW
    elif is_filler_input(text):
        return  # Completely ignore
    else:
        session.interrupt()  # Real input - allow interrupt
```

#### Case 3: Agent Is Idle
```python
if not kelly.is_speaking:
    if is_filler_input(text):
        return  # Suppress from LLM
    # Otherwise process normally
```

---

## Key Code Changes

### 1. Word Lists Configuration

**Filler Words** (acknowledgments to ignore):
```python
FILLER_WORDS = {
    "uhhuh", "okay", "alright", "mhm", "yeah", "yep", "yup",
    "hmm", "right", "uh", "um", "ah", "gotit", "isee", "ok",
    # ... more
}

FILLER_PHRASES = {
    "all right", "got it", "i see", "uh huh", "oh okay"
}
```

**Command Words** (explicit stop requests):
```python
STOP_WORDS = {
    "wait", "stop", "finish", "hold", "pause", "halt"
}
```

### 2. Detection Functions

**`is_filler_input(transcript)`** — Returns `True` if input is purely acknowledgment:
- Removes punctuation
- Checks against filler word/phrase lists
- Validates all words are filler tokens

**`contains_command(transcript)`** — Returns `True` if input contains stop command:
- Checks if sentence starts with stop word
- Detects "filler + command" patterns ("yeah wait", "okay stop")
- Avoids false positives in longer sentences

### 3. State Tracking

```python
class IntelligentAgent(Agent):
    def __init__(self):
        self.is_speaking = False           # Currently generating speech
        self.was_interrupted_by_vad = False  # Just got interrupted by VAD
        self.last_speech_content = ""      # Content being spoken
```

### 4. Event Handlers

**`on_speech_created`** — Tracks when agent starts speaking:
```python
@session.on("speech_created")
def on_speech_created(ev):
    kelly.is_speaking = True
    kelly.was_interrupted_by_vad = False
```

**`on_agent_state_changed`** — Detects interruptions:
```python
if ev.old_state == "speaking" and ev.new_state == "listening":
    if kelly.is_speaking:
        kelly.was_interrupted_by_vad = True
```

**`on_user_input_transcribed`** — Main interruption logic (see Layer 3 above)

---

## Configuration Parameters

### AgentSession Settings

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `allow_interruptions` | `True` | Enable VAD-based interruptions |
| `min_interruption_duration` | `0.6` | Require 0.6s of speech to interrupt |
| `min_interruption_words` | `2` | Require 2+ words to interrupt |
| `resume_false_interruption` | `True` | Auto-resume after false interruptions |
| `false_interruption_timeout` | `1.0` | Wait 1s before resuming |
| `preemptive_generation` | `False` | Disabled for more predictable flow |
| `min_endpointing_delay` | `0.5` | Min silence before turn ends |
| `max_endpointing_delay` | `2.5` | Max silence before turn ends |

---

## How It All Works Together

### Scenario 1: User says "yeah" (0.3s, quick acknowledgment)
1. ✅ **VAD Layer:** Too short (0.3s < 0.6s) → No interrupt
2. ✅ **Transcript Handler:** Detects filler while speaking → Ignores
3. ✅ **Result:** Agent continues speaking smoothly

### Scenario 2: User says "okaaaay" (1.5s, slow filler)
1. ❌ **VAD Layer:** Long enough (1.5s > 0.6s) → Interrupts agent
2. ✅ **Resume Layer:** Waits 1s for more speech, nothing comes → Resumes
3. ✅ **Transcript Handler:** Marks as filler → Suppresses from LLM
4. ✅ **Result:** Brief pause (1s), then agent resumes

### Scenario 3: User says "stop" (0.5s, quick command)
1. ✅ **VAD Layer:** Too short (0.5s < 0.6s) → No interrupt
2. ✅ **Transcript Handler:** Detects command → `session.interrupt()`
3. ✅ **Result:** Agent stops immediately via manual interrupt

### Scenario 4: User says "wait a second" (1.2s, clear command)
1. ✅ **VAD Layer:** Long enough (1.2s > 0.6s) → Interrupts agent
2. ✅ **Transcript Handler:** Detects command → Stays stopped
3. ✅ **Result:** Agent stops, processes user's request

---

## Testing the Solution

### Test Cases

1. **Filler while speaking:**
   - Say "yeah", "okay", "hmm" while agent is talking
   - **Expected:** Agent continues without stopping

2. **Command while speaking:**
   - Say "wait", "stop", "hold on" while agent is talking
   - **Expected:** Agent stops immediately

3. **Mixed input:**
   - Say "yeah wait" while agent is talking
   - **Expected:** Agent stops (command wins)

4. **Filler while silent:**
   - Say "okay" when agent is idle
   - **Expected:** Ignored, doesn't trigger new response

5. **Normal conversation:**
   - Ask questions when agent is idle
   - **Expected:** Normal response flow

### Logs to Watch For

```
🎤 KELLY STARTED SPEAKING
📝 TRANSCRIPT: 'yeah' | Kelly speaking: True
🔇 FILLER while speaking: 'yeah' - completely ignored
```

```
📝 TRANSCRIPT: 'wait' | Kelly speaking: True  
🛑 STOP COMMAND while speaking: 'wait' - forcing interrupt NOW
```

```
⚠️ KELLY INTERRUPTED - waiting for transcript...
📝 TRANSCRIPT: 'okay' | Just interrupted: True
🔄 FALSE INTERRUPT: 'okay' was just a filler - should resume
```

---

## Files Modified

- **`basic_agent.py`** — Main implementation with all intelligent interruption logic

## Dependencies

No additional dependencies required beyond standard LiveKit Agents SDK.

---

## Limitations

1. **Brief pause on slow fillers:** If user says a filler slowly (>0.6s), there may be a ~1s pause before auto-resume
2. **Language-specific:** Word lists are currently English-focused (though some Hindi words are included)
3. **Context-unaware:** Doesn't understand semantic context (e.g., "no" as answer vs. "no" as stop command)

---

## Future Improvements

1. **Sentiment analysis:** Use LLM to determine if "no" is a stop command or an answer
2. **Adaptive thresholds:** Learn user's speech patterns and adjust thresholds
3. **Multi-language support:** Extended word lists for other languages
4. **Prosody analysis:** Use tone/pitch to distinguish acknowledgments from commands

---

## Credits

Implementation for the **LiveKit Intelligent Interruption Handling Challenge**.
