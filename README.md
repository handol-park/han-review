# han-review

A coding-agent-agnostic, multi-angle pull-request reviewer. A PR is a proposed
amendment to a product's blueprint; `han-review` convenes a panel of independent
experts who each interrogate that change from one vantage point — in its own
isolated context — then consolidates their findings into one severity-ranked
verdict.

## Two ways to run

- **Local skill** (`han-review`) — works in Claude Code and Codex. Installed via
  [dotfiles](https://github.com/handol-park/dotfiles) (this repo is a submodule
  there).
- **GitHub bot** — a reusable workflow other repos call from a tiny stub,
  invoked on-demand by commenting `@claude review`, `@codex review`, or
  `@gemini-cli /review` on a PR.

## The panel

`product` · `architect` · `engineer` · `qa` · `security` · `reliability` · `docs`
— see [`angles/`](angles/). Each angle is one agent-agnostic spec, read verbatim
by every host.

## Value ladder

Findings are weighted and conflicts resolved by:

**Safety > Effectiveness > Correctness > Simplicity > Efficiency**

This is the *operational* ladder. The authoritative definitions live in the
consuming environment's `AGENTS.md` (the repo owner's value system); this repo
carries only the ordering it needs to rank and resolve.

## Contract

Each angle returns an `AngleResult` per [`schema/finding.schema.json`](schema/finding.schema.json).

## Maintaining

See [MAINTAINING.md](MAINTAINING.md) — architecture, how to change angles/schema/
workflow, the release + versioning process, rollout, testing recipes, and gotchas.
