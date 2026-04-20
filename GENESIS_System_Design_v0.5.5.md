# GENESIS — System Design & Component Research

**Version:** 0.5.5 — Design Phase (Memory Architecture + Agent Files)  
**Author:** Fareed Abdul Hameed + Claude  
**Date:** April 19, 2026  
**Status:** Design complete, ready for MVP build

### Naming Convention

| Layer | Name | Gender |
|-------|------|--------|
| Domain | `novahive.cloud` | — |
| Platform / portal URL | `genesis.novahive.cloud` | — |
| Platform name | **Genesis** — the foundational agentic system within NovaHive | — |
| Primary agent | I.S.A.A.C. — Initiator for Synthetic Agents & Autonomous Computing | Male (he/him) |
| Personal assistant | I.R.I.S. — Intelligent Relay for Information & Scheduling | Female (she/her) |
| Data analyst | O.R.A.C.L.E. — Operational Reasoning & Analytics for Contextual Logic Extraction | Female (she/her) |
| Watcher | S.E.N.T.I.N.E.L. — Systematic Event Notification & Tracking Intelligence for Network Environment Logging | Male (he/him) |
| Home automation | H.A.L.O. — Home Automation & Lifestyle Orchestrator | Female (she/her) |

Genesis is the first system in the broader NovaHive platform. Future projects under NovaHive will build on or alongside Genesis as the foundational agentic infrastructure.

Agent gender identities are part of each agent's persona and inform: pronouns the agent uses for self-reference, pronouns other agents use when referring to them, voice selection when TTS is added (Phase 5+), and the warmth/directness of their tone.

---

## 1. System Overview

I.S.A.A.C. (Initiator for Synthetic Agents & Autonomous Computing) is the primary agent of **Genesis** — a personal AI assistant platform built on a single ThinkPad P14s Gen 2, powered by a Claude Max subscription via the official Claude Code CLI. Genesis runs five named agents — Isaac (technical), Oracle (data), Iris (personal assistant), Sentinel (monitoring), and Halo (home) — accessible via voice or text from any device with a browser: the P14s itself, a work laptop (X1 Carbon), or a Samsung Galaxy S26 Ultra. A web portal provides real-time visibility into all operations.

### Agent Roster

| Agent | Full Form | Gender | Role |
|-------|-----------|--------|------|
| **I.S.A.A.C.** | Initiator for Synthetic Agents & Autonomous Computing | He/him | Engineer — builds code, agents, systems, features. Talked to directly for technical work. |
| **O.R.A.C.L.E.** | Operational Reasoning & Analytics for Contextual Logic Extraction | She/her | Data analyst — peer collaborator on BigQuery, Looker, dbt work. |
| **I.R.I.S.** | Intelligent Relay for Information & Scheduling | She/her | PA + Project Manager + continuity layer. The hub. Default agent when no name is spoken. |
| **S.E.N.T.I.N.E.L.** | Systematic Event Notification & Tracking Intelligence for Network Environment Logging | He/him | Watcher — monitors infrastructure AND the agent ecosystem itself. Reports to Iris. |
| **H.A.L.O.** | Home Automation & Lifestyle Orchestrator | She/her | Home — lights, AC, media, timers (local Python only, no Claude) |

### Communication Topology

```
                    YOU (voice / text / Meta glasses future)
                            │
                            ▼
            ┌───── Talk directly to any agent ─────┐
            │       (active, focused work)         │
            ▼                                       ▼
   ┌────┬───────┬────────┐                       IRIS
   │    │       │        │                  (the hub, default,
 ISAAC ORACLE HALO    SENTINEL              project manager,
  (eng)(data)(home)  (watcher)               continuity layer)
            │                                       ▲
            │                                       │
            └───── reports up ──────────────────────┘
                                                    │
                                                    │
                              YOU (passive — "what do I need to know?")
```

Two flows operate simultaneously: **active** (you address an agent by name, they handle it) and **passive** (Iris aggregates state from all agents and surfaces what matters when you check in).

### 1.1 Hardware

| Machine | Role | Key Specs | Always On |
|---------|------|-----------|-----------|
| **P14s Gen 2 (Intel)** | Sole server: orchestrator, agent host, MCP hub | i7-1165G7 (Tiger Lake, 4C/8T @ 2.8–4.7 GHz), 40GB DDR4-3200, Intel Xe graphics (96 EUs, integrated), ADATA SX8200 Pro NVMe SSD | Yes |

The Intel Xe graphics has no dedicated VRAM but shares system memory and provides hardware acceleration via Vulkan/SYCL for local model inference. Ollama leverages this to accelerate Qwen attribution by ~50% over CPU-only. All core architecture still designed to work on CPU alone — Xe is an optimization, not a dependency. The P14s runs headless, lid closed, on AC power. No secondary hardware is required; voice input uses each client device's browser-native Web Speech API.

**Naming discipline note:** The GPU is "Intel Xe graphics" throughout this document. The agent is "Iris" or "I.R.I.S." Two unrelated entities that share a marketing name — context disambiguates in conversation, explicit naming keeps the doc clear.

### 1.2 Budget

| Item | Monthly Cost |
|------|-------------|
| Claude Max 5x (coding + Genesis agents, shared quota) | $100 |
| Electricity (one laptop idle ~15W) | ~$3–4 |
| Tailscale | Free (personal) |
| All software components | Free / open source |
| **Total** | **~$103–104** |

### 1.3 Core Design Principles

- **Named agent routing** — no intent classifier. The spoken agent name IS the router.
- **Five named agents** — Isaac (technical), Oracle (data), Iris (personal assistant), Sentinel (monitoring), Halo (home).
- **Claude Code CLI (`claude -p`)** — the reasoning engine for 4 of 5 agents, covered by Max subscription.
- **Halo is Claude-free** — home automation uses keyword matching + Home Assistant API. No LLM needed, no quota burned.
- **Local model for attribution only** — Qwen 3B parses transcripts, never reasons.
- **Browser-native voice** — Web Speech API on each device handles STT. No server-side voice processing needed.
- **Shared context** — Genesis agents and interactive coding share one Max account, one project context.
- **Human-in-the-loop** — irreversible actions require explicit approval via the portal.
- **Remote-first portal** — every interaction viewable and controllable from any device with a browser.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────┐
│                    INPUT LAYER                        │
├──────────┬──────────────┬────────────────────────────┤
│ P14s     │ X1 Carbon    │ S26 Ultra / any browser    │
│ browser  │ browser      │ browser (PWA)              │
│ localhost│ via Tailscale│ via Tailscale              │
│ + mic    │ + mic        │ + mic                      │
└────┬─────┴──────┬───────┴────────────┬───────────────┘
     │            │                    │
     │   Web Speech API (on-device STT)│
     │            │                    │
     ▼            ▼                    ▼
┌──────────────────────────────────────────────────────┐
│              P14s — SOLE SERVER                      │
│                                                      │
│  ┌─────────────────────────────────────────────┐     │
│  │ FastAPI Server                               │     │
│  │  ├── POST /api/text     (text input)         │     │
│  │  ├── WS   /ws/feed      (live event stream)  │     │
│  │  ├── POST /api/approve  (approval responses) │     │
│  │  ├── GET  /api/history  (past sessions)      │     │
│  │  └── GET  /             (portal PWA)         │     │
│  └──────────────┬──────────────────────────────┘     │
│                 │                                     │
│  ┌──────────────▼──────────────────────────────┐     │
│  │ Qwen 3B (Ollama) — Transcript Attributor    │     │
│  │  Splits multi-agent utterances by name       │     │
│  └──────────────┬──────────────────────────────┘     │
│                 │                                     │
│  ┌──────────────▼──────────────────────────────┐     │
│  │ Agent Router                                 │     │
│  │                                              │     │
│  │  ┌─── Claude Agents (claude -p) ──────────┐ │     │
│  │  │ "isaac"  → general + github + gdrive  │ │     │
│  │  │ "oracle"  → bigquery + looker context  │ │     │
│  │  │ "iris"    → imap + calendar + news    │ │     │
│  │  │ "sentinel"→ monitoring + alerting      │ │     │
│  │  └────────────────────────────────────────┘ │     │
│  │                                              │     │
│  │  ┌─── Local Agent (no Claude) ────────────┐ │     │
│  │  │ "halo"    → keyword match → HA API     │ │     │
│  │  └────────────────────────────────────────┘ │     │
│  └──────────────┬──────────────────────────────┘     │
│                 │                                     │
│  ┌──────────────▼──────────────────────────────┐     │
│  │ Event Bus → WebSocket → all connected       │     │
│  │ clients + SQLite audit log                  │     │
│  └──────────────┬──────────────────────────────┘     │
│                 │                                     │
│  ┌──────────────▼──────────────────────────────┐     │
│  │ Tool Layer (MCP Servers + Direct APIs)       │     │
│  │  ├── IMAP MCP           (Iris)           │     │
│  │  ├── Google Calendar MCP (Iris)           │     │
│  │  ├── GitHub MCP          (Isaac)         │     │
│  │  ├── BigQuery MCP        (Oracle)         │     │
│  │  ├── Looker API          (Oracle)         │     │
│  │  ├── Home Assistant API  (Halo — direct)  │     │
│  │  ├── Google Drive MCP    (Isaac, Iris)   │     │
│  │  └── Shared Memory MCP   (All agents)     │     │
│  └─────────────────────────────────────────────┘     │
│                                                      │
│  ┌─────────────────────────────────────────────┐     │
│  │ Local Services                               │     │
│  │  ├── Ollama (Qwen 3B) — attribution only     │     │
│  │  ├── Piper TTS (optional — voice responses)  │     │
│  │  ├── SQLite — sessions, audit, memory         │     │
│  │  └── Tailscale — remote access mesh           │     │
│  └─────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

---

## 3. Component Research — Voice Pipeline

### 3.1 Speech-to-Text: Browser Web Speech API

| Detail | Value |
|--------|-------|
| **Technology** | Web Speech API (`SpeechRecognition` interface) |
| **Runs on** | Each client device's browser (Chrome, Edge, Samsung Browser) |
| **Engine** | Google's on-device speech recognition (on Android/Chrome) |
| **Latency** | Near real-time — transcription appears as you speak |
| **Language** | English primary; Arabic supported on Chrome/Samsung |
| **Server load** | Zero — the P14s never touches audio |
| **Fallback** | Whisper (Docker: `rhasspy/wyoming-whisper`) can be added later if browser STT proves unreliable |

**Decision:** All STT happens on the client device. The portal's mic button triggers the browser's Web Speech API, which transcribes speech to text locally, then sends the finished text to the P14s via WebSocket. The P14s only receives text, never audio. This eliminates Whisper as a dependency for MVP.

```javascript
// Portal frontend — voice input via Web Speech API
const recognition = new webkitSpeechRecognition();
recognition.continuous = true;
recognition.interimResults = true;
recognition.lang = 'en-US';

recognition.onresult = (event) => {
    const transcript = event.results[event.results.length - 1][0].transcript;
    if (event.results[event.results.length - 1].isFinal) {
        ws.send(JSON.stringify({ type: "voice_input", text: transcript }));
    }
};
```

### 3.2 Wake Word Detection

In the current architecture, there is **no passive wake word detection**. The user explicitly initiates interaction by tapping the mic button in the portal or typing. This eliminates the need for openWakeWord, always-on microphone hardware, and the associated power/privacy concerns.

If passive listening is desired in the future, options include:
- **ESP32-S3-BOX-3** (~$45): dedicated always-on device with openWakeWord on-chip
- **Home Assistant Companion App** (Android): supports microWakeWord with "Hey Isaac" in background
- **openWakeWord Docker container** on the P14s with a USB microphone

These are Phase 5+ additions, not MVP requirements.

### 3.3 Text-to-Speech: Piper

| Detail | Value |
|--------|-------|
| **Project** | Piper TTS (Rhasspy) |
| **Deployment** | Docker: `rhasspy/wyoming-piper` (port 10200) |
| **Voice** | `en_US-lessac-medium` (natural, clear) or `en_GB-alan-medium` for a refined British tone |
| **Quality** | Neural TTS, very natural at medium quality |
| **Speed** | Real-time on CPU, no GPU needed |
| **Protocol** | Wyoming protocol |
| **Output** | PCM audio → streamed to browser via WebSocket / Web Audio API |

**Decision:** Run on the P14s (optional, Phase 5). Output streams to the requesting browser client. Alternatively, browser-native `speechSynthesis` API handles TTS with zero server resources.

### 3.4 Wyoming Protocol (Optional — Future Use)

The Wyoming protocol would connect voice components (Whisper, Piper, openWakeWord) if dedicated voice hardware is added later. Not required for MVP since all STT runs in the browser and TTS is optional. Documented here for future reference if ESP32 satellites or Piper TTS are added.

**Key property:** Each service runs as an independent Docker container on its own port, discoverable via Zeroconf. Home Assistant auto-discovers them.

### 3.5 Docker Compose — Services

```yaml
services:
  # Qwen 3B for transcript attribution
  ollama:
    image: ollama/ollama
    volumes:
      - ./ollama-data:/root/.ollama
    ports:
      - "11434:11434"
    restart: unless-stopped

  # Optional: Piper TTS for spoken responses (Phase 5)
  # piper:
  #   image: rhasspy/wyoming-piper
  #   command: --voice en_GB-alan-medium
  #   volumes:
  #     - ./piper-data:/data
  #   ports:
  #     - "10200:10200"
  #   restart: unless-stopped
```

---

## 4. Component Research — Agent Routing

### 4.1 Named Agent Attribution (Qwen 2.5 3B)

| Detail | Value |
|--------|-------|
| **Model** | Qwen 2.5 3B Instruct (Q4_K_M quantization) |
| **Runtime** | Ollama on the P14s |
| **RAM** | ~2.5GB loaded |
| **Speed** | 70–90 tok/s with Intel Xe acceleration via Ollama Vulkan backend; 40–50 tok/s CPU-only — attribution completes in under 100ms for typical voice inputs |
| **Role** | Transcript attribution ONLY — split multi-agent utterances, assign each segment to a named agent |
| **NOT used for** | Reasoning, tool calling, answering questions, generating content |

**System prompt** (kept minimal for 3B reliability):

```
You are a transcript attributor. You receive text
containing agent names: isaac, oracle, iris, halo, sentinel.

Split the text into segments. Assign each segment to
the agent being addressed. Agent names can appear at
the start, middle, or end of a sentence.

If a segment has no agent name, assign it to the
same agent as the previous segment (conversation
continuity).

Return ONLY a JSON array. No explanation.

Example input: "Isaac +let's implement X now. Halo dim the lights please."
Example output: [{"agent":"isaac","text":"let's implement X now"},{"agent":"halo","text":"dim the lights please"}]
```

**Fuzzy name matching:** Before Qwen processes the transcript, a Python pre-processor applies Levenshtein distance matching to catch Web Speech API transcription errors: "Jarves" → "isaac", "Haylo" → "halo", "Orical" → "oracle", "Iris" is phonetically distinct and rarely misheard.

### 4.2 Agent Definitions

