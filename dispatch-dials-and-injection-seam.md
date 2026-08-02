---
title: "Kickoff/swarm dispatch dials and prompt-injection seam"
tags: ["design-doc"]
sources: []
contributors: ["unknown"]
created: 2026-08-02
updated: 2026-08-02
---


## Design Specification

### Summary

Add per-dispatch reasoning/spend dials (`--effort`, `--budget-usd`) to the
kickoff/swarm launch path and make them recorded, and make a configured
`agent.kickoff_template` compose *with* crosslink's tool-maintained prompt
(placeholder interpolation + a per-dispatch `--template` flag + swarm per-agent
scoping) instead of wholesale-replacing it. This closes upstream
dollspace-gay/crosslink #61 (effort/thinking-budget dial) and #62 (injection
seam), which vsdd's R1–R5 (magnificentlycursed/crosslink#2) require to ride
crosslink for conformant, verifiable dispatch. Delivered as two implementation
slices sharing `KickoffOpts`/`lifecycle.rs`: the effort/budget dial first
(independent, ships on `develop` today), then the seam (gated on the
`kickoff_template` mechanism reaching `develop`).

### Requirements

- REQ-1: `crosslink kickoff run` and `kickoff plan` accept `--effort <level>` (`low|medium|high|xhigh|max`), threaded through `KickoffOpts`/`PlanOpts` (`src/commands/kickoff/types.rs:81`, `:212`) into `build_agent_command` (`src/commands/kickoff/launch.rs:186`) and the container command in `launch_container` (`launch.rs` ~`:811`), emitted as `claude --effort <level>` immediately after the existing `--model {model}`.
- REQ-2: The same two commands accept `--budget-usd <amount>`, emitted as `claude --max-budget-usd <amount>` on both launch paths — a fail-closed spend ceiling for unattended dispatch. Both flags are optional; omitting them reproduces today's invocation byte-for-byte. Under swarm the cap is **per-agent** (each dispatched session receives the given cap), matching the per-session nature of the claude flag — see Decision D1.
- REQ-3: `effort`, `budget_usd`, and the resolved `model` are recorded at launch in `.kickoff-metadata.json` by extending `KickoffMetadata` (`src/commands/kickoff/types.rs:57`), so the run-record oracle (see the `run-record-capability-inventory` knowledge page) captures the dispatch dials — the WAS-side gap R5 names as "effort absent on kickoff-path records".
- REQ-4: Swarm de-hardcodes `model: "opus"` (`src/commands/swarm/lifecycle.rs:734`) and threads `effort`/`budget_usd`/`model` from swarm config into the `KickoffOpts` it builds at `lifecycle.rs:729`, so swarm-dispatched agents carry and record the same dials as direct kickoff.
- REQ-5: A configured `agent.kickoff_template` is interpolated at prompt assembly rather than replacing the built prompt: the placeholders `{{built_prompt}}`, `{{issue_id}}`, `{{branch}}`, `{{description}}`, plus the dispatch-dial placeholders `{{model}}`, `{{effort}}`, `{{doc_path}}`, and `{{allowed_tools}}` (Decision D2, so a consumer can echo the dials into its injected manifest for R4/R5), are substituted at the `run.rs` step-5 seam (the point that today calls `build_prompt`, `src/commands/kickoff/run.rs:114`). The consumer supplies only its layer; crosslink's 15-step protocol stays tool-maintained. (BLOCKED: the `kickoff_template` mechanism is main-only at commit `26ee1885` and absent on `develop`; see Open Question Q3.)
- REQ-6: `kickoff run`/`plan` accept `--template <path>` flowing into `KickoffOpts`/`PlanOpts`, with the `agent.kickoff_template` config value as fallback, so the template is dispatch-scoped and parallel dispatches do not race on the single repo-global `hook-config.json` path.
- REQ-7: Swarm supports per-phase template files keyed by phase name, so a configured template no longer applies uniformly to every agent in a wave and discards the per-bullet `description` that differentiates them (`lifecycle.rs:729`, `build_prompt` at `src/commands/kickoff/prompt.rs:186`).
- REQ-8: The fully-assembled prompt continues to be written to each agent's `KICKOFF.md` (`run.rs:117`) after any templating, preserving it as the injection audit record R5 depends on.

### Acceptance Criteria

