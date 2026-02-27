# architecture

how KidBlocksOS works from power on to creation.

## the stack

```
Raspberry Pi
├── Raspberry Pi OS Lite (Bookworm, arm64)
│   └── kidblocks.target (custom systemd boot target)
│
├── Cage (Wayland compositor)
│   └── Electron (kiosk mode)
│       ├── main process (node.js)
│       │   ├── IPC handlers (projects, config, TTS, wifi)
│       │   └── gateway client (talks to OpenClaw)
│       └── renderer (chromium)
│           ├── OS UI (lock screen, home, studios, wizard, settings)
│           ├── template engine (13 patterns)
│           ├── content safety filter
│           └── sandboxed iframe (where generated apps run)
│
└── OpenClaw gateway (localhost:8089)
    └── imagination engine agent
        └── kidblocks-engine skill
```

there's no desktop environment. the Pi boots directly into the kiosk. the only thing on screen is KidBlocksOS.

## boot sequence

### first boot (fresh install)

```
power on
  -> Pi OS boots to multi-user.target
  -> kidblocks-firstboot.service kicks in
    -> installs system packages (cage, fonts, audio libs)
    -> creates a dedicated kidblocks user
    -> installs Node.js 22
    -> runs npm install for Electron
    -> installs OpenClaw globally
    -> drops in systemd service files
    -> sets kidblocks.target as default boot
    -> deletes itself
    -> reboots
```

### normal boot (after setup)

```
power on
  -> Pi OS boots to kidblocks.target
  -> kidblocks-gateway.service starts
    -> OpenClaw agent comes online at port 8089
    -> loads the kidblocks-engine skill
  -> kidblocks-kiosk.service starts
    -> cage compositor grabs tty7
    -> electron launches in kiosk mode
    -> lock screen shows up
```

### first user session (setup wizard)

when the kid taps the lock screen and no profile exists yet, the wizard runs. two phases:

**parent zone (8 steps):**
1. "hand this to a grown-up"
2. language (10 options)
3. timezone
4. wifi setup
5. screen time limits and bedtime
6. content safety level
7. parent pin (4 digit, confirmed, hashed)
8. API key for the imagination engine

**kid zone (4 steps):**
9. "now hand this to your kid!"
10. what's your name?
11. how old are you? (5 through 10)
12. pick your buddy (emoji avatar)

## the creation pipeline

what happens when a kid says "make a penguin game where I slide on ice":

```
"make a penguin game where I slide on ice"
  |
  +-- content safety filter (client-side regex)
  |   blocked? -> gentle redirect ("how about a superhero who saves everyone?")
  |   passed? -> continue
  |
  +-- template matching (engine.js matchIntent function)
  |   matched? -> generate HTML from template (instant, offline)
  |   no match + AI healthy? -> continue
  |   no match + AI down? -> "that one needs the AI brain! try connecting to wifi"
  |
  +-- AI generation
  |   electron main process POSTs to OpenClaw gateway:
  |
  |   POST http://127.0.0.1:8089/v1/chat/completions
  |   {
  |     model: "openclaw:main",
  |     messages: [
  |       { role: "system", content: "read kidblocks-engine skill..." },
  |       { role: "user", content: '{"description":"penguin sliding...","studio":"games","age":7}' }
  |     ]
  |   }
  |
  |   agent reads SKILL.md, generates JSON with html + logic fields
  |
  +-- launch
      html goes into sandboxed iframe (allow-scripts only)
      logic goes into the KidBlocks visual programming drawer
      activity gets logged
      creation can be saved, modified, or rebuilt
```

## data layout

### projects (saved creations)

```
/opt/kidblocks/data/projects/{id}/
├── manifest.json    # name, studio, template, icon, timestamps
├── app.html         # the generated HTML
└── logic.json       # visual programming data (things + rules)
```

### configuration

```
/opt/kidblocks/data/config/
├── profile.json     # child's name, age, avatar
├── parent.json      # pin hash and salt (scrypt)
├── device.json      # language, timezone
├── timekeeper.json  # daily limit, break interval, bedtime, wake
├── safety.json      # level, blocked topics
├── engine.json      # gateway token, configured flag
├── tts.json         # ElevenLabs key, voice choice
├── wifi.json        # ssid, password
└── usage-YYYY-MM-DD.json  # daily screen time tracking
```

### logs

```
/opt/kidblocks/data/logs/
├── activity-YYYY-MM-DD.jsonl   # everything (creations, voice, sessions)
├── first-boot.log              # first boot setup output
└── electron.log                # electron stderr
```

## the OpenClaw agent