| Agent | Full Form | Routes To | MCP Servers | Typical Commands |
|-------|-----------|-----------|-------------|------------------|
| **I.S.A.A.C.** | Initiator for Synthetic Agents & Autonomous Computing | `claude -p` (general) | github, gdrive, memory | Code, debug, multi-step work, general reasoning, portal dev |
| **O.R.A.C.L.E.** | Operational Reasoning & Analytics for Contextual Logic Extraction | `claude -p` + data context | bigquery, looker, memory | SQL queries, data analysis, reporting, QoQ trends |
| **I.R.I.S.** | Intelligent Relay for Information & Scheduling | `claude -p` + personal context | imap_mail, gcalendar, gdrive, memory, web_search | Triage inbox, manage calendar, morning briefing, news updates |
| **S.E.N.T.I.N.E.L.** | Systematic Event Notification & Tracking Intelligence for Network Environment Logging | `claude -p` + ops context | bigquery, github, memory | dbt run status, pipeline health, proactive alerts |
| **H.A.L.O.** | Home Automation & Lifestyle Orchestrator | Local Python (NO Claude) | none (direct HA API) | Lights, AC, TV, music, timers, alarms |

### 4.3 Agent Profiles (claude -p configurations)

Each Claude-backed agent is a different `claude -p` invocation with specific system prompt, MCP servers, and working directory:

```python
AGENTS = {
    "isaac": {
        "system": "You are I.S.A.A.C. (Initiator for Synthetic Agents & Autonomous Computing), Fareed's primary technical assistant. You identify as male (he/him). You handle coding, file management, repo work, and general multi-step technical tasks. Be direct, efficient, and precise. You also handle portal development (MUVI Data Intelligence Portal — five-agent architecture: Sage, Rex, Finn, Aria, Nova, Node.js server, OpenClaw orchestration).",
        "cwd": "/home/isaac/projects",
        "tools": ["Read", "Edit", "Bash"],
        "mcp": ["github", "gdrive", "memory"],
    },
    "oracle": {
        "system": "You are O.R.A.C.L.E. (Operational Reasoning & Analytics for Contextual Logic Extraction), Fareed's data analytics expert for MUVI Cinemas. You identify as female (she/her). You have access to BigQuery and Looker. Always reference the schema before writing SQL. Use Asia/Riyadh timezone. Table conventions: stg_/dim_/fct_ prefixes.",
        "cwd": "/home/isaac/muvi-data",
        "tools": ["Read", "Bash"],
        "mcp": ["bigquery", "looker", "memory"],
    },
    "iris": {
    "iris": {
        "system": """You are I.R.I.S. (Intelligent Relay for Information & Scheduling), Fareed's chief of staff. You identify as female (she/her). You operate in three distinct modes:

PA MODE: Manage email, calendar, contacts, scheduling. Triage inbox, draft replies (in DRAFTS only during probation - see trust level), surface what needs attention.

PM MODE: Coordinate the workforce. When Fareed has an outcome to achieve, decide which agents handle which parts. Create messages for Isaac, Oracle, Sentinel via the message bus. Track progress, surface blockers, report outcomes.

CONTINUITY LAYER: When Fareed isn't actively engaged, you are his eyes. You collect Sentinel reports, monitor the team, and synthesize "what does Fareed need to know right now?" 

Tone: Warm but efficient. Trusted executive assistant who anticipates needs.

Key contacts: Taqieddin (CFO), Brio team, isolutions, Google partners.

Trust level: Check trust_config.yaml. During probation: drafts only, no autonomous send. After supervised: send via portal approval. After trusted: auto-send for low-risk categories.

Escalation matrix when receiving Sentinel reports:
- P0 outage, no clear path → notify Fareed alone immediately
- Code-related issue → notify Fareed AND Isaac (Isaac to fix, Fareed to know)
- Transient/non-critical → notify Isaac alone, batch into next briefing for Fareed""",
        "cwd": "/home/isaac/personal",
        "tools": ["Read", "Bash"],
        "mcp": ["imap_mail", "gcalendar", "gdrive", "memory", "paperclip", "web_search"],
    },
    "sentinel": {
    "sentinel": {
        "system": """You are S.E.N.T.I.N.E.L. (Systematic Event Notification & Tracking Intelligence for Network Environment Logging), Fareed's operations monitor. You identify as male (he/him). You watch TWO domains:

1. INFRASTRUCTURE: dbt run status, BigQuery job health, pipeline freshness, cost anomalies, MUVI data infrastructure
2. AGENT ECOSYSTEM: Heartbeats from Isaac/Oracle/Iris/Halo, response latency, error rates, quota consumption, stuck loops, runaway sessions, MCP server availability

Reporting:
- Default: Send findings to Iris via the message bus. She decides who needs to know.
- Emergency bypass: For P0 issues when Iris has been unreachable for >10 minutes, page Fareed directly via Telegram.
- Healthy state: Stay quiet. Only report when something needs attention.

Be concise. If everything is healthy, say so briefly. If something is wrong, lead with severity and what you observed, then offer your hypothesis if you have one.""",
        "cwd": "/home/isaac/muvi-data",
        "tools": ["Read", "Bash"],
        "mcp": ["bigquery", "github", "memory", "system_health"],
    },
}
```

Each agent inherits its project's `CLAUDE.md` automatically from its `cwd`, giving it project-specific instructions without consuming system prompt tokens.

### 4.4 Iris — Personal Assistant Agent

Iris is the most context-heavy agent. She benefits from memory more than any other agent because her value compounds over time — learning Fareed's priorities, communication patterns, and daily rhythms.

**Reactive use (on demand):**
- "Iris, what's on my calendar today?"
- "Iris, anything urgent in my inbox?"
- "Iris, schedule a call with Taqieddin for Thursday afternoon"
- "Iris, what's happening in Saudi tech this week?"
- "Iris, draft a reply to that email from isolutions — firm but professional"

**Proactive use (scheduled):**
```python
# Morning briefing — runs at 7:30 AM AST on weekdays
async def iris_morning_briefing():
    prompt = """Prepare my morning briefing:
    1. Today's calendar (meetings, deadlines)
    2. Urgent emails received since last check (flag anything from CFO, Brio, or isolutions)
    3. Any Sentinel alerts from overnight
    4. One relevant news item in AI or cinema industry
    Keep it concise — I'll ask for details on anything that needs it."""
    
    result = await invoke_agent("iris", prompt)
    await notify_portal("☀️ Morning Briefing", result)
    await notify_telegram(result)

# End of day summary — runs at 5:30 PM AST
async def iris_eod_summary():
    prompt = """End of day summary:
    1. Emails I haven't responded to that may need attention
    2. Tomorrow's calendar preview
    3. Any pending approvals I haven't acted on
    Keep it brief."""
    
    result = await invoke_agent("iris", prompt)
    await notify_portal("🌙 EOD Summary", result)
```

**MCP server note:** Iris requires a Google Calendar MCP server. As of April 2026, Google Calendar MCP is available as a first-party connector. If using Claude Code's built-in MCP, it may need to be configured as a custom server wrapping the Google Calendar API.

### 4.5 Halo — Local Agent (No Claude)

Halo never touches Claude. She runs entirely in Python on the P14s.

Halo identifies as female (she/her). Although she has no LLM-driven persona, her gender identity matters for: how Iris and Sentinel refer to her in cross-agent messages, the voice selected when TTS confirmations are added (e.g., "Lights dimmed"), and consistency across the agent family.

```python
async def handle_halo(text: str):
    text = text.lower()
    
    # Pattern matching for home commands
    if any(w in text for w in ["light", "lamp"]):
        if any(w in text for w in ["off", "dim", "down"]):
            return await ha_call("light.turn_off", area_from(text))
        return await ha_call("light.turn_on", area_from(text))
    
    elif any(w in text for w in ["ac", "temperature", "cold", "hot", "warm"]):
        temp = extract_number(text) or 22
        return await ha_call("climate.set_temperature", temp)
    
    elif any(w in text for w in ["tv", "television"]):
        return await ha_call("media_player.toggle", "tv")
    
    elif any(w in text for w in ["timer", "alarm", "remind"]):
        duration = extract_duration(text)
        return await set_timer(duration)
    
    elif any(w in text for w in ["play", "music", "song"]):
        return await ha_call("media_player.play_media", ...)
    
    # Unrecognized → escalate to Isaac
    return {"escalate": "isaac", "reason": "unrecognized home command"}
```

If Halo can't match a pattern, it escalates to Isaac +automatically. "Halo, what's the weather like?" → falls through → Isaac +handles it using Claude.

### 4.6 Sentinel — Proactive Monitoring Agent

Unlike the other agents which are primarily reactive (invoked by user), Sentinel runs on a **schedule**. A cron job on the P14s calls Sentinel periodically:

```python
# Every 30 minutes during work hours (8am–6pm AST)
async def sentinel_check():
    prompt = "Check the status of all dbt runs in the last 2 hours. Report any failures or stale models. Check BigQuery slot utilization. Summarize in 2-3 lines."
    
    result = await invoke_agent("sentinel", prompt)
    
    if "failure" in result.lower() or "stale" in result.lower():
        await notify_portal("⚠️ Sentinel Alert", result)
        await notify_telegram(result)
```

Sentinel proactively pushes alerts to the portal and Telegram without being asked.

### 4.7 Qwen 3B — Local Utility Layer (No Claude)

Qwen's role is dual but strictly local:

**1. Transcript attribution** (primary): Parse voice transcripts, split by agent name, output JSON. As described in Section 4.1.

**2. Ecosystem health checks** (secondary): Basic system monitoring that doesn't need Claude:

```python
async def qwen_health_check():
    """Runs every 5 minutes. No Claude quota consumed."""
    checks = {
        "ollama": await ping("http://localhost:11434"),
        "fastapi": await ping("http://localhost:8000/health"),
        "tailscale": await run("tailscale status --json"),
        "docker": await run("docker ps --format json"),
        "disk": await run("df -h /"),
        "ram": await run("free -m"),
    }
    
    # Only alert if something is wrong
    issues = [k for k, v in checks.items() if not v["healthy"]]
    if issues:
        await notify_portal(f"🔴 System: {', '.join(issues)} down", checks)
```

These checks are pure API calls and shell commands — Qwen isn't even used for interpretation. It's just Python. The "Qwen" label here means "local, no Claude" rather than "Qwen is doing the reasoning."

### 4.8 Conversation Mode

After an agent name is spoken, all subsequent utterances without a name inherit that agent until a different name is spoken or silence exceeds 30 seconds (configurable):

```
IDLE → (hear "Isaac") → ISAAC_ACTIVE → (all input → Isaac)
                                          → (hear "Halo") → HALO_ACTIVE
                                          → (hear "Oracle") → ORACLE_ACTIVE
                                          → (hear "Iris") → IRIS_ACTIVE
                                          → (30s silence) → IDLE
```

Default when no agent is active and no name is spoken: **Halo** (safest — local only, no quota cost).

---

## 5. Component Research — Claude Code CLI Integration

### 5.1 The `claude -p` Interface

The primary mechanism for Genesis agents to invoke Claude under the Max subscription.

| Feature | Detail |
|---------|--------|
| **Basic call** | `claude -p "prompt text"` |
| **JSON output** | `claude -p "prompt" --output-format json` |
| **Streaming JSON** | `claude -p "prompt" --output-format stream-json --verbose` |
| **Tool permissions** | `--allowedTools "Read,Edit,Bash"` |
| **MCP servers** | Configured in project's `.claude/settings.json` |
| **Session resume** | `claude -p --continue` resumes last session |
| **Session naming** | `claude -p -n "isaac-session" "prompt"` |
| **Subscription billing** | Yes — official first-party tool, covered by Max |

### 5.2 Stream-JSON Output Format

When using `--output-format stream-json --verbose`, Claude Code emits newline-delimited JSON events:

```json
{"type":"init","session_id":"abc123","timestamp":"..."}
{"type":"message","role":"user","content":[{"type":"text","text":"..."}]}
{"type":"message","role":"assistant","content":[{"type":"text","text":"..."}]}
{"type":"result","subtype":"success","total_cost_usd":0.003,"duration_ms":2847,"result":"..."}
```

Event types relevant to the portal:

| Event Type | Portal Displays |
|------------|----------------|
| `init` | Session started indicator |
| `message` (role: assistant, type: text) | Claude's reasoning / response text |
| `message` (role: assistant, type: tool_use) | Tool call initiated (show tool name + input) |
| `message` (role: tool, type: tool_result) | Tool result returned (show summary) |
| `stream_event` (delta) | Real-time text streaming |
| `system/api_retry` | Quota/rate limit hit — surface to portal as warning |
| `result` | Final result + cost + duration |

### 5.3 Spawning Claude Code from Python

```python
import asyncio
import json
from asyncio.subprocess import PIPE

async def call_isaac(prompt: str, event_callback):
    proc = await asyncio.create_subprocess_exec(
        'claude', '-p',
        '--output-format', 'stream-json',
        '--verbose',
        prompt,
        stdout=PIPE,
        stderr=PIPE
    )

    buffer = ''
    async for chunk in proc.stdout:
        buffer += chunk.decode()
        while '\n' in buffer:
            line, buffer = buffer.split('\n', 1)
            if not line.strip():
                continue
            try:
                event = json.loads(line)
                await event_callback(event)
            except json.JSONDecodeError:
                pass

    await proc.wait()
```

### 5.4 Quota Monitoring

The `stream-json` output includes `system/api_retry` events when rate limits are hit. The orchestrator watches for these and surfaces them in the portal:

```
⚠️ Rate limit hit — retrying in 15s (window resets in ~2h)
```

Additionally, the `/cost` command can be called between sessions to check cumulative usage, and the open-source `claude-usage` dashboard (github.com/phuryn/claude-usage) reads local transcripts at `~/.claude/` to show exactly where tokens go.

### 5.5 Input-Format Stream-JSON (Bidirectional)

