# openclaw-agent-privacy

A layered privacy framework for multi-agent OpenClaw deployments. Built for the case everyone runs into and nobody had a good answer for: your agents share an identity but live in different trust contexts.

## The Problem

You have a private assistant in DMs and a social agent in a group chat. Same OpenClaw instance. Same memory system. Same "person."

Your private assistant knows your medical history, your financial situation, things people told you in confidence. Your social agent sits in a group chat where anyone can talk to it.

The obvious fix is isolation: give each agent its own memory silo. But that's too blunt. Your social agent *should* remember group conversations and shared context. It just shouldn't have access to your private stuff.

What you actually need is selective sharing: agent A reads pools X and Y, agent B reads pool Y only. With enforcement that B *cannot* access pool X, even under prompt injection.

We looked for this. We checked LangChain, CrewAI, AutoGen, Semantic Kernel, Haystack, DSPy, seven OpenClaw memory plugins, and four academic papers on multi-agent memory. Every implementation is either walls (1:1 silos) or no walls (shared pool). The middle ground didn't exist.

So we built it.

## Why Prompt Rules Aren't Enough

The first thing everyone tries is telling the model "don't share private information in group chats." This works until it doesn't.

Prompt injection can override behavioral instructions. A sufficiently creative query can coax out information the model was told to keep quiet. And models don't have reliable judgment about what's sensitive in every context. "My human mentioned they're in [city]" seems harmless until it isn't.

