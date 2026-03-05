# changelog

## [0.2.0-beta.1] - 2026-03-04

marketplace and device onboarding.

### added
- marketplace watcher daemon (systemd service, auto-restart)
  - polls XMTP for new device DMs every 10 seconds
  - auto-adds devices to marketplace group
  - delivers full catalog via encrypted DM reply
- device wallet generation during setup wizard (self-custody, key never leaves device)
- XMTP registration and marketplace join during setup wizard
  - runs as child process (system Node.js 22, not Electron's bundled Node 20)
  - progress bar with 5 steps visible to parent
- encrypted catalog delivery via DM (solves XMTP E2E encryption limitation for new members)
- Lumen voice guide (ElevenLabs TTS for every wizard step)
- WiFi signal strength and volume control widgets in setup wizard
- 6 marketplace listings: Cookie Catcher, Space Dinosaur, Rainbow Painter, Ocean Adventure, Beat Maker, Volcano Lab

### changed
- setup wizard expanded from 12 to 14 steps (added wallet + marketplace)
- marketplace architecture: catalog delivered via DM reply instead of group message sync
- all XMTP operations run as child processes using system Node.js runtime

### fixed
- Electron Node.js ABI mismatch with XMTP native bindings (child process workaround)
- XMTP SDK v5.4.0 API: Client.create takes 2 args not 3
- WiFi connect freeze (execSync to async execFile)
- double tap sounds on kid keyboard
- DNS resolution on fresh devices (NetworkManager generating empty resolv.conf)

### documentation
- updated architecture with wallet, XMTP, and watcher details
- updated marketplace docs with DM-based catalog delivery and watcher architecture
- updated XMTP docs with SDK v5.4.0 API notes and child process architecture
- added encryption key derivation details

## [0.1.0-beta.1] - 2026-02-27

first public release.

### added
- imagination engine skill (kidblocks-engine/SKILL.md)
  - six studios: games, stories, music, art, science, tinker
  - 40+ generation patterns
  - content safety with reinterpretation
  - age-adaptive output (5-6, 7-8, 9-10)
  - touch-first design requirements
  - Web Audio API sound patterns
  - performance constraints for Raspberry Pi
  - structured JSON output (html + visual logic)
- architecture documentation
  - full system stack
  - boot sequence
  - creation pipeline
  - data model
  - OpenClaw integration
  - security model and threat analysis
- skill usage guide
  - OpenClaw setup
  - raw API usage with Python example
  - paste-and-go usage
  - input/output format reference
  - testing guide

### security
- three layer content safety (client filter, skill rules, iframe sandbox)
- iframe sandbox: allow-scripts only (no allow-same-origin)
- parent PIN hashed with scrypt
- random gateway token per device
- systemd hardening