KidBlocksOS runs its own OpenClaw instance. this is not the host system's agent. it's a separate thing with its own config, workspace, and purpose.

| property | value |
|----------|-------|
| config | /opt/kidblocks/openclaw-config |
| port | 8089 |
| binding | 127.0.0.1 only |
| auth | random token generated per device |
| agent | single agent ("main") |
| skill | kidblocks-engine |
| model | set by parent during setup |
| workspace | /opt/kidblocks/openclaw-config/workspace |

the electron app talks to it through the standard Chat Completions API. the agent reads the skill, follows the patterns, and returns structured JSON. it has no tools, no file access, no internet. just the skill and the model.

## security model

### who we're protecting against

this is a children's device. the threats that matter:

1. **bad content** - kid types something the AI shouldn't generate
2. **sandbox escape** - generated HTML tries to reach the OS
3. **credential theft** - API keys or pins get exposed
4. **physical access** - someone plugs in a keyboard or SSHes in
5. **network attacks** - malicious wifi, MITM

### how each is handled

| threat | mitigation |
|--------|-----------|
| bad content | three layer filter: regex -> skill rules -> sandbox |
| sandbox escape | iframe sandbox="allow-scripts" (no allow-same-origin) |
| credential theft | hashed pin (scrypt), random gateway token, file permissions |
| physical access | kiosk blocks shortcuts, pin gates settings |
| network attacks | gateway on localhost only, electron blocks navigation |

### electron hardening

```javascript
// renderer can't touch node.js
contextIsolation: true
nodeIntegration: false

// can't navigate away
mainWindow.webContents.on('will-navigate', (e) => e.preventDefault());
mainWindow.webContents.setWindowOpenHandler(() => ({ action: 'deny' }));

// content security policy
"default-src 'self' 'unsafe-inline' 'unsafe-eval' data: blob: file:;
 connect-src 'self' http://127.0.0.1:* https://api.elevenlabs.io"
```

### systemd hardening

```ini
ProtectSystem=strict          # filesystem is read-only except data dir
ReadWritePaths=/opt/kidblocks/data
ProtectHome=yes               # /home is invisible
NoNewPrivileges=yes           # can't escalate
ProtectKernelTunables=yes     # can't touch /proc/sys
ProtectKernelModules=yes      # can't load modules
ProtectControlGroups=yes      # can't modify cgroups
```

## template engine

13 built-in templates handle common requests without touching the AI.

the matching works through regex on the input text. "penguin sliding on ice" matches the platformer template because it detects a character (penguin emoji) plus a game-like context. the template function takes the extracted parameters and returns a complete HTML page.

```
input: "catch falling pizza"
  -> regex: /catch|fall|basket|collect/ -> catcher template
  -> regex: /pizza/ -> items = pizza emoji set
  -> TEMPLATES.catcher({ catcherEmoji: '🧺', items: '🍕,🍕,🧀,🍅,🫓', speed: 3 })
  -> complete HTML5 game
```

each template also has a logic generator that produces the visual programming view:

```javascript
LOGIC_GEN.catcher = (params) => ({
  things: [
    { name: 'Basket', icon: '🧺', props: [{ label: 'Speed', type: 'slider' }] }
  ],
  rules: [
    { when: 'item falls into basket', then: 'score goes up' },
    { when: 'item hits the ground', then: 'missed +1' }
  ]
});
```

## KidBlocks visual programming

the "Inside" button on any creation opens a drawer that shows kids how their app works. it uses two concepts:

**things** - characters, objects, and their adjustable properties. sliders change values in real time by sending postMessage to the iframe.

**rules** - when/then pairs written in plain language. "when the cat catches a fish, score goes up by 1."

this is not drag-and-drop coding. it's a transparency layer. kids see the logic behind what they built and can tweak properties without understanding code.

## voice

input: Web Speech API (SpeechRecognition). respects the language setting from the wizard.

output: ElevenLabs API if configured (cached to disk by content hash). falls back to Web Speech API SpeechSynthesis.

## screen time (TimeKeeper)

ticks every 30 seconds once a session starts:

- checks daily limit (warns at 5 min and 2 min remaining, locks at zero)
- checks break interval (nudges the kid to stretch)
- checks bedtime (locks the entire device with a dark sleep screen)
- saves usage to disk per day

## guardian channel (planned)

not implemented yet. the plan is XMTP-based parental notifications:

- what the child built
- content flags (blocked inputs)
- screen time milestones
- bedtime enforcement triggers
- daily summary

parent commands coming later: extend time, lock now, adjust safety level remotely.

currently everything logs locally and is viewable in the settings guardian report.
