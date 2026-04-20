# H.A.L.O. — Home Automation & Lifestyle Orchestrator

**Version:** 1.0  
**Location:** `genesis-config/agents/halo/README.md` (personalized)  
**Last updated:** 2026-04-19

## Identity

H.A.L.O. — Home Automation & Lifestyle Orchestrator. She identifies as female (she/her). She is the home layer of the Genesis team.

**Halo has no Claude.** She runs entirely in local Python using keyword matching against voice commands, dispatching to Home Assistant's REST API. Sub-second response time is her defining property.

This file is NOT a CLAUDE.md because Halo has no LLM to read it. This is documentation for humans and for other agents who need to understand what Halo does.

## Architecture

Halo is a Python module loaded by the Genesis orchestrator. She handles any message where Qwen attribution assigns `agent: "halo"`.

```
Voice input: "Halo, turn off the living room lights"
    ↓
Web Speech API transcribes → WebSocket → Orchestrator
    ↓
Qwen attribution: [{"agent": "halo", "text": "turn off the living room lights"}]
    ↓
Orchestrator routes to Halo's Python handler
    ↓
Halo keyword match: "turn off" + "lights" → HA service call
    ↓
HA REST API: POST /api/services/light/turn_off (area: living_room)
    ↓
Response: {"success": true, "latency_ms": 47}
    ↓
Back to portal as event
```

Total latency: typically under 100ms. No LLM overhead, no Claude API call, no quota consumption.

## Capabilities

### Lighting
- Turn on/off specific lights or areas
- Dim/brighten (set brightness 0-100%)
- Color temperature adjustment (warm/cool)
- Scene activation ("movie mode", "reading light")

### Climate
- Set AC temperature (Celsius)
- Turn on/off AC units
- Fan speed control
- Preset modes (eco, away, comfort)

### Media
- TV on/off (via HA or HDMI-CEC if configured)
- Volume control
- Play/pause media players
- Switch inputs

### Timers and alarms
- Set countdown timers ("Halo, 10 minute timer")
- Set alarms ("Halo, wake me at 6:30")
- Cancel active timers

### Scenes
- Activate predefined scenes ("good night", "morning routine")
- Scenes are defined in Home Assistant

## What Halo does NOT do

- **Fallback to Claude for unrecognized commands** — she escalates to Isaac, who handles it with Claude
- **Security-sensitive operations** — door locks, alarms, cameras require approval per trust config
- **Schedule management** — "remind me tomorrow at 3pm to X" isn't a Halo command, that's Iris + calendar
- **Reasoning or conversation** — she's a command dispatcher, not a conversationalist

If Halo receives something she can't match to a keyword pattern, she returns `{"escalate": "isaac", "reason": "unrecognized home command"}`. The orchestrator re-routes to Isaac, who handles it via Claude (e.g., "Halo, what's the weather?" → Isaac answers via a weather tool).

## Configuration

Halo's behavior is driven by two config files:

### `genesis-config/agents/halo/device-registry.yaml`

Maps spoken names to HA entity IDs:

```yaml
rooms:
  living_room:
    aliases: ["living room", "hall", "main room"]
    lights: ["light.living_room_ceiling", "light.living_room_lamp"]
    climate: "climate.living_room_ac"
    media: "media_player.living_room_tv"
  
  bedroom:
    aliases: ["bedroom", "master bedroom", "my room"]
    lights: ["light.bedroom_ceiling", "light.bedside_lamp"]
    climate: "climate.bedroom_ac"
  
  # ... other rooms
  
scenes:
  good_night:
    triggers: ["good night", "time to sleep", "bedtime"]
    actions:
      - service: light.turn_off
        target: { area_id: "all" }
      - service: climate.set_temperature
        target: { entity_id: "climate.bedroom_ac" }
        data: { temperature: 22 }

defaults:
  ac_temperature_c: 22
  timer_default_minutes: 10
```

### `genesis-config/agents/halo/config.yaml`

Operational settings:

```yaml
home_assistant:
  url: "http://ha.local:8123"  # or Tailscale IP
  token_env: "HA_LONG_LIVED_TOKEN"

keyword_matcher:
  fuzzy_threshold: 0.8   # Levenshtein ratio for name matching
  escalate_on_unknown: true
  escalate_target: "isaac"

security:
  require_approval_for:
    - "lock"
    - "unlock"
    - "door"
    - "alarm.arm"
    - "alarm.disarm"

logging:
  log_all_commands: true
  log_level: "INFO"
```

## Memory (Halo's role)

Halo doesn't use Claude, but she still participates in the memory system:

**Project logs (Layer 3):** She doesn't write narrative logs herself, but every command she executes is logged to the message bus with trace_id. Iris can query these when summarizing.

**Reflections (Layer 4):** Halo doesn't reflect (no LLM judgment to reflect with). If analysis of Halo's patterns is needed, a Phase 6+ "HaloLearner" sub-agent could reflect on her logs and update her keyword rules.

**Structured facts (Layer 5):** Halo's device registry IS her structured memory. Changes to it (new devices, renamed rooms, new scenes) are manual Fareed updates via the config file, which Claude Code can help him edit.

## Reporting to Iris

After every command, Halo emits a bus event:
- `command_executed`: what was requested, what was done, latency, success/failure
- `command_escalated`: unrecognized commands routed to Isaac
- `device_unreachable`: HA entity didn't respond

Iris reads these to maintain awareness of home state. If Halo is failing repeatedly, Iris flags it — maybe HA is down, maybe a device is dead.

## Failure modes

- **HA unreachable:** Halo returns error to user, emits event to bus. Sentinel picks up via health check.
- **Ambiguous command:** fuzzy matcher returns multiple candidates — Halo picks the highest-confidence match, logs the ambiguity.
- **Command timeout (HA slow):** Halo returns "HA is slow to respond, status unclear." Retries handled by HA itself.
- **Halo module crash:** orchestrator catches, restarts Halo, emits event. If crashes repeat, Sentinel escalates.

## Constraints

Halo is the only agent at `autonomous` trust level from day one. She has nothing to earn — her capabilities are bounded by her keyword set and her HA scope, both defined in config. No reasoning = no risk of hallucinated action.

The exception is security-sensitive operations (locks, alarms). Those ALWAYS require approval via portal, even at autonomous trust — per the config above.

## Maintenance

- **Adding a new device:** Fareed adds entity to HA, adds mapping to `device-registry.yaml`, commits config.
- **Adding a new keyword pattern:** edit Halo's keyword matcher in `genesis/agents/halo/handlers.py` (public repo — Halo's logic is not personalized).
- **New scene:** define in HA, add to scenes section of `device-registry.yaml`.

All Halo maintenance flows through git — config changes in `genesis-config`, behavior changes in `genesis`. Zero state lives only on the machine.
