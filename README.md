# JobBobber Harness Spec

This repository is a fork of [openai/symphony](https://github.com/openai/symphony), adapted into the working specification for the **JobBobber agent harness**. It is a spec-and-docs repo: there is no runtime here, no service to deploy, and no code dependency to install. The deliverable is the spec itself.

The fork is pinned to upstream commit [`58cf97d`](https://github.com/openai/symphony/commit/58cf97d) (`fix(elixir): configure Codex app-server model via config`) — the last upstream commit before this fork. We are not tracking upstream — Symphony's value to us is the harness *pattern*, captured once at this commit and adapted from there. OpenAI has stated they don't plan to maintain Symphony as a product, so re-evaluation against any future upstream change is a fresh research task rather than a continuous integration burden.

## What we kept and why

Symphony's contribution is the **harness pattern**: the structured envelope around an autonomous agent that makes its output safe to launch, legible while running, and verifiable when complete. That envelope has six elements that travel across domains — a versioned agent contract, a sandboxed execution context, a defined tool surface, an untrusted-input posture, a run-to-completion rule, and a proof-of-work artifact. Those are what we are adapting.

Everything else in Symphony — Linear as the control plane, the Codex App Server JSON-RPC protocol, the Elixir reference implementation, the polling scheduler — is a deployment choice we don't share. Those pieces have been removed from this fork. `SPEC.md` remains untouched in this pass and is the source material for the editorial transformation that follows.

## What this is for: JobBobber

[JobBobber](https://github.com/Little-Town-Labs) is a two-sided AI hiring marketplace. Job seekers and employers each have an autonomous agent acting on their behalf. When a seeker ticket and an employer ticket meet, a **match ticket** is created and the two agents conduct a structured negotiation through a privacy filter. Each agent independently scores the fit against its own versioned rubric, and each side's human has a notification threshold.

The harness specified in this repo is what wraps each negotiation run. It defines the agent contract, the execution context, the privacy-filter boundary, the run-to-completion rule, and the **match dossier** that serves as proof-of-work — scores with rubric breakdowns, redacted transcript, agent rationale, flags, and version metadata. Auditability against EEOC, NYC Local Law 144, and similar AI-hiring rules is a first-class requirement of the spec, not a future concern.

## Repository contents

- [`SPEC.md`](SPEC.md) — the upstream Symphony spec, preserved verbatim. The next editorial pass will transform this into the JobBobber harness spec.
- [`JOBBOBBER_ADAPTATIONS.md`](JOBBOBBER_ADAPTATIONS.md) — companion doc to `SPEC.md` recording which Symphony patterns we're adopting, which we're explicitly not, and what's still open. The reasoning behind the pass-two transformation lives here.
- [`LICENSE`](LICENSE) — Apache License 2.0, inherited from upstream.
- [`NOTICE`](NOTICE) — upstream copyright notice, preserved as required by Apache 2.0.

## Attribution

Originally forked from [openai/symphony](https://github.com/openai/symphony), pinned to commit [`58cf97d`](https://github.com/openai/symphony/commit/58cf97d). Symphony is © 2025 OpenAI, licensed under Apache 2.0. The original `LICENSE` and `NOTICE` files are preserved in this repository unchanged.

For background on the harness pattern this spec builds on, see OpenAI's [harness engineering post](https://openai.com/index/harness-engineering/) and the [Symphony announcement](https://openai.com/index/open-source-codex-orchestration-symphony/).

## License

This project is licensed under the [Apache License 2.0](LICENSE), inherited from the upstream Symphony project.
