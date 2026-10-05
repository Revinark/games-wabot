# AGENTS.md

## Project Overview
Games-Wabot — a customizable WhatsApp bot built with Node.js and `@adiwajshing/baileys` (v3.5.3, legacy WhatsApp Web API). When run with `--server`, it starts an Express web server on port 3000 that displays a QR code for WhatsApp authentication.

## Setup Quirks (Non-Obvious)

### Node Version
- Requires Node 14.x (per `package.json` engines). Uses `node:14-buster` Docker image.
- Debian Buster repos are EOL — the Dockerfile patches apt sources to `archive.debian.org`.

### Baileys Build
- `@adiwajshing/baileys` is installed from `github:BochilGaming/Baileys#fix` (a fork).
- npm 6 (bundled with Node 14) does NOT run the `prepare` script (`tsc`) properly for git deps — TypeScript (a devDependency) isn't available at prepare time, and the `files` field in baileys' package.json excludes `src/`, so the compiled `lib/` directory is never created.
- **Workaround**: The compose startup clones the Baileys repo separately, copies `src/` + `tsconfig.json` into the installed package, and runs `tsc --skipLibCheck --noEmitOnError false || true` (TypeScript 5 is installed globally in the image). The `|| true` is needed because the Baileys source has minor type errors under TS5 that don't prevent emission.

### re2 Native Module
- `url-regex-safe` pulls in `re2` as a transitive dependency. The latest `re2` requires Node 22+ and fails to build on Node 14.
- **Workaround**: `npm install --ignore-scripts` skips the re2 native build. re2 is optional — `url-regex-safe` falls back to native RegExp.

### git:// Protocol
- GitHub deprecated the `git://` protocol. The Dockerfile configures `git config --global url."https://github.com/".insteadOf git://github.com/` so npm can resolve GitHub dependencies.

### WhatsApp Connection
- The legacy WhatsApp Web API (used by Baileys v3.5.3) returns 404 — WhatsApp has discontinued it. The bot's web UI (QR code page) still loads, but WhatsApp authentication will not succeed with this old protocol version. This is a known limitation of the upstream project, not a setup issue.

## Verification
- `curl http://localhost:3000/` returns 200 with the QR code web page.
- Container healthcheck: `node -e "require('http').get('http://localhost:3000/', r => process.exit(r.statusCode === 200 ? 0 : 1))"`
- Start command: `docker compose -f docker-compose.base44.yml up -d`
