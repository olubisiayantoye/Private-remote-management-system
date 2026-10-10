# Private-remote-management-system

<div align="center">

# CBT Remote Management &amp; Monitoring System

### Enterprise-grade remote administration for computer-based-testing centres — built from scratch in pure PHP, with a signed Windows agent, real-time telemetry, WebRTC screen control and a tamper-evident audit trail.

[![Version](https://img.shields.io/badge/version-1.0.0-0e7c86)](#project-status)
[![PHP](https://img.shields.io/badge/PHP-8.3%2B-777bb4?logo=php&logoColor=white)](https://www.php.net/)
[![MySQL](https://img.shields.io/badge/MySQL-8-4479a1?logo=mysql&logoColor=white)](https://www.mysql.com/)
[![.NET](https://img.shields.io/badge/.NET-10-512bd4?logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![MQTT](https://img.shields.io/badge/MQTT-Mosquitto-660066?logo=eclipsemosquitto&logoColor=white)](https://mosquitto.org/)
[![WebRTC](https://img.shields.io/badge/WebRTC-screen%20%2B%20control-333333?logo=webrtc&logoColor=white)](#what-it-does)
[![Tests](https://img.shields.io/badge/tests-268%20passing-1f7a4d)](#quality--testing)
[![Framework](https://img.shields.io/badge/framework-none%20(by%20design)-a6322f)](#engineering-highlights)

**A self-hosted platform designed for ~50 centres and 12,500+ Windows PCs.**

</div>

---

## Overview

CBT exam centres run large fleets of locked-down Windows PCs that have to be **enrolled, watched, supported and proven trustworthy** — often during live, high-stakes exams. This project is a complete, self-hosted system that does all of that from a single web portal: it enrols each PC through a signed agent, streams live health telemetry over MQTT, lets staff **screen-share and take keyboard/mouse control** over WebRTC, runs permission-gated remote commands, and records every privileged action in a **hash-chained, tamper-evident audit log**.

It was built **phase by phase, end to end** — 26 phases from empty repository to a v1.0.0 release with a deployment kit, load-test harness and disaster-recovery runbook — and deliberately **without a web framework**, to demonstrate command of the fundamentals: routing, dependency injection, an ORM-free data layer, security headers, session management and a real-time event pipeline, all hand-built and tested.

> **Why it's interesting as an engineering artifact:** it spans four runtimes that have to agree on one security model — a pure-PHP portal, a MySQL 8 data layer, a long-running PHP real-time engine (MQTT bridge + WebSocket gateway), and a C#/.NET 10 Windows service — plus a WebRTC media path and a cryptographically verifiable audit chain.

---

## What it does

| Capability | Summary |
|---|---|
| 🖥️ **Fleet enrolment** | A C#/.NET agent registers each PC; an administrator approves it. Install is driven by a single enrollment token (`install.config.json` + VBScript/PowerShell installer). |
| 📡 **Live monitoring** | Every PC heartbeats over MQTT; the portal shows online/offline, CPU/RAM/disk health and alerts in real time via a WebSocket gateway. |
| 🎥 **Screen share + remote control** | Staff start a WebRTC screen-share session and can optionally take keyboard/mouse control — gated by permission, requester identity and control mode; Ctrl-Alt-Del is not injectable. |
| ⚡ **Remote commands** | Lock, restart, shut down, request status / hardware info / heartbeat, and self-update — nonce-deduplicated, expiring, audited, and blocked in exam mode. |
| 🔄 **Signed agent self-update** | Publish a signed build, deploy `UPDATE_AGENT`; the agent verifies **SHA-256 + Authenticode publisher** before swapping and restarting itself. |
| 🧾 **Tamper-evident audit** | Append-only, hash-chained `audit_logs` (`row_hash = SHA-256(prev_hash + row)`); DB triggers reject UPDATE/DELETE; `audit:verify` detects tampering, `audit:anchor` checkpoints off-box. |
| 📊 **Reports & exports** | Session, command, fleet-inventory and availability reports, centre-scoped, with CSV export. |
| 🏫 **Exam mode** | Per-centre lockdown that blocks disruptive actions and non-consented sessions during live exams. |

---

## Architecture

```
            Internet
  browsers │                              │ agents (Windows CBT PCs)
           ▼                              ▼
 ┌───────────────────────┐        ┌───────────────────────────────────┐
 │  PHP PORTAL            │        │  REAL-TIME ENGINE                  │
 │  • Web UI + JSON APIs  │        │  • Mosquitto + mosquitto-go-auth   │
 │  • RBAC · MFA · CSP    │◀──────▶│  • mqtt:bridge  (presence/ingest)  │
 │  • MySQL 8 data layer  │   DB   │  • ws:gateway   (live dashboards)  │
 │  • Hash-chained audit  │        │  • coturn  (WebRTC TURN fallback)  │
 └───────────────────────┘        └───────────────────────────────────┘
           ▲                                   ▲
           │ HTTPS register / rotate / update  │ MQTT presence · commands · WebRTC signalling
           └───────────────── C#/.NET 10 RemoteAgent (LocalSystem service) ─────────────────┘
```

- **Browser → portal:** HTTPS, server-side sessions, CSRF-protected, MFA-gated.
- **Agent → portal:** registration and credential rotation over HTTPS; everything else over MQTT.
- **Broker → portal:** every CONNECT/SUBSCRIBE/PUBLISH is authorised by a loopback broker-auth endpoint (`TopicPolicy`), so a PC can only ever touch its own topics.
- **Live dashboards:** bridge → MQTT events → gateway → browsers over `wss://…/ws` (ticket-authenticated, no cookies forwarded).
- **Screen share / control:** signalling over MQTT; media is **peer-to-peer WebRTC**, relaying through coturn only when a direct path fails.

---

## Tech stack

**Portal** · PHP 8.3 (no framework, PSR-4), MySQL 8 (RANGE partitioning, triggers), hand-built core: router, DI container, PDO data layer, view engine, middleware pipeline.
**Real-time** · Mosquitto 2.x + [mosquitto-go-auth](https://github.com/iegomez/mosquitto-go-auth), two long-running PHP daemons (`mqtt:bridge`, `ws:gateway`), coturn for TURN/STUN.
**Media** · WebRTC (VP8) for screen share, a WebRTC data channel for input control, TURN REST (HMAC) credentials.
**Agent** · C#/.NET 10, single-file self-contained Windows service; Authenticode signing (`signtool`) + `WinVerifyTrust` publisher verification; VBScript/PowerShell installer.
**Ops** · nginx + PHP-FPM, systemd units & timers, encrypted `mysqldump` backups, dependency-free CLI (`bin/console`).

---

## Engineering highlights

- **Framework-free by design.** Routing, DI, the data layer, sessions, CSP with per-request nonces, and the real-time pipeline are all hand-built — no Laravel/Symfony — to show the fundamentals rather than hide them.
- **Defense in depth.** TOTP MFA with recovery codes and step-up re-auth; RBAC across four roles scoped per centre and per device; strict CSP (no inline scripts/styles), CSRF on every form, HSTS; per-PC MQTT credentials (PBKDF2-SHA512) rotated monthly.
- **Cryptographic integrity.** The audit log is append-only and hash-chained at the database level; tampering is mathematically detectable and anchorable off-box.
- **Supply-chain-aware updates.** Agent updates are verified by SHA-256 **and** Authenticode publisher before they are ever executed.
- **Built to scale and to prove it.** Partitioned telemetry with automated retention, a load-test harness, and a documented capacity budget (~2,500 heartbeats/s per node ingest; 12,500 PCs @ 60 s ≈ 208/s).
- **Operable from day one.** VPS deployment kit, two-host (shared hosting + AWS Free Tier) install manual, nightly encrypted backups with a restore drill, and a disaster-recovery runbook (RPO ≤ 24 h).

---

## Quick start

```bash
cp .env.example .env
php bin/console key:generate      # paste the APP_KEY line into .env
php bin/console env:check
php bin/console db:setup-sql > setup.local.sql   # run in MySQL, copy passwords to .env, delete the file
php bin/console migrate --seed --demo            # demo logins: *@demo.example.com / DemoPass!2026
php tests/run.php
php -S 127.0.0.1:8000 -t public public/index.php
```

Then open `http://127.0.0.1:8000` and `http://127.0.0.1:8000/api/health`.
Full production deployment (single VPS, or shared hosting + AWS Free Tier EC2) is in the [Installation & Production Manual](docs/DEVELOPER-MANUAL.md).

---

## Quality &amp; testing

- **268 passing tests, 0 failing** (2 skipped — MySQL-8-only partition/optimizer checks) via a **dependency-free** runner: `php tests/run.php`.
- Unit and integration suites run against a MySQL-compatible engine locally and real MySQL 8 in CI-style runs.
- The C#/.NET agent builds and tests on Windows with `agent\RemoteAgent\build.ps1` (restore → build → unit tests → signed publish → SHA-256 → smoke tests), with verified build reports recorded in [`docs/BUILD-REPORTS.md`](docs/BUILD-REPORTS.md).

---

## Documentation

| Document | What's inside |
|---|---|
| [Installation & Production Manual](docs/DEVELOPER-MANUAL.md) | End-to-end install, configuration and go-live for a single VPS or a shared-hosting + AWS split. |
| [`docs/PHASE-02…26.md`](docs/) | The full phased design record — one guide per subsystem. |
| [`SESSION-SIGNALLING.md`](docs/SESSION-SIGNALLING.md) · [`SESSION-CONTROL.md`](docs/SESSION-CONTROL.md) | WebRTC screen-share and remote-control contracts. |
| [`DEVICE-COMMANDS.md`](docs/DEVICE-COMMANDS.md) · [`AGENT-RELEASES.md`](docs/AGENT-RELEASES.md) | Remote-command and signed-release/self-update contracts. |

---

## Project status

**All 26 build phases complete — v1.0.0.** A self-hosted portal (auth, RBAC, centres, devices, enrolment), MQTT presence/heartbeats, live dashboards, WebRTC screen share + control, remote commands, a signed Windows agent with self-update, a tamper-evident audit log, reports, a load-test harness, a full VPS deployment kit, and backup/retention hardening.

<details>
<summary><strong>Phase-by-phase breakdown</strong></summary>

Authentication, RBAC & user management · centre & device management · device registration API · C#/.NET agent foundation · secure device authentication · MQTT integration · heartbeat & online/offline monitoring · admin dashboard · WebSocket gateway · real-time dashboard · remote session rules · WebRTC signalling · agent screen sharing · remote control · remote commands · installer & enrollment · signing & release (self-update) · audit log · reports & exports · load testing · VPS deployment & scheduling · backup & retention hardening.

</details>

---

## Authors

Designed and built by **Olubisi** and **Adebamigbe**.

| Author | Role | Links |
|---|---|---|
| **Olubisi** | _add role, e.g. Backend / Platform_ | `GitHub` · `LinkedIn` · `email` |
| **Adebamigbe** | _add role, e.g. Agent / Frontend_ | `GitHub` · `LinkedIn` · `email` |

> _Fill in the roles and links above — the placeholders are ready for your GitHub handles, LinkedIn profiles and contact emails._

---

## License

This is a private, self-hosted project. All rights reserved unless a `LICENSE` file states otherwise. Please contact the authors before reuse or redistribution.
