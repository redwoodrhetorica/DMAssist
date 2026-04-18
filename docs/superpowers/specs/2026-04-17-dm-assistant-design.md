# DM Assistant — Design Spec
**Date:** 2026-04-17  
**Status:** Approved

---

## Overview

DM Assistant is a desktop application for Dungeon Masters that monitors game sessions (Discord voice/text and/or local microphone), transcribes what happens, classifies events as in-game or out-of-character, and generates structured logs and narrative summaries on demand. It is designed for non-technical users — the default experience requires zero configuration.

---

## Goals

- Support online (Discord), in-person (local mic), and hybrid sessions
- Differentiate in-game dialogue/actions from out-of-character player chat
- Identify and attribute speech to known characters
- Generate structured event logs and narrative session summaries on demand
- Be approachable to users who barely know how to use a computer

## Non-Goals (v1)

- Cloud hosting or SaaS model
- Google Docs / Notion / Obsidian output
- Mobile companion app
- Automated session scheduling

---

## Architecture

**Pattern:** Event-driven with plugin system  
**Platform:** Electron desktop app (Windows/Mac)  
**Stack:** TypeScript, React, Vite (via electron-vite), shadcn/ui, Tailwind CSS

```
┌─────────────────────────────────────────────────────────┐
│                    ELECTRON MAIN PROCESS                 │
│                                                         │
│  ┌──────────────┐    ┌───────────────────────────────┐  │
│  │ Input Plugins│    │         Event Bus             │  │
│  │              │───▶│                               │  │
│  │ • Discord    │    │  Raw events → Transcription → │  │
│  │   Voice      │    │  Classification → Event Store │  │
│  │ • Discord    │    │                               │  │
│  │   Text       │    └──────────────┬────────────────┘  │
│  │ • Local Mic  │                   │                   │
│  └──────────────┘    ┌──────────────▼────────────────┐  │
│                      │       Output Plugins          │  │
│                      │ • Discord Channel             │  │
│                      │ • Markdown File               │  │
│                      │ • PDF Export                  │  │
│                      │ • Clipboard / Email           │  │
│                      └───────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────┘
                              │ IPC
┌─────────────────────────────▼───────────────────────────┐
│                  ELECTRON RENDERER (Web UI)              │
│   Session Control │ Live Transcript │ Characters │ History│
└─────────────────────────────────────────────────────────┘
```

**Data flow for a single voice utterance:**
1. Input plugin captures audio chunk
2. Whisper transcribes it to text
3. Classifier (Claude API) tags it: speaker, in-game/OOC, event type
4. Event written to session SQLite database with timestamp
5. UI updates live via IPC
6. On DM command → Summary Generator reads full session log + character registry → Claude generates narrative → delivered to configured outputs

---

## Input Plugins

Each plugin implements a common `InputPlugin` interface: `start()`, `stop()`, emits `RawEvent` onto the event bus.

### Local Microphone Plugin (Default)
- Captures system audio via `naudiodon` (Node.js PortAudio bindings)
- Voice activity detection (VAD) to detect utterance boundaries
- Speaker identity unknown — classification infers character from context
- DM can optionally configure multiple mic inputs for better attribution
- **Default enabled. Zero setup required.**

### Discord Voice Plugin (Advanced)
- Connects via `discord.js` to a configured voice channel
- Captures per-user audio streams — Discord provides separate streams per speaker, giving free speaker identity
- Buffers audio and sends to transcription on silence detection
- Requires Discord bot token (configured via in-app wizard)

### Discord Text Plugin (Advanced)
- Monitors one or more configured Discord text channels
- Passes messages directly to classification — no transcription needed
- Respects `//ooc` prefix as a strong OOC signal, but classifies contextually regardless
- Requires same Discord bot token as voice plugin

---

## Processing Pipeline

### Stage 1 — Transcription Service
- Audio events → Whisper local inference via `whisper-node` (wraps whisper.cpp)
- Text events (Discord text, companion web input) skip this stage
- Whisper model auto-selected based on detected hardware
- Exposed to user as "Transcription Quality: Good / Better / Best" — no technical labels

### Stage 2 — Classification Engine
Sends a structured prompt to Claude API containing:
- Transcribed text
- Recent event history (last N events for context window)
- Character registry
- Detected signals (`//ooc`, tone indicators)

Claude returns structured JSON:
```json
{
  "type": "dialogue" | "action" | "combat" | "ooc" | "meta",
  "character": "Theron" | "player:Jake" | null,
  "confidence": 0.92,
  "in_game": true
}
```

Low-confidence events (below configurable threshold) are flagged for optional DM review in the UI rather than silently included.

### Stage 3 — Event Store
- Writes classified events to a SQLite database (one file per session)
- Schema: `id, timestamp, speaker, raw_text, type, character, in_game, confidence`
- Session files stored in `Documents/DMAssist/[Campaign]/sessions/`

