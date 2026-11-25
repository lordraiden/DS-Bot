# Per-Bot capability policy over inherited DSH capabilities

- Date: 2026-11-17 (rescoped 2026-11-25)
- Related: lordraiden/DS-Bot#1, #2, #3, #4; upstream Fei2411/DS-Bot#1341

> Draft skeleton. `docs/plans/README.md` asks for Chinese; this draft is in English for review and is translated before the PR (an English plan has precedent: `2026-11-04-local-agents.md`).

## Why

Every Bot runs in the same DSH profile, so every Bot inherits the profile's skills and MCP tools. Today the only per-Bot control is DS Bot's own capability library, which is additive: it cannot take away what the profile already provides. A team therefore cannot give each Bot a least-privilege surface without duplicating configuration or changing the shared profile.

**v1 is skills only.** DSH 0.2.3-alpha moved MCP tools to on-demand loading, where `tools.restrict()` no longer filters them; the MCP half is deferred to v2 (#4) so it is designed against the loader model instead of the eager one.

## Decided behavior

### Scope boundary

- v1 covers inherited **model-invocable skills** (deployment-global and ancestor-scope).
- v2 (#4) covers inherited **MCP servers** at server granularity, on-demand and eager.
- DS Bot's team tools (registered in the Agent's own scope) and the DS Bot capability library stay additive and outside the policy. A capability skill shadows a same-name inherited skill (Agent layer wins); the policy must never shadow capability names.
- A Bot running on a local agent CLI has no DSH policy: the CLI owns its tools.
- The policy never hides the `skill` tool; it only masks skills.

### Persistence

- `bot.capabilityPolicy = { mode: 'all' | 'selected', grants: Grant[] }`.
- `state.capabilitySets = { [id]: { id, name, description, grants, createdAt, updatedAt } }`; IDs are generated UUIDs, never names.
- v1 grants: `{ type: 'skill', name }` and the Bot-only reference `{ type: 'set', id }`. `mcp` and `tool` are reserved for #4 and are rejected as unknown in v1 (ignored + reported).
- An absent policy means `all`. Existing Bots are not rewritten merely to store the default.
- `selected` with an empty grant list is distinct from a missing policy and expands to nothing.
- A malformed policy object fails closed (no grants) with a diagnostic. An unknown grant `type` is ignored and reported, so a future grant type does not brick Bots on downgrade.
- A missing set ID or an unavailable skill name is unavailable, not fatal; it becomes effective again when its source returns with the same identity.
- Duplicating a Bot copies its policy; deleting a Bot leaves sets in place.

### Catalog

- Skills: `ctx.skills.snapshot({ cwd, scope })`, grouped by `source`. Live Agents use `parentScope = scopeParentOf(scopeOf(agent.ctx))` and `ctx.workingDirectory.ensure(agent, signal)`; a cold Bot uses `ctx.agentPresets.acquireScope(presetId)` (dispose the lease) and its folder or the team workspace. Never enumerate skills at host level: in the web profile the base `skill-filesystem` row is disabled and presets own discovery, so a host-level snapshot is empty (rml4791lol-lang/DS-Bot#62).
- The inherited catalog excludes names provided by attached capability skills: they are always on and are shown in the "From DS Bot" section, so the same skill is not offered twice.
- v2 adds inherited MCP servers (name, tool count, loader presence, connector state), grouped by server.
- The catalog is bounded (names, descriptions, groups) and cached, invalidated by `skills/change` (and `tools/change` in v2).

### Expansion

- One pure function used by apply, preview and tests; the client never reimplements it.
- Union of direct grants and referenced sets; duplicates deduplicated; nested sets rejected.
- v1 returns `{ skills, unavailable }`; v2 adds `{ servers, unavailableServers }`.
- Never throws on a stale grant.

### Enforcement

- Skills, primary: an Agent-scoped shadow mask. For every inherited skill the Bot may not use, register a runtime skill with the same name in the Agent's scope, with `invocation: { modelInvocable: false, userInvocable: false }`, a non-empty description and empty content. The nearest layer wins, so the model catalog omits it, the `skill` tool rejects it and `/name` skips the gesture, with no text rewriting and no ordering dependency. Skip names that are DS Bot capability skills; recompute on `skills/change`; remove with the policy.
- Skills, fail-closed fallback: if the mask cannot be established, deny the `skill` tool and register a `{ prepend: true }` `agent/pre-step` listener that removes disallowed `skill-invocation` messages and adds a model-facing note. The prepend ordering is only needed here.
- Keep the `skill` tool guard by name as defense in depth, for a skill that appears between a `skills/change` and the next step.
- v2 MCP: deny the loader (`mcp__<server>__load_tools`) for on-demand servers, or the expanded `mcp__<server>__*` names for eager servers. `restrict()` does not filter tools loaded on demand, so the loader is the boundary; per-tool selection is out of scope.
- If applying fails, log, surface a diagnostic and let the Agent start — never throw out of `agent/created`, never silently fall back to `all`.

### Reconciliation

- Apply in `agent/created` before the first step; reconcile live Agents after a policy or set change.
- Cover the Bot's chat, every group Session it owns, and every live Agent; archived parts need nothing until resumed.
- Adopt already-live Agents on startup/reload; reapply on `skills/change` (and `tools/change` in v2).

### Preview

- v1: effective and available skill counts; unavailable grants; available-but-not-assigned; set usage; capability-provided names; grounded warnings.
- v2: per-server effective/hidden state, loader/eager mode, unavailable servers.
- The client renders the host preview; it does not recompute expansion.

## Out of scope

- Per-tool MCP selection (use separate server entries/credentials for read-only vs mutate).
- Nested sets.
- Moving the DS Bot capability library into the policy.
- Skill version pinning and a promotion flow (raised by @lnLiChang on #1341): a separate lifecycle feature, not selection.
- Any DSH change or patch.

## Known limits

- A skill that appears between a `skills/change` and the next step can be visible for one step; the `skill` tool guard still refuses it.
- A DS Bot capability skill stores its body in `state.json` and shadows a same-name inherited skill; the two copies can drift (reported on #1341).
- A cold Bot's catalog uses its folder, which may differ from a live Session's current `working_directory`.
- Editing a shared set changes every Bot assigned to it; the UI shows usage and warns when grants change.
- v2 depends on the on-demand loader, which is alpha in 0.2.3; the eager fallback keeps rc.5 working.

## Prior art

- rml4791lol-lang's merged fork work (rml4791lol-lang/DS-Bot#60 and #62) already uses an Agent-scoped shadow mask for a per-Bot skill shortlist, and found that the host-level file catalog is empty in the web profile. This plan reuses the mask technique and the cold-scope read instead of the catalog text filter; coordinate before submitting overlapping changes upstream.

## Before the PR

- [ ] Translate this note to Chinese (the plan convention) and keep the English draft only if the maintainer accepts it.
- [ ] Add the row to the table in `docs/plans/README.md`.
- [ ] Update the **Capabilities** section of `docs/plans/2026-10-10-bot-runtime.md`: inherited skills are no longer additive-only; DS Bot capabilities stay additive.
- [ ] Fill in the exact API names and the test names once the spike proves the shadow mask hides a skill from the catalog, the `skill` tool and `/name` (and, only if needed, that a prepended gate sees the catalog in the fallback).
- [ ] `npm test`, `node scripts/build-client.mjs --check`, `npm run lint`, and the host smoke test for `src/host/` changes.
- [ ] Browser GIF for the interface change (synthetic data, no secrets).

## Changes

- 2026-11-17: draft skeleton (issues #1–#3).
- 2026-11-25: rescoped v1 to skills only; MCP server visibility moved to v2 (#4) after DSH 0.2.3-alpha introduced on-demand MCP tool loading, where `restrict()` no longer filters loaded tools.
