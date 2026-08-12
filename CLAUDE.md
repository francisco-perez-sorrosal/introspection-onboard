# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An **Introspection agent monorepo**: one repository holding several independently deployable agent *recipes*, each in its own top-level directory and registered by a manifest in `.introspection/`. Today it holds one — `everyday-muse`.

There is no build step, no compiler, and no test suite. TypeScript extensions are loaded directly by the Pi runtime; `introspection check` is the only validation gate, and it runs in the pre-commit hook and in CI (`.github/workflows/recipe-validation.yml`, Node 24).

## Commands

```bash
introspection local --runtime everyday-muse              # interactive session
introspection local --runtime everyday-muse -p "Go."     # one-shot smoke test
introspection check                                      # validate every recipe package
```

`--runtime` is required whenever more than one recipe exists; `.introspection/local.json` (gitignored) records the default for a directory. The CLI is not a project dependency — invoke it as `npx -y -p @introspection-ai/cli introspection <cmd>` if it is not on `PATH`.

Git hooks are activated by `npm run prepare` inside a recipe directory (sets `core.hooksPath=.githooks`); `.githooks/pre-commit` runs `introspection check`.

## Use the Introspection plugin skills

The `introspection:create`, `improve`, `deploy`, `operate`, and `migrate` skills are the authoritative workflows for this repo — they carry the current CLI surface and resolve versioned references at runtime. Load the matching skill before scaffolding a recipe, changing a deployed agent, configuring bindings, or investigating a live runtime. Do not reconstruct those procedures from memory or from this file.

## Monorepo contract

| Layer | File | Effect of a change |
|---|---|---|
| Registration | `.introspection/<recipe>.yaml` — `name`, `path`, `description`, `runtime.llm_mode` | Requires a **new runtime version** |
| Behavior | `<recipe>/SYSTEM.md`, `<recipe>/agents/*.yaml` | Requires a **fresh session** |
| Capabilities | `<recipe>/extensions/*.ts` | Requires a **fresh session** |

A recipe's `package.json` `pi` field declares which agent YAMLs and extension entrypoints load; `peerDependencies` on `@earendil-works/pi-*` are supplied by the Pi runtime — never install them. Packages are `private: true` and are not published to npm.

Recipes deploy independently by path, so a change confined to one directory does not touch the others.

## Invariants to preserve

These are deliberate and easy to "fix" into breakage:

- **Absence of a credential is not an error.** `extensions/parallel-search.ts` sends `x-api-key` only when `PARALLEL_API_KEY` is in the environment. Locally the shell supplies it; on a deployed runtime an endpoint binding injects it at the egress boundary and the key never enters the sandbox. Do not add a guard that fails the call when the variable is unset — that breaks every deployed run.
- **`ok` / `empty` / `error` stay distinct.** A tool result that collapses "the search broke" into "the web had nothing" makes the agent report absence with false confidence. Both tools and `SYSTEM.md` depend on the three-way distinction.
- **Quote only what search returned.** `everyday-muse`'s single product guarantee is that every quotation and every attribution came from a search excerpt. A run ending "I could not find a sourced quote" is a *correct* run. Any edit to `SYSTEM.md` must keep that boundary and the parallel rule for the author lookup.
- **Search results are untrusted content.** Results are rendered as labelled data with an explicit untrusted-content heading, never as free prose. Keep that framing when changing `renderResults`.
- **API request shape.** `/v1/search` rejects `max_results` / `max_chars_per_result` as `extra_forbidden` — trimming happens on the response, not in the request.

## Deploying

Deployment runs through the `deploy` skill. Note: opening a PR repoints the staging runtime to `pr/N` on its own — the manual `runtimes pin` step some docs describe is stale.

## Design direction

`introspection_self_improving_agent_mvp.md` (untracked, design-stage) specifies a planned self-improving-agent MVP: a target recipe evaluated by τ-bench `banking_knowledge`, with Claude Code plus the Introspection plugin as an improvement orchestrator using `operate` for diagnosis and `improve` for harness mutation. None of it is implemented. If work starts on it, its stated constraints govern: the benchmark and evaluator are immutable, the model is frozen so that measured deltas attribute to the harness, evidence is open-coded before any failure taxonomy is imposed, and recipe changes reach `main` through PRs rather than direct writes.
