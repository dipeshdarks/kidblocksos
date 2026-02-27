# KidBlocksOS

> A children's creative tablet OS powered by AI — built on Raspberry Pi and OpenClaw.

⚠️ **Status: Beta (v0.1.0)** — Proof of concept. Not production-ready.

---

## What Is This?

KidBlocksOS turns a Raspberry Pi 5 with a 7" touchscreen into an AI-powered creative tablet for kids ages 5-10.

A child says what they want — *"a penguin game where I jump on ice"* or *"teach me about the solar system"* — and the system builds it. In seconds, they have a working interactive app they can play, modify, and learn from.

No app store. No downloads. Just imagination → creation.

### How It Works

```
Kid says "make a maze with a cat"
         │
         ▼
┌─────────────────────┐
│  KidBlocksOS Shell   │  Electron kiosk on Raspberry Pi
│  (touch UI, studios) │  Content filter, templates, voice input
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Template Engine     │  13 built-in patterns (platformer, maze, piano...)
│  (instant, offline)  │  Works without internet
└─────────┬───────────┘
          │ If template can't handle it...
          ▼
┌─────────────────────┐
│  Imagination Engine  │  OpenClaw agent with kidblocks-engine skill
│  (AI generation)     │  Generates custom HTML apps from descriptions
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Sandboxed iframe    │  allow-scripts only — no DOM escape
│  (safe execution)    │  Kid plays the creation
└─────────────────────┘
```

## What's Open Source Here

This repo contains:

- **`skill/kidblocks-engine/SKILL.md`** — The Imagination Engine skill that powers AI generation
- **`docs/`** — Full documentation on the architecture, how to build your own, and how the pieces fit together

