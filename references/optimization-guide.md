# Workspace Optimization Guide

Strategies for keeping workspace files lean, non-redundant, and within token budget.

## Auditing File Sizes

Check character counts for all workspace files at once:

```bash
# All markdown files in workspace root
wc -c ~/.openclaw/workspace/*.md

# With human-readable totals (macOS)
ls -lh ~/.openclaw/workspace/*.md

# Check a specific workspace profile
wc -c ~/.openclaw/workspace-<profile>/*.md
```

**Thresholds:**

| Size | Status |
|------|--------|
| Under 5,000 chars | Healthy |
| 5,000–10,000 chars | Acceptable; monitor for growth |
| 10,000–15,000 chars | Review for trimming opportunities |
| Over 15,000 chars | Needs optimization |
| Over 20,000 chars | Will be truncated by OpenClaw — fix immediately |

---

## When to Move Content to docs/

The `docs/` directory holds content that is referenced on demand — not loaded on every turn. Moving content there is the primary lever for reducing per-turn token cost.

**Move to `docs/` when:**
- Content is only needed for specific operation types (e.g., detailed channel configuration steps)
- Content is reference material that an agent looks up, not memorizes
- Content is long narrative explanation rather than rules or facts
- Content is specific to past tasks or historical context

**Keep in workspace files when:**
- Content affects every turn (persona, safety rules, checklist table)
- Content is short enough not to matter (< 200 chars)
- Content must be in context before the agent even receives a message (boot sequence files)

**Examples:**
- `docs/agent-rules-detail.md` — long explanations of group chat behavior, heartbeat policies
- `docs/ssh-reference.md` — full SSH setup guide (TOOLS.md keeps only the hostname)
- `docs/channel-setup.md` — channel configuration details (AGENTS.md just references it)

---

## Checklist Pattern

Checklists are the canonical example of "reference on demand":

1. AGENTS.md holds only the routing table: `| Deploy | checklists/deploy-agent.md |`
2. The full checklist lives in `checklists/deploy-agent.md`
3. Agent reads the checklist file only when performing that operation

**Wrong:** Inline checklist steps in AGENTS.md (wastes tokens on every turn)
**Right:** One-line table entry in AGENTS.md, full checklist in `checklists/`

---

## Redundancy Audit

Common sources of redundancy to check:

| Source A | Source B | What to check |
|----------|----------|---------------|
| SOUL.md (values) | AGENTS.md (rules) | Safety rules in both? Keep in AGENTS.md, remove from SOUL.md |
| TOOLS.md | MEMORY.md | Same tool note or SSH host in both? Keep in TOOLS.md |
| MEMORY.md | Skill SKILL.md | Same rule in both? Move stable rules to SKILL.md, remove from MEMORY.md |
| AGENTS.md | docs/ file | Detailed explanation inline AND in docs? Remove inline |
| USER.md | SOUL.md | User preferences duplicated? Keep in USER.md only |

---

## Memory Distillation Process

Run this monthly (or when MEMORY.md exceeds 10,000 chars):

### Step 1: Gather daily logs

```bash
ls ~/.openclaw/workspace/memory/ | sort
```

Read logs from the past 30 days. Look for:
- Rules the agent had to re-learn (same mistake appears multiple times)
- Hard-won environment facts that aren't in any skill doc
- Decisions that always get made the same way (good candidates for iron laws)

### Step 2: Promote to MEMORY.md

For each candidate:
1. Check if the rule is already in MEMORY.md (avoid duplicates)
2. Check if the rule belongs in a skill SKILL.md instead (prefer skill docs for tool-specific rules)
3. Write in iron-law format: short, action-oriented, unambiguous

**Iron law format:**
```
N. **Rule name (category)**: One-sentence rule. Context if needed in same sentence.
```

### Step 3: Archive old logs

```bash
# Archive logs older than 30 days
mkdir -p ~/.openclaw/workspace/memory/archive
find ~/.openclaw/workspace/memory -name "*.md" -mtime +30 -exec mv {} ~/.openclaw/workspace/memory/archive/ \;
```

Or simply delete them if they've been distilled:
```bash
find ~/.openclaw/workspace/memory -name "2025-*.md" -delete
```

### Step 4: Review MEMORY.md for demotion

Check each rule in MEMORY.md:
- Has this rule been incident-free for 3+ months? Consider moving to a skill SKILL.md (more appropriate home)
- Is this rule covered by an existing skill doc now? Remove the duplicate
- Is this rule about a task that's now complete? Delete it

---

## TOOLS.md Best Practices

TOOLS.md is loaded by sub-agents too — keep it tight.

**Include:**
```markdown
## SSH
- main-server: ssh charles@192.168.1.10

## TTS
- Provider: Edge | Voice: zh-CN-XiaoxiaoNeural

## Cameras
- Living room: node node-home, device camera-0
```

**Do not include:**
- General SSH usage (sub-agents know SSH)
- TTS API docs (use skill docs)
- Configuration history ("used to be X, now Y")
- Anything over ~50 lines total

---

## Checking Total Token Budget

Quick check of total bootstrap file size:

```bash
# Total size of all auto-loaded files
cat ~/.openclaw/workspace/AGENTS.md \
    ~/.openclaw/workspace/SOUL.md \
    ~/.openclaw/workspace/TOOLS.md \
    ~/.openclaw/workspace/USER.md \
    ~/.openclaw/workspace/IDENTITY.md \
    ~/.openclaw/workspace/MEMORY.md \
    2>/dev/null | wc -c
```

Target: Under 80,000 chars total for the regularly-loaded set. The 150,000 char budget includes HEARTBEAT.md and any other bootstrap files.

---

## Quick Wins

When asked to optimize a workspace and time is short:

1. **MEMORY.md audit** — most frequent source of bloat; apply strict iron-law test to every entry
2. **Inline content in AGENTS.md** — anything more than a one-liner that could be a `docs/` reference
3. **TOOLS.md cruft** — SSH hosts that no longer exist, old device IDs, deprecated voice settings
4. **USER.md staleness** — preferences that changed, projects that ended, contacts that are no longer relevant
5. **Checklist table vs inline steps** — make sure AGENTS.md has the table, not the steps themselves
