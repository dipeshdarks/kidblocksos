# KidBlocksOS Architecture

> Technical deep dive into how KidBlocksOS works, from boot to creation.

## Overview

KidBlocksOS is a single-purpose kiosk OS built on Raspberry Pi OS Lite. It boots directly into a full-screen Electron app running inside the Cage Wayland compositor. There is no desktop environment — the child sees only KidBlocksOS.

```
┌─────────────────────────────────────────────────┐
│                 Raspberry Pi 5                   │
├─────────────────────────────────────────────────┤
│  Raspberry Pi OS Lite (Bookworm, arm64)          │
│  ├── systemd: kidblocks.target                   │
│  ├── kidblocks-kiosk.service (Cage + Electron)   │
│  └── kidblocks-gateway.service (OpenClaw)        │
├─────────────────────────────────────────────────┤
│  Cage Compositor (Wayland)                       │
│  └── Electron (kiosk mode, no sandbox*)          │
│      ├── Main Process (Node.js)                  │
│      │   ├── IPC handlers (projects, config...)  │
│      │   ├── OpenClaw gateway client             │
│      │   ├── TTS (ElevenLabs API, cached)        │
│      │   └── System (WiFi, info)                 │
│      └── Renderer (Chromium)                     │
│          ├── OS UI (lock, home, studios, wizard) │
│          ├── Template Engine (13 patterns)        │
│          ├── Content Safety Filter               │
│          └── Sandboxed iframe (generated apps)   │
├─────────────────────────────────────────────────┤
│  OpenClaw Gateway (port 8089, localhost)          │
│  └── Agent: Imagination Engine                   │
│      └── Skill: kidblocks-engine                 │
└─────────────────────────────────────────────────┘
```

*\*`--no-sandbox` is required on Pi with Cage. Mitigated by systemd hardening and dedicated user account.*

## Boot Sequence

### Fresh Install (First Boot)

```
Power on
  → Pi OS boots to multi-user.target
  → kidblocks-firstboot.service runs
    → Installs system packages (cage, fonts, audio)
    → Creates kidblocks user
    → Installs Node.js 22
    → Installs Electron (npm install)
    → Installs OpenClaw (npm install -g)
    → Configures systemd services
    → Sets kidblocks.target as default
    → Self-deletes the setup script
    → Reboots
```

### Normal Boot (After Setup)

```
Power on
  → Pi OS boots to kidblocks.target
  → kidblocks-gateway.service starts
    → OpenClaw gateway on port 8089
    → Loads kidblocks-engine skill
  → kidblocks-kiosk.service starts
    → Cage compositor on tty7
    → Electron in kiosk mode
    → Lock screen appears
```

### First User Session (Setup Wizard)

When the child taps the lock screen for the first time, if no profile exists, the setup wizard runs:

```
Parent Zone (8 steps):
  1. "Hand this to a grown-up"
  2. Language selection (10 languages)
  3. Timezone
  4. WiFi configuration
  5. Screen time limits + bedtime
  6. Content safety level (strict/standard/creative)
  7. Parent PIN (4-digit, confirmed, hashed with scrypt)
  8. Anthropic API key (starts OpenClaw gateway)

Kid Zone (4 steps):
  9. "Hand this to your kid!"
  10. Name
  11. Age (5-10)
  12. Avatar (emoji buddy)
```

## Creation Flow

When a child asks to build something, here's the full pipeline:

```
Child: "make a penguin game where I slide on ice"
  │
  ├── Content Safety Filter (client-side regex)
  │   └── Blocked? → Gentle redirect ("How about...")
  │   └── Passed? ↓
  │
  ├── Template Matching (engine.js: matchIntent)
  │   └── Matched? → Generate from template (instant, offline)
  │   └── No match + AI healthy? ↓
  │
  ├── AI Generation (Imagination Engine)
  │   ├── Electron main process sends to OpenClaw gateway
  │   │   POST http://127.0.0.1:8089/v1/chat/completions
  │   │   {
  │   │     model: "openclaw:main",
  │   │     messages: [
  │   │       { role: "system", content: "Read kidblocks-engine skill..." },
  │   │       { role: "user", content: '{"description":"penguin sliding...","studio":"games","age":7}' }
  │   │     ]
  │   │   }
  │   ├── OpenClaw agent reads SKILL.md
  │   ├── Agent generates JSON: { html: "...", logic: {...} }
  │   └── Response parsed, validated
  │
  ├── App Launch
  │   ├── HTML loaded into sandboxed iframe (allow-scripts only)
  │   ├── Logic loaded into KidBlocks drawer (visual programming view)
  │   ├── Activity logged (studio, input, template/ai, timestamp)
  │   └── Creation can be saved, modified, or shared
  │
  └── No AI + No template?
      └── "That one needs the AI brain! Try connecting to WiFi."
```

## Data Model

### Projects (saved creations)

```
/opt/kidblocks/data/projects/{id}/
├── manifest.json    # name, studio, template, icon, timestamps
├── app.html         # the generated HTML app
└── logic.json       # KidBlocks visual programming data
```

### Configuration

```
/opt/kidblocks/data/config/
├── profile.json     # child's name, age, avatar
├── parent.json      # PIN hash + salt (scrypt)
├── device.json      # language, timezone, device name
├── timekeeper.json  # daily limit, break interval, bedtime, wake time
├── safety.json      # level (strict/standard/relaxed), blocked topics
├── engine.json      # API key (encrypted at rest planned), gateway token
├── tts.json         # ElevenLabs API key, voice preference
├── wifi.json        # SSID, PSK
├── guardian.json    # Guardian channel config (future: XMTP)
└── usage-YYYY-MM-DD.json  # daily screen time tracking
```

### Activity Logs

```
/opt/kidblocks/data/logs/
├── activity-YYYY-MM-DD.jsonl   # all events (creation, voice, session)
├── first-boot.log              # first boot setup output
└── electron.log                # Electron stderr
```

## OpenClaw Integration

KidBlocksOS embeds its own OpenClaw instance. This is **not** the host's OpenClaw — it's a separate agent with its own:

| Property | Value |
|----------|-------|
| Config home | `/opt/kidblocks/openclaw-config` |
| Gateway port | 8089 |
| Gateway bind | 127.0.0.1 only |
| Auth | Random token per device |
| Agent | `main` (single agent) |
| Skill | `kidblocks-engine` |
| Model | Set by parent (Anthropic API key) |
| Workspace | `/opt/kidblocks/openclaw-config/workspace` |

### Chat Completions API

The Electron main process talks to the OpenClaw gateway via the standard Chat Completions API:

```
POST /v1/chat/completions
Authorization: Bearer {random-gateway-token}
Content-Type: application/json

{
  "model": "openclaw:main",
  "user": "kidblocks-engine",
  "messages": [
    {
      "role": "system",
      "content": "You are the KidBlocksOS Imagination Engine. Read skills/kidblocks-engine/SKILL.md..."
    },
    {
      "role": "user",
      "content": "{\"description\":\"...\",\"studio\":\"games\",\"age\":7,\"safety\":\"standard\"}"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 8000
}
```

The agent receives this, reads the skill, generates the app, and returns JSON in the standard Chat Completions response format.

## Security Architecture

### Threat Model

The primary threats for a children's device:

1. **Inappropriate content** — child inputs something the AI shouldn't generate
2. **App sandbox escape** — generated HTML tries to access the OS
3. **Credential exposure** — API keys leaked or accessible
4. **Physical access** — someone plugs in a keyboard or SSHes in
5. **Network attacks** — malicious WiFi or MITM

### Mitigations

| Threat | Mitigation |
|--------|-----------|
| Inappropriate content | 3-layer safety: regex filter → skill-level rules → sandbox |
| Sandbox escape | iframe `sandbox="allow-scripts"` (no `allow-same-origin`) |
| Credential exposure | Hashed PIN (scrypt), random gateway token, filesystem permissions |
| Physical access | PIN-gated settings, kiosk mode blocks keyboard shortcuts |
| Network attacks | Gateway on localhost only, Electron blocks navigation |

