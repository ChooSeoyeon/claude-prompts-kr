# Google Meet Recorder + Auto Transcript & Translation

Auto-record Google Meet audio and transcribe it to text. Translate to your language with one command.

[한국어](./README.ko.md)

---

## How it works

1. `record-meeting` — records system audio + microphone simultaneously
2. After stopping (Ctrl+C), Whisper runs automatically → saves `.txt` transcript
3. `/translate-meeting file.txt ko` — Claude translates to your language

**What runs locally (free):**
- Audio recording (BlackHole + Python)
- Transcription (mlx-whisper, runs on Apple Silicon GPU)

**What uses tokens (minimal):**
- Translation only (~$0.03–0.05 per hour of meeting)

---

## Prerequisites (manual steps before setup)

### 1. Install BlackHole and restart Mac

```bash
brew install blackhole-2ch
```

**Restart your Mac after installation.**

### 2. Audio MIDI Setup (GUI)

1. Spotlight (Cmd+Space) → open "Audio MIDI Setup"
2. Click `+` at bottom left → "Create Multi-Output Device" (do this **twice**)
3. First device — rename to `Meeting-AirPods`:
   - ✅ AirPods
   - ✅ BlackHole 2ch
4. Second device — rename to `Meeting-Speakers`:
   - ✅ MacBook Pro Speakers
   - ✅ BlackHole 2ch
5. Double-click each device to rename it
6. No need to change your default output — `record-meeting` switches automatically and restores when done

---

## Setup

Paste the contents of [prompt.md](./prompt.md) into Claude Code. Claude will handle everything else automatically.

---

## Usage

```
Before meeting:  record-meeting
After meeting:   Ctrl+C
Translate (KO):  /translate-meeting ~/meetings/meeting_FILENAME.txt ko
Translate (EN):  /translate-meeting ~/meetings/meeting_FILENAME.txt en
```
