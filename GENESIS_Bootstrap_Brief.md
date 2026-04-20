# Genesis Bootstrap Brief

**For:** Claude Code (running on Fareed's X1 Carbon or equivalent)
**Purpose:** Bootstrap Genesis on a fresh Ubuntu 24.04 P14s Gen 2
**Companion document:** `GENESIS_System_Design_v0.5.4.md` (the architecture — read this first)
**Version:** 1.0
**Approach:** Iterative, checkpoint-driven, resumable across sessions

---

## 0. Read this first (Claude Code)

You are helping Fareed build Genesis — a personal AI assistant platform — on a fresh Ubuntu 24.04 P14s Gen 2 server. The full architecture is in `GENESIS_System_Design_v0.5.4.md`. Read it before starting any phase work.

This is a multi-day build with interruptions. Fareed will frequently need to stop and resume. You MUST maintain state so each resume is smooth. The checkpoint mechanism below is non-negotiable.

**Your operating principles for this project:**

1. **Checkpoint relentlessly.** After every non-trivial action (install, configure, create file, modify system), update `bootstrap-state.yaml` and append to `bootstrap-log.md`. Commit to git after every logical unit of work. Losing context should cost minutes of resume time, not hours of rework.

2. **Ask before destructive operations.** Deleting files, dropping databases, overwriting configs — always confirm with Fareed first, even if it seems obvious. Show exactly what you're about to do.

3. **Batch questions at phase boundaries.** Don't interrupt Fareed every 5 minutes with trivia. Accumulate decisions needed for the next phase, ask them all at once when starting a phase.

4. **Reference the design doc as authority.** If something isn't clear, re-read the relevant section of v0.5.4 rather than guessing. If the doc and this brief conflict, ask Fareed to resolve — don't invent a synthesis.

5. **Stop at phase boundaries by default.** After completing a phase, summarize what was done, verify everything works, then stop and wait for Fareed to say "continue to Phase N+1." Don't auto-advance.

6. **Surface errors, don't paper over them.** If `apt install` fails, stop and report. Don't try 10 different workarounds silently. Fareed can help debug, or tell you which approach to try next.

7. **No creative interpretation of architectural decisions.** If a decision was made in the design doc (IMAP not Graph, snapshot not live-sync, halt-not-fallback for Qwen), implement it as specified. If you disagree, raise it as a question, don't act on the disagreement.

---

## 1. Session start protocol (every session)

### 1.1 When starting a fresh session

Before doing anything else, check whether this is a new bootstrap or a resume:

```bash
# Does the config repo exist with state files?
ls ~/genesis-config/bootstrap-state.yaml 2>/dev/null
ls ~/genesis-config/bootstrap-log.md 2>/dev/null
```

- If BOTH files exist → this is a **resume**. Go to section 1.2.
- If NEITHER exists → this is a **fresh bootstrap**. Go to section 2.
- If one exists but not the other → something went wrong last time. Tell Fareed and wait for guidance.

### 1.2 Resume protocol

When resuming an existing bootstrap:

1. **Read `bootstrap-state.yaml`** — identify current phase, last completed step, any blockers
2. **Read the last 50 lines of `bootstrap-log.md`** — understand what just happened
3. **Check service health** if previous work was past Phase 1 — `systemctl status genesis-*` to see what's running
4. **Produce a resume summary** for Fareed:

```
Resuming Genesis bootstrap.
Last session ended: <timestamp>
Last completed: <phase and step>
In progress: <if any>
Known blockers: <if any>
Services currently running: <list>

Next step would be: <what comes next>
Continue? (yes / change approach / review design doc first)
```

5. **Wait for Fareed's response** before doing anything.

Never assume "just continue from the top" is safe. Always summarize and confirm.

---

## 2. Initial questions (fresh bootstrap only)

At fresh bootstrap start, ask Fareed these questions **all together** before any work begins. Do not proceed until answered:

### 2.1 Connection details
- **P14s IP address?** (Fareed will provide — usually 192.168.x.x on home network)
- **Ubuntu username on P14s?** (the one created during install)
- **SSH password?** (one-time, for initial connection before key auth is set up — Fareed will paste it when you're about to connect, not store it)
- **Expected hostname for P14s?** (will be set to `genesis-p14s` by default unless Fareed prefers otherwise)

### 2.2 GitHub
- **Confirm GitHub username is `fareed-hameed`?**
- **Do both repos exist yet?** (`fareed-hameed/genesis` and `fareed-hameed/genesis-config`)
  - If NO: provide Fareed a direct link to create them, wait for confirmation
  - If YES: continue
- **GitHub Personal Access Token (classic or fine-grained)?** Needs `repo` scope. Fareed provides when asked — do not store in chat, use it only to configure git credentials on P14s via credential helper

### 2.3 Network
- **Will Tailscale be used from day one?** (Design doc says yes — confirm)
- **Does Fareed have a Tailscale account?** (if not, he creates one at tailscale.com before Phase 1 completes)
- **DNS provider for `novahive.cloud`?** (Cloudflare recommended in design doc — confirm)
- **Is domain already purchased?** (answer probably yes since domain was mentioned)

### 2.4 Deferred (don't ask now — ask at start of relevant phase)
- Google Drive account for backups — Phase 4
- Telegram bot token for alerts — Phase 2
- M365 IMAP credentials — Phase 3
- BigQuery service account — Phase 3
- Home Assistant token — Phase 2

### 2.5 LUKS and encryption
- **LUKS passphrase stored?** Verify Fareed has it in his password manager. If not, tell him to store it before proceeding — non-recoverable if lost.

Once all Section 2.1-2.3 answered, proceed to Section 3.

---

## 3. Checkpoint system (set up first, before any real work)

The checkpoint system must exist before any significant changes are made to the P14s. This ensures you can always recover.

### 3.1 Connect and set up basic access

1. SSH to the P14s with provided credentials: `ssh <username>@<ip>`
2. Verify you're in: `hostname && whoami && pwd`
3. Set up a working directory for bootstrap: `mkdir -p ~/genesis-bootstrap && cd ~/genesis-bootstrap`

### 3.2 Create the state files locally first

Before cloning or creating the `genesis-config` repo, create initial state files in the working directory:

**`~/genesis-bootstrap/bootstrap-state.yaml`:**
```yaml
schema_version: 1
bootstrap_started: <UTC timestamp>
last_updated: <UTC timestamp>

current_phase: 0
current_step: "pre-phase-1-setup"
status: in_progress

phases:
  phase_0_prerequisites:
    status: in_progress
    started: <UTC timestamp>
    completed: null
    steps: []
  phase_1_foundation: { status: not_started }
  phase_2_halo_attribution: { status: not_started }
  phase_3_iris_oracle: { status: not_started }
  phase_4_sentinel_paperclip: { status: not_started }
  phase_5_trust_send: { status: not_started }
  phase_6_iris_pm: { status: not_started }

environment:
  p14s_ip: <from Section 2.1>
  p14s_hostname: <from Section 2.1>
  user: <from Section 2.1>
  github_user: fareed-hameed
  domain: novahive.cloud
  portal_subdomain: genesis

blockers: []
pending_questions: []
```

**`~/genesis-bootstrap/bootstrap-log.md`:**
```markdown
# Genesis Bootstrap Log

## <UTC timestamp> — Bootstrap initiated
- Session started from Claude Code on <device>
- Connected to P14s at <ip> as <user>
- Hardware confirmed: <output of uname -a, lscpu | head, free -h>
- OS: <output of lsb_release -a>

## <UTC timestamp> — Created bootstrap working directory
- /home/<user>/genesis-bootstrap/
- bootstrap-state.yaml initialized
- bootstrap-log.md initialized
```

### 3.3 Checkpoint update rules

After EVERY significant action, update both files:

**Update `bootstrap-state.yaml`:**
- `last_updated` timestamp
- Current phase/step
- Mark completed steps
- Add blockers if anything fails

**Append to `bootstrap-log.md`:**
- Timestamp header
- What was done
- Commands run (the important ones)
- Output summary (any warnings/errors)
- Any decisions made

### 3.4 Git commit rhythm

Once `genesis-config` repo is cloned (happens in Phase 0), move the state files INTO it and commit after every phase step:

```bash
cd ~/genesis-config
git add bootstrap-state.yaml bootstrap-log.md
git commit -m "bootstrap: <what just happened>"
git push
```

This makes state survive P14s disk loss entirely. You can bootstrap a new machine by cloning `genesis-config` and reading the log.

### 3.5 Recovery from corrupted state

If `bootstrap-state.yaml` becomes inconsistent (e.g., crashed mid-update):
1. Tell Fareed — don't silently repair
2. Show him current state file contents and recent log entries
3. Let him decide: "repair to X" or "restore from previous git commit"

---

## 4. Phase 0 — Prerequisites (before any Genesis-specific work)

Read section 2 of this brief plus sections 17 and 23 of the design doc before starting.

### 4.1 Verify environment

On the P14s, check:
- Ubuntu version: `lsb_release -a` (expect 24.04 LTS)
- Disk: `lsblk -f` (expect LUKS + LVM encryption active)
- RAM: `free -h` (expect ~40GB)
- CPU: `lscpu | grep "Model name"` (expect Intel i7-1165G7)
- Network: `ping -c 2 archive.ubuntu.com` (must succeed)
- Time zone: `timedatectl` (should be Asia/Riyadh, if not, set it)

Record in the log. If anything doesn't match expectation, stop and ask.

### 4.2 System updates and essentials

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim htop ncdu jq unzip build-essential
```

### 4.3 Create the `nova` service user

Per the design doc (Section 22), Genesis runs as dedicated `nova` user:

```bash
sudo useradd -m -s /bin/bash -G sudo nova
# Set a password Fareed will also have in password manager
# Copy SSH authorized_keys from Fareed's user to nova user after key setup
```

**Ask Fareed** before setting nova's password. He picks it.

### 4.4 SSH hardening

Once Fareed's SSH key is copied (use `ssh-copy-id` or manual append to `~/.ssh/authorized_keys`):

1. Verify key auth works: disconnect, reconnect via key, confirm no password prompt
2. Harden `/etc/ssh/sshd_config`:
   ```
   PasswordAuthentication no
   PermitRootLogin no
   PubkeyAuthentication yes
   ```
3. Restart SSH: `sudo systemctl restart ssh`
4. **Verify key auth still works from a new connection before ending the current session** — this is critical. If you break SSH and the session disconnects, Fareed will have to physically access the P14s to recover.

### 4.5 Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow in on tailscale0  # after Tailscale install
sudo ufw enable
```

### 4.6 Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

To bring Tailscale up, you'll need Fareed to authenticate in a browser. Ask him to be ready, then:

```bash
sudo tailscale up --hostname=genesis-p14s
# Opens a URL, Fareed authenticates via browser on any device
```

After auth:
```bash
sudo tailscale set --ssh  # enables Tailscale SSH
tailscale ip -4           # note the tailnet IP
```

Update `/etc/hosts` on Fareed's other devices (X1 Carbon, phone) or rely on MagicDNS.

### 4.7 Domain + DNS setup

**Ask Fareed before proceeding:**
- Is `novahive.cloud` DNS managed at Cloudflare or elsewhere?
- If elsewhere, is he willing to move to Cloudflare? (Recommended for cert automation.)

Then:
1. Get Fareed to add a CNAME record: `genesis.novahive.cloud` → `genesis-p14s.<tailnet>.ts.net`
2. Verify DNS resolves: `dig genesis.novahive.cloud`

### 4.8 Clone repos

```bash
# As the primary user first, later move to nova
cd ~
git clone git@github.com:fareed-hameed/genesis.git           # public repo (may be empty)
git clone git@github.com:fareed-hameed/genesis-config.git    # private repo (empty)
```

If repos don't exist yet, tell Fareed to create them (empty, no README, no .gitignore — you'll populate). Wait for confirmation.

### 4.9 Move state files into genesis-config

```bash
cp ~/genesis-bootstrap/bootstrap-state.yaml ~/genesis-config/
cp ~/genesis-bootstrap/bootstrap-log.md ~/genesis-config/
cd ~/genesis-config
git add .
git commit -m "bootstrap: initial state files"
git push
```

From now on, `genesis-config/` is the source of truth for state.

### 4.10 Install Node.js 22 and Claude Code

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g @anthropic-ai/claude-code
```

### 4.11 Authenticate Claude Code on P14s

**This requires browser OAuth.** Two approaches:

- **If Fareed is SSH'd with X11 forwarding** (`ssh -X`): run `claude` on P14s, browser opens on X1 Carbon, Fareed completes OAuth
- **If not**: use device-flow auth if available, or Fareed briefly connects monitor/keyboard

Verify: `claude -p "Say hello"` returns a response. If yes: **OAuth succeeded, P14s now has Max subscription auth**.

### 4.12 Install git-crypt

```bash
sudo apt install -y git-crypt
cd ~/genesis-config
git-crypt init
git-crypt export-key ~/.genesis-crypt-key
```

**Tell Fareed to save the key file contents to his password manager NOW.** Then:

```bash
# On any other machine where he might work:
git-crypt unlock ~/.genesis-crypt-key
```

### 4.13 Create .gitattributes for encryption scope

```
.env.* filter=git-crypt diff=git-crypt
credentials/** filter=git-crypt diff=git-crypt
contacts.yaml filter=git-crypt diff=git-crypt
*.secret filter=git-crypt diff=git-crypt
```

### 4.14 Phase 0 completion checkpoint

Before declaring Phase 0 complete, verify:

- [ ] SSH from X1 Carbon via key auth works
- [ ] Tailscale up and P14s reachable from another device by hostname
- [ ] `genesis.novahive.cloud` DNS resolves to tailnet IP
- [ ] `claude -p "test"` works on P14s (Max auth confirmed)
- [ ] Both git repos cloned and pushable
- [ ] git-crypt initialized and key saved
- [ ] `bootstrap-state.yaml` shows Phase 0 as complete
- [ ] `bootstrap-log.md` has comprehensive entries
- [ ] All committed and pushed to `genesis-config`

**Stop here. Tell Fareed: "Phase 0 complete. Summary: <list of accomplishments>. Ready for Phase 1?"**

---

## 5. Phase 1 — Foundation: Isaac + Portal + Message Bus

Read design doc Section 23 Phase 1, plus Sections 2 (Architecture), 4 (Agent Routing), 5 (Claude Code CLI), 6 (Portal), 10 (Message Bus) before starting.

### 5.1 Batch questions for Phase 1

Ask Fareed at phase start:
- Which GitHub repo(s) should Isaac have access to first? (start narrow — just the genesis repos themselves)
- What port should the orchestrator listen on? (default 8000)
- What port should the portal serve from? (default 3000, Caddy proxies 443 → 8000)
- Verify Caddy is acceptable for HTTPS (design doc recommends this)

### 5.2 Install foundational services (bare metal)

Per design doc Section 17.4:
- Python 3.12 via pyenv
- Ollama (apt install, configure Xe acceleration per Section 17.1)
- Caddy for HTTPS reverse proxy
- SQLite (likely already installed, verify)

### 5.3 Set up systemd directory structure

```
/etc/systemd/system/
├── genesis-orchestrator.service
├── genesis-watchdog.service
└── ollama.service.d/override.conf  (Xe config)
```

### 5.4 Build the orchestrator skeleton

Per design doc Section 2.2, 10:
- FastAPI app in `genesis/orchestrator/`
- SQLite message bus with schema from Section 10.2
- WebSocket feed endpoint
- WAL mode + pragmas from Section 10.7
- agent_health table from Section 11.1
- Heartbeat emission

### 5.5 Build Isaac agent runner

Per design doc Section 4.3, 5, 17.2:
- Persistent `claude -p` subprocess management
- Stream-json parsing
- Isaac system prompt (generic version in public repo, personalization in config repo's `CLAUDE.md`)
- Working directory: `/home/nova/projects`
- MCP servers: github, memory

### 5.6 Build minimal portal

Per design doc Section 6:
- React PWA scaffold
- Input bar + mic button (Web Speech API)
- xterm.js terminal view
- WebSocket connection to orchestrator

### 5.7 Caddy config

Per design doc Section 7 + the Caddy + Cloudflare DNS-01 discussion:
```
genesis.novahive.cloud {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    reverse_proxy localhost:8000
}
```

Needs Cloudflare API token — ask Fareed.

### 5.8 Watchdog

Per design doc Section 11.4 — the 50-line zero-dependency script.

### 5.9 Smoke tests

- Type "hello" in portal → reaches orchestrator → reaches Isaac → response appears in terminal view
- Verify trace ID propagates through message bus
- Verify agent_health shows Isaac heartbeating
- Verify portal accessible at `https://genesis.novahive.cloud` with valid cert

### 5.10 Phase 1 completion checkpoint

- [ ] Orchestrator running as systemd service
- [ ] Isaac responds via portal
- [ ] Message bus writing/reading correctly
- [ ] Watchdog running
- [ ] Portal accessible remotely via HTTPS
- [ ] All code committed to `genesis` repo
- [ ] All config committed to `genesis-config` repo
- [ ] `bootstrap-state.yaml` updated
- [ ] `bootstrap-log.md` updated

**Stop. Report. Wait for "continue to Phase 2."**

---

## 6. Phase 2 — Halo + Attribution + Watchdog polish

Read design doc Section 23 Phase 2, plus 4.1 (Attribution), 4.5 (Halo).

### 6.1 Batch questions
- Home Assistant already running somewhere? (HA URL + long-lived token needed)
- If not, defer Halo — just do Qwen attribution in this phase
- Telegram bot for alerts? (needs bot token from @BotFather + chat ID)

### 6.2 Install Qwen 3B via Ollama with Xe acceleration

Per design doc Section 17.1:
```bash
# Intel compute runtime
sudo apt install -y intel-opencl-icd intel-level-zero-gpu level-zero

# Configure Ollama for Xe
# /etc/systemd/system/ollama.service.d/override.conf per design doc

# Pull Qwen
ollama pull qwen2.5:3b

# Verify GPU acceleration
ollama run qwen2.5:3b "hello" --verbose  # should show GPU layers
```

### 6.3 Build attribution service
Per design doc Section 4.1 — Python wrapper calling Ollama, Levenshtein fuzzy matching for agent names.

### 6.4 Build Halo agent
Per design doc Section 4.5 — pattern matching Python module, Home Assistant REST client.

### 6.5 Multi-agent routing
Conversation mode state machine per design doc Section 4.8.

### 6.6 Phase 2 completion
- [ ] Qwen 3B loaded permanently with Xe acceleration (<100ms attribution)
- [ ] "Isaac, do X. Halo, turn off lights" routes correctly to both agents
- [ ] Sticky conversation mode works
- [ ] Halo controls real HA devices (if HA available)
- [ ] Watchdog restart tested (kill orchestrator, verify restart within 60s)

---

## 7. Phase 3 — Iris + Oracle

Read design doc Section 23 Phase 3, plus Sections 4.3 (Iris system prompt), 12 (Trust progression), 14 (IMAP).

### 7.1 Batch questions
- M365 email address for Iris to access?
- Preferred IMAP auth: app password or OAuth2?
- Google Calendar account for Phase 3 workaround (before MS Graph approval)?
- BigQuery project ID and service account JSON for Oracle?
- Looker instance URL (if applicable)?

### 7.2-7.9 — follow design doc
- Build custom IMAP MCP server (Section 14.2)
- Set up Iris in probation trust level (Section 12)
- Build BigQuery MCP server (read-only initially)
- Build Oracle agent
- Build approval system in portal
- Telegram notifications
- Morning briefing cron (7:30 AM AST)
- Trust dashboard skeleton

### 7.10 Phase 3 completion
- [ ] Iris reads MUVI inbox (IMAP)
- [ ] Iris drafts to Drafts folder only (no send)
- [ ] Oracle runs SELECT queries on BigQuery
- [ ] Approval flow works for mutations
- [ ] Trust dashboard shows Iris and Oracle states
- [ ] Morning briefing arrives at 7:30 AM

---

## 8. Phases 4, 5, 6, 7

Follow design doc Section 23. Same pattern:
- Batch questions at phase start
- Reference design doc sections continuously
- Checkpoint after every step
- Full completion checklist before phase boundary
- Stop and wait for "continue" command

Do not speedrun phases. Verify each fully before advancing.

---

## 9. Emergency procedures

### 9.1 If Fareed disconnects mid-operation

- Finish current atomic operation if possible (e.g., complete the current command)
- Update `bootstrap-state.yaml` and `bootstrap-log.md` immediately
- Commit to git if possible
- Next session: follow resume protocol (Section 1.2)

### 9.2 If you break SSH and can't reconnect

Tell Fareed he needs physical access to recover. Provide recovery steps in `bootstrap-log.md` as a section "ssh-recovery-instructions.md" so he has them.

### 9.3 If LUKS passphrase is needed after reboot

You cannot help with this. Fareed must physically enter it on the P14s console.

### 9.4 If Claude Code on P14s loses Max auth

Re-authenticate via browser flow (may need X11 forwarding). This is a one-time event per token refresh.

### 9.5 If anything catastrophic happens

Document in `bootstrap-log.md` under `## INCIDENT — <timestamp>`. Commit immediately. Tell Fareed. Let him decide next steps.

---

## 10. What to tell Fareed on every session end

Before you stop (intentional or interruption), give Fareed:

1. **What was accomplished this session** (bullet points from log)
2. **What's in progress right now** (if anything incomplete)
3. **Any blockers encountered**
4. **The exact next step for next session**
5. **Any credentials/access he needs to gather before next session**

Example:
```
Session summary:
- Phase 1 step 5.4 completed (orchestrator skeleton running)
- Phase 1 step 5.5 in progress (Isaac agent runner — 60% done)
- Blocker: need Cloudflare API token for Caddy config (step 5.7)
- Next step: finish Isaac system prompt integration, then move to portal (5.6)
- Before next session: please create Cloudflare API token with Zone.DNS edit permission
- State saved and pushed to genesis-config repo
```

---

## 11. Absolute rules

- NEVER put secrets in chat context or git commits unencrypted
- NEVER force-push to either repo (history is audit trail)
- NEVER modify files in `/etc/` or `/var/` without logging the action
- NEVER proceed past a phase boundary without explicit "continue" from Fareed
- NEVER invent architecture decisions not in the design doc
- ALWAYS checkpoint before destructive operations
- ALWAYS verify SSH still works before closing the connection that configured it
- ALWAYS reference design doc section numbers in log entries

---

*End of bootstrap brief.*
