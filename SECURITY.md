# security

this is a children's product. security is not optional.

## reporting vulnerabilities

do not open a public issue. email **w2927864@gmail.com** with:

- what you found
- steps to reproduce
- potential impact (especially around child safety)
- suggested fix if you have one

we'll respond within 48 hours and work with you on a fix before any public disclosure.

## what counts

**in scope:**
- content safety bypass (making the skill generate inappropriate content)
- sandbox escape (generated HTML accessing the parent frame or OS)
- credential exposure (API keys, PINs, or tokens leaking)
- parental control bypass (getting around PIN, screen time, bedtime, or safety levels)

**out of scope:**
- vulnerabilities in upstream dependencies (Electron, Node.js, Pi OS) should go to those projects
- physical access attacks (root on the Pi means game over regardless)
- social engineering the parent into sharing their PIN

## the security model

full details in [docs/architecture.md](docs/architecture.md).

the short version:

1. three layers of content safety (client filter, skill rules, iframe sandbox)
2. dedicated system user with minimal permissions
3. systemd hardening (ProtectSystem, NoNewPrivileges, read-only filesystem)
4. generated content runs in `iframe sandbox="allow-scripts"` only. no allow-same-origin.
5. parent PIN hashed with scrypt. gateway tokens randomized per device.
6. everything runs on localhost. no external network exposure.

## supported versions

| version | supported |
|---------|-----------|
| 0.1.x (beta) | yes |