---

## Character Registry

Stored as a JSON file per campaign. Never exposed as raw JSON to the user — managed entirely through the UI.

**Character record:**
```json
{
  "id": "uuid",
  "name": "Theron",
  "player": "Jake",
  "class": "Paladin",
  "race": "Human",
  "description": "Lawful good, serious, speaks formally",
  "aliases": ["Theron", "Sir Theron", "the paladin"],
  "archived": false
}
```

Characters are archived, never deleted — past sessions reference them by ID.

The app supports multiple campaigns, each with its own character registry and session history. Campaign switching is a top-level UI action.

---

## Summary Generator

Triggered on demand by the DM — works mid-session or after a session ends.

### Structured Event Log
Cleaned and formatted version of the raw transcript, grouped into scenes. OOC entries stripped (or shown in a collapsible sidebar). Each entry: timestamp, character name, event text.

### Narrative Summary
Claude receives the full event log + character registry and writes a prose recap in campaign journal style. Third-person, past tense.

**Prompt configuration:**
- System prompt establishes tone (campaign journal style)
- Character registry injected as context
- DM sets tone per campaign: Serious / Heroic / Humorous
- Low-confidence events presented as ambiguous, not stated as fact

**DM controls:**
- Preview before export — can regenerate or manually edit
- Summary length: Short / Medium / Detailed
- Option to exclude specific characters/players from summary

---

## User Interface

### Philosophy
Session-first. The most important action is always one click away. No jargon, no config files, no technical settings on the main screen.

### Views

**Home / Session Control**
- Large "Start Session" button, campaign name shown prominently
- Active session: live scrolling transcript, color-coded by event type (dialogue, action, combat, OOC)
- "Generate Summary" always visible during and after a session

**Characters**
- Card grid — one card per character with portrait placeholder, name, player, class/race
- "Add Character" is a simple friendly form
- Edit and archive actions on each card

**Session History**
- List of past sessions by date
- Click a session → view structured log or narrative summary
- Export button (markdown, PDF) per session

**Settings**
- Output destinations: checkboxes with plain-language labels
- Transcription Quality selector (Good / Better / Best)
- Advanced section (collapsed by default): Discord setup wizard

### Onboarding
- First launch: 3-step wizard
  1. Campaign name
  2. Add players and characters
  3. Ready — "Start your first session"
- Estimated time: "Takes about 2 minutes"

---

## Output Plugins

Each plugin implements `OutputPlugin` interface: `deliver(summary, format)`.

| Plugin | Default | Setup Required |
|--------|---------|---------------|
| Local Markdown File | Enabled | None |
| PDF Export | On-demand | None |
| Discord Channel | Disabled | Discord wizard |
| Clipboard / Email | On-demand | None |

**Local Markdown File:** Saves to `Documents/DMAssist/[Campaign]/[Date]-session.md` automatically.

**PDF Export:** Uses `puppeteer` to render a styled HTML template. Available from Session History view.

**Discord Channel:** Posts narrative summary (not raw log) to a configured channel. DM previews before posting.

**Clipboard / Email:** Copies summary to clipboard or opens default mail client with summary pre-filled.

**Future (not v1):** Google Docs, Notion, Obsidian.

---

## Data Storage

All data stored locally under `Documents/DMAssist/`:

```
Documents/DMAssist/
  campaigns/
    [campaign-id]/
      campaign.json       # campaign metadata + character registry
      sessions/
        [date]-[id].db    # SQLite session database
        [date]-[id].md    # auto-saved markdown log
  settings.json           # app-wide settings (output prefs, API keys, etc.)
```

API keys (Claude, Discord bot token) stored in OS keychain via `keytar`, not in plain files.

---

## Error Handling

- Transcription failure: event marked as `transcription_failed`, shown in UI as unreadable entry — DM can manually annotate
- Classification failure / API timeout: event stored with `unclassified` type, flagged for review
- Mic unavailable at session start: clear friendly error with troubleshooting steps, not a raw error message
- Claude API unavailable: summary generation fails gracefully with retry option — session data is never lost

---

## Technology Summary

| Concern | Technology |
|---------|-----------|
| Desktop app | Electron |
| Build tooling | electron-vite |
| Language | TypeScript |
| UI framework | React |
| Component library | shadcn/ui + Tailwind CSS |
| Discord integration | discord.js |
| Voice transcription | whisper-node (whisper.cpp) |
| AI classification/summary | Claude API (claude-sonnet-4-6) |
| Local database | SQLite via better-sqlite3 |
| PDF generation | Puppeteer |
| Audio capture | naudiodon |
| Secrets storage | keytar (OS keychain) |
| Installer | Electron Forge (NSIS / DMG) |