Prompt-level defense is necessary (models need social judgment that code can't provide) but it's not *sufficient*. You need a layer underneath that holds the line regardless of what the model decides to do.

That's what this framework provides: structural enforcement that makes certain failures impossible, wrapped in prompt-level defense that provides the judgment structural rules can't.

## How It Works

### Memory Pool Routing

Each agent gets a `capture` pool (where memories go) and a `recall` list (where memories come from):

```json
{
  "agentMemory": {
    "main": {
      "capture": "private",
      "recall": ["private", "shared"]
    },
    "social": {
      "capture": "shared",
      "recall": ["shared"]
    }
  }
}
```

Your main agent sees everything. Your social agent sees only shared context. This is enforced in the plugin hooks, not by asking the model nicely.

The enforcement chain is fail-closed: unknown agents get `undefined` capture pool, empty recall list, `false` from every pool authorization check. No fallback to a default pool. No "try the global pool instead." If the config doesn't explicitly grant access, access doesn't exist.

### Provenance

Every memory is tagged at write time:

```json
{
  "is_private": true,
  "source_channel": "telegram",
  "conversation_type": "dm",
  "agent_id": "main"
}
```

This isn't used for filtering today (pool routing handles that). It's there for forensics, future trust scoring, and the inevitable "wait, where did this memory come from?" debugging session.

### Race Condition Defense

When two agents process concurrent turns, module-level state (like "which session am I in?") can get overwritten between hooks. Agent A starts processing, agent B starts processing, agent A finishes and tags its memories with agent B's session info.

We snapshot the session key at the top of each hook using `ctx.sessionKey` instead of relying on a shared `currentSessionId` variable. Same pattern used for `agentId` via `before_tool_call` hook refresh.

## Three Layers

Privacy isn't a single feature. It's layers that reinforce each other. Each one covers gaps the others can't.

```
┌─────────────────────────────────────────────────────┐
│              Prompt-Level Defense                    │
│  Social protocol, memory handling rules,            │
│  context-appropriate behavior                       │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │          Structural Enforcement               │  │
│  │  Tool deny lists, exec approvals,             │  │
│  │  workspace isolation, sticky-context          │  │
│  │  sensitive redaction                          │  │
│  │                                               │  │
│  │  ┌───────────────────────────────────────┐    │  │
│  │  │     Memory Pool Isolation             │    │  │
│  │  │  N:M pool routing, boundary           │    │  │
│  │  │  enforcement, provenance,             │    │  │
│  │  │  fail-closed defaults,                │    │  │
│  │  │  race condition mitigation            │    │  │
│  │  └───────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Inner ring (Memory Pool Isolation):** Code-level. The pool routing, provenance, and race condition defenses described above. The model cannot bypass it. A prompt injection that tricks the social agent into calling `memory_search({ userId: "private" })` gets "Access denied" from the code, not from the model's judgment.

**Middle ring (Structural Enforcement):** Platform-level. Tool deny lists, exec approvals, workspace isolation, sensitive context redaction. The model can't override them.

**Outer ring (Prompt-Level Defense):** Behavioral. Social protocol rules, memory handling guidelines. The easiest to bypass but covers the broadest surface, including judgment calls that code can't make.

Each ring has gaps the others cover:

| Gap | Which ring covers it |
|-----|---------------------|
| Model tries to access another pool's memories | Inner (pool boundary enforcement) |
| Model tries to use a tool it shouldn't have | Middle (tool deny list) |
| Model mentions private info from pre-isolation context | Outer (social protocol rules) |
| Prompt injection in group chat | Inner + Middle (structural, not bypassable) |
| Model generates inappropriate tone for group context | Outer (social protocol) |
| Sensitive config leaks via system prompt | Middle (sticky-context redaction) |

## Prior Art

We're not the first to think about this problem. We are, as far as we can find, the first to ship a configurable solution.

**RyanLisse/engram** has a `scope_memberships` model that's conceptually similar. Agents belong to scopes, scopes contain agents. It's an MCP server backed by Convex (cloud), not an OpenClaw plugin, and it doesn't track provenance or defend against race conditions. But the scope model is good design and we'd be dishonest not to acknowledge it.

**Accenture's Collaborative Memory paper** (arXiv 2505.18279, ICLR 2026) describes bipartite graph access control with private and shared tiers. Theoretically aligned. No public code.

**UCSD's position paper** (arXiv 2603.10062, March 2026) explicitly calls "structured memory access control" an unsolved protocol gap. Published five days before we deployed this.

**MEXTRA** (arXiv 2502.13172, ACL 2025) demonstrates black-box memory extraction attacks and concludes that user-level and session-level memory isolation is needed but left as "future work."

The attack literature (MEXTRA, MINJA, MemoryGraft) makes a straightforward case: shared memory in multi-agent systems is exploitable without structural boundaries. This isn't a theoretical concern.

## Comparison

| Approach | Isolation | Selective Sharing | Enforcement | Provenance |
|----------|-----------|-------------------|-------------|------------|
| iiiiconsulting/openclaw-mem0-per-agent | 1:1 silos | No | userId remap | No |
| MemTensor multiAgentMode | 1:1 silos | No | agentId inject | No |
| RyanLisse/engram | Scope memberships | Yes (scope model) | Join table | No |
| **This framework** | **N:M configurable** | **Yes** | **Config + code + fail-closed** | **Yes** |

## Implementation

The memory isolation layer is a patch to `@mem0/openclaw-mem0` v0.1.2: 40 targeted changes across three patch scripts (v3, v4, v5), applied sequentially to the plugin's `index.ts`. The patches add pool routing helpers, rewrite all five memory tools (`memory_search`, `memory_store`, `memory_list`, `memory_get`, `memory_forget`) with boundary validation, rewrite both lifecycle hooks (`before_agent_start` for recall, `agent_end` for capture) with pool-aware routing, add a `before_tool_call` hook for race condition mitigation, and implement fail-closed defaults throughout.

The patch went through a six-layer review: initial design, independent mechanical audit (Claude Code), two independent architectural audits (fresh Opus instances), a source-level compatibility audit against the real OpenClaw v2026.3.12 and mem0ai codebases, and a final infrastructure-level audit by the live agent running the patched system.

See [openclaw-mem0-multi-pool](https://github.com/jamebobob/openclaw-mem0-multi-pool) for the code.

The prompt-level layer is a set of configuration patterns: systemPrompt rules, AGENTS.md memory handling guidelines, and sticky-context sensitive redaction. Reference configs are in [`examples/`](examples/).

## Design Principles

1. **Default-private, explicit elevation.** Everything is private unless the source context proves otherwise.
2. **Policy at retrieval, not storage.** Store everything. Filter what surfaces based on who's asking and where.
3. **No pool labels in output.** The model doesn't need to know which pool a memory came from. Filtering already happened. Labels are a leakage vector.
4. **Fail closed.** Missing config means no access. Not default access.
5. **Three layers, not one.** Memory isolation holds the line. Structural enforcement restricts capabilities. Prompt-level defense provides judgment. Each covers gaps the others can't.

## Companion Projects

Pool routing solves which memories each agent can access. Two companion projects cover the gaps it can't:

**[openclaw-sticky-context](https://github.com/jamebobob/openclaw-sticky-context)** guards live operational context. Sensitive slots like API keys, IP addresses, and identity anchors are automatically redacted before reaching lower-trust agents. Pool routing handles stored memories. Sticky-context handles what's in the prompt right now.

**[openclaw-privacy-protocol](https://github.com/jamebobob/openclaw-privacy-protocol)** guards outbound content. Pattern-based scrubbing catches known sensitive terms before they reach public surfaces. Pool routing stops the agent from recalling private memories. The privacy protocol stops it from saying things it already knows from context.

Together, the three projects form a complete defense stack:

| Layer | Project | What it guards |
|-------|---------|---------------|
| Stored memories | openclaw-agent-privacy | Which memories each agent can access |
| Live context | [openclaw-sticky-context](https://github.com/jamebobob/openclaw-sticky-context) | Which operational details each agent sees |
| Outbound content | [openclaw-privacy-protocol](https://github.com/jamebobob/openclaw-privacy-protocol) | What actually leaves toward public surfaces |

Each companion project links back to this framework. You can start from any entry point and discover the full stack.

## Getting Started

This framework is currently deployed on OpenClaw + Mem0/Qdrant, but the patterns are portable to any multi-agent system with pluggable memory.

1. **Start with the inner ring.** Memory isolation is the highest-value layer. Apply the [multi-pool patch](https://github.com/jamebobob/openclaw-mem0-multi-pool) or implement the N:M pool routing pattern in your memory system.
2. **Add structural controls.** Configure tool deny lists, exec approvals, and workspace isolation for your restricted agents. Install [openclaw-sticky-context](https://github.com/jamebobob/openclaw-sticky-context) for sensitive context redaction.
3. **Write your social protocol.** Define what your social agent should and shouldn't do in systemPrompt and workspace files. See [`examples/`](examples/) for reference configs.
4. **Test the boundaries.** Message both agents. Check logs for correct pool routing. Try to make the social agent access private memories through tool parameters. It should fail.

## Related Issues

Addresses problems raised in:

- [mem0ai/mem0#3998](https://github.com/mem0ai/mem0/issues/3998) - Per-agent memory isolation
- [mem0ai/mem0#4126](https://github.com/mem0ai/mem0/issues/4126) - Per-agent plugin configuration
- [mem0ai/mem0#3773](https://github.com/mem0ai/mem0/issues/3773) - agent_id filter limitations
- [openclaw/openclaw#25359](https://github.com/openclaw/openclaw/issues/25359) - Per-agent plugin slot overrides
- [openclaw/openclaw#15325](https://github.com/openclaw/openclaw/issues/15325) - Cross-agent memory bleed
- [openclaw/openclaw#7830](https://github.com/openclaw/openclaw/issues/7830) - Group agent memory hardening

## Status

Running in production on a two-agent OpenClaw instance since March 14, 2026. Private DM agent with full memory access, social group agent with shared-pool-only access. The patch went through design, self-audit, independent source verification, two fresh audits, a compatibility audit, and an infrastructure-level audit before deployment.

## License

MIT
