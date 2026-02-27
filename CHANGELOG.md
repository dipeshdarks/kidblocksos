# changelog

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