This repo does **not** contain:
- The KidBlocksOS Electron shell
- The pre-built OS image
- OpenClaw itself (see [openclaw.ai](https://openclaw.ai))

### Why?

The skill is the brain. We're open-sourcing it because:
1. Anyone with OpenClaw can use it to build kid-safe interactive apps
2. Educators and parents can inspect exactly what the AI is told to do
3. The community can improve the patterns, add templates, and tighten safety rules
4. It demonstrates how an OpenClaw skill can power an entire product

The OS shell is a specific hardware integration (Pi + touchscreen + kiosk mode). The skill works anywhere OpenClaw runs.

## The Imagination Engine Skill

The skill at [`skill/kidblocks-engine/SKILL.md`](skill/kidblocks-engine/SKILL.md) teaches an AI agent to:

- Generate complete, self-contained HTML5 apps from natural language
- Follow **6 creative studios**: Games, Stories, Music, Art, Science, Tinker
- Use **40+ patterns** across those studios (platformer, maze, piano, solar system, etc.)
- Enforce **content safety** — violence → "help friends", guns → "water balloons"
- Adapt difficulty by age (5-6, 7-8, 9-10)
- Target a **7" touchscreen** with big tap targets and no tiny text
- Generate all audio with **Web Audio API** (no external assets)
- Output structured JSON with visual programming logic (KidBlocks)

### Using the Skill

#### With OpenClaw (recommended)

Install the skill into any OpenClaw agent:

```bash
# Copy into your OpenClaw workspace
mkdir -p ~/.openclaw/workspace/skills/kidblocks-engine
cp skill/kidblocks-engine/SKILL.md ~/.openclaw/workspace/skills/kidblocks-engine/
```

Then tell your agent:

> "Read the kidblocks-engine skill. Make me a game where a fox collects gems in a forest."

The agent reads the skill, follows the patterns, and returns a JSON object with `html` and `logic` fields.

#### With Any LLM

The skill is a markdown file. You can paste it as a system prompt into any LLM:

```
System: [contents of SKILL.md]
User: Make a piano app for a 7 year old
```

The model will follow the patterns and output structured JSON.

## Architecture Overview

See [docs/architecture.md](docs/architecture.md) for the full technical breakdown.

### Components

| Component | Description | Location |
|-----------|-------------|----------|
| **KidBlocksOS Shell** | Electron kiosk app with studios, wizard, settings | Closed source |
| **Template Engine** | 13 built-in offline templates | Closed source |
| **Imagination Engine** | OpenClaw agent + kidblocks-engine skill | **This repo (skill)** |
| **Content Safety** | Client-side filter + AI-level safety rules | Skill is open, filter is closed |
| **Guardian Channel** | Parental activity monitoring (XMTP planned) | Closed source |
| **TimeKeeper** | Screen time, break reminders, bedtime enforcement | Closed source |
| **KidBlocks** | Visual programming layer showing app logic | Closed source |

### System Requirements (for the full OS)

- Raspberry Pi 5 (4GB+ RAM)
- 7" official touchscreen (1024x600)
- 32GB+ SD card
- WiFi for AI features (templates work offline)
- Anthropic API key (for Imagination Engine)

### The OpenClaw Agent

KidBlocksOS runs its own dedicated OpenClaw agent — separate from any other agent on the system. This agent:

- Has a single skill: `kidblocks-engine`
- Runs a local gateway on port 8089 (localhost only)
- Receives structured prompts from the Electron shell via Chat Completions API
- Returns JSON with HTML + visual logic
- Has no access to the internet, no tools, no file system — just the skill and the model

The agent is configured during the first-boot setup wizard, where the parent provides an Anthropic API key.

## Security Model

### Content Safety (Three Layers)

1. **Client-side filter** — Regex patterns block violent/sexual/inappropriate terms before they reach the AI. Blocked inputs get gentle redirects ("How about a superhero who SAVES everyone?")

2. **Skill-level safety** — The AI prompt explicitly lists banned categories and provides safe reinterpretations. "Kill enemies" → "help friends". "Gun game" → "water balloon launcher".

3. **Sandbox execution** — Generated HTML runs in an iframe with `sandbox="allow-scripts"` only. No `allow-same-origin` — the generated app cannot access the parent DOM, IPC bridge, or any OS functionality.

### Parental Controls

- **PIN-protected settings** (hashed, not plaintext)
- **Three safety levels**: Strict (templates only), Standard (AI + filtering), Creative (more freedom, 9+)
- **Screen time limits** with break reminders
- **Bedtime enforcement** — device locks at scheduled time
- **Activity log** — every AI interaction logged for parent review
- **Guardian Report** — viewable in settings behind PIN

### System Hardening

- Dedicated `kidblocks` user with minimal permissions
- SystemD hardening: `ProtectSystem=strict`, `NoNewPrivileges`, `ProtectKernelTunables`
- Gateway bound to `127.0.0.1` only
- Random gateway auth token per device
- Content Security Policy on Electron window

## Templates (Built-in, Offline)

The OS includes 13 pre-built templates that work without AI or internet:

| Template | Studio | Description |
|----------|--------|-------------|
| Platformer | Games | Side-scrolling with emoji characters |
| Catcher | Games | Catch falling items |
| Maze | Games | Generated maze with touch controls |
| Whack-a-mole | Games | Tap targets before they disappear |
| Pong | Games | Classic paddle game |
| Memory | Games | Card matching |
| Drawing | Art | Freehand canvas with brushes/colors |
| Fireworks | Art | Tap to create particle explosions |
| Color Mixer | Art/Science | RGB sliders, learn color theory |
| Beat Maker | Music | Step sequencer with 5 instruments |
| Piano | Music | Playable keyboard |
| Solar System | Science | Orbiting planets with facts |
| Story | Stories | Branching narrative with choices |

Templates are pattern-matched from the child's input. AI is only invoked when no template matches.

## Contributing

We welcome contributions to the Imagination Engine skill:

- **New patterns** — Add studio patterns the AI can follow
- **Better safety rules** — Improve content filtering and reinterpretation
- **Age adaptation** — Better difficulty scaling per age group
- **Localization** — Translate prompts and safety rules
- **Performance** — Optimize generated HTML for Pi hardware

### How to Test Changes

1. Edit `skill/kidblocks-engine/SKILL.md`
2. Paste the skill as a system prompt to any LLM
3. Send test prompts ("make a dinosaur game for a 6 year old")
4. Verify the output is valid JSON with working HTML
5. Test the HTML in a browser at 1024x600 resolution

## Roadmap

- [ ] Guardian Channel via XMTP (parent notifications)
- [ ] More studio patterns (coding studio, robotics studio)
- [ ] Skill marketplace (community-contributed studios)
- [ ] Multi-language skill variants
- [ ] OTA updates
- [ ] Accessibility (screen reader, high contrast, switch access)

## License

The Imagination Engine skill is released under **MIT License**.

KidBlocksOS (the shell, templates, and image) is proprietary.

---

Built with 🧱 by the KidBlocksOS team. Powered by [OpenClaw](https://openclaw.ai).
