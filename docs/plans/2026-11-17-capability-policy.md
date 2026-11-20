# Per-Bot capability policy over inherited DSH capabilities

- Date: 2026-11-17
- Related: lordraiden/DS-Bot#1, #2, #3; upstream Fei2411/DS-Bot#1341

> Draft skeleton. `docs/plans/README.md` asks for Chinese; this draft is in English for review and is translated before the PR (an English plan has precedent: `2026-11-04-local-agents.md`).

## Why

Every Bot runs in the same DSH profile, so every Bot inherits the profile's MCP tools and skills. Today the only per-Bot control is DS Bot's own capability library, which is additive: it cannot take away what the profile already provides. A team therefore cannot give each Bot a least-privilege surface without duplicating MCP/skill configuration or changing the shared profile.

This plan adds a per-Bot policy over the **inherited** surface, plus reusable **capability sets**, without moving DS Bot's own capabilities into it.

## Decided behavior

### Scope boundary

- The policy covers inherited tools (deployment-global and ancestor-scope) and inherited model-invocable skills.
- DS Bot's team tools (registered in the Agent's own scope) and the DS Bot capability library stay additive and outside the policy. A capability skill shadows a same-name inherited skill (Agent layer wins); the policy must never shadow capability names.
- A Bot running on a local agent CLI has no DSH policy: the CLI owns its tools.
- The policy never removes `skill`: without it `dsh-tool-skill` stops publishing its catalog and granted skills cannot be loaded.

### Persistence

- `bot.capabilityPolicy = { mode: 'all' | 'selected', grants: BotGrant[] }`.
- `state.capabilitySets = { [id]: { id, name, description, grants, createdAt, updatedAt } }`; IDs are generated UUIDs, never names.
- Leaf grants: `{ type: 'mcp', server, tools: 'all' | string[] }`, `{ type: 'tool', name }`, `{ type: 'skill', name }`; Bot-only reference: `{ type: 'set', id }`.
- An absent policy means `all`. Existing Bots are not rewritten merely to store the default.
- `selected` with an empty grant list is distinct from a missing policy and expands to nothing.
- A malformed policy object fails closed (no grants) with a diagnostic. An unknown grant `type` is ignored and reported, so a future grant type does not brick Bots on downgrade.
- A missing set ID or an unavailable tool/skill name is unavailable, not fatal; it becomes effective again when its source returns with the same identity.
- Duplicating a Bot copies its policy; deleting a Bot leaves sets in place.

### Catalog

