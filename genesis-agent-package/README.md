# Genesis Agent Package

This package contains the complete set of agent identity files, visual architecture diagrams, skill examples, and templates needed to bootstrap Genesis's agent layer.

**Target audience:** Claude Code executing the Phase 1 bootstrap on Fareed's P14s.

**Companion documents:**
- `GENESIS_System_Design_v0.5.5.md` — the architecture (read first)
- `GENESIS_Bootstrap_Brief.md` — the deployment procedure (reference continuously)

---

## Package structure

```
genesis-agent-package/
├── README.md                          ← this file
│
├── diagrams/                          ← Mermaid source for visual architecture
│   ├── 01-system-architecture.mmd
│   ├── 02-agent-topology.mmd
│   ├── 03-request-flow.mmd
│   └── 04-deployment.mmd
│
├── agents/                            ← Personalized agent identity files
│   ├── isaac/
│   │   └── CLAUDE.md                  ← Isaac's identity + MUVI context
│   ├── oracle/
│   │   └── CLAUDE.md                  ← Oracle's identity + data ecosystem
│   ├── iris/
│   │   └── CLAUDE.md                  ← Iris's identity + contacts + PM logic
│   ├── sentinel/
│   │   └── CLAUDE.md                  ← Sentinel's identity + monitoring mandate
│   └── halo/
│       └── README.md                  ← Halo docs (she has no LLM, no CLAUDE.md)
│
├── skills/                            ← Example skills (recurring tasks)
│   ├── iris-morning-briefing.md       ← Iris's daily 7:30 AM briefing
│   ├── oracle-weekly-kpi-report.md    ← Oracle's Sunday 10 AM MUVI KPIs
│   └── isaac-refactor-task.md         ← Isaac's refactor discipline
│
└── templates/                         ← Templates for generated memory files
    ├── project-log-template.md        ← Per-project, per-date execution log
    └── reflection-template.md         ← Per-agent, per-date EOD reflection
```

---

## How to deploy this package

These files don't all live in the same place after deployment. Different pieces go to different locations:

### 1. Diagrams → `genesis` (public repo)

```bash
mkdir -p ~/genesis/docs/diagrams
cp genesis-agent-package/diagrams/*.mmd ~/genesis/docs/diagrams/
cd ~/genesis
git add docs/diagrams/
git commit -m "docs: add system architecture diagrams"
git push
```

GitHub auto-renders `.mmd` files. Visit `github.com/fareed-hameed/genesis/tree/main/docs/diagrams` after push to verify.

### 2. Agent CLAUDE.md files → `genesis-config` (private repo)

These are personalized with MUVI context and real contacts. They MUST live in the private repo.

```bash
mkdir -p ~/genesis-config/agents/{isaac,oracle,iris,sentinel,halo}
cp genesis-agent-package/agents/isaac/CLAUDE.md ~/genesis-config/agents/isaac/
cp genesis-agent-package/agents/oracle/CLAUDE.md ~/genesis-config/agents/oracle/
cp genesis-agent-package/agents/iris/CLAUDE.md ~/genesis-config/agents/iris/
cp genesis-agent-package/agents/sentinel/CLAUDE.md ~/genesis-config/agents/sentinel/
cp genesis-agent-package/agents/halo/README.md ~/genesis-config/agents/halo/

cd ~/genesis-config
git add agents/
git commit -m "agents: add personalized CLAUDE.md files for all agents"
git push
```

### 3. Symlinks from agent working directories → personalized files

Each agent's persistent `claude -p` subprocess runs in its own working directory. Claude Code loads `CLAUDE.md` from that directory at session start. Symlink the personalized versions in:

```bash
# Create working directories for each agent
mkdir -p /home/nova/agents/{isaac,oracle,iris,sentinel}

# Symlink personalized CLAUDE.md files
ln -sf ~/genesis-config/agents/isaac/CLAUDE.md /home/nova/agents/isaac/CLAUDE.md
ln -sf ~/genesis-config/agents/oracle/CLAUDE.md /home/nova/agents/oracle/CLAUDE.md
ln -sf ~/genesis-config/agents/iris/CLAUDE.md /home/nova/agents/iris/CLAUDE.md
ln -sf ~/genesis-config/agents/sentinel/CLAUDE.md /home/nova/agents/sentinel/CLAUDE.md
```

Halo doesn't get a CLAUDE.md (no LLM). Her config lives separately:
```bash
mkdir -p /home/nova/agents/halo
cp ~/genesis-config/agents/halo/README.md /home/nova/agents/halo/
# Halo's device-registry.yaml and config.yaml come from separate personalization
```

### 4. Skill files → per-agent skills directories in `genesis-config`

```bash
mkdir -p ~/genesis-config/agents/{iris,oracle,isaac}/skills/

cp genesis-agent-package/skills/iris-morning-briefing.md ~/genesis-config/agents/iris/skills/
cp genesis-agent-package/skills/oracle-weekly-kpi-report.md ~/genesis-config/agents/oracle/skills/
cp genesis-agent-package/skills/isaac-refactor-task.md ~/genesis-config/agents/isaac/skills/

cd ~/genesis-config
git add agents/*/skills/
git commit -m "skills: add initial skill examples for each agent"
git push
```

