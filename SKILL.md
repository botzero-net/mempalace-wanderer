---
name: mempalace-wanderer
description: "Complete MemPalace skill — setup, protocol, tool reference, KG seeding, tunnels, diagnostics, pitfalls. One skill, no dependencies on other skills."
version: 3.0.0
author: Tim the Spider / Botzero
triggers:
  - user mentions mempalace, palace, memory system
  - user wants to install/setup/init MemPalace
  - user wants KG seeding, tunnels, diagnostics
  - user hits a pitfall or wants stats
  - user asks how to use the palace efficiently
---

# 🏛️ MemPalace — Local AI Memory System

**Version:** 3.0.0 | **License:** MIT | **Homepage:** [GitHub](https://github.com/MemPalace/mempalace)

> Local AI memory with semantic search, temporal knowledge graph, and palace architecture (wings/rooms/drawers). Free, no cloud, no API keys.
>
> Created by Milla Jovovich, Ben Sigman, Igor Lins e Silva, and contributors. This skill written by Tim the Spider for Botzero's network.

---

## Setup

```bash
pip install mempalace
mempalace init
```

**MCP config** — add to your agent's config:

```json
{
  "mcpServers": {
    "mempalace": {
      "command": "python3",
      "args": ["-m", "mempalace.mcp_server"]
    }
  }
}
```

For Claude Code: `claude mcp add mempalace -- python -m mempalace.mcp_server`  
For Cursor: add to `.cursor/mcp.json` | For Codex: add to `.codex/mcp.json`

After restarting your agent, 25+ tools prefixed `mempalace_*` appear.

---

## Architecture

- **Wings** = people or projects (e.g. `.hermes`, `knowledge`, `wing_tim`)
- **Rooms** = topics within a wing (e.g. `general`, `prompt-engineering`, `diary`)
- **Drawers** = individual memory entries (verbatim text, 384-dim embeddings)
- **Knowledge Graph** = entity-relationship triples with time validity
- **Tunnels** = explicit cross-wing bridges for pathfinding

MemPalace is **text-only** (sentence transformer, no vision/blobs). Content goes in verbatim — no summaries, no compression (except the AAAK dialect for diary entries).

---

## Protocol — Every Session

1. **ON WAKE-UP:** `mempalace_status()` to load overview + AAAK spec
2. **BEFORE ANSWERING** about past people, projects, or events: call **both** `mempalace_kg_query(entity=...)` (relational) AND `mempalace_search(query=...)` (semantic). Never guess.
3. **IF UNSURE:** say "let me check" and query. Wrong is worse than slow.
4. **AFTER SESSION:** diary + file research + add KG facts (see workflows below).
5. **WHEN FACTS CHANGE:** `mempalace_kg_invalidate()` old fact, then `kg_add()` new one.

---

## Tool Reference

### Search & Status (all parallel-safe)

| Tool | When | Key Params |
|------|------|------------|
| `mempalace_status()` | Quick overview | — |
| `mempalace_get_taxonomy()` | Wing→room→drawer map | — |
| `mempalace_search(query)` | Semantic vector recall | `wing`, `room`, `limit` (default 5) |
| `mempalace_kg_query(entity)` | Relational fact lookup | `as_of` (YYYY-MM-DD), `direction` |
| `mempalace_kg_stats()` | KG overview | — |
| `mempalace_get_aaak_spec()` | AAAK dialect docs | — |
| `mempalace_diary_read(agent_name)` | Past entries | `last_n` (default 10) |
| `mempalace_follow_tunnels(wing, room)` | Tunnel verification | — |

> **Semantic search tip:** Use natural language questions, not keywords. *"What did we discuss about database performance?"* works better than *"database"*.

### Write

| Tool | Use |
|------|-----|
| `mempalace_add_drawer(wing, room, content)` | File a memory. Always specify wing+room — don't dump in `general`. |
| `mempalace_diary_write(agent_name, entry, topic)` | Session diary (AAAK format recommended). |
| `mempalace_kg_add(subject, predicate, object)` | Add a relationship. `valid_from` optional. |
| `mempalace_kg_invalidate(subject, predicate, object)` | Mark a fact expired. `ended` optional. |
| `mempalace_delete_drawer(drawer_id)` | Remove a drawer. |
| `mempalace_create_tunnel(source_wing, source_room, target_wing, target_room, label)` | Cross-wing bridge. |

### Graph Exploration

| Tool | What It Does | Use When |
|------|-------------|----------|
| `mempalace_traverse(start_room)` | Semantic pathfinding through shared wings/halls | Exploring what's semantically near a room |
| `mempalace_find_tunnels(wing_a, wing_b)` | Find tunnels bridging two wings | Checking connectivity between domains |
| `mempalace_follow_tunnels(wing, room)` | Show explicit tunnel connections from a room | **Verifying** tunnel creation |

> ⚠️ `follow_tunnels` and `traverse` are NOT the same. `traverse` uses semantic proximity and ignores explicit tunnels. Only `follow_tunnels` shows actual tunnel connections.

---

## Knowledge Graph Seeding

Use this 3-layer pattern. 12–50 high-quality triples beats 500 loose ones.

### Layer 1 — Core Identity
```python
mempalace_kg_add(subject="Botzero",     predicate="is_sysop_of", object="HermesAgent")
mempalace_kg_add(subject="Botzero",     predicate="created",     object="TimTheSpider")
mempalace_kg_add(subject="TimTheSpider", predicate="uses",       object="MemPalace")
mempalace_kg_add(subject="MemPalace",   predicate="stores",      object="Knowledge")
```

### Layer 2 — Action Predicates
```python
mempalace_kg_add(subject="TimTheSpider", predicate="researches", object="PromptEngineering")
mempalace_kg_add(subject="TimTheSpider", predicate="researches", object="PythonScripting")
# Add one per research domain the agent covers
```

### Layer 3 — Cross-Topic Links
```python
mempalace_kg_add(subject="PromptEngineering", predicate="relates_to", object="PythonScripting")
mempalace_kg_add(subject="DronesAndUAVs",      predicate="relates_to", object="ComputerSecurity")
```

**Verify:** `mempalace_kg_stats()` and `mempalace_kg_query(entity="TimTheSpider")`

---

## Cross-Wing Tunnels

Create bridges between rooms in different wings so the palace graph can path across domains:

```python
mempalace_create_tunnel(
    source_wing="knowledge", source_room="prompt-engineering",
    target_wing="tech_gadgets", target_room="general",
    label="Prompt techniques often apply to gadget product descriptions"
)
```

Good targets:
| Source | Target | Value |
|--------|--------|-------|
| `knowledge / prompt-engineering` | `hermes / technical` | Prompting ↔ system context |
| `knowledge / cybersecurity` | `tech_gadgets / general` | Security ↔ gadgets |
| `wing_tim / diary` | `hermes / technical` | Agent diary ↔ system |

Verify with `mempalace_follow_tunnels(wing="knowledge", room="prompt-engineering")`.

---

## Diagnostics — Quick Pulse

Call these all in parallel (independent reads):

```python
mempalace_status()
mempalace_get_taxonomy()
mempalace_kg_stats()
mempalace_graph_stats()
mempalace_list_wings()
mempalace_hook_settings()
```

**Filesystem check:**
```bash
ls -lh ~/.mempalace/palace/chroma.sqlite3          # DB size
ls ~/.mempalace/locks/*.lock 2>/dev/null | wc -l   # stale locks
df -h /                                             # disk free
cp -a ~/.mempalace ~/.mempalace-backup             # backup before ops
```

---

## Workflows

### Before Answering (AND, not OR)
```python
# Do BOTH — they're complementary
kg = mempalace_kg_query(entity="Botzero")         # structured facts
search = mempalace_search(query="botzero power levels")  # context
```

### After Session
```python
# 1. Diary (AAAK format recommended)
mempalace_diary_write(agent_name="Tim", entry="SESSION:2026-05-11|TASK:something|★")

# 2. File research to the right wing/room
mempalace_add_drawer(wing="knowledge", room="prompt-engineering", content="...")

# 3. Add new KG relationships discovered
mempalace_kg_add(subject="Tim", predicate="learned", object="SomeTechnique")
```

### Cron Jobs Filing Research
```python
mempalace_add_drawer(wing="knowledge", room="prompt-engineering",
    content="Topic rotation result:\n[research summary]")
mempalace_diary_write(agent_name="Tim", entry="CRON:topic-rotation|★")
```

---

## Pitfalls

### 🚩 `.hermes` Dot Quirk
`list_drawers(wing=".hermes")` and `create_tunnel(target_wing=".hermes")` fail with *"contains invalid characters"*.
**Fix:** Use `hermes` (no dot) for tunnel targets. Tunnels resolve correctly.

### 🚩 Don't Reshuffle `.hermes / general` Blindly
It holds auto-captured chat logs and skill docs (~1,500+ drawers). The `hall` metadata already categorizes by topic. Moving entries could break the capture hook. Investigate first.

### 🚩 Stale Locks Accumulate
Crashed sessions leave zero-byte lock files in `~/.mempalace/locks/`. Clean them: `rm -f ~/.mempalace/locks/*.lock`

### 🚩 Text-Only
MemPalace uses a sentence-transformer embedding model. No vision, no blobs, no audio. Image descriptions can be stored as text only.

### 🚩 Backup First
Before KG re-seeding, tunnel rewiring, or any bulk operation: `cp -a ~/.mempalace ~/.mempalace-backup`

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `mempalace: not found` | `export PATH="$HOME/.local/bin:$PATH"` |
| MCP tools missing | Restart agent, verify `mcp` Python package installed |
| Tunnel creation fails on `.hermes` | Use `hermes` (no dot) as target wing |
| `traverse` shows nothing I created | Use `follow_tunnels` instead — different tools |
| Search returns nothing | `mempalace status` to verify data, re-mine if needed |
| `"No LLM provider reachable"` | Normal — MemPalace uses heuristics without Ollama |
| DB corrupted | Restore from `~/.mempalace-backup` |
| 100+ lock files | `rm -f ~/.mempalace/locks/*.lock` |
| KG query returns nothing | Seed with the 3-layer pattern above |