- Tools: `ctx.tools.definitions(parentScope)` where `parentScope = scopeParentOf(scopeOf(agent.ctx))` (global definitions when there is no parent). That list is a safe subset of what `restrict()` accepts.
- Exclude `HIDDEN_TOOLS` (already denied by DS Bot), the reserved `run_code` transport, and `skill` (always admitted, never offered).
- Group by MCP server prefix; connector tools (`mcp__<connector>__*`) are shown as connector groups, grounded in `connectorOffFor`.
- Skills: `ctx.skills.snapshot({ cwd, scope })`, grouped by `source`. Live Agents use `parentScope = scopeParentOf(scopeOf(agent.ctx))` and `ctx.workingDirectory.ensure(agent, signal)`; a cold Bot uses `ctx.agentPresets.acquireScope(presetId)` (dispose the lease) and its folder or the team workspace. Never enumerate skills at host level: in the web profile the base `skill-filesystem` row is disabled and presets own discovery, so a host-level snapshot is empty (rml4791lol-lang/DS-Bot#62).
- The catalog is bounded (names, descriptions, groups) and cached, invalidated by `tools/change` and `skills/change`.

### Expansion

- One pure function used by apply, preview and tests; the client never reimplements it.
- Union of direct grants and referenced sets; duplicates deduplicated; nested sets rejected.
- `mcp: all` expands from the current catalog on every preview/reconcile; it is not frozen at save time.
- Always admits `skill`; returns `{ tools, skills, unavailable }`; never throws on a stale grant.

### Enforcement

- `agent.ctx.tools.restrict({ allow: [...] })` on the Agent's inherited surface; `allow: []` is a valid empty selection.
- Filter the allow-list against the inherited names at apply time (`restrict()` throws on unknown names); a stale grant is omitted, never fatal.
- The policy disposer is separate from connector masks and `HIDDEN_TOOLS` restrictions; DSH restrictions intersect.
- A monotonic `guard` denies stale-schema and PTC/program calls; a guard may deny but never grant.
- Skills, primary: an Agent-scoped shadow mask. For every inherited skill the Bot may not use, register a runtime skill with the same name in the Agent's scope, with `invocation: { modelInvocable: false, userInvocable: false }`, a non-empty description and empty content. The nearest layer wins, so the model catalog omits it, the `skill` tool rejects it and `/name` skips the gesture, with no text rewriting and no ordering dependency. Skip names that are DS Bot capability skills; recompute on `skills/change`; remove with the policy.
- Skills, fail-closed fallback: if the mask cannot be established, deny the `skill` tool and register a `{ prepend: true }` `agent/pre-step` listener that removes disallowed `skill-invocation` messages and adds a model-facing note. The prepend ordering is only needed here.
- Keep the `skill` tool guard by name as defense in depth, for a skill that appears between a `skills/change` and the next step.
- If applying fails, try `restrict({ allow: [] })`; if even that fails, log, surface a diagnostic and let the Agent start — never throw out of `agent/created`, never silently fall back to `all`.

### Reconciliation

- Apply in `agent/created` before the first step; reconcile live Agents after a policy or set change.
- Cover the Bot's chat, every group Session it owns, and every live Agent; archived parts need nothing until resumed.
- Adopt already-live Agents on startup/reload; reapply on `tools/change` and `skills/change`.

### Preview

- Effective and available tool/skill counts; unavailable grants; available-but-not-assigned; set usage; connector state; grounded warnings.
- The client renders the host preview; it does not recompute expansion.

## Out of scope

- A per-Bot deny/exception layer on top of sets (v1 is union-only).
- Nested sets.
- Moving the DS Bot capability library into the policy.
- Per-tool selection in the Bot card outside a set (granularity lives in sets).
- Any DSH change or patch.

## Known limits

- Tools registered by a preset in its own scope are not in the global catalog; `selected` hides them and they cannot be re-assigned (today DS Bot already hides the relevant ones).
- A skill that appears between a `skills/change` and the next step can be visible for one step; the `skill` tool guard still refuses it.
- A DS Bot capability skill stores its body in `state.json` and shadows a same-name inherited skill; the two copies can drift (reported on #1341).
- Connector tools are governed by two controls: the policy allow-list and the connector switch (intersection).
- A cold Bot's catalog uses its folder, which may differ from a live Session's current `working_directory`.
- Editing a shared set changes every Bot assigned to it; the UI shows usage and warns when grants change.

## Prior art

- rml4791lol-lang's merged fork work (rml4791lol-lang/DS-Bot#60 and #62) already uses an Agent-scoped shadow mask for a per-Bot skill shortlist, and found that the host-level file catalog is empty in the web profile. This plan reuses the mask technique and the cold-scope read instead of the catalog text filter; coordinate before submitting overlapping changes upstream.

## Before the PR

- [ ] Translate this note to Chinese (the plan convention) and keep the English draft only if the maintainer accepts it.
- [ ] Add the row to the table in `docs/plans/README.md`.
- [ ] Update the **Capabilities** section of `docs/plans/2026-10-10-bot-runtime.md`: inherited capabilities are no longer additive-only; DS Bot capabilities stay additive.
- [ ] Fill in the exact API names and the test names once the spike proves the shadow mask hides a skill from the catalog, the `skill` tool and `/name` (and, only if needed, that a prepended gate sees the catalog in the fallback).
- [ ] `npm test`, `node scripts/build-client.mjs --check`, `npm run lint`, and the host smoke test for `src/host/` changes.
- [ ] Browser GIF for the interface change (synthetic data, no secrets).

## Changes

- 2026-11-17: draft skeleton (issues #1–#3).