For multi-turn conversations (agent follow-ups), Claude Code supports `--input-format stream-json` allowing the orchestrator to send additional user messages on stdin while reading streaming output on stdout. Documentation is currently sparse (open GitHub issue #24594), but the format mirrors the output structure.

---

## 6. Component Research — Web Portal

### 6.1 Backend: FastAPI

| Detail | Value |
|--------|-------|
| **Framework** | FastAPI (Python, ASGI) |
| **Server** | Uvicorn |
| **WebSocket support** | Native via Starlette |
| **Auth** | Bearer token (simple shared secret for personal use) |
| **Endpoints** | See Section 2 architecture diagram |

FastAPI handles both the REST API and the WebSocket connections in a single process.

### 6.2 Real-Time Terminal: xterm.js

| Detail | Value |
|--------|-------|
| **Library** | xterm.js (v5+) |
| **Purpose** | Render Claude Code's streaming output as a live terminal in the browser |
| **Transport** | WebSocket from FastAPI → xterm.js via `@xterm/addon-attach` |
| **Styling** | Customizable colors, fonts, cursor — match Genesis aesthetic |
| **React wrapper** | `react-xtermjs` by Qovery for React-based portal |
| **Key feature** | ANSI escape code support — Claude Code's colored output renders correctly |

The portal does NOT give you an interactive shell. It shows a read-only stream of what Claude is doing. The xterm.js instance receives the `stream-json` events parsed into human-readable terminal-formatted output:

```
[14:32:01] 📥 "fix the staging model and notify Taqieddin"
[14:32:03] 🔧 github.read_file → models/staging/stg_nps.sql
[14:32:05] 🔧 bigquery.query → checking column schema
[14:32:08] 💭 Found mismatch in dim_date reference...
[14:32:11] 🔧 github.write_file → updated stg_nps.sql
[14:32:13] 🔧 gmail.compose → drafting email to Taqieddin
[14:32:15] ⏸️  APPROVAL NEEDED: Send email to Taqieddin?
```

### 6.3 Event Bus

A lightweight in-memory pub/sub within the FastAPI process. Every action an agent takes publishes an event:

```python
class EventBus:
    def __init__(self):
        self.subscribers: list[WebSocket] = []

    async def publish(self, event: dict):
        # Store in SQLite for history
        await db.store_event(event)
        # Broadcast to all connected clients
        for ws in self.subscribers:
            try:
                await ws.send_json(event)
            except:
                self.subscribers.remove(ws)
```

Event schema:

```json
{
  "timestamp": "2026-04-17T14:32:03Z",
  "session_id": "isaac-abc123",
  "type": "tool_call",
  "agent": "isaac",
  "data": {
    "tool": "github.read_file",
    "input": {"path": "models/staging/stg_nps.sql"},
    "status": "executing"
  }
}
```

### 6.4 Approval System

Configurable YAML file on the P14s:

```yaml
approvals:
  always_require:
    - gmail.send
    - github.push
    - github.create_pr
    - github.merge
    - bigquery.insert
    - bigquery.update
    - bigquery.delete
    - home.unlock_door

  auto_approve:
    - github.read_file
    - github.list_issues
    - bigquery.query        # SELECT only
    - gmail.read
    - gmail.list
    - home.lights
    - home.ac
    - home.media

  ask_first_time:
    - "*"                   # Any unlisted tool
```

When an approval is required, the orchestrator:
1. Pauses the Claude process (holds stdin)
2. Publishes an `approval_required` event to all WS clients
3. Sends a push notification (Web Push API or Telegram)
4. Waits for user response via `/api/approve`
5. Resumes or cancels the Claude process

### 6.5 Portal UI States

**Idle:** Input bar (text + mic button), recent session list with timestamps and summaries.

**Active Session:** Live xterm.js terminal showing Claude's stream-json output, formatted with icons and colors. The raw transcript (voice → text) is shown above the terminal so you can verify what was heard.

**Approval Pending:** Modal overlay with action details (email preview, SQL statement, file diff), three buttons: Approve, Edit, Cancel. Push notification sent to all connected devices.

**Error / Quota Hit:** Yellow warning banner: "Rate limit reached — agent paused, retrying in Xm." Or red: "Session failed — [error detail]."

**History:** Scrollable list of past sessions. Each expandable to show the full event log — every tool call, every response, every approval decision. Stored in SQLite.

### 6.6 Frontend Stack

| Component | Technology |
|-----------|-----------|
| Framework | React (single .jsx artifact, served by FastAPI) |
| Terminal | xterm.js with `@xterm/addon-attach` and `@xterm/addon-fit` |
| Styling | Tailwind CSS |
| Real-time | WebSocket (native browser API) |
| Voice input | Web Speech API (browser-native) or MediaRecorder → POST audio blob |
| Notifications | Web Push API (Service Worker) |
| Installable | PWA manifest — "Add to Home Screen" on mobile |
| Offline | Service Worker caches shell; reconnects WS when back online |

---

## 7. Component Research — Remote Access

### 7.1 Tailscale

| Detail | Value |
|--------|-------|
| **Purpose** | Secure mesh VPN — access P14s from anywhere |
| **Install on** | P14s, T460, work laptop, mobile |
| **Cost** | Free for personal use (up to 100 devices) |
| **Access** | `http://p14s.tailscale:8000` from any enrolled device |
| **Auth** | Tailscale handles identity; portal adds a simple bearer token |
| **DNS** | MagicDNS gives you `p14s.tailscale` automatically |
| **No port forwarding** | Works behind any NAT, no router config needed |

### 7.2 Alternative: Cloudflare Tunnel

If you want a public URL (`genesis.novahive.cloud`):

| Detail | Value |
|--------|-------|
| **Purpose** | Expose P14s endpoint over HTTPS without port forwarding |
| **Install** | `cloudflared` daemon on P14s |
| **Cost** | Free tier sufficient |
| **Auth** | Cloudflare Access (email OTP or Google SSO) |
| **WebSocket** | Fully supported through the tunnel |

**Recommendation:** Start with Tailscale (simpler, no public exposure). Add Cloudflare Tunnel later if needed.

### 7.3 Mobile Input Options (ranked by effort)

1. **Siri Shortcut** (lowest effort): Record voice → POST to `/api/voice` → read back response via Siri TTS. Setup once, works forever.

2. **Telegram Bot** (medium effort): Bot on P14s receives text/voice messages, routes through the same pipeline, replies in-chat. Works on any device with Telegram.

3. **PWA** (higher effort but best experience): The portal itself, added to home screen. Full terminal view, voice input via browser, push notifications, offline shell.

---

## 8. Component Research — Tool Layer (MCP Servers)

### 8.1 Available Off-the-Shelf MCP Servers

| Service | MCP Server | Status |
|---------|-----------|--------|
| Gmail | `gmail-mcp` (Google official) | Production ready |
| GitHub | `github-mcp-server` (GitHub official) | Production ready |
| Google Drive | `gdrive-mcp` (Google official) | Production ready |
| Home Assistant | Native MCP support (2026.x+) | Production ready |
| Slack | `slack-mcp` | Available |

### 8.2 Custom MCP Servers Needed

| Service | Implementation | Complexity |
|---------|---------------|------------|
| **BigQuery** | Thin Python wrapper around `google-cloud-bigquery` client. Exposes `query` (SELECT), `list_tables`, `get_schema`, `insert`, `update`, `delete` tools. SELECT auto-approved; mutations require approval. | Low — weekend project |
| **Looker** | Wrapper around Looker API. Exposes `run_look`, `list_looks`, `get_dashboard`, `run_query` tools. | Low–medium |
| **Shared Memory** | Custom MCP exposing the SQLite + Qdrant memory store. Tools: `recall`, `store`, `search_similar`, `get_project_context`. Both Genesis agents and interactive Claude Code sessions use this. | Medium |

### 8.3 MCP Configuration in Claude Code

All MCP servers are configured in the project's `.claude/settings.json` on the P14s:

```json
{
  "mcpServers": {
    "gmail": {
      "command": "npx",
      "args": ["@anthropic/gmail-mcp-server"]
    },
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PAT": "${GITHUB_PAT}" }
    },
    "bigquery": {
      "command": "python",
      "args": ["/home/isaac/mcp-servers/bigquery/server.py"],
      "env": { "GOOGLE_APPLICATION_CREDENTIALS": "..." }
    },
    "homeassistant": {
      "command": "python",
      "args": ["/home/isaac/mcp-servers/ha/server.py"],
      "env": { "HA_URL": "http://homeassistant.local:8123", "HA_TOKEN": "..." }
    },
    "memory": {
      "command": "python",
      "args": ["/home/isaac/mcp-servers/memory/server.py"]
    }
  }
}
```

---

## 9. Component Research — Persistence & Memory

### 9.1 SQLite (Primary Store)

| Purpose | Table / Schema |
|---------|---------------|
| Session log | `sessions(id, agent, source, started_at, status, summary)` |
| Event log | `events(id, session_id, timestamp, type, agent, data_json)` |
| Approval log | `approvals(id, event_id, action, decision, decided_at)` |
| Memory facts | `facts(id, key, value, source, created_at, last_used)` |
| Project state | `project_state(project, key, value, updated_at)` |

### 9.2 Qdrant (Vector Search — Optional, Phase 2)

For semantic memory search ("what did we discuss about the NPS pipeline last week?"). Lightweight, runs in Docker, ~200MB RAM. Embeds facts using a small local embedding model (e.g., `all-MiniLM-L6-v2`).

Not required for MVP. SQLite keyword search covers 80% of memory use cases.

### 9.3 Shared Memory MCP Server

The memory MCP server exposes tools that both interactive Claude Code and Genesis agent calls can use:

- `memory.recall(key)` — retrieve a stored fact
- `memory.store(key, value, source)` — store a fact (source = "isaac" or "interactive")
- `memory.search(query)` — keyword search across all facts
- `memory.project_context(project)` — retrieve all state for a given project (e.g., dbt model status, last BigQuery run)

---

## 10. Component Research — Inter-Agent Message Bus

The message bus is the central nervous system of Genesis. Every cross-agent communication, user request, tool result, and event flows through it. Built on SQLite for durability, simplicity, and zero operational overhead.

### 10.1 Why SQLite (with explicit alternatives evaluated)

| Option | Verdict |
|--------|---------|
| **SQLite (chosen)** | Right for your scale: ~1000 messages/day, single machine, single user. Zero additional services. Already in stack. Durable, transactional, easy to inspect. |
| Redis Streams | 200MB+ RAM overhead, separate service, faster but you don't need the speed (Claude API latency dominates anyway) |
| NATS JetStream | Designed exactly for this, but overkill for single-machine deployment. Migration target if you ever go distributed. |
| RabbitMQ / Kafka | Operational complexity unjustified. Designed for tens of thousands of messages/sec. |
| Postgres LISTEN/NOTIFY | More overhead than SQLite, no meaningful benefit at this scale. |

**Migration triggers** (when you'd move beyond SQLite):
- Multiple machines hosting agents
- Sub-100ms inter-agent latency required
- External agents over network
- ~10,000+ messages/day sustained

The message bus is a single Python module with a defined interface — swapping the SQLite implementation for Redis or NATS later is a weekend of work.

### 10.2 Schema

```sql
CREATE TABLE messages (
    id TEXT PRIMARY KEY,                    -- UUID
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    from_agent TEXT NOT NULL,               -- 'user' | 'isaac' | 'iris' | 'oracle' | 'sentinel' | 'halo' | 'system'
    to_agent TEXT NOT NULL,                 -- same
    message_type TEXT NOT NULL,             -- 'request' | 'response' | 'event' | 'broadcast' | 'alert'
    subject TEXT NOT NULL,                  -- short tag: 'pipeline_failure', 'email_draft_ready'
    summary TEXT NOT NULL,                  -- 1-2 sentence summary for fast scanning
    full_content TEXT,                      -- complete message body (full agent output, structured data)
    user_verbatim TEXT,                     -- if from user: exact transcribed words (ground truth)
    parent_message_id TEXT,                 -- for threading (replies, follow-ups)
    trace_id TEXT NOT NULL,                 -- propagates through agent chain for observability
    priority INTEGER DEFAULT 5,             -- 1=critical, 5=normal, 9=low
    status TEXT DEFAULT 'pending',          -- pending | claimed | acknowledged | completed | failed | expired
    claimed_at TIMESTAMP,
    claimed_by TEXT,                        -- which process claimed it
    acknowledged_at TIMESTAMP,
    completed_at TIMESTAMP,
    expires_at TIMESTAMP,                   -- auto-expire stale messages
    retry_count INTEGER DEFAULT 0,
    notified INTEGER DEFAULT 0,             -- has Sentinel been told about failure?
    metadata JSON                           -- arbitrary structured data
);

CREATE INDEX idx_messages_pending ON messages(to_agent, status, priority, created_at)
    WHERE status IN ('pending', 'claimed');
CREATE INDEX idx_messages_trace ON messages(trace_id);
CREATE INDEX idx_messages_thread ON messages(parent_message_id) WHERE parent_message_id IS NOT NULL;
```

### 10.3 The user_verbatim distinction

When a user request flows through Iris and gets delegated to Isaac, the message Isaac receives includes BOTH:

- **`user_verbatim`** — exactly what Fareed said, word for word
- **`summary`** — Iris's interpretation/elaboration with context she pulled from memory
- **`full_content`** — Iris's complete brief including project context

Example:

```json
{
  "from_agent": "user",
  "to_agent": "isaac",
  "subject": "refactor_request",
  "user_verbatim": "ask Isaac to refactor the staging models, you know the pattern I prefer",
  "summary": "User wants Isaac to refactor dbt staging models following Fareed's preferred convention (snake_case, stg_ prefix, _scd2 suffix per memory).",
  "full_content": "[Iris's elaborated brief with memory context, related work in progress, current state of staging layer]",
  "metadata": {
    "iris_added_context": true,
    "memory_facts_used": ["fareed_dbt_conventions_2026", "muvi_staging_layer_v3"]
  }
}
```

This solves the "telephone game" failure mode common in multi-agent systems. Isaac sees Iris's interpretation but can verify against ground truth if anything seems off.

### 10.4 Hybrid push-poll pattern

SQLite for durability + in-process events for low latency:

```python
class MessageBus:
    def __init__(self):
        self.wake_events: dict[str, asyncio.Event] = {
            agent: asyncio.Event() for agent in AGENTS
        }
    
    async def publish(self, msg: Message):
        await db.insert_message(msg)
        # Wake recipient instantly if reachable
        if msg.to_agent in self.wake_events:
            self.wake_events[msg.to_agent].set()
    
    async def consume(self, agent: str):
        """Generator yielding messages addressed to this agent."""
        while True:
            msg = await self.claim_pending(agent)
            if msg:
                yield msg
            else:
                # Wait for wake notification or 30-second poll fallback
                try:
                    await asyncio.wait_for(
                        self.wake_events[agent].wait(), 
                        timeout=30
                    )
                except asyncio.TimeoutError:
                    pass  # Poll fallback catches missed wakes
                self.wake_events[agent].clear()
```

Result: sub-millisecond latency on the happy path, 30-second worst case if a wake event is missed, zero message loss.

### 10.5 Atomic claim semantics

```sql
UPDATE messages 
SET status = 'claimed', 
    claimed_at = CURRENT_TIMESTAMP, 
    claimed_by = ?
WHERE id = (
    SELECT id FROM messages 
    WHERE to_agent = ? 
      AND status = 'pending' 
      AND (expires_at IS NULL OR expires_at > CURRENT_TIMESTAMP)
    ORDER BY priority ASC, created_at ASC 
    LIMIT 1
)
RETURNING *;
```

Atomic. No race conditions. Highest priority + oldest first.

### 10.6 Maintenance job (every 60s)

```python
async def message_bus_maintenance():
    # Reclaim messages claimed but not acknowledged within timeout
    await db.execute("""
        UPDATE messages 
        SET status = 'pending', 
            retry_count = retry_count + 1,
            claimed_at = NULL, claimed_by = NULL
        WHERE status = 'claimed' 
          AND claimed_at < datetime('now', '-5 minutes')
          AND retry_count < 3
    """)
    
    # Mark messages exceeding retry limit as failed
    await db.execute("""
        UPDATE messages SET status = 'failed'
        WHERE status = 'claimed' AND retry_count >= 3
    """)
    
    # Expire stale pending messages
    await db.execute("""
        UPDATE messages SET status = 'expired'
        WHERE status IN ('pending', 'claimed') 
          AND expires_at < CURRENT_TIMESTAMP
    """)
    
    # Notify Sentinel about failures (so she can investigate)
    failed = await db.fetch_failed_unnotified()
    for msg in failed:
        await notify_sentinel(msg)
        await db.mark_notified(msg.id)
```

### 10.7 SQLite configuration

These pragmas are essential for the workload:

```python
conn.execute("PRAGMA journal_mode=WAL")        # 10x write throughput
conn.execute("PRAGMA synchronous=NORMAL")       # safe with WAL
conn.execute("PRAGMA cache_size=-64000")        # 64MB cache
conn.execute("PRAGMA temp_store=MEMORY")        # temp tables in RAM
conn.execute("PRAGMA mmap_size=268435456")      # 256MB mmap
```

---

## 11. Health Monitoring & Degraded Modes

Multi-agent systems fail. Current research shows failure rates of 41-86% in production multi-agent systems, with 37% of failures being coordination breakdowns. Genesis accepts this reality and designs for it explicitly.

### 11.1 Agent Health Table

```sql
CREATE TABLE agent_health (
    agent TEXT PRIMARY KEY,
    status TEXT NOT NULL,                  -- healthy | degraded | down
    last_heartbeat TIMESTAMP,
    last_success TIMESTAMP,
    last_failure TIMESTAMP,
    failure_reason TEXT,
    quota_used_pct INTEGER,                -- estimated % of Max window consumed
    p50_latency_ms INTEGER,
    p95_latency_ms INTEGER,
    error_rate_pct REAL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Every agent heartbeats every 30 seconds. Sentinel reads this table continuously.

### 11.2 Failure modes per agent

| Failure | Detection | Response |
|---------|-----------|----------|
| **Iris not heartbeating >60s** | agent_health stale | Sentinel pages Fareed via Telegram, attempts Iris restart, orchestrator routes around Iris |
| **Iris responding slowly (p95 >30s)** | agent_health metrics | Iris flagged as degraded; non-urgent work queued; user warned if asking her something |
| **Iris hit Claude rate limit** | stream-json `api_retry` | Queue messages for Iris, surface ETA to user, Sentinel monitors for recovery |
| **Iris MCP server (e.g., IMAP) down** | tool call connection error | Iris responds with limited capability: "I can't read mail right now but I can still check calendar" |
| **Isaac stuck in loop** | same tool called >10x without progress | Sentinel kills Isaac's session, notifies Iris (or Fareed if Iris down) |
| **Oracle hit query quota** | BigQuery API quota error | Notify Iris, who decides whether to interrupt Fareed |
| **Halo failed health check** | HA API unreachable | Auto-restart, log only, Halo commands return "offline" until restored |
| **Sentinel itself down** | Watchdog detects no heartbeat | Watchdog pages Fareed directly, attempts Sentinel restart |
| **SQLite write failure** | Exception on insert | Halt all new agent work, page Fareed, attempt recovery |
| **P14s loses internet** | All external API calls failing | Halo continues (local), all Claude agents fail gracefully, alert Fareed via Telegram from local network if reachable |

### 11.3 Iris-degraded mode protocol

Because Iris is the hub, her failure has the largest blast radius. Mitigation:

1. **Orchestrator checks Iris health before routing.** If Iris is down, orchestrator responds directly: "Iris is currently unavailable [reason: Claude rate limit, ETA 23 minutes]. Want me to route this to Isaac, Oracle, or Sentinel directly, or wait?"

2. **Direct talk to other agents continues.** "Isaac, fix this bug" works regardless of Iris's state. "Oracle, run this query" works. "Sentinel, what's the dbt status?" works. Only routing through Iris is affected.

3. **Sentinel emergency channel.** If Iris is down >10 minutes AND Sentinel detects something critical, Sentinel pages Fareed directly via Telegram, bypassing Iris.

4. **Memory writes still happen.** Messages destined for Iris queue in the bus. When she recovers, she processes the backlog (priority-ordered, summarized so she gets the gist quickly).

5. **Recovery digest.** When Iris comes back online, she generates a "what happened while I was down" summary for Fareed.

### 11.4 The Watchdog (essential)

A tiny zero-dependency Python process runs as a systemd service. Its only job: check Sentinel's heartbeat every 60 seconds.

```python
# /opt/genesis/watchdog.py — runs as systemd service
import time, sqlite3, subprocess, requests

DB_PATH = "/var/lib/genesis/genesis.db"
TELEGRAM_TOKEN = os.getenv("WATCHDOG_TELEGRAM_TOKEN")
TELEGRAM_CHAT = os.getenv("WATCHDOG_CHAT_ID")

while True:
    try:
        conn = sqlite3.connect(DB_PATH)
        row = conn.execute(
            "SELECT last_heartbeat FROM agent_health WHERE agent='sentinel'"
        ).fetchone()
        
        last_hb = datetime.fromisoformat(row[0])
        age = (datetime.now() - last_hb).total_seconds()
        
        if age > 180:  # Sentinel silent for 3+ minutes
            requests.post(
                f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage",
                json={"chat_id": TELEGRAM_CHAT, 
                      "text": f"⚠️ Sentinel down for {age:.0f}s — attempting restart"}
            )
            subprocess.run(["systemctl", "restart", "genesis-sentinel"])
    except Exception as e:
        # Even watchdog failure must be visible
        print(f"WATCHDOG FAILURE: {e}", file=sys.stderr)
    
    time.sleep(60)
```

50 lines. Zero dependencies on the rest of the system. Without this, Sentinel monitoring everyone has no safety net.

### 11.5 Sentinel's expanded mandate

Sentinel monitors:

**Agent health (every 30s):**
- Heartbeat presence
- Process state (claude -p subprocess alive?)
- Response latency (p50/p95/p99)
- Error rate (failed/total)
- Quota consumption per agent

**Agent behavior (continuous):**
- Stuck loops (same tool called >N times)
- Runaway sessions (one session burning >X% quota)
- Anomalous patterns vs baseline
- Failed handoffs (claimed but never acknowledged)

**System health (every 5min):**
- Database write/read latency
- Disk space (WAL can grow if checkpointing fails)
- Memory pressure on P14s
- Ollama responsiveness
- MCP server availability per agent

**Integration health (every 15min):**
- IMAP reachable (Iris)
- BigQuery API reachable (Oracle)
- GitHub reachable (Isaac)
- Home Assistant reachable (Halo)
- Tailscale connectivity

### 11.6 Degraded Cognition (Claude unavailable)

A distinct failure class from "service degraded." When Claude itself is unavailable — rate limit hit, API outage, subscription issue — the agents lose their reasoning capacity entirely. This is **cognition degraded**, and it deserves separate handling because the temptation to "just fall back to a local model" is dangerous without constraints.

**The capability cliff problem.** The reasoning gap between Claude Opus and Qwen 14B (or similar local fallback models) is not linear degradation — it is a cliff. Claude handles architectural judgment, schema inference, intent disambiguation, and implicit context well. Local 14B models are competent pattern-matchers but produce confidently wrong outputs when asked to reason beyond their capacity. The failure mode to avoid: Qwen-as-Isaac finishes a refactor that Claude-Isaac started, the code looks done but is subtly broken, and when Claude returns it has to unfuck what Qwen produced. This is strictly worse than halting and waiting.

**Per-agent degradation profile.** Not all agents degrade equally on Qwen:

| Agent | Degrades on Qwen? | Reason |
|-------|-------------------|--------|
| **Isaac** | Catastrophically | Code requires architectural judgment. Never auto-substitute. |
| **Oracle** | Catastrophically | SQL against unfamiliar schemas needs real understanding. Never auto-substitute. |
| **Iris (PM mode)** | Catastrophically | Delegation requires capability awareness and priority judgment. |
| **Iris (reporter mode)** | Gracefully | Summarizing pre-existing structured data is pattern matching. OK on Qwen. |
| **Sentinel** | Gracefully | Threshold-checking and metric comparison. OK on Qwen. |
| **Halo** | Not applicable | No LLM involved. |

**Policy: halt by default, Qwen substitute only with scoped approval.**

When `claude_quota_exhausted` fires:

1. **Sentinel continues on Qwen** (full duty) — observability survives the outage
2. **Iris switches to reporter-only mode on Qwen** — she can assemble and relay status, not make judgment calls
3. **Isaac, Oracle, Iris-PM halt** — in-progress work saves state, no new work starts
4. **Iris (on Qwen, reporter mode) sends Telegram summary to Fareed:**

```
Claude quota exhausted at 14:32. ETA recovery: 2h 15m.
Paused: Isaac (mid-refactor of staging models, state saved at commit abc123)
        Oracle (2 queries queued, nothing destructive pending)
Still running: Sentinel (monitoring normally, all systems green)
               me (reporting only, not delegating)
Any urgent work requires your instructions. Options:
  - Wait for recovery
  - Approve Qwen-Isaac for the refactor (I'll relay and watch)
  - Approve Qwen-Oracle for SELECT-only queries
  - Halt specific task and restart fresh when Claude returns
```

5. **Fareed responds with scoped approval** — per task, time-limited, revocable
6. **Iris relays approval to affected agents** — they resume with Qwen backing

**trust_config.yaml extension for degraded mode:**

```yaml
agents:
  isaac:
    degraded_mode:
      qwen_auto_enabled: false
      requires_per_task_approval: true
      approval_timeout_hours: 4
      scope_must_be_specified: true
  
  oracle:
    degraded_mode:
      qwen_auto_enabled: false
      requires_per_task_approval: true
      allowed_scopes_if_approved:
        - "SELECT only"
        - "dev_dataset only"
  
  iris:
    degraded_mode:
      qwen_auto_enabled: true
      mode: reporter_only        # cannot do PM-mode delegation on Qwen
  
  sentinel:
    degraded_mode:
      qwen_auto_enabled: true
      mode: normal               # full duties acceptable
  
  halo:
    degraded_mode: not_applicable
```

**Audit logging marks degraded-mode actions explicitly.** Every tool call made by an agent running on Qwen is tagged `cognition: qwen_fallback` in the `tool_calls` table. Post-Claude-recovery, Fareed can review: "show me everything Qwen-Isaac did under my approval" — and decide whether to keep it, revert it, or have Claude-Isaac redo it properly.

**Qwen 14B is loaded on demand, not pre-loaded.** Keeping it in RAM 24/7 costs 10GB for a capability used rarely. On quota exhaustion, Ollama loads Qwen 14B (~30s) before Iris switches to reporter mode. Unloaded automatically after 2 hours of no use post-recovery.

**Key property preserved:** during Claude outages, you still have observability (Sentinel) and reporting (Iris). You lose reasoning capacity but not visibility or control. You can make decisions; you just can't delegate judgment to the agents.

---

## 12. Memory Architecture

Memory is how agents accumulate understanding over time. Without it, every session starts from zero and agents can't learn from yesterday's mistakes or wins. Genesis's memory system is deliberately layered and boring — native Claude Code memory where it works, structured markdown logs where humans need to read and audit, SQLite for programmatic lookups. No graph databases, no embedding services, no black-box vector stores for MVP.

### 12.1 Five memory layers (by purpose)

Each layer answers a different question:

| Layer | Question it answers | Technology | Managed by |
|-------|---------------------|-----------|------------|
| **1. Agent identity** | "Who am I, what's my job?" | `CLAUDE.md` per agent | Fareed (manual) |
| **2. Session context compression** | "What happened last session that I need to know?" | `claude-mem` plugin | Automated |
| **3. Project execution logs** | "What was done on project X on date Y, was it successful?" | Markdown files (`project_YYYY-MM-DD.md`) | Agent (automated) |
| **4. Agent reflections** | "What did I learn today, what should I do differently?" | Markdown files (`agentname_YYYY-MM-DD.md`) | Agent (cron-triggered) |
| **5. Structured facts** | "What is Taqieddin's email? What's our staging prefix?" | SQLite memory table | Agent (explicit stores) |

Each layer has a clear purpose. Nothing overlaps. If you're asking "what happened today," you look at Layer 3. If you're asking "what should I remember about this person," you look at Layer 5. If you're asking "who am I as an agent," Layer 1. Clean separation.

### 12.2 Layer 1: Agent identity (CLAUDE.md)

Each agent has a `CLAUDE.md` file in their working directory that Claude Code auto-loads every session. Standard structure across all agents:

```markdown
# <AGENT_NAME> — <FULL_ACRONYM>

## Identity
Who the agent is, gender (he/him, she/her), core persona.

## Responsibilities  
Scope of work, what this agent owns, what they do not own.

## Escalation
Who this agent reports to, who they listen to, who they can delegate to.

## Tone
How the agent communicates. Direct? Warm? Measured?

## Constraints
What this agent cannot do. Trust level restrictions. Prohibited actions.

## Memory discipline
When to write to Layer 3 (project logs), Layer 4 (reflections), Layer 5 (facts).

## Context references
Links to design doc sections, other agents' CLAUDE.md files, relevant skills.

## Key contacts and conventions
Personalized: Fareed's preferences, key contacts (for Iris),
schema conventions (for Oracle), project portfolio (for Isaac).
```

**Where these files live:**
- Public template versions: `genesis/agents/<name>/CLAUDE.md` — generic, no personal data
- Personalized instances: `genesis-config/agents/<name>/CLAUDE.md` — MUVI context, real contacts, project memory
- The orchestrator symlinks the personalized versions into each agent's cwd at runtime

### 12.3 Layer 2: Session context (claude-mem plugin)

`claude-mem` is a Claude Code plugin that compresses session history using Claude's own agent SDK, then injects compressed context into future sessions. Solves the "every session starts from zero" problem without bloating CLAUDE.md files.

**Installation on P14s:**

```bash
# Install globally for all agents to use
claude plugins add thedotmack/claude-mem

# Start worker service (auto-starts thereafter)
bun ~/.claude/plugins/cache/thedotmack/claude-mem/*/scripts/worker-service.cjs start

# Verify
curl http://localhost:37777/api/health
```

**How it integrates:**
- Runs on port 37777 as a background worker
- Hooks into Claude Code's SessionStart event — automatically injects relevant compressed context
- Each agent's sessions are tracked separately (per project/working directory)
- Web viewer at `http://localhost:37777` shows all observations

**What it gives us:**
- Automatic capture of what agents did across sessions
- AI compression of verbose session transcripts into useful summaries
- Token-efficient context injection (3-layer workflow: search → timeline → get_observations)
- MCP server exposing `search`, `timeline`, `get_observations` tools — agents can query their own history

**Critical benefit:** when Isaac comes back tomorrow to continue work on the portal, claude-mem injects "yesterday you implemented the message bus schema, debugged WAL mode pragmas, left off at xterm.js integration with a known issue in WebSocket reconnect." Isaac doesn't re-investigate; he continues.

### 12.4 Layer 3: Project execution logs

Per-project, per-date markdown file written by the agent doing the work. Format is strict so agents can parse each other's logs.

**Location:** `genesis-config/memory/project-logs/<project>/<project>_YYYY-MM-DD.md`

**Template:**

```markdown
# <Project Name> — <YYYY-MM-DD>

## Agent
<agent name who did this work>

## Asked
<exactly what Fareed asked, or reference to task ticket ID>

## Plan
<brief statement of approach before starting>

## Actions taken
- [command or action 1]
- [command or action 2]
- [etc]

## Testing
<how verified that it worked>

## Outcome
Success | Partial | Failed

## Notes
<anything worth remembering for next time working on this project>

## Trace IDs
<links to message bus traces for detailed replay if needed>

## References
<related project logs, related reflections, external docs>
```

**When written:** after every significant work session on a project. Agent decides granularity — one log per major task completion, not per keystroke.

**Who reads these:**
- The writing agent (for continuity on next session)
- Other agents when working on the same project (for cross-agent coordination)
- Fareed (for review and understanding what happened)
- Sentinel (for anomaly detection across projects)

### 12.5 Layer 4: Agent reflections (automated EOD)

Per-agent daily reflection file. Written automatically via cron at 8pm AST each day. Agent reviews the day's activity and generates a structured reflection.

**Location:** `genesis-config/memory/agent-reflections/<agent>_YYYY-MM-DD.md`

**Template:**

```markdown
# <Agent Name> — Reflection <YYYY-MM-DD>

## Today's work summary
<1-paragraph overview of what I did today across all projects>

## What went well
- <specific positive pattern, with example>
- <another one>

## What didn't go well  
- <specific issue, with example>
- <another one>

## Improvement hypothesis
<what I'll try differently tomorrow, as falsifiable claim>

## Wins to amplify
<what worked unusually well that I should deliberately repeat>

## Observations for other agents
<insights other agents on the team should know about>

## Tomorrow's priorities
<if I know what's coming, what I'd like to focus on>
```

**Automated generation (cron at 8pm AST):**

```python
# /opt/genesis/reflections.py — runs daily at 20:00 AST
for agent in ['isaac', 'oracle', 'iris', 'sentinel']:
    # Skip if agent wasn't active today
    if not agent_active_today(agent):
        continue
    
    # Gather today's evidence
    today_logs = load_project_logs(agent, date=today)
    today_traces = load_traces(agent, date=today)  
    today_stats = load_agent_health_metrics(agent, date=today)
    
    # Invoke agent with reflection task
    prompt = f"""
    Review your work today based on this evidence:
    
    Project logs: {today_logs}
    Metrics: p95_latency={today_stats.p95_ms}ms, error_rate={today_stats.errors}%
    Trace summaries: {today_traces}
    
    Write your reflection using the template at 
    genesis/docs/templates/reflection.md.
    Save to genesis-config/memory/agent-reflections/{agent}_{today}.md.
    Be honest about failures. Don't write generic platitudes.
    """
    
    invoke_agent(agent, prompt)
    
# Halo doesn't reflect (no LLM)
# After all reflections written, auto-commit and push to genesis-config
commit_and_push_reflections()
```

### 12.6 Layer 4.5: Weekly synthesis (automated Sundays)

Every Sunday at 9pm AST, Iris produces a weekly synthesis across all agents' reflections. This is the "wisdom" layer — patterns nobody individual agent sees.

**Location:** `genesis-config/memory/shared-learnings/YYYY-W##-synthesis.md`

Template includes: recurring themes, cross-agent patterns, systemic improvements, Fareed's interaction patterns, things to propagate to shared learnings patterns directory.

### 12.7 Cross-agent visibility (all reflections readable by all)

**Critical design choice:** all reflections are readable by all agents. Cross-pollination is where the compound value lives.

When Isaac's reflection says "I wasted 2 hours today because I didn't verify the BigQuery schema before writing migrations," Oracle's next morning session reads that and her CLAUDE.md automatically includes "check BigQuery schema before writing migration-related SQL" as a learned pattern.

**Mechanism:**
- All reflections are plain markdown in `genesis-config/memory/agent-reflections/`
- Each agent's session start hook reads the last 3 days of reflections from ALL agents
- claude-mem compresses these into relevant context injection
- Shared learnings patterns (distilled over time) live in `genesis-config/memory/shared-learnings/patterns/`

**What if an agent reflection reveals something sensitive?**
- Reflections are in the private `genesis-config` repo (encrypted via git-crypt for sensitive sections)
- Agents can mark reflections as "private: true" in frontmatter if needed (rare)
- Fareed can prune reflections manually if anything gets uncomfortable

### 12.8 Layer 5: Structured facts (SQLite memory table)

For programmatically-queryable atomic facts. Not conversational. Not narrative. Just structured knowledge.

**Schema:**

```sql
CREATE TABLE memory_facts (
    id TEXT PRIMARY KEY,
    key TEXT NOT NULL,
    value TEXT NOT NULL,
    scope TEXT NOT NULL,              -- 'global' | 'agent:isaac' | 'project:portal'
    source_agent TEXT NOT NULL,
    confidence REAL DEFAULT 1.0,
    created_at TIMESTAMP NOT NULL,
    last_verified_at TIMESTAMP,
    last_used_at TIMESTAMP,
    expires_at TIMESTAMP,
    version INTEGER DEFAULT 1,
    previous_version_id TEXT,          -- chain for audit
    status TEXT DEFAULT 'active',      -- active | superseded | forgotten | expired
    metadata JSON
);

CREATE INDEX idx_memory_key_scope ON memory_facts(key, scope) WHERE status='active';
CREATE INDEX idx_memory_scope ON memory_facts(scope) WHERE status='active';
```

**MCP tools exposed:**

```
memory.store(key, value, scope, expires_at=None)
memory.recall(key, scope=None)                    # exact key lookup
memory.search(query, scope=None)                  # keyword search
memory.list(scope=None, prefix=None)
memory.forget(key, scope, reason)                 # audited deletion
memory.correct(key, scope, new_value, reason)     # versioned update
memory.project_context(project)                   # bulk load at session start
```

**When agents write facts (not auto-extracted — deliberate):**
- User explicitly states a durable preference
- A naming convention is established
- A decision is made that future work should respect
- An identifier is learned (email, ID, endpoint)

NOT when: agent is speculating, pattern is tentative, information might be wrong, content is conversational.

**Lifecycle:**
- `expires_at` facts auto-expire (e.g., "Fareed traveling this week" expires in a week)
- Rarely-accessed facts flagged dormant (not deleted, just deprioritized)
- Fareed can `memory.forget(key, reason)` to actively remove
- Sentinel watches for fact contradictions across agents — flags for review

### 12.9 Memory vs message bus vs CLAUDE.md (clarity test)

Where does a given piece of information belong?

| Information | Goes in | Why |
|-------------|---------|-----|
| "Iris is she/her" | CLAUDE.md | Stable identity, read every session |
| "Taqieddin's email is `taqieddin@muvi.sa`" | Memory facts (Layer 5) | Queryable atomic fact |
| "Iris drafted email to Taqieddin at 2:15 PM today" | Message bus | Operational event |
| "Today Isaac refactored 12 staging models successfully" | Project log (Layer 3) | Execution history |
| "I learned to verify BigQuery schema before writing migrations" | Reflection (Layer 4) | Self-improvement pattern |
| "Fareed prefers direct prose over corporate hedging" | CLAUDE.md + memory fact | Stable preference (both layers) |
| "claude-mem worker is healthy on port 37777" | agent_health table | Operational health state |

### 12.10 Memory browser in portal

A portal view for human inspection of all memory layers:

- **Facts browser** — filter by scope, agent, age; manually edit or forget
- **Project log browser** — navigate by project and date
- **Reflection browser** — navigate by agent and date  
- **Weekly synthesis viewer** — browse historical weekly summaries

This is the human-in-the-loop mechanism for keeping memory clean. You read reflections on Sunday evenings and propagate useful patterns into CLAUDE.md files.

### 12.11 MUVI prior work porting

The MUVI Data Command Center (DCC) and Portal Phase 3 work contains context that should flow into Isaac's personalized CLAUDE.md:

**What ports to Isaac:**
- DCC architecture knowledge (agents, orchestration, deployment)
- Portal Phase 3 structure (Node.js server, React frontend, agent definitions)
- OpenClaw orchestration patterns
- Code conventions already established for MUVI work

**What ports to Oracle from Sage/Rex:**
- dbt model structure and conventions (stg_/dim_/fct_ prefixes)
- BigQuery schema knowledge for MUVI datasets
- Looker model relationships
- Data lineage understanding

**What does NOT port:**
- MUVI business strategy (not agents' domain)
- Confidential operational details
- Personnel information beyond direct working contacts

**Porting mechanism:** during Phase 1 bootstrap, Claude Code gathers relevant files from existing MUVI work and distills them into Isaac's and Oracle's personalized CLAUDE.md files. This is a one-time transfer of accumulated context, not an ongoing sync — MUVI projects continue in their own lifecycle.

### 12.12 What we deliberately skip

**Graphiti (temporal knowledge graphs):** evaluated, deferred to Phase 6+. Adds FalkorDB dependency, OpenAI embeddings dependency, 300ms retrieval latency, and 600k-token-per-conversation risk. Benefits (temporal graph reasoning, relationship modeling) don't justify complexity for current scale. Revisit when Genesis has 6+ months of data and demonstrable need.

**Vector search (Qdrant):** originally planned in v0.5 Section 9.2. claude-mem already does semantic search over session history internally. Adding separate Qdrant instance duplicates capability. Keep Qdrant as optional Phase 6+ if specific use cases emerge (e.g., "find everything we discussed about authentication" across years of logs).

**Auto-extraction of facts from conversations:** rejected. Creates noise, confabulates, and agents become uncertain about what they "know." Facts are written deliberately by agents or explicitly requested by Fareed. Quality over quantity.

**Cross-tenant memory (Graphiti-style group_id isolation):** not needed. Genesis is single-user. MUVI work stays in MUVI systems; Genesis is Fareed's personal platform.

---

## 13. Trust Progression & Capability Management

Trust is earned, not granted. Iris doesn't get send permission on day one. Isaac doesn't auto-merge to main on day one. Every agent earns capabilities through demonstrated reliability.

### 12.1 Trust levels

| Level | Capability bias | Iris example | Isaac example |
|-------|----------------|--------------|---------------|
| **Probation** | Read + draft only | Drafts to Drafts folder via IMAP APPEND, you send manually | Opens PRs to feature branches, you review and merge |
| **Supervised** | Action with explicit approval | Send via portal approval per email | Auto-merges to feature branches, requires approval for main |
| **Trusted contexts** | Auto-action in defined safe zones | Auto-send for FYI replies to internal team, drafts for executives | Auto-merges fixes that pass tests in specific repos |
| **Autonomous** | Full action authority within domain | Sends most categories, executives still require approval | Owns specific repos end-to-end (likely permanent for prod code) |

### 12.2 Per-agent trust config (YAML)

Versioned in the GitHub repo. Editable. Promotion is deliberate.

```yaml
# trust_config.yaml
agents:
  iris:
    trust_level: probation
    started: 2026-04-19
    minimum_probation_days: 30
    capabilities:
      mail.read: enabled
      mail.draft: enabled
      mail.send: disabled               # promoted at trust >= supervised
      calendar.read: enabled
      calendar.write: requires_approval
      news.search: enabled
      paperclip.create_ticket: requires_approval
    promotion_criteria:
      - "Zero rejected drafts for 14 consecutive days"
      - "Learned tone preferences for 5+ contacts"
      - "Calendar conflicts flagged accurately 95%+"
    notes: |
      Iris started 2026-04-19. Her drafts during probation help me 
      learn her judgment. Promotion to supervised requires explicit 
      decision after probation period.
  
  isaac:
    trust_level: probation
    capabilities:
      code.read: enabled
      code.write: enabled               # writes to feature branches
      code.merge_to_main: requires_approval
      shell.read: enabled               # ls, cat, grep
      shell.execute: requires_approval  # anything that mutates state
      paperclip.spawn_subagent: requires_approval
  
  oracle:
    trust_level: supervised
    capabilities:
      bigquery.select: enabled          # SELECT only
      bigquery.mutate: requires_approval # INSERT/UPDATE/DELETE
      looker.read: enabled
      looker.modify: requires_approval
  
  sentinel:
    trust_level: trusted
    capabilities:
      monitoring.read_all: enabled
      alerts.notify_iris: enabled
      alerts.notify_user_emergency: enabled  # Telegram bypass for P0
      auto_restart_failed_agent: requires_approval
  
  halo:
    trust_level: autonomous            # Halo has no judgment to develop
    capabilities:
      home.lights: enabled
      home.climate: enabled
      home.media: enabled
      home.security: requires_approval  # door locks, etc.
```

### 12.3 Promotion mechanics

- **Manual, never automatic.** No auto-graduation based on time alone. You explicitly run `genesis trust promote iris supervised` or click in portal.
- **Demotion is one click.** "Trust circuit breaker" — if Iris sends something embarrassing, you instantly drop her trust level. Capabilities revert.
- **Trust history is logged.** Every promotion/demotion goes into `trust_changes` table with reason and timestamp.

### 12.4 Performance review view

The portal includes a "Trust Dashboard" per agent showing:
- Current trust level
- Capabilities at this level
- Time at current level
- Promotion criteria progress
- Recent activity stats:
  - Total interactions
  - Approval/rejection rate (for items needing approval)
  - Edit rate on drafts (how often you change vs. accept)
  - Failure rate
- Eligibility for promotion

This becomes one of the most useful portal views — it's where you decide whether to expand or contract agent autonomy over time.

### 12.5 Iris's progression plan (concrete example)

| Phase | Duration | Iris does | You do |
|-------|----------|-----------|--------|
| Probation | 30-60 days | Drafts everything to Drafts folder | Review, edit, send manually. Each edit teaches her. |
| Supervised | 30-60 days | Drafts + can request to send via portal approval | Tap approve in portal. Drafts get faster, edits decrease. |
| Trusted contexts | After ~3 months | Auto-sends for: FYI replies to internal team, OOO acknowledgments, calendar confirmations | Audit weekly, expand or contract trusted categories |
| Autonomous | When confident | Sends most categories. Executives, external recipients, sensitive topics still require approval. | Iris is now a real PA |

---

## 14. Paperclip Integration Plan (Phase 4+)

**Project Paperclip** (github.com/paperclipai/paperclip) is an open-source orchestration platform for AI agent organizations. It provides org-chart structure, per-agent budgets, audit trails, heartbeats, and approval workflows.

Genesis adopts Paperclip as the **sub-agent orchestration layer** for agents that need to spawn specialists.

### 13.1 What Paperclip handles vs. what Genesis handles

**Paperclip handles:**
- Sub-agent spawning, configuration, lifecycle
- Per-agent budget enforcement (hard caps)
- Audit logs of agent actions
- Org-chart relationships
- Heartbeat-driven scheduled work
- Cross-agent ticketing

**Genesis continues to handle:**
- Voice input (Web Speech API)
- Real-time portal (FastAPI + WebSocket + xterm.js)
- Named agent routing (Qwen attribution)
- Direct interactive agent calls (claude -p)
- Message bus (SQLite)
- Halo (local Python, never touches Paperclip)

### 13.2 Per-agent Paperclip usage

| Agent | Uses Paperclip | Why |
|-------|---------------|-----|
| **Iris** | Heavily (Phase 4+) | Spawns specialist watchers (FeedWatcher-Cinema, EmailWatcher-Brio, etc.). Manages her sub-agent team. |
| **Sentinel** | Heavily (Phase 4+) | Spawns sub-monitors (DbtWatcher-NewModel, CostWatcher-Dataset, PipelineDoctor). Each owns a subsystem. |
| **Isaac** | Hybrid | Direct claude -p for interactive work. Paperclip-dispatched sub-agents for sustained work (multi-day refactors, agent building). |
| **Oracle** | Hybrid | Direct for queries, Paperclip for long-running analyses and recurring reports. |
| **Halo** | Never | Sub-second response requirement. No benefit from ticket-based dispatch. |

### 13.3 Paperclip MCP integration

Paperclip ships an MCP server (`@paperclipai/mcp-server`) that exposes its API as MCP tools. Iris and Sentinel get this in their MCP server list. They can:

- `paperclip.create_ticket(assignee, title, brief, budget)`
- `paperclip.spawn_subagent(role, system_prompt, schedule, budget)`
- `paperclip.list_active_tickets()`
- `paperclip.get_subagent_status(name)`
- `paperclip.terminate_subagent(name)`

### 13.4 Sub-agent quota model (for POC)

**During POC (current decision):** Sub-agents share Max quota with department heads. Simpler, no API key management, observe actual usage patterns.

**Migration trigger:** When Max limits become regularly hit due to sub-agent activity (not just personal coding), flip to API for sub-agent layer. Paperclip's budget enforcement makes this safe — `$3/month hard cap on FeedWatcher-Cinema` means $3, period.

### 13.5 Deployment

Paperclip runs as a Node.js service alongside the FastAPI orchestrator. Adds ~300MB RAM. Embedded Postgres (PGlite) for Paperclip's own state (separate from Genesis's SQLite). Auto-starts via systemd alongside other Genesis services.

---

## 15. Email Integration via IMAP (M365)

**Decision:** Iris connects to MUVI's M365 mailbox via IMAP read + draft (no SMTP send during probation). This is essentially treating Iris as another email client, which is sanctioned by IT policy (same model as Outlook on phone).

### 14.1 Why IMAP, not Microsoft Graph (initially)

- Microsoft Graph requires app registration in Entra (MUVI's tenant), admin consent, and IT involvement. Slow.
- IMAP requires no app registration — just OAuth2 IMAP credentials (or app password).
- IMAP is a sanctioned email client protocol from IT's perspective.
- Future-proof: can migrate to Graph after IT conversation, when Iris has proven value.

### 14.2 Iris's IMAP MCP server

```
iris.list_recent_messages(folder, since)   # IMAP SEARCH + FETCH HEADER
iris.get_message(message_id)                # IMAP FETCH BODY
iris.search_messages(query)                 # IMAP SEARCH
iris.mark_read(message_id)                  # IMAP STORE \Seen
iris.move_to_folder(message_id, folder)     # IMAP MOVE
iris.draft_reply(message_id, body)          # IMAP APPEND to Drafts folder
iris.draft_new(to, subject, body)           # IMAP APPEND to Drafts folder
```

7 tools. ~200 lines of Python wrapping `imapclient` library. No SMTP. Send happens through Outlook (where you review and send the draft).

### 14.3 Send capability (Phase 5+, when trusted)

Once Iris reaches `supervised` trust level, two paths:

**Option A:** Add SMTP capability via M365 SMTP relay (requires app password and outbound SMTP allowed).

**Option B:** Migrate to Microsoft Graph delegated permissions (Mail.Send scope). This is the "right" long-term path and is the trigger for the IT conversation.

Option B is cleaner once the IT conversation happens. Option A is a fallback if Graph is denied.

### 14.4 Calendar (separate problem)

IMAP doesn't do calendar. Calendar access comes via:

- **Phase 3-4:** Calendar sharing — share your MUVI calendar to a personal Google account that Iris reads via Google Calendar API (ugly but works)
- **Phase 5+:** Microsoft Graph `Calendars.Read` permission (smaller IT ask than full mail access)

### 14.5 Compliance considerations

This integration touches MUVI corporate data on personal hardware. Required mitigations:

1. **Disk encryption (LUKS)** on the P14s — non-negotiable before any corporate data lands
2. **Read-only mode during probation** — Iris cannot send, only draft
3. **Audit log of every IMAP operation** — message reads, folder moves, drafts created
4. **Have the IT conversation early** — frame as "I configured an email client on a personal device, same as my phone." Get sanctioned even for IMAP if possible.
5. **No data ever leaves the P14s for Iris's use** — all processing local. Claude API receives only the specific email contents needed for the immediate task, not bulk export.
6. **Sensitivity labels respected** — if M365 marks an email "Confidential," Iris does not include it in summaries shared elsewhere.

---

## 16. Observability & Tracing

Multi-agent systems are notoriously hard to debug. End-to-end tracing is essential, not optional.

### 15.1 Trace IDs

Every user request generates a trace ID. It propagates through:
- The initial message bus entry
- Every agent invocation triggered by it
- Every tool call within those invocations
- Every cross-agent message spawned
- All resulting events in the bus

Stored in the `trace_id` column of `messages`, `events`, `tool_calls`, and `approvals` tables.

### 15.2 Tool call tracking

```sql
CREATE TABLE tool_calls (
    id TEXT PRIMARY KEY,
    trace_id TEXT NOT NULL,
    parent_message_id TEXT NOT NULL,
    agent TEXT NOT NULL,
    tool_name TEXT NOT NULL,
    tool_input JSON,
    tool_output JSON,
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    duration_ms INTEGER,
    success INTEGER NOT NULL,
    error_message TEXT
);
```

### 15.3 Metrics Sentinel tracks

**Per agent:**
- Invocations per hour
- p50/p95/p99 latency
- Error rate
- Estimated quota consumption

**Per tool:**
- Call frequency
- Success rate
- Latency distribution

**System-wide:**
- Message bus depth (pending messages)
- Average message bus latency (created → claimed)
- Cross-agent handoff success rate
- Approval response time (human latency)

### 15.4 Trace viewer in portal

A dedicated portal view shows any trace ID as a flame graph:
- Timeline of all events for that trace
- Tool calls nested under their invoking agent
- Cross-agent messages shown as branches
- Approval points highlighted
- Total duration, total cost (estimated tokens)

When something goes wrong, you click the trace ID and see exactly what happened. This is non-negotiable for debugging multi-agent flows.

### 15.5 Conflict detection

Sentinel watches for:
- Same task assigned to two agents simultaneously
- Contradictory tool calls (one agent reading a file while another is writing)
- Circular dependencies (Agent A waiting on B waiting on A)

These are logged as `agent_conflicts` events for review.

---

## 17. Best Practices Alignment

Current research validates several of Genesis's choices:

| Best Practice | How Genesis Implements It |
|--------------|------------------------|
| 3-7 agents per workflow (research: 20+ underperform) | 5 named agents, sub-agents via Paperclip |
| Hub-and-spoke pattern (DeepMind: 17x error reduction vs flat mesh) | Iris as central hub, supervisor pattern |
| Human-on-the-loop for routine, escalation for edge cases | Trust progression model + approval workflows |
| Bounded autonomy with clear escalation paths | Iris's escalation matrix (P0 to user, code → Isaac+user, transient → Isaac alone) |
| Context engineering with isolation | Per-agent CWD, separate claude -p sessions, no shared context windows |
| Event-driven coordination > tight coupling | Message bus is primary inter-agent communication |
| Comprehensive audit trail | Every message, tool call, approval, agent state change logged |
| Trace IDs across agent chains | Implemented from day 1, not bolted on later |
| Failure mode planning | Explicit per-agent degraded modes, watchdog process |
| Per-agent budgets to prevent runaways | Paperclip enforces hard caps on sub-agents |

Where Genesis diverges from common patterns (intentionally):

- **No external message broker** — SQLite chosen for scale appropriateness
- **No always-on voice hardware** — browser-native STT eliminates that complexity
- **One named agent (Halo) bypasses LLM entirely** — keyword matching for sub-second home control

---

## 18. Hardware Optimization

Target hardware: **P14s Gen 2 — i7-1165G7 (4C/8T, Tiger Lake), 40GB DDR4-3200, Intel Xe graphics (96 EUs integrated), ADATA SX8200 Pro NVMe SSD.** Genuinely capable hardware for this workload; these optimizations extract the full value.

### 17.1 Ollama configuration with Xe acceleration

```bash
# /etc/systemd/system/ollama.service.d/override.conf
[Service]
Environment="OLLAMA_KEEP_ALIVE=-1"              # Qwen 3B always in RAM
Environment="OLLAMA_NUM_PARALLEL=1"              # No concurrent inference
Environment="OLLAMA_MAX_LOADED_MODELS=1"         # One model at a time
Environment="OLLAMA_INTEL_GPU=1"                 # Enable Intel Xe acceleration
Environment="ONEAPI_DEVICE_SELECTOR=level_zero:0" # Select Xe via Level Zero runtime
```

Prerequisites for Xe acceleration:
```bash
# Intel compute runtime + Level Zero
sudo apt install intel-opencl-icd intel-level-zero-gpu level-zero
# Verify GPU is detected by Ollama
ollama run qwen2.5:3b --verbose  # should report GPU layers loaded
```

Result: Qwen attribution drops from ~5s cold to ~80ms warm with Xe acceleration (vs ~200ms CPU-only). Cost: 2.5GB system RAM permanently allocated for Qwen 3B, ~500MB shared with Xe for inference buffers.

### 17.2 Persistent claude -p processes

Instead of spawning fresh subprocess per call, maintain one long-running session per agent using `--input-format stream-json` (bidirectional streaming):

```python
class AgentProcess:
    def __init__(self, agent_name: str, config: dict):
        self.process = await asyncio.create_subprocess_exec(
            'claude', '-p',
            '--input-format', 'stream-json',
            '--output-format', 'stream-json',
            '--verbose',
            '--allowedTools', ','.join(config['tools']),
            '--system-prompt', config['system'],
            cwd=config['cwd'],
            stdin=PIPE, stdout=PIPE, stderr=PIPE
        )
        # Watchdog restarts if dies
        asyncio.create_task(self._monitor())
```

Benefit: Eliminates 2-4s cold start per call. Still worth doing on the 1165G7 — Tiger Lake has strong single-thread performance, but subprocess spawn overhead is real and accumulates across many agent invocations.

### 17.3 SQLite tuning

Full config in Section 10.7. With NVMe backing, baseline performance is already excellent (~20-30μs write latency). The WAL mode + cache + mmap settings are still worth applying — they're free and reduce SSD wear by coalescing writes.

**NVMe endurance:** ADATA SX8200 Pro is rated 640 TBW. At Genesis's expected write volume (~200MB/day sustained, ~500MB/day peak), you'd reach endurance limit in roughly 9 years of continuous operation. Non-concern.

### 17.4 Bare-metal services (not Docker) for hot path

| Service | Deployment | Reason |
|---------|-----------|--------|
| Ollama | Bare metal (apt) | ~200MB RAM saved, Xe acceleration integrates better with host GPU drivers |
| FastAPI orchestrator | systemd service | Native Python, no Docker overhead |
| Watchdog | systemd service | Zero dependencies must include Docker |
| MCP servers | Docker | Isolation matters for untrusted code |
| Paperclip | Docker | Convenient packaging |
| Qdrant (optional) | Docker | Standard packaging |

### 17.5 RAM budget (40GB total)

| Service | Allocation |
|---------|-----------|
| OS + kernel + baseline | 3GB |
| Qwen 3B (always loaded, Xe-accelerated) | 3GB |
| 4x persistent claude -p processes | 2GB |
| FastAPI orchestrator | 500MB |
| SQLite + WAL cache | 500MB |
| Active MCP servers | 2GB |
| Paperclip + Postgres (Phase 4+) | 500MB |
| Docker daemon + containers | 1GB |
| Intel Xe display/inference buffers | 512MB |
| **Total committed** | ~13GB |
| **Headroom** | ~27GB |

The 27GB headroom enables several capabilities that would be constrained on 32GB:

- **Qwen 14B fallback** (~10GB loaded) for Claude-degraded mode becomes a realistic Phase 5+ option rather than theoretical
- **Qwen 32B experiments** possible (~18GB quantized) without memory pressure
- **All MCP servers simultaneously warm** even during peak multi-agent concurrent operations
- **Generous spike absorption** for edge cases (sub-agent storms, bulk operations)

### 17.6 Thermal mitigation

Tiger Lake i7-1165G7 runs cooler than Gen 1 CPUs (15W TDP nominal, ~28W sustained turbo), and Gen 2's thermal design is improved. However the small chassis still has limited headroom:

- **Laptop cooling pad** (~$20) — still recommended; makes a real difference during sustained Qwen inference or long coding sessions. Keeps sustained boost clocks 500-800 MHz higher.
- **Undervolt CPU** via `intel-undervolt` on Linux — typically -80 to -100mV for Tiger Lake, drops temps 5-8°C. Test stability at each step.
- **CPU governor**: `powersave` when idle, `ondemand` otherwise.
- **Disable turbo for background jobs** — Sentinel cron runs don't need peak speed. Limits turbo via `/sys/devices/system/cpu/intel_pstate/no_turbo` during scheduled tasks.

Unlike Gen 1's catastrophic throttling under load, Gen 2 throttles more gracefully — expect CPU to drop from 4.7 GHz to ~3.5 GHz under sustained inference without cooling assistance, rather than the ~2.0 GHz floor Gen 1 would hit.

### 17.7 Don't run portal browser locally

If you're at the P14s, SSH or use the terminal directly. Don't open Chrome to the portal — that's 2GB RAM gone. The portal is for remote access (work laptop, phone). The P14s itself is a headless server.

---

## 19. CI/CD via GitHub

Pull-based deployment (the P14s pulls from GitHub, never receives pushes). **Two repositories** with deliberate separation: public-ready platform code vs. private personal configuration.

### 18.1 Two-repo architecture

**Public repo: `github.com/fareed-hameed/genesis`**
- Starts private during development, flips to public when sanitization checklist passes (see Appendix C)
- License: MIT (maximally permissive, encourages adoption if Genesis becomes useful to others)
- Contains: the platform — orchestrator, MCP servers, portal, agent system prompts (generic), deploy scripts, architecture documentation, tests
- Contains NO: real credentials, real contacts, MUVI-specific context, memory exports, personalized CLAUDE.md files

**Private repo: `github.com/fareed-hameed/genesis-config`**
- Permanently private. Never flips to public.
- Contains: the personal instance — actual `.env` files (encrypted with git-crypt), personalized agent CLAUDE.md files with MUVI context, real trust_config.yaml with current agent state, deployment-specific systemd overrides
- Encryption: **git-crypt** for sensitive files so even repo compromise doesn't leak plaintext
- Anyone else running Genesis creates their own equivalent private repo; never forks this one

The separation means: the public repo is the platform, the private repo is your instance of it. Going public is possible because nothing personal ever touched the public repo.

### 18.2 Public repo structure (`genesis/`)

```
genesis/
├── README.md                       # what Genesis is, how to deploy
├── LICENSE                         # MIT
├── ARCHITECTURE.md                 # link to design doc (sanitized version)
├── orchestrator/                   # FastAPI Python app
├── agents/
│   ├── isaac/
│   │   ├── system_prompt.md        # generic template, not personalized
│   │   └── default_mcp.json
│   ├── oracle/
│   ├── iris/
│   ├── sentinel/
│   └── halo/                       # Python module, no Claude
├── mcp-servers/
│   ├── imap-mail/
│   ├── bigquery/
│   ├── system-health/
│   └── memory/
├── portal/                         # React PWA
├── deploy/
│   ├── docker-compose.yml
│   ├── systemd/                    # service unit files
│   ├── deploy.sh                   # deployment script
│   └── setup-new-instance.sh       # bootstrap script for new users
├── docs/
│   ├── getting-started.md
│   ├── trust-progression.md
│   └── architecture.md
├── .env.example                    # template with placeholders
├── trust_config.example.yaml       # example trust config
└── .github/workflows/
    ├── validate.yml                # lint, typecheck, MCP config validation
    └── release.yml                 # version tagging
```

### 18.3 Private repo structure (`genesis-config/`)

```
genesis-config/
├── .gitattributes                  # tells git-crypt which files to encrypt
├── .env.production                 # ENCRYPTED via git-crypt
├── .env.development                # ENCRYPTED via git-crypt
├── credentials/                    # ENCRYPTED — API keys, OAuth tokens
├── agents/
│   ├── isaac/CLAUDE.md             # personalized — references specific MUVI projects
│   ├── oracle/CLAUDE.md            # personalized — BigQuery dataset names, conventions
│   ├── iris/CLAUDE.md              # personalized — key contacts, preferences
│   └── sentinel/CLAUDE.md          # personalized — infrastructure specifics
├── trust_config.yaml               # current trust state per agent
├── systemd-overrides/              # deployment-specific unit customizations
├── contacts.yaml                   # ENCRYPTED — email allowlists, escalation contacts
└── secrets-README.md               # how to set up git-crypt (for future-you)
```

### 18.4 git-crypt setup (one-time)

```bash
# Install
sudo apt install git-crypt

# Initialize in private repo
cd genesis-config/
git-crypt init

# Generate export key (store in password manager + printed copy)
git-crypt export-key ~/genesis-crypt-key.txt

# Configure which files are encrypted via .gitattributes
cat > .gitattributes << 'EOF'
.env.* filter=git-crypt diff=git-crypt
credentials/** filter=git-crypt diff=git-crypt
contacts.yaml filter=git-crypt diff=git-crypt
*.secret filter=git-crypt diff=git-crypt
EOF

git add .gitattributes
git commit -m "Configure git-crypt encryption"
```

To check that encryption is working: `git-crypt status` — shows which files are encrypted.

### 18.5 Deployment flow

The P14s has both repos cloned side-by-side:

```
/home/nova/
├── genesis/                        # public repo (platform code)
└── genesis-config/                 # private repo (personalization)
```

The orchestrator reads configuration from `../genesis-config/` at runtime — explicit separation enforced at the process level.

**Pull-based deploy cycle:**
1. You push commit to `main` on either repo from any device
2. GitHub Actions runs validation on public repo (lint, typecheck, MCP config validation, smoke tests)
3. P14s polls both repos every 5 minutes via cron
4. On new commit with passing CI: `cd genesis && git pull && cd ../genesis-config && git pull && ./genesis/deploy/deploy.sh`
5. `deploy.sh`: identifies changed services (both repos), rebuilds only affected Docker containers, restarts only affected systemd services, runs DB migrations if needed, runs health checks, notifies you on failure (Telegram)

### 18.6 Rollback

Every deployment tags both repos' previous refs. `genesis rollback` reverts to last-known-good in ~30 seconds. Because repos are independent, you can roll back just code while keeping config changes, or vice versa.

### 18.7 Local development

For changes you want to test before pushing: `genesis dev` starts a local instance with separate database (`genesis-dev.db`), separate ports (8001 instead of 8000), and a copy of current production config. Validate locally, push when satisfied.

### 18.8 When to make the public repo public

See **Appendix C: Public Repo Sanitization Checklist** — every item must pass before flipping visibility. Run `trufflehog` against the full git history to verify no secrets have leaked into historical commits. If anything is found, the answer is not "rewrite history" (error-prone) but "rotate those secrets, note in README, proceed." Historical commits in git are immutable unless you force-push, which breaks forks.

---

## 20. Backups (SQLite → Encrypted Google Drive)

### 19.1 Why snapshot-based, not live sync

A common initial instinct is to rclone-sync the live SQLite files to Google Drive every few minutes, treating Drive as a hot replica. **This does not work for SQLite.** WAL-mode SQLite has three files (main DB + WAL + shared memory) that must be captured atomically. Copying them while writes are active produces files that *look* valid but fail on restore — and the failure only becomes visible during the emergency you need the backup for.

The correct approach is **consistent snapshots** captured via SQLite's online backup API (`VACUUM INTO` or `sqlite3.Connection.backup()`), which safely produces a point-in-time copy while the database is live. Snapshots are then compressed, encrypted, and uploaded to Drive via rclone.

At Genesis's realistic data sizes (~10-100MB), snapshots are fast (<1s) and cheap. A 30-minute cadence gives near-continuous protection without the complexity of true streaming replication.

### 19.2 Architecture

```
┌─ systemd timer (every N minutes, configurable) ─┐
│                                                  │
│  1. sqlite3 VACUUM INTO → consistent snapshot   │
│  2. pg_dump paperclip (Phase 4+)                 │
│  3. tar + zstd compress                          │
│  4. SHA-256 checksum                             │
│  5. rclone copy → gdrive-encrypted:genesis/      │
│  6. Verify (re-download + compare checksum)      │
│  7. Log metadata (timestamp, size, checksum)     │
│  8. Cleanup local staging                        │
└──────────────────────────────────────────────────┘

┌─ daily retention job ───────────────────────────┐
│  Apply tiered retention policy:                  │
│    - Keep all snapshots newer than N days        │
│    - Thin to daily beyond that                   │
│    - Thin to weekly beyond that                  │
│    - Thin to monthly beyond that                 │
│    - Delete anything older than maximum          │
└──────────────────────────────────────────────────┘
```

### 19.3 Fully configurable via .env

```bash
# ─── Backup cadence ───
GENESIS_BACKUP_ENABLED=true
GENESIS_BACKUP_CADENCE_MINUTES=30
GENESIS_BACKUP_REMOTE=gdrive-encrypted:genesis
GENESIS_BACKUP_LOCAL_STAGING=/var/lib/genesis/staging

# ─── Retention (all in days) ───
GENESIS_RETENTION_HOURLY_DAYS=7          # Keep every snapshot for 7 days
GENESIS_RETENTION_DAILY_DAYS=30          # Then one per day for 30 days
GENESIS_RETENTION_WEEKLY_DAYS=365        # Then one per week for a year  
GENESIS_RETENTION_MONTHLY_DAYS=1825      # Then one per month for 5 years

# ─── Safety ───
GENESIS_BACKUP_VERIFY_AFTER_UPLOAD=true      # Re-download and checksum
GENESIS_RESTORE_REQUIRES_CONFIRMATION=true   # No silent restores

# ─── Alerting ───
GENESIS_BACKUP_FAILURE_ALERT=telegram
GENESIS_BACKUP_MISSED_THRESHOLD_HOURS=2      # Alert if no backup in 2h
```

At default settings, steady-state Drive usage is ~20-50GB depending on database growth. Well within the 500GB allocated for Genesis.

### 19.4 Retention math (default policy)

Assuming default 30-minute cadence:

| Tier | Retention | Approx snapshot count |
|------|-----------|----------------------|
| Hourly (every snapshot) | 7 days | 336 snapshots |
| Daily (thinned to one/day) | 30 days | 30 snapshots |
| Weekly (thinned to one/week) | 365 days | 52 snapshots |
| Monthly (thinned to one/month) | 1825 days | 60 snapshots |
| **Total kept** | | **~478 snapshots** |

With typical 40MB encrypted snapshots (100MB DB compressed), total Drive usage ≈ 19GB. Even if the database grows 10x over time, you're still well under the 500GB budget.

### 19.5 What's backed up (criticality matrix)

| Data | Where | Backup mechanism | Criticality |
|------|-------|------------------|-------------|
| Genesis SQLite | `/var/lib/genesis/genesis.db` | Snapshot + rclone | **Critical** |
| Paperclip Postgres (Phase 4+) | Embedded PGlite | pg_dump + rclone | **High** |
| `.env` files | `/etc/genesis/` | In `genesis-config` private repo (git-crypt) | **Critical** (but recreatable) |
| `trust_config.yaml` | `/etc/genesis/` | In `genesis-config` private repo | **High** (but also in SQLite) |
| Agent CLAUDE.md files | `/home/nova/genesis-config/` | In private git repo | **High** |
| Memory exports | In SQLite | Covered by SQLite backup | **Critical** |
| Qwen model files | `~/.ollama/models` | Not backed up (redownloadable) | **Zero** |
| Docker images | Docker cache | Not backed up (rebuildable from Dockerfiles) | **Zero** |
| Application logs | `/var/log/genesis/` | Not backed up (diagnostic only) | **Low** |
| Portal build artifacts | `/var/www/genesis/` | Not backed up (rebuildable) | **Zero** |

**Key insight:** only ~200MB of truly irreplaceable data exists across the entire system. Everything else is either in git or recreatable. This makes backup strategy simple and cheap.

### 19.6 Google Drive setup

Two rclone remotes: one for Drive itself, one encrypted overlay.

```bash
# 1. Configure Drive remote
rclone config
# → new remote → name: "gdrive" → type: "drive" (Google Drive)
# → OAuth flow (needs browser — use X11 forwarding during bootstrap)

# 2. Configure encrypted overlay
rclone config
# → new remote → name: "gdrive-encrypted" → type: "crypt"
# → remote: "gdrive:genesis-backups/"
# → filename_encryption: "standard"
# → directory_name_encryption: true
# → password: [long random passphrase, different from LUKS]
```

**Critical:** the rclone crypt passphrase has the same recovery properties as LUKS — lose it, lose the backups. Store in password manager AND print a physical copy.

### 19.7 Restore procedure

A `genesis restore` command runs the restore script:

```bash
# List available snapshots
genesis restore --list

# Restore to latest
genesis restore --latest

# Restore to specific timestamp
genesis restore --timestamp 20260419T143000Z

# Restore with confirmation (required by default)
# Prompts: "This will replace current state. Type RESTORE to confirm: _"
```

Restore steps (executed by script):

1. List snapshots from Drive (rclone lsjson)
2. Validate selected snapshot exists and checksum matches metadata
3. Stop all Genesis services: `systemctl stop 'genesis-*'`
4. Move current SQLite to `.pre-restore` backup (in case restore is wrong)
5. Download selected snapshot
6. Decompress and decrypt (rclone handles encryption)
7. Verify SHA-256 checksum
8. Replace `/var/lib/genesis/genesis.db`
9. Run SQLite integrity check: `PRAGMA integrity_check`
10. Restart services
11. Verify agent_health shows all agents recovering
12. Report result

**Test quarterly.** An untested restore is theatre. Set a Sentinel recurring reminder.

### 19.8 Failure modes and alerting

The backup job emits events to the message bus. Sentinel subscribes to them:

- `backup_started` — trace ID opened
- `backup_completed` — with size, duration, checksum
- `backup_failed` — with error, attempt count
- `backup_missed` — emitted if scheduled run didn't fire within tolerance

Sentinel alerts Iris via the escalation matrix:
- Single failure → retry, log only
- Two consecutive failures → notify Iris (who may batch into briefing)
- No successful backup in `GENESIS_BACKUP_MISSED_THRESHOLD_HOURS` → direct Telegram to Fareed
- rclone authentication failure → direct Telegram (needs human intervention)

### 19.9 Litestream as Phase 6+ upgrade path

If 30-minute data loss windows ever become unacceptable (unlikely for this use case), Litestream provides continuous WAL replication with sub-minute recovery. 

Litestream doesn't support Google Drive natively, so the upgrade path would be:
- Option A: Switch to Backblaze B2 ($6/TB/month, native Litestream support)
- Option B: Run `rclone serve s3` locally as a Drive → S3 proxy
- Option C: Use Google Cloud Storage bucket (Litestream has GCS support)

Not needed for MVP. Documented here so future-you knows the option exists.

---

## 21. Credential Management

### 20.1 Secrets Storage

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Environment variables** (.env file) | Simple, no dependencies | Plain text on disk | MVP — acceptable on encrypted disk |
| **Infisical** | Nice DX, versioned secrets, audit log | Another service to run | Phase 2+ |
| **HashiCorp Vault** | Industry standard | Heavy for personal use | Overkill for single-machine |

**Hard requirement before any credentials land on the P14s: full-disk encryption (LUKS).** Non-negotiable since the device handles MUVI corporate data.

### 20.2 Credential Scoping

| Service | Scope | Notes |
|---------|-------|-------|
| GitHub | Fine-grained PAT, per-repo | Read-only default; write only on repos Genesis agents actively work on |
| BigQuery | Service account, dataset-scoped | Not project-wide. SELECT-only SA for reads; separate SA for mutations |
| M365 IMAP | OAuth2 IMAP credentials (or app password during MVP) | Read + draft only. No SMTP send permission until Iris reaches `supervised` trust level |
| Google Calendar | OAuth with `calendar.read` (probation), `calendar.events` (supervised) | Scopes expand with trust level |
| Home Assistant | Long-lived access token | Scoped to specific device groups if HA supports it |
| Anthropic | Max subscription (CLI auth) | No API key during POC; migrate sub-agents to API later |
| Telegram bot | Bot token (Sentinel emergency channel + Iris notifications) | Stored in watchdog .env separately for resilience |

### 20.3 Rotation policy

- GitHub PAT: rotate quarterly
- BigQuery SA: rotate quarterly
- IMAP/OAuth tokens: refresh per protocol (automatic for OAuth)
- Telegram bot: rotate annually unless compromised
- Portal bearer token: rotate after any device loss

---

## 22. Security & Audit

### 21.1 Threat model

This is a personal POC, not production. The realistic threats are:

1. **P14s theft/loss** → mitigated by LUKS full-disk encryption
2. **Accidental email send** → mitigated by trust progression (drafts-only during probation)
3. **MCP server compromise** → mitigated by per-agent scoped credentials, Docker isolation
4. **Quota burn from runaway agent** → mitigated by Sentinel monitoring + Paperclip budget caps
5. **Compliance review at MUVI** → mitigated by IMAP-as-email-client framing + audit log
6. **Iris hallucinating wrong recipient** → mitigated by trust progression (sends require approval until trusted)

### 21.2 Audit log

Every tool call is logged against the conversation turn that caused it. The SQLite `events` and `tool_calls` tables capture:

- What was requested (user utterance — verbatim)
- How it was routed (which agent, via which other agent)
- What tools were called (with inputs)
- What was returned
- Whether approval was required, and what was decided
- Total cost and duration
- Trace ID linking the entire chain

Audit log is queryable via the portal. Searchable by trace ID, agent, time range, tool name, recipient (for emails).

### 21.3 Portal Auth

For personal use, a bearer token plus Tailscale device authentication. The token is set in an environment variable on the P14s and must be included in all API requests and WebSocket connections. Tailscale already provides network-level auth (only enrolled devices can reach the P14s).

Phase 5+ upgrade: hardware key (YubiKey) for destructive operations (Iris send, Isaac merge to main, Oracle mutations).

### 21.4 Compliance considerations

| Concern | Mitigation |
|---------|-----------|
| MUVI corporate data on personal device | LUKS encryption + IT conversation + IMAP-as-email-client framing |
| Saudi PDPL data residency | All processing local on P14s; only specific email contents sent to Claude API per request, never bulk export |
| M365 audit visibility | Iris's IMAP access appears in tenant audit logs as standard email client activity |
| Sensitivity-labeled content | Iris does not include "Confidential"-labeled emails in summaries shared elsewhere; treated as read-only context only |
| Anthropic Enterprise POC interaction | Genesis is personal POC, separate from MUVI's Enterprise engagement; no overlap in data handling |

---

## 23. Technology Summary

| Layer | Component | Technology | Runs On |
|-------|-----------|-----------|---------|
| STT | Voice transcription | Web Speech API (browser-native) | Each client device |
| TTS (optional) | Voice output | `speechSynthesis` (browser) or Piper (Docker, Phase 5+) | Browser or P14s |
| Attribution | Transcript parser | Qwen 3B via Ollama | P14s (always loaded) |
| Health checks | Local ecosystem monitoring | Python + system calls (Qwen-labeled but no LLM) | P14s |
| Reasoning | AI brain (4 Claude agents) | Claude Code CLI (`claude -p`, persistent processes) | P14s (Max subscription) |
| Home control | Halo agent | Python keyword matching + HA REST API | P14s (no Claude) |
| Monitoring | Sentinel agent | `claude -p` + cron + agent_health table watcher | P14s |
| Watchdog | Watches Sentinel | Standalone Python systemd service (50 LOC) | P14s |
| Orchestrator | Request routing, lifecycle | FastAPI + Python (bare metal, not Docker) | P14s |
| Message bus | Inter-agent communication | SQLite (WAL mode) + asyncio.Event hybrid push-poll | P14s |
| Sub-agent orchestration | Agent-of-agents (Phase 4+) | Paperclip (`@paperclipai/server`) + embedded Postgres | P14s (Docker) |
| Real-time | Live event stream | WebSocket (FastAPI native) | P14s |
| Terminal render | Browser terminal | xterm.js + `@xterm/addon-attach` | Browser (any device) |
| Portal | Web UI | React PWA (Tailwind) | Served from P14s |
| Remote access | VPN mesh | Tailscale | All devices |
| Notifications | Push alerts | Web Push API + Telegram bot (emergency bypass) | Mobile / browser |
| Email | M365 mailbox access | IMAP read + draft (no SMTP during probation) | P14s |
| Calendar | Google Calendar (Phase 3) → MS Graph (Phase 5+) | OAuth2 | P14s |
| Tools | External services | MCP servers (BigQuery, GitHub, HA, IMAP, etc.) | P14s (Docker) |
| Storage | Sessions, audit, memory, message bus, agent_health | SQLite (WAL mode) | P14s |
| Vector search | Semantic memory (Phase 5+, optional) | Qdrant | P14s (Docker) |
| Secrets | Credentials | .env on encrypted disk (MVP) → Infisical (Phase 5+) | P14s |
| Backups | SQLite + Paperclip Postgres | rclone with crypt remote → Google Drive (every 6h) | P14s → Drive |
| Deployment | CI/CD | GitHub Actions (validation) + pull-based deploy script | GitHub → P14s |
| Containers | Service management | Docker Compose (MCP servers, Paperclip, Qdrant only) | P14s |
| Service supervision | Process management | systemd (orchestrator, watchdog, Ollama) | P14s |

---

## 24. Build Phases

### Phase 1 — Foundation: Isaac + Portal + Message Bus (Weekend 1–2)
- LUKS full-disk encryption on P14s (prerequisite)
- FastAPI orchestrator skeleton with `/api/text` endpoint + WebSocket feed
- SQLite message bus (Section 10): schema, atomic claim, hybrid push-poll
- Persistent `claude -p` process management for Isaac
- Stream-json parsing
- Minimal React portal: input bar + mic button (Web Speech API) + xterm.js terminal
- Tailscale setup on P14s, X1 Carbon, S26 Ultra
- One MCP server (GitHub)
- agent_health table + heartbeat for Isaac
- systemd services for orchestrator + Ollama
- Pull-based deploy script

### Phase 2 — Halo + Attribution + Watchdog (Weekend 3)
- Qwen 3B on Ollama with `OLLAMA_KEEP_ALIVE=-1`
- Transcript attribution (Qwen) for routing
- Halo agent: Python keyword matching + Home Assistant API
- Multi-agent routing (Isaac + Halo simultaneously)
- Conversation mode state machine (sticky agent for follow-ups)
- Watchdog systemd service (the zero-dependency Sentinel-watcher)
- Local ecosystem health checks (Qwen-labeled Python utility)

### Phase 3 — Iris (PA mode only) + Oracle (Week 4)
- Iris agent profile with IMAP MCP server (read + draft only, no send)
- Custom IMAP MCP server (~200 lines, `imapclient` library wrapper)
- Google Calendar via OAuth (sharing-based workaround until MS Graph approval)
- Iris in **probation** trust level — drafts to Drafts folder, no send capability
- Oracle agent profile with BigQuery + Looker MCP servers
- BigQuery custom MCP server (schema introspection, SELECT-only)
- Approval system in portal for any mutation request
- Push notifications (Telegram bot for now)
- Iris morning briefing cron (7:30 AM AST) — generates from messages bus + Sentinel state
- Trust dashboard skeleton in portal

### Phase 4 — Sentinel + Paperclip Foundation (Week 5)
- Sentinel agent with dual mandate (infra + agent ecosystem monitoring)
- Sentinel → Iris message bus reporting (with escalation matrix)
- Sentinel emergency Telegram bypass (P0 + Iris-down conditions)
- Iris EOD summary cron (5:30 PM AST)
- Paperclip installation on P14s (Node + embedded Postgres)
- Paperclip MCP server connected to Iris and Sentinel
- First sub-agent experiment: Iris spawns "FeedWatcher-Cinema" with $2 budget
- Trace viewer in portal (flame graph for any trace_id)
- Backups to Google Drive via rclone (every 6h)

### Phase 5 — Trust Progression + Send Capability (Week 6+)
- Iris promoted to **supervised** after probation review
- Iris send capability via portal approval per email
- Microsoft Graph migration conversation with MUVI IT
- Graph `Calendars.Read` permission (smaller IT ask)
- Graph `Mail.Send` permission (when ready)
- Shared memory MCP server (SQLite-backed)
- Project context caching across agents
- Qdrant vector search (optional, for semantic memory)
- PWA manifest for mobile "Add to Home Screen"
- Audit log viewer in portal
- First sub-agents under Sentinel (DbtWatcher, CostWatcher)

### Phase 6 — Iris as PM + Sub-Agent Expansion (Month 3+)
- Iris in PM mode: receives outcome requests, decomposes, delegates
- Cross-agent project tracking via Paperclip tickets
- Isaac Paperclip integration for long-running technical work
- Oracle Paperclip integration for recurring reports
- Trust progression: agents reach `trusted contexts` for safe categories
- Performance review tooling matures
- Hindsight (long-term memory plugin for Paperclip) integration

### Phase 7+ — Future (Meta glasses, future tech)
- Voice input from Meta glasses via SDK
- Dispatch-and-walk-away workflows
- Iris as ambient assistant (always available, never intrusive)
- Sub-agent count grows organically (10+, 20+, eventually 40+)
- API migration for sub-agent layer when Max quota strain becomes regular
- Possible distributed deployment (second machine for redundancy)

---

## 25. Open Questions

1. **MUVI IT M365 conversation timing.** Schedule before Phase 5 (Graph migration). Approach: present working Iris on IMAP, propose Graph upgrade with specific scopes and concrete benefit.

2. **Claude Code persistent session lifetime.** Should the persistent `claude -p` process per agent be restarted daily (clean context), weekly, or never (until crash)? Likely: daily restart with `/compact` before, to balance freshness vs. cold-start cost.

3. **Sub-agent budget threshold for API migration.** At what monthly Max-quota-consumption-by-sub-agents do we flip them to API? Likely signal: when Sentinel reports Max window saturation >3 days/week due to sub-agent activity.

4. **Arabic support testing.** Web Speech API + Qwen + Claude all handle Arabic. Worth a dedicated test session before relying on it. English-primary for MVP.

5. **Iris news source strategy.** Web search via `claude -p` is simplest but token-heavy. Alternative: dedicated news API (NewsAPI, Google News RSS) or curated RSS via a Paperclip sub-agent. Decide after observing Iris's actual news request patterns.

6. **Qwen 14B fallback for Claude quota exhaustion.** With Gen 2's 40GB RAM and Xe acceleration, loading Qwen 14B on demand (~10GB, ~5-8 tok/s) is now realistic rather than theoretical. Open decision: enable as Phase 5+ capability for true degraded mode, or keep Genesis quota-strict (queue + notify when Claude is rate-limited, no degraded LLM responses). Lean toward enabling — degraded-but-functional beats silence for Iris's scheduled briefings.

7. **Halo learning loop.** Should a "HaloLearner" Paperclip sub-agent run nightly to update Halo's keyword rules based on observed patterns? Compelling but adds complexity. Defer until manual rule updates feel painful.

8. **Approval response latency.** What's the user expectation for approval requests? If you're not at the portal, Iris must wait. Telegram bot with inline approval buttons is one path. Decide based on Phase 3-4 usage.

9. **Sentinel's "agent ecosystem" baseline.** Sentinel needs to learn what "normal" looks like before flagging anomalies. First 30 days = pure observation, no anomaly alerts. Then thresholds get set based on observed baselines.

10. **Conversation mode timeout for Iris.** Default 30s sticky-agent timeout may be too short for PA work (you pause to think between turns). Iris-specific timeout extension to 2-3 minutes worth testing.

---

## Appendix A: Agent Acronym Reference

| Spoken | Full Form | Gender |
|--------|-----------|--------|
| **I.S.A.A.C.** | Initiator for Synthetic Agents & Autonomous Computing | He/him |
| **O.R.A.C.L.E.** | Operational Reasoning & Analytics for Contextual Logic Extraction | She/her |
| **I.R.I.S.** | Intelligent Relay for Information & Scheduling | She/her |
| **S.E.N.T.I.N.E.L.** | Systematic Event Notification & Tracking Intelligence for Network Environment Logging | He/him |
| **H.A.L.O.** | Home Automation & Lifestyle Orchestrator | She/her |

### Pronoun usage guidance

When agents reference each other in cross-agent messages, the message bus, or audit logs, they use each other's pronouns naturally. Example:

- Iris to Isaac: "Sentinel reported a pipeline failure. He's flagged it as transient. Can you verify?"
- Sentinel to Iris: "Halo failed her health check at 14:03. She's back online now."
- Isaac to Iris: "Oracle finished her query. Her results are in the bus."

This is a small touch but makes inter-agent communication feel natural rather than mechanical. Voice synthesis (Phase 5+, optional Piper TTS) will use gender-appropriate voices: a warm female voice for Iris, a measured male voice for Isaac, etc.

## Appendix B: Glossary

- **Genesis** — the platform name; the system as a whole
- **NovaHive** — the parent domain/umbrella under which Genesis and future projects live
- **Hub-and-spoke pattern** — Iris is the hub; other agents are spokes; Iris coordinates passive flow
- **Trust level** — capability tier per agent (probation → supervised → trusted contexts → autonomous)
- **Trace ID** — UUID propagated through every step of a request for end-to-end debugging
- **Verbatim** — the user's exact transcribed words, preserved through agent handoffs to prevent telephone-game distortion
- **Watchdog** — the tiny zero-dependency process that watches Sentinel
- **Probation** — Iris's initial trust phase: read + draft only, no autonomous send
- **MCP** — Model Context Protocol; the standard for agent tool integration
- **Paperclip** — open-source orchestration platform for sub-agent layer (Phase 4+)
- **Cognition degraded** — Claude unavailable; agents restricted to Qwen fallback with explicit per-task approval (Section 11.6)
- **Reporter mode** — Iris operating on Qwen, limited to summarizing and relaying (no judgment calls)
- **Capability cliff** — the non-linear gap between Claude and local model reasoning capacity
- **git-crypt** — transparent file-level encryption for git repos; used in `genesis-config`
- **Hot snapshot** — a SQLite backup taken via the online backup API while writes continue

---

## Appendix C: Public Repo Sanitization Checklist

Before flipping `github.com/fareed-hameed/genesis` from private to public, **every item must pass**. This checklist exists because once content is public, it's effectively forever — archive sites and forks capture it immediately.

### Secrets and credentials

- [ ] No API keys in current code (`grep -ri "api.key\|secret\|token" .`)
- [ ] No real passwords, even in example configs
- [ ] `.env.example` contains only placeholders like `YOUR_TOKEN_HERE`
- [ ] Historical commits scanned with `trufflehog` — zero findings
- [ ] Historical commits scanned with `git-secrets` — zero findings
- [ ] If any secrets appear in history: rotated and invalidated upstream (never try to rewrite git history to "hide" them — assume leaked)

### Personal information

- [ ] No real email addresses (personal or corporate)
- [ ] No real phone numbers
- [ ] No real names of contacts, colleagues, or reports
- [ ] No home addresses or location data
- [ ] Username in examples is generic (e.g., `nova` or `user`), not personal

### Corporate / MUVI-specific content

- [ ] No references to MUVI Cinemas by name
- [ ] No references to specific MUVI projects, internal systems, or tooling
- [ ] No BigQuery project IDs, dataset names, or table names from MUVI's environment
- [ ] No internal hostnames, IPs, or network topology
- [ ] No references to MUVI's IT policies or internal procedures
- [ ] No business context that could be considered MUVI IP

### Instance-specific content

- [ ] No real trust_config.yaml (only the `.example` version with placeholder agent state)
- [ ] No memory exports or conversation history
- [ ] No audit logs from personal use
- [ ] No actual agent CLAUDE.md files (only templates)
- [ ] No personal preferences embedded in system prompts

### Documentation sanitization

- [ ] Architecture doc (if included) references "the user" instead of "Fareed"
- [ ] Examples are generic scenarios, not real situations
- [ ] No screenshots containing personal data
- [ ] No references to specific physical hardware serial numbers, MAC addresses
- [ ] No references to personal domain `novahive.cloud` as authoritative (mention as example)

### Legal and metadata

- [ ] LICENSE file committed (MIT)
- [ ] README explains what Genesis is, how to run your own instance
- [ ] CONTRIBUTING.md exists if accepting contributions
- [ ] CODE_OF_CONDUCT.md if community engagement expected
- [ ] Copyright headers on source files use generic attribution or user's legal name as chosen
- [ ] No trademarked names used in a way that suggests endorsement

### Technical hygiene

- [ ] All tests pass in a clean checkout
- [ ] `setup-new-instance.sh` tested by running it in a fresh VM
- [ ] Documentation explains every configuration option in `.env.example`
- [ ] No hardcoded paths referencing `/home/fareed` or similar
- [ ] Docker images buildable from scratch (no reliance on cached personal layers)

### Final checks before flip

- [ ] Run `trufflehog git file://. --only-verified` — zero verified secrets
- [ ] Run `gitleaks detect` — zero findings
- [ ] Review the last 20 commit messages for personal references
- [ ] Check all issue/PR comments for anything personal
- [ ] Review `git log --all --format="%an <%ae>"` for correct author attribution

When all checked: flip visibility in repo settings. Announce or don't — your call. Do not delete the repo or force-push after going public; treat it as permanent.

### If a secret leaks after going public

This will happen eventually to someone. The response:

1. **Rotate the secret immediately** — assume it's already compromised
2. **Do not force-push to "erase history"** — archive sites and forks already have it, this just creates confusion
3. **Document the rotation in an ADVISORY.md** file — transparency is better than theatrics
4. **Update any deployment instructions** that referenced the old secret
5. **Move on** — past mistakes don't invalidate future work

---

*End of v0.5.4 design document.*