- [ ] AC-1: `build_agent_command(.., effort = Some("high"), ..)` produces a command string containing ` --effort high` positioned after `--model`; `effort = None` produces no `--effort` token (unit test alongside the existing `test_build_agent_command_*` cases in `src/commands/kickoff/tests.rs`).
- [ ] AC-2: `budget_usd = Some("5.00")` produces ` --max-budget-usd 5.00`; `None` produces no such token (same test family).
- [ ] AC-3: `crosslink kickoff run --effort bogus <desc>` exits non-zero with a clap value error naming the allowed set (integration test in `tests/cli_integration.rs`, mirroring the gh#66 conflict test).
- [ ] AC-4: After a non-dry-run `kickoff run --effort high --budget-usd 5 --model opus`, `.kickoff-metadata.json` deserializes to a `KickoffMetadata` whose `effort == "high"`, `budget_usd == "5"`, `model == "opus"` (serde round-trip unit test).
- [ ] AC-5: The swarm `KickoffOpts` built at `lifecycle.rs:729` carries the swarm-config `model`/`effort`/`budget_usd` (no literal `"opus"` remains; unit test on the opts-construction helper).
- [ ] AC-6: With the `kickoff_template` mechanism present, a template body containing `{{built_prompt}}` yields an assembled prompt in which the token is replaced by the full `build_prompt` output, and each of `{{issue_id}}`, `{{branch}}`, `{{description}}`, `{{model}}`, `{{effort}}`, `{{doc_path}}`, `{{allowed_tools}}` is replaced by its value (an unset optional like `{{effort}}` with no `--effort` renders empty) (unit test; gated on REQ-5's dependency).
- [ ] AC-7: When both `--template <path>` and `agent.kickoff_template` are set, the CLI flag wins; with only the config set, the config path is used (unit test on the resolution helper).
- [ ] AC-8: A swarm run configured with a per-phase template applies the phase-matched template to that phase's agents and leaves other phases on the built prompt, with each agent's `description` still present in its assembled prompt (integration test).
- [ ] AC-9: After a templated dispatch, the worktree's `KICKOFF.md` contains the fully-assembled (post-interpolation) prompt (assertion in the AC-8 test).

### Architecture

Two slices share one surface — `KickoffOpts` (`src/commands/kickoff/types.rs:81`)
and the swarm `lifecycle.rs` opts construction (`:729`) — so they are designed
together and implemented in sequence.

**Slice 1 — dials (#61), ships on `develop` now.** `claude` exposes both
`--effort <low|medium|high|xhigh|max>` and `--max-budget-usd <amount>` as
session flags (verified against the installed CLI; see the
`runtime-harness-surface` knowledge page for the broader effort/model dial
inventory). The change mirrors how `--model` already flows: add
`effort: Option<&str>` and `budget_usd: Option<&str>` to `KickoffOpts` and
`PlanOpts`; add the clap args to the `Run`, `Plan`, and swarm subcommands in
`src/main.rs` (the `Plan` command already gained `--skip-permissions`/
`--permission-mode` via gh#66, so this follows that pattern), with
`--effort`'s `value_parser` a `PossibleValuesParser` over the five levels (same
idiom as `--permission-mode`, `main.rs:1700`); thread them into
`build_agent_command` (`launch.rs:186`) and the `launch_container` command
string (`launch.rs` ~`:811`), each value passed through `shell_escape_arg`
(`src/utils.rs`, the shell-safety idiom in the `architecture-code-patterns`
page) and emitted after `--model`. Recording (REQ-3, R5): extend
`KickoffMetadata` (`types.rs:57`, currently `started_at`/`timeout_secs`) with
optional `model`/`effort`/`budget_usd`, written at `run.rs:158` where
`.kickoff-metadata.json` is already produced; the write stays best-effort (the
existing pattern), since a missing dial record must not abort a launch. Swarm
(REQ-4): replace the `model: "opus"` literal and add the dials at
`lifecycle.rs:734`, sourced from swarm config. Error philosophy follows the
codebase (`architecture-code-patterns` / `config-filesystem-conventions`):
clap validates the effort level fail-closed; metadata writes are best-effort.

**Slice 2 — seam (#62), gated on the template mechanism reaching `develop`.**
Today `develop`'s `run.rs:114` calls `build_prompt` unconditionally and writes
its output to `KICKOFF.md` (`:117`); the `agent.kickoff_template` full-
replacement mechanism exists only on `main` (`26ee1885`). Once that syncs to
`develop`, REQ-5 changes the replacement into interpolation at that seam:
compute `built = build_prompt(..)`, read the template, and substitute
`{{built_prompt}}` → `built`, `{{issue_id}}`/`{{branch}}`/`{{description}}` →
their values; the result is what gets written to `KICKOFF.md`, preserving the
audit record (REQ-8). REQ-6 adds `template: Option<&Path>` to
`KickoffOpts`/`PlanOpts` and a `--template` clap arg, resolved as
CLI-flag-else-`agent.kickoff_template`-config (config read via the layered
`hook-config.json` path documented in `config-filesystem-conventions`). REQ-7
(swarm per-agent): because swarm rides `kickoff::run` (`lifecycle.rs` calls
`kickoff::run` per agent) and the template check sits upstream of
`build_prompt`, a repo-global template currently overwrites every wave agent's
differentiated `description`; per-phase template files keyed by phase name let
each phase's agents interpolate a phase-specific template while retaining their
own `description` via `{{description}}`. The dynamic per-agent **exec-hook**
(crosslink invokes a consumer command per agent, injecting its stdout) is the
vsdd end-state but is a larger, separate surface — documented as Q2/Out of
Scope, implemented after per-phase templates.

**R4 coherence:** both slices mutate `KickoffOpts` and the swarm opts builder;
doing them in one design avoids two passes over `lifecycle.rs`'s hardcoding and
`KickoffOpts`'s field set. **R5 auditability:** `KICKOFF.md` remains the
per-agent record of exactly what was injected, and `.kickoff-metadata.json`
gains the dial record — together the WAS-side oracle the consumer verifies
against.

### Out of Scope

- The dynamic per-agent **exec-hook** for swarm (crosslink invoking a consumer command per dispatch to source each prompt). It is the vsdd composition-by- construction end-state but a distinct, larger surface (per-dispatch process execution, failure/timeout policy, argument/stdin contract); it is designed as a follow-on after per-phase templates (REQ-7), not implemented in this cycle.
- Any change to the built-in prompt's *content* (the 15-step protocol text). The seam only changes how a consumer template composes with it.
- Consumer-specific logic inside crosslink. The seam and dials are generic — any tool composing dispatch context or setting reasoning dials can use them.
- New model-selection behavior beyond de-hardcoding swarm's `"opus"` — model resolution/defaults are unchanged.

### decisions

All open questions from the design interview are resolved (no `<!-- OPEN -->`
blocks remain).

### D1: `--budget-usd` is a per-agent cap under swarm
`claude --max-budget-usd` caps a single session's spend, so `--budget-usd`
applies **per dispatched agent**: each session in a swarm wave receives the
given cap (total wave spend up to N×amount). This matches the flag's
per-session semantics, needs no division policy, and cannot starve a mid-wave
agent. A hard wave-level ceiling is explicitly not attempted (Out of Scope).

### D2: Extended interpolation placeholder set
The seam exposes, in addition to `{{built_prompt}}`/`{{issue_id}}`/`{{branch}}`/
`{{description}}`, the dispatch-dial placeholders `{{model}}`, `{{effort}}`,
`{{doc_path}}`, and `{{allowed_tools}}`. This lets a consumer echo the exact
dispatch dials into its injected manifest, which the consumer's R4/R5
verification reads back from the run record — closing the loop between what was
dialed and what the manifest claims. Each placeholder is escaped at
substitution; an unset optional (e.g. `{{effort}}` with no `--effort`) renders
empty.

### D3: Sequencing — Slice 1 now, Slice 2 held for the `kickoff_template` sync
The interpolation seam is a small edit on top of the `kickoff_template`
mechanism, which is main-only (`26ee1885`) and absent on `develop`. Slice 1
(the effort/budget dials, REQ-1–REQ-4) is independent and lands on `develop`
immediately. Slice 2 (REQ-5–REQ-7) is held until `kickoff_template` reaches
`develop` through a normal `main`→`develop` sync — not front-run by basing the
PR on `main` (which would split the bundle across base branches). Track the sync
as the gate on Slice 2's PR.