### Electron Security

```javascript
// Context isolation — renderer can't access Node.js
contextIsolation: true
nodeIntegration: false

// No navigation — can't go to external URLs
mainWindow.webContents.on('will-navigate', (e) => e.preventDefault());
mainWindow.webContents.setWindowOpenHandler(() => ({ action: 'deny' }));

// CSP header
"default-src 'self' 'unsafe-inline' 'unsafe-eval' data: blob: file:;
 connect-src 'self' http://127.0.0.1:* https://api.elevenlabs.io"

// IPC — only exposed via contextBridge preload
// Renderer has no direct access to fs, child_process, or network
```

### SystemD Hardening

```ini
# kidblocks-kiosk.service
ProtectSystem=strict          # /usr, /boot, /etc are read-only
ReadWritePaths=/opt/kidblocks/data  # only data dir is writable
ProtectHome=yes               # /home is invisible
NoNewPrivileges=yes           # can't gain privileges via setuid
ProtectKernelTunables=yes     # can't modify /proc/sys
ProtectKernelModules=yes      # can't load kernel modules
ProtectControlGroups=yes      # can't modify cgroups
```

## Template Engine

The template engine uses pattern matching to convert natural language into structured parameters, then renders pre-built HTML templates.

### Intent Matching (engine.js)

```
Input: "penguin sliding on ice"
  │
  ├── Check studio context (games? art? music?)
  ├── Regex match: /penguin/ → emoji 🐧
  ├── No specific game pattern match
  └── Default: platformer template
      params: { emoji: '🐧', itemEmoji: '⭐', speed: 5 }
```

### Template → HTML

Each template is a JavaScript function that takes parameters and returns a complete HTML string:

```javascript
TEMPLATES.platformer = (params) => `<!DOCTYPE html>...
  // Uses params.emoji for player character
  // Uses params.speed for movement speed
  // Uses params.itemEmoji for collectibles
...`;
```

### Logic Generation

Each template also has a logic generator that produces the KidBlocks visual programming data:

```javascript
LOGIC_GEN.platformer = (params) => ({
  things: [
    { name: 'Player', icon: '🐧', props: [{ label: 'Speed', type: 'slider', value: 5 }] }
  ],
  rules: [
    { when: 'Player touches a star', then: 'Score goes up by 1' }
  ]
});
```

## KidBlocks Visual Programming

The "Inside" drawer shows kids how their creation works using a simple visual language:

- **Things** — Characters, objects, and their adjustable properties (sliders, toggles)
- **Rules** — When/then pairs in plain English ("When cat catches fish → Score +1")

Sliders in the Things section send `postMessage` events to the app iframe, allowing real-time parameter changes.

## Voice Pipeline

```
Tap microphone → Web Speech API (SpeechRecognition)
  → Transcript → same pipeline as text input
  → Respects device language setting (10 languages)
```

TTS output (spoken responses to the child):
```
Text → ElevenLabs API (if configured)
  → Cache to disk (MD5 hash of text + voice)
  → Play cached file
  
Fallback: Web Speech API SpeechSynthesis
```

## Screen Time System (TimeKeeper)

```
Session start → tick every 30s
  → Check daily limit (configurable, default 60 min)
     → 5 min warning → 2 min warning → time's up → lock screen
  → Check break interval (configurable, default 20 min)
     → "Time for a stretch!" reminder
  → Check bedtime (configurable, default 23:00)
     → Bedtime mode: dark lock screen, all interaction disabled
  → Save usage to config/usage-YYYY-MM-DD.json
```

## Future: Guardian Channel

Planned XMTP-based parental notification channel:

```
KidBlocksOS device wallet ←→ XMTP ←→ Parent wallet
  
Messages:
  - creation: what the child built
  - content_flag: blocked input
  - screen_time: usage milestone
  - bedtime: enforcement triggered
  - daily_summary: end-of-day report
  
Parent commands (future):
  - extend_time: add 15 minutes
  - lock_now: immediate lock
  - adjust_safety: change safety level
```

Currently stubbed — logs messages locally, XMTP transport not yet integrated.
