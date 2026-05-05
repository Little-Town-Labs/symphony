# Parley

**Parley** is the negotiation agent harness for [JobBobber](https://github.com/Little-Town-Labs). It wraps each match-ticket negotiation between a seeker-side agent and an employer-side agent, mediates their cross-side communication through a privacy filter, and produces a versioned **match dossier** as the proof-of-work artifact for human review and audit.

This repository is the **Parley specification**. It is a spec-and-docs repo: no runtime, no service to deploy, no code dependency to install. The deliverable is `SPEC.md`.

This fork was originally adapted from [openai/symphony](https://github.com/openai/symphony) (pinned to commit [`58cf97d`](https://github.com/openai/symphony/commit/58cf97d)). Symphony's contribution was the **harness pattern** — the structured envelope around an autonomous agent that makes its output safe to launch, legible while running, and verifiable when complete. Parley adopts that pattern and adapts it for JobBobber's two-sided negotiation runs. We are not tracking upstream; OpenAI has stated they don't plan to maintain Symphony as a product.

## What Parley is for

JobBobber is a two-sided AI hiring marketplace. Job seekers and employers each have an autonomous agent acting on their behalf. When a seeker ticket and an employer ticket meet, a match ticket is created — that's the trigger Parley listens for. Two agents conduct a structured, round-bounded negotiation through a privacy filter; each agent independently scores the fit against its own versioned rubric; the harness assembles a single canonical dossier with per-audience transcript projections, deterministic weighted totals, agent rationale, and full version metadata.

Auditability under EEOC, NYC Local Law 144, and similar AI-hiring regulation is a first-class correctness requirement of Parley — not an observability nice-to-have. Compliance hooks (versioned rubrics with bias-test artifacts, hash-chained audit log, dossier signing in production posture) are normative in the spec, not optional.

## Repository contents

- [`SPEC.md`](SPEC.md) — the Parley specification. Eighteen normative sections covering the domain model, agent contract registry, run-state machine, Inngest event topology, privacy filter, side runner protocol, audit log, regulatory posture, and reference algorithms. This is the contract.
- [`PARLEY_ADAPTATIONS.md`](PARLEY_ADAPTATIONS.md) — companion doc recording which Symphony patterns Parley adopted (and where each one landed in `SPEC.md`), which patterns were explicitly not adopted, and what remains genuinely open. Decisions-taken log, not a forward-looking plan.
- [`LICENSE`](LICENSE) — Apache License 2.0, inherited from the upstream Symphony fork point.
- [`NOTICE`](NOTICE) — upstream copyright notice, preserved as required by Apache 2.0.

## Attribution

Parley's specification originated as a fork of [openai/symphony](https://github.com/openai/symphony), pinned to commit [`58cf97d`](https://github.com/openai/symphony/commit/58cf97d). Symphony is © 2025 OpenAI, licensed under Apache 2.0. The original `LICENSE` and `NOTICE` files are preserved in this repository unchanged. Symphony's specification has been transformed section-by-section into Parley's; git history preserves the original.

For background on the harness pattern Parley builds on, see OpenAI's [harness engineering post](https://openai.com/index/harness-engineering/) and the [Symphony announcement](https://openai.com/index/open-source-codex-orchestration-symphony/).

## License

This project is licensed under the [Apache License 2.0](LICENSE), inherited from Symphony.
