# Genesis Bundle

Complete deliverable for bootstrapping Genesis — Fareed's personal AI assistant platform — on a fresh Ubuntu Server 24.04 P14s Gen 2.

**Target audience:** Claude Code (running on X1 Carbon) executing the bootstrap on the P14s.

---

## What's in this bundle

```
genesis-bundle/
├── README.md                              ← this file (start here)
├── GENESIS_System_Design_v0.5.5.md        ← the architecture (2,675 lines)
├── GENESIS_Bootstrap_Brief.md             ← build instructions with checkpoints
│
└── genesis-agent-package/
    ├── README.md                          ← how to deploy the agent files
    │
    ├── agents/                            ← personalized agent identities
    │   ├── isaac/CLAUDE.md
    │   ├── oracle/CLAUDE.md
    │   ├── iris/CLAUDE.md
    │   ├── sentinel/CLAUDE.md
    │   └── halo/README.md
    │
    ├── diagrams/                          ← visual architecture (Mermaid)
    │   ├── 01-system-architecture.mmd
    │   ├── 02-agent-topology.mmd
    │   ├── 03-request-flow.mmd
    │   └── 04-deployment.mmd
    │
    ├── skills/                            ← example recurring-task skills
    │   ├── iris-morning-briefing.md
    │   ├── oracle-weekly-kpi-report.md
    │   └── isaac-refactor-task.md
    │
    └── templates/                         ← memory file templates
        ├── project-log-template.md
        └── reflection-template.md
```

**Total files:** 17 (plus this README)

---

## How to start

### 1. Read the architecture first

`GENESIS_System_Design_v0.5.5.md` is the source of truth for what Genesis is and how it's designed. Read it before doing anything else. It covers:

- System overview and goals
- Agent definitions and responsibilities
- Memory architecture (5 layers: identity, session compression, project logs, reflections, structured facts)
- Trust progression model
- MCP tool layer
- Deployment architecture
- Backup strategy (Google Drive via rclone)
- Two-repo pattern (public `genesis` + private `genesis-config`)
- Degraded cognition handling
- Build phases (0 through 7)

### 2. Follow the bootstrap brief

`GENESIS_Bootstrap_Brief.md` is the actual procedure for building Genesis. It:

- Defines the checkpoint system (`bootstrap-state.yaml` + `bootstrap-log.md` in `genesis-config`)
- Lists initial questions to ask Fareed upfront (IP, GitHub credentials, etc.)
- Walks through Phase 0 (prerequisites) through Phase 7
- Specifies session-resume protocol for interrupted builds

### 3. Deploy the agent package (during Phase 1)

When the bootstrap reaches Phase 1 step 5.5 (agent deployment), use `genesis-agent-package/README.md` for exact file placement instructions. Different files go to different repos:

- Diagrams → public `genesis` repo under `docs/diagrams/`
- Agent CLAUDE.md files → private `genesis-config` repo under `agents/<name>/`
- Skills → private `genesis-config` repo under `agents/<name>/skills/`
- Templates → public `genesis` repo under `docs/templates/`

Then symlinks from each agent's working directory point to the personalized files.

---

## Critical reading order for Claude Code

1. This README (orientation)
2. `GENESIS_Bootstrap_Brief.md` sections 0-3 (operating principles, checkpoint system)
3. `GENESIS_System_Design_v0.5.5.md` sections 1-2 (system overview, architecture)
4. `GENESIS_Bootstrap_Brief.md` section 2 (ask initial questions to Fareed)
5. Proceed with Phase 0 per the bootstrap brief

Later phases reference specific design doc sections — read those on demand, not upfront.

---

## Key architectural decisions (summary)

- **Hardware:** Single P14s Gen 2 (i7-1165G7, 40GB RAM, Intel Xe graphics)
- **Reasoning:** Claude Code CLI persistent processes via Max subscription (shared across all agents during POC)
- **Local model:** Qwen 3B via Ollama with Intel Xe acceleration (attribution only)
- **Bus:** SQLite WAL mode with hybrid push-poll
- **Portal:** React PWA + xterm.js + Web Speech API + FastAPI backend
- **Remote access:** Tailscale mesh VPN, Caddy reverse proxy with Cloudflare DNS-01
- **Email:** M365 IMAP read + draft (no SMTP during probation), Graph API Phase 5+
- **Memory:** Native CLAUDE.md + claude-mem plugin + structured markdown logs + SQLite facts
- **Repos:** Public `genesis` (MIT eventual) + permanently-private `genesis-config` (git-crypt)
- **Backups:** Snapshot-based via rclone to Google Drive, 30-minute cadence, configurable retention

---

## Agent roster

| Agent | Acronym meaning | Gender | Role |
|-------|-----------------|--------|------|
| **I.S.A.A.C.** | Initiator for Synthetic Agents & Autonomous Computing | he/him | Engineer |
| **O.R.A.C.L.E.** | Operational Reasoning & Analytics for Contextual Logic Extraction | she/her | Data analyst |
| **I.R.I.S.** | Intelligent Relay for Information & Scheduling | she/her | PA + PM + continuity hub |
| **S.E.N.T.I.N.E.L.** | Systematic Event Notification & Tracking Intelligence for Network Environment Logging | he/him | Watcher |
| **H.A.L.O.** | Home Automation & Lifestyle Orchestrator | she/her | Home automation (no LLM) |

---

## Initial questions Claude Code will ask Fareed

At fresh bootstrap start, Claude Code asks these together before any work begins:

**Connection:**
- P14s IP address (look it up via router or `ip addr` on P14s console)
- Ubuntu username on P14s
- SSH password (one-time, for initial key setup)
- Expected hostname for P14s (default: `genesis-p14s`)

**GitHub:**
- Confirm GitHub username is `fareed-hameed`
- Confirm both repos exist (`genesis` + `genesis-config`, both empty)
- GitHub PAT with `repo` scope

**Network:**
- Tailscale account ready?
- DNS provider for `novahive.cloud` (Cloudflare recommended)
- Domain already purchased?

See bootstrap brief section 2 for complete list.

---

## What's NOT in this bundle

- Genesis platform code (orchestrator, portal, MCP servers) — built by Claude Code during Phase 1+
- Halo's personalized `device-registry.yaml` — Fareed fills in as he registers HA devices
- `trust_config.yaml` initial values — created during Phase 1
- Real memory content (project logs, reflections, facts) — accumulates as agents work
- Credentials, API keys, tokens — never in any shared artifact; Fareed provides at deployment time

---

## If something's missing or broken

Contact Fareed via the chat. He'll either provide the missing piece or re-run the bundling step. Don't proceed past a missing critical file — flag and wait.