### 5. Templates → `genesis` (public) for reference

Templates are generic — they go in the public repo so they can be referenced by any instance of Genesis:

```bash
mkdir -p ~/genesis/docs/templates
cp genesis-agent-package/templates/*.md ~/genesis/docs/templates/

cd ~/genesis
git add docs/templates/
git commit -m "docs: add memory file templates (project log, reflection)"
git push
```

### 6. Create the memory directory structure in `genesis-config`

The memory directories must exist so agents can write to them from day one:

```bash
mkdir -p ~/genesis-config/memory/{project-logs,agent-reflections,shared-learnings/patterns}

# Create README files so git tracks the empty directories
cat > ~/genesis-config/memory/project-logs/README.md << 'EOF'
# Project Execution Logs

Per-project, per-date markdown files following the pattern:
  project-logs/<project>/<project>_YYYY-MM-DD.md

Template: ../docs/templates/project-log-template.md (in genesis public repo)

Agents write these after significant work sessions on a project.
All agents can read all project logs for coordination and context.
EOF

cat > ~/genesis-config/memory/agent-reflections/README.md << 'EOF'
# Agent Daily Reflections

Per-agent, per-date markdown files following the pattern:
  agent-reflections/<agent>_YYYY-MM-DD.md

Template: ../docs/templates/reflection-template.md (in genesis public repo)

Generated automatically at 8 PM AST by EOD cron.
All agents read all reflections at session start (cross-pollination).
EOF

cat > ~/genesis-config/memory/shared-learnings/README.md << 'EOF'
# Shared Learnings

Weekly synthesis files: YYYY-WXX-weekly-synthesis.md
Generated by Iris every Sunday 9 PM AST.

Patterns directory: patterns/
Distilled wisdom that has crossed the reflection → synthesis → canonical threshold.
Referenced from agent CLAUDE.md files as appropriate.
EOF

cd ~/genesis-config
git add memory/
git commit -m "memory: initialize structured memory directories"
git push
```

---

## Verification after deployment

Before declaring the package deployed, verify:

- [ ] All four diagrams render correctly on GitHub (visit `github.com/fareed-hameed/genesis/tree/main/docs/diagrams`)
- [ ] Each agent's working directory contains a symlinked CLAUDE.md that resolves to the personalized version in `genesis-config`
- [ ] Running `cat /home/nova/agents/isaac/CLAUDE.md` shows Isaac's personalized content
- [ ] Skills directories contain the example skill files under each relevant agent
- [ ] Memory directory structure exists and is committed to `genesis-config`
- [ ] Templates are in the public `genesis` repo under `docs/templates/`
- [ ] All commits pushed to respective repos

---

## What this package does NOT include

Things that are generated or created during later phases:

- **The actual orchestrator code** — built in Phase 1 step 5.4 per bootstrap brief
- **The portal code** — built in Phase 1 step 5.6
- **Halo's device registry** — Fareed fills this out manually as he registers devices with HA
- **trust_config.yaml** — created during Phase 1, evolves over time
- **Real memory content** — accumulates as agents do actual work; starts empty
- **Sub-agent definitions** — Phase 4+ when Paperclip is introduced

---

## Notes for Claude Code

### Symlinks vs copies

The agent CLAUDE.md files are SYMLINKED from working directories to `genesis-config/agents/<n>/CLAUDE.md`. This ensures:
- Agents always read the latest personalized content
- Changes Fareed makes to CLAUDE.md files take effect immediately
- No drift between "deployed" version and "repo" version

Never copy these files into the working directories — always symlink.

### What if the symlink is broken?

If `/home/nova/agents/isaac/CLAUDE.md` doesn't resolve (broken symlink, missing target):
1. Check `~/genesis-config/agents/isaac/CLAUDE.md` exists
2. If missing, pull latest from genesis-config repo
3. If still missing, Fareed needs to be told — the personalized file has been lost
4. Fall back to the generic template in `genesis/agents/isaac/CLAUDE.md.example` (public repo) temporarily

### MUVI context porting

The personalized CLAUDE.md files assume Fareed has MUVI-related context (DCC, dbt, Looker, NPS translator). If Fareed is setting up for someone else or a different context, the MUVI-specific sections need to be replaced.

Generic versions with `[REPLACE]` markers exist in the public `genesis` repo as `.example` files for other users.

### When agents read these files

- **CLAUDE.md:** loaded automatically by Claude Code at session start for each agent
- **Skills:** loaded on demand when the skill is triggered (by cron, by request, or by pattern match)
- **Templates:** read when generating a new memory file
- **Diagrams:** reference material, not loaded into context — Fareed and reviewers look at them

---

## Change management

If any of these files need updating:

**Diagrams:** edit the `.mmd` source, commit to `genesis`, GitHub re-renders automatically.

**Agent CLAUDE.md files:** edit in `genesis-config`, commit, push. Next session picks up changes.

**Skills:** edit in `genesis-config/agents/<agent>/skills/`, commit, push.

**Templates:** edit in `genesis`, commit, push.

Never edit symlinked or copied versions on the filesystem — always edit the canonical location in the repo and let git + the deploy cycle propagate.
