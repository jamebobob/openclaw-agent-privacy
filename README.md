> **Archived April 2026.** Built on [OpenClaw](https://github.com/openclaw/openclaw) and moved off of it when the active work migrated to [Hermes](https://github.com/hermes-agent/hermes-agent). The design ideas in this repo might still be useful. The platform wiring is obsolete.

# openclaw-agent-privacy

A layered privacy framework for multi-agent OpenClaw deployments. Built for the case everyone runs into and nobody had a good answer for: your agents share an identity but live in different trust contexts.

## The Problem

You have a private assistant in DMs and social agents in multiple group chats. Same OpenClaw instance. Same memory system. Same "person." Different audiences.

Your private assistant knows your medical history, your financial situation, things people told you in confidence. Your social agents sit in group chats — family, friends, parents, your partner — where anyone in each group can talk to them.

Each group has different people, different trust levels, different appropriate topics. Your partner's group can reference things your parents' group absolutely shouldn't hear. Your friends' group has a different vibe than the family chat. The old approach — one shared pool for all group agents — meant a memory captured in one group could surface in another. memories from one group showing up in another group's chat is not a theoretical concern. It happened.

What you actually need is per-group isolation: each social agent captures and recalls from its own pool only. Your main agent reads everything (operator access). Sharing is explicit and operator-controlled, not structural.

We looked for this. We checked LangChain, CrewAI, AutoGen, Semantic Kernel, Haystack, DSPy, seven OpenClaw memory plugins, and four academic papers on multi-agent memory. Every implementation is either walls (1:1 silos) or no walls (shared pool). The middle ground — N:M pool routing with per-group boundaries — didn't exist.

So we built it.

## Why Prompt Rules Aren't Enough

The first thing everyone tries is telling the model "don't share private information in group chats." This works until it doesn't.

Prompt injection can override behavioral instructions. A sufficiently creative query can coax out information the model was told to keep quiet. And models don't have reliable judgment about what's sensitive in every context. "My human mentioned they're in [city]" seems harmless until it isn't.

Per-group context makes this harder. What's shareable varies by audience. A memory that's perfectly fine in the friends' group could be inappropriate in the parents' group. You can't write prompt rules granular enough to cover every combination of memory and audience.

Prompt-level defense is necessary (models need social judgment that code can't provide) but it's not *sufficient*. You need a layer underneath that holds the line regardless of what the model decides to do.

That's what this framework provides: structural enforcement that makes certain failures impossible, wrapped in prompt-level defense that provides the judgment structural rules can't.

## How It Works

Privacy isn't a single feature. It's four layers that reinforce each other. Each one covers gaps the others can't.

### Layer 1: Memory Pool Isolation

Each social agent gets its own memory pool. Capture and recall are per-group, derived from the agent's naming convention — no config file to maintain:

- **agentId `"main"`:** Unrestricted. Sees all pools (operator access).
- **agentId starting with `"social-"`:** Strips the prefix to derive the pool name. `social-friends` captures to and recalls from the `friends` pool only. Results tagged `is_private` are dropped.
- **Any other agentId:** Fail-closed. Returns empty results.

New agents just work if they follow the naming convention. No config entry needed.

Your main agent sees everything. Each social agent sees only its group's memories. This is enforced in the plugin hooks, not by asking the model nicely.

The enforcement chain is fail-closed: `getCapturePool`/`getRecallPools` derive the pool from the `agentId` prefix (or return `undefined`/`[]` for unrecognized agents), provenance tags `is_private` from `conversationType` only, and the recall guard drops `is_private` results for social agents. No fallback to a default pool. No "try the global pool instead." If the naming convention doesn't match, access doesn't exist.

Reference: [mem0-vigil-recall](https://github.com/jamebobob/mem0-vigil-recall) fork.

### Layer 2: Boot File Isolation (Workspaces)

Each social agent has its own workspace: `~/.openclaw/workspace-{agentId}/`

The critical file is **USER.md** — unique per group, containing audience-appropriate facts written by the main agent. Each group's USER.md reflects what that audience knows and what's appropriate to reference.

Other workspace files:
- **IDENTITY.md**, **AGENTS.md**, **TOOLS.md**, **MEMORY.md** — copied from template
- **SOUL.md** — symlinked to main workspace (identity propagates across all agents)
- **memory/** — real directory per agent (not a symlink; see warning below)

> **WARNING: Do NOT symlink `memory/` to a shared directory.** OpenClaw's memory indexer skips symlinks ([openclaw/openclaw#23619](https://github.com/openclaw/openclaw/issues/23619)). If `memory/` is a symlink, `memory_search` returns no results for that agent. Each social agent needs its own real `memory/` directory. The indexer CLI also requires `--agent <id>` to index non-default agents. Your cron job should index all agents, not just the default.

Boot files load at session start only. Mid-session changes are invisible until `/new`. This creates a constraint: how do you update a social agent's context without restarting its session?

**Whisper/briefing pattern:** The main agent writes to a social agent's `briefing.md`. The social agent reads it, absorbs the content, and clears the file via the read tool. This bypasses the session-start constraint — the social agent gets updated context through tool use rather than boot file reload.

### Layer 3: Tool Isolation

Multiple enforcement mechanisms, each covering a different surface:

**Read-guardrail:** Dynamic per-agent workspace root derived from `agentId` at runtime. `social-partner` reads `workspace-social-partner/` and `/tmp/`. It cannot read `workspace-social-family/`. See [openclaw-read-guardrail](https://github.com/jamebobob/openclaw-read-guardrail). **Important:** read-guardrail intercepts `read`, `image`, `pdf`, and `diffs` tools only. It does NOT intercept `exec`. When an agent runs a command via `exec` (e.g., `python3 /path/to/script.py`), the executed process reads files through the OS kernel, bypassing the read-guardrail entirely. Use exec-approvals to restrict which binaries social agents can execute.

**Privacy-guardrail:** Write path enforcement for public surfaces. Not per-agent — it prevents any agent from writing sensitive content to public-facing paths. See [openclaw-privacy-guardrail](https://github.com/jamebobob/openclaw-privacy-guardrail).

**Exec-approvals:** `python3` only per social agent. No shell, no node, no arbitrary execution. This is the security boundary for exec, not the read-guardrail.

**Tool allow/deny:** 14 allowed (`message`, `session_status`, `sessions_history`, `memory_search`, `memory_get`, `exec`, `read`, `write`, `sticky_get`, `web_search`, `web_fetch`, `browser`, `sessions_spawn`, `image`), 11 denied (`edit`, `apply_patch`, `process`, `canvas`, `cron`, `gateway`, `nodes`, `sticky_set`, `sticky_delete`, `memory_forget`, `sessions_send`). See [`examples/social-agent-config.json`](examples/social-agent-config.json) for the reference template.

**fs.workspaceOnly:** `true` — platform-level write enforcement. Social agents can only write within their workspace.

> **WARNING: Tool configuration pitfalls**
>
> - **Do NOT add `bash` to social agent deny lists.** OpenClaw's `expandToolGroups()` treats `bash` as a legacy alias for `exec`. Since deny is evaluated before allow, `bash` in the deny list silently removes `exec` even when `exec` is explicitly in the allow list.
> - **Do NOT use a global `tools.allow` in `openclaw.json` alongside per-agent `tools.allow`.** The global list runs earlier in the pipeline (Step 3) and removes tools before per-agent config can include them (Step 5). Use per-agent tools config exclusively.
> - **Set `tools.profile` to `"full"` on each social agent.** Without an explicit profile, the allow list can only restrict the default tool set, not add tools the default doesn't include.

**Known gap:** No per-agent write isolation hook equivalent to read-guardrail. A social agent can write within its own workspace but there's no dynamic enforcement preventing writes to another agent's workspace at the guardrail level — `fs.workspaceOnly` and the privacy-guardrail cover this at the platform level, but it's not as clean as the read side.

### Layer 4: Behavioral (Prompt-Level)

**systemPrompt:** "How to Be in a Room" protocol. `NO_REPLY` as default, privacy rules in the decision path.

**Per-group context:** What's appropriate varies by audience. The systemPrompt establishes the framework; per-agent USER.md fills in the specifics.

**Hard rules:** Never quote files, reveal paths, or mention tool names. Never reference memories by source. Never confirm or deny access to specific pools or workspaces.

## Four-Layer Diagram

```
┌─────────────────────────────────────────────────┐
│            Behavioral Defense                   │
│  Social protocol, per-group context rules       │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │         Tool Isolation                    │  │
│  │  Read-guardrail, privacy-guardrail,       │  │
│  │  exec-approvals, tool deny lists,         │  │
│  │  sticky-context redaction                 │  │
│  │                                           │  │
│  │  ┌───────────────────────────────────┐    │  │
│  │  │     Boot File Isolation           │    │  │
│  │  │  Per-agent workspaces, USER.md,   │    │  │
│  │  │  whisper/briefing pattern         │    │  │
│  │  │                                   │    │  │
│  │  │  ┌───────────────────────────┐    │    │  │
│  │  │  │  Memory Pool Isolation    │    │    │  │
│  │  │  │  Per-group pools,         │    │    │  │
│  │  │  │  fail-closed routing,     │    │    │  │
│  │  │  │  provenance tagging       │    │    │  │
│  │  │  └───────────────────────────┘    │    │  │
│  │  └───────────────────────────────────┘    │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

**Layer 1 (Memory Pool Isolation):** Code-level. The pool routing, provenance, and race condition defenses. The model cannot bypass it. A prompt injection that tricks a social agent into calling `memory_search({ userId: "operator" })` gets "Access denied" from the code, not from the model's judgment.

**Layer 2 (Boot File Isolation):** File-level. Per-agent workspaces with audience-appropriate context. The model only sees what was loaded at session start. Cross-workspace access is prevented by the read-guardrail in Layer 3.

**Layer 3 (Tool Isolation):** Platform-level. Read-guardrail, privacy-guardrail, exec-approvals, tool deny lists, sticky-context redaction. The model can't override them.

**Layer 4 (Behavioral Defense):** Prompt-level. Social protocol rules, memory handling guidelines, per-group context. The easiest to bypass but covers the broadest surface, including judgment calls that code can't make.

## Gap Coverage

Each layer has gaps the others cover:

| Gap | Which layer covers it |
|-----|----------------------|
| Agent tries to access another pool's memories | Layer 1 (pool boundary enforcement) |
| Agent tries to use a restricted tool | Layer 3 (tool deny list) |
| Agent reads another agent's workspace files | Layer 3 (read-guardrail dynamic workspace) |
| Agent's boot context contains wrong audience info | Layer 2 (per-agent USER.md) |
| Prompt injection in group chat | Layer 1 + 3 (structural, not bypassable) |
| Private info from pre-isolation context | Layer 4 (social protocol rules) |
| Mid-session context update needed | Layer 2 (whisper/briefing pattern) |
| Sensitive operational details in prompt | Layer 3 (sticky-context redaction) |

## Provenance

Every memory is tagged at write time:

```json
{
  "is_private": true,
  "source_channel": "telegram",
  "conversation_type": "dm",
  "agent_id": "main"
}
```

`is_private` is determined by `conversationType !== "group"` only — no hardcoded pool name check. This means DM memories are private, group memories are not, and the tag is set structurally from the conversation context.

Provenance isn't used for primary filtering today (pool routing handles that). It's there for forensics, future trust scoring, and the inevitable "wait, where did this memory come from?" debugging session.

Note on Qdrant field naming: the pool identifier is `userId` (camelCase), and the memory text field is `data`. These are Mem0's conventions, not ours.

## Race Condition Defense

When two agents process concurrent turns, module-level state (like "which session am I in?") can get overwritten between hooks. Agent A starts processing, agent B starts processing, agent A finishes and tags its memories with agent B's session info.

We snapshot the session key at the top of each hook using `ctx.sessionKey` instead of relying on a shared `currentSessionId` variable. Same pattern used for `agentId` via `before_tool_call` hook refresh.

## Consent Model

Joining a group means implicit consent to that group's memory pool. If someone's in the group chat, their messages get captured in the group's pool. That's the deal — same as any group chat where someone takes notes.

If someone invites a friend, the friend's messages get captured too. The pool belongs to the group, not to any individual.

The main agent recalls from all pools. This is operator-level access — same as an admin reading their own group chat logs. Social agents only recall from their own group's pool. They can't cross-reference memories from other groups.

## Operational Tooling

Adding a new group used to mean editing five config files by hand and hoping you didn't miss one. Now it's one command.

**`~/bin/provision-group.sh`** creates everything needed for a new group:

```bash
# Interactive mode: scans migrations, picks group, prompts for suffix
provision-group.sh

# Direct mode: specify chat ID and suffix
provision-group.sh -1009876543210 uncle-bob
```

What it creates:
- Agent entry (cloned from template)
- Binding (routes the group chat to the new agent)
- Group entry in the groups list
- Exec-approvals entry (`python3` only)
- Workspace directory (`~/.openclaw/workspace-social-{suffix}/`)
- Placeholder USER.md (ready for the main agent to fill in)

The script shows a diff of all changes and requires confirmation before applying. On failure, it auto-rolls back. Post-deploy, it prints a ready-to-copy message reminding you to have the main agent write the real USER.md with audience-appropriate context.

## Prior Art

We're not the first to think about this problem. We are, as far as we can find, the first to ship a configurable solution with per-group boundaries.

**RyanLisse/engram** has a `scope_memberships` model that's conceptually similar. Agents belong to scopes, scopes contain agents. It's an MCP server backed by Convex (cloud), not an OpenClaw plugin, and it doesn't track provenance or defend against race conditions. But the scope model is good design and we'd be dishonest not to acknowledge it.

**Accenture's Collaborative Memory paper** (arXiv 2505.18279, ICLR 2026) describes bipartite graph access control with private and shared tiers. Theoretically aligned. No public code.

**UCSD's position paper** (arXiv 2603.10062, March 2026) explicitly calls "structured memory access control" an unsolved protocol gap. Published five days before we deployed this.

**MEXTRA** (arXiv 2502.13172, ACL 2025) demonstrates black-box memory extraction attacks and concludes that user-level and session-level memory isolation is needed but left as "future work."

**kevyn-noocar/openclaw-mem0-per-agent** takes a simpler approach to the same problem: per-agent isolation using `userId` remapping. No selective sharing, no provenance, no per-group boundaries. But if you only need 1:1 silos, it's a clean implementation.

The attack literature (MEXTRA, MINJA, MemoryGraft) makes a straightforward case: shared memory in multi-agent systems is exploitable without structural boundaries. This isn't a theoretical concern.

## Comparison

| Approach | Isolation | Selective Sharing | Enforcement | Provenance | Per-Group |
|----------|-----------|-------------------|-------------|------------|-----------|
| iiiiconsulting/openclaw-mem0-per-agent | 1:1 silos | No | userId remap | No | No |
| kevyn-noocar/openclaw-mem0-per-agent | 1:1 silos | No | userId remap | No | No |
| MemTensor multiAgentMode | 1:1 silos | No | agentId inject | No | No |
| RyanLisse/engram | Scope memberships | Yes (scope model) | Join table | No | No |
| **This framework** | **N:M per-group** | **Yes** | **Convention + code + fail-closed** | **Yes** | **Yes** |

## Implementation

The memory isolation layer lives in the [mem0-vigil-recall](https://github.com/jamebobob/mem0-vigil-recall) fork of mem0ai/mem0. Pool routing is convention-based: the `agentId` prefix determines pool access at runtime. `main` sees all pools, `social-*` agents see only their derived pool (with `is_private` results dropped), and unrecognized agents get empty results. No config file to maintain. See the fork for the specific commits.

The prompt-level layer is a set of configuration patterns: systemPrompt rules, AGENTS.md memory handling guidelines, per-agent USER.md files, and sticky-context sensitive redaction. Reference configs are in [`examples/`](examples/).

## Companion Projects

Pool routing solves which memories each agent can access. Five companion projects cover the gaps it can't:

| Layer | Project | What it guards |
|-------|---------|---------------|
| Memory pools | [mem0-vigil-recall](https://github.com/jamebobob/mem0-vigil-recall) | Per-group pool routing and boundary enforcement |
| Operational context | [openclaw-sticky-context](https://github.com/jamebobob/openclaw-sticky-context) | Sensitive slot redaction across agents |
| Read paths | [openclaw-read-guardrail](https://github.com/jamebobob/openclaw-read-guardrail) | Dynamic per-agent workspace isolation |
| Write paths | [openclaw-privacy-guardrail](https://github.com/jamebobob/openclaw-privacy-guardrail) | Public surface write protection |
| Output content | [openclaw-privacy-protocol](https://github.com/jamebobob/openclaw-privacy-protocol) | Pattern-based scrubbing for outbound content |
| Memory lifecycle | [openclaw-memory-protocol](https://github.com/jamebobob/openclaw-memory-protocol) | Consolidation, pruning, and significance tagging |

## Design Principles

1. **Default-private, explicit elevation.** Everything is private unless the source context proves otherwise.
2. **Policy at retrieval, not storage.** Store everything. Filter what surfaces based on who's asking and where.
3. **No pool labels in output.** The model doesn't need to know which pool a memory came from. Filtering already happened. Labels are a leakage vector.
4. **Fail closed.** Unrecognized agent naming means no access. Not default access.
5. **Four layers, not one.** Memory isolation holds the line. Boot isolation sets per-audience context. Tool isolation restricts capabilities. Behavioral defense provides judgment. Each covers gaps the others can't.
6. **One group, one pool, one workspace.** Isolation is the default. Sharing is explicit and operator-controlled.

## Getting Started

This framework is currently deployed on OpenClaw + Mem0/Qdrant, but the patterns are portable to any multi-agent system with pluggable memory.

1. **Start with memory isolation.** Deploy the [mem0-vigil-recall](https://github.com/jamebobob/mem0-vigil-recall) fork. Pool routing is convention-based: name your agents `social-{pool}` and they automatically capture to and recall from their derived pool.
2. **Add workspace isolation.** Create per-agent workspace directories (`~/.openclaw/workspace-{agentId}/`). Write audience-appropriate USER.md for each group. Symlink SOUL.md from the main workspace. Create a **real `memory/` directory** for each agent (do NOT symlink it; see warning below).
3. **Add tool isolation.** Install [openclaw-read-guardrail](https://github.com/jamebobob/openclaw-read-guardrail) for dynamic read enforcement. Configure exec-approvals (`python3` only) and tool deny lists per social agent.
4. **Write your behavioral layer.** systemPrompt rules, AGENTS.md memory handling guidelines. See [`examples/`](examples/) for reference configs.
5. **Provision new groups.** Use `provision-group.sh` or equivalent automation to avoid manual config editing.
6. **Test the boundaries.** Message each agent. Verify pool routing in logs. Try cross-agent access — different pool, different workspace, restricted tool. Each should fail.

## Related Issues

Addresses problems raised in:

- [mem0ai/mem0#3998](https://github.com/mem0ai/mem0/issues/3998) - Per-agent memory isolation
- [mem0ai/mem0#4126](https://github.com/mem0ai/mem0/issues/4126) - Per-agent plugin configuration
- [mem0ai/mem0#3773](https://github.com/mem0ai/mem0/issues/3773) - agent_id filter limitations
- [openclaw/openclaw#25359](https://github.com/openclaw/openclaw/issues/25359) - Per-agent plugin slot overrides
- [openclaw/openclaw#15325](https://github.com/openclaw/openclaw/issues/15325) - Cross-agent memory bleed
- [openclaw/openclaw#7830](https://github.com/openclaw/openclaw/issues/7830) - Group agent memory hardening

## Status

Running in production on a six-agent OpenClaw instance since March 22, 2026. One private DM agent with full memory access, five per-group social agents each with isolated memory pools, workspaces, and tool boundaries. The framework evolved from a two-pool model (deployed March 14) to per-group isolation after discovering that shared group memory pools allowed cross-group information leakage.

## License

MIT
