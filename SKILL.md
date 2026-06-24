---
name: han-review
description: Multi-angle PR review — convene independent angle agents (product, architect, engineer, qa, security, reliability, docs), each in its own isolated context, then consolidate into one severity-ranked verdict. Use when the user says "han-review", "review this PR/branch", or runs /han-review [<PR#>].
allowed-tools: ["Bash", "Read", "Grep", "Glob", "Task", "TodoWrite"]
---

# han-review

Convene a panel of independent reviewers over a change and return ONE
consolidated, severity-ranked verdict. Each angle runs in its own context so it
can't be anchored by the others.

**Value ladder (operational):** Safety > Effectiveness > Correctness >
Simplicity > Efficiency. Authoritative definitions live in the consuming repo's
`AGENTS.md` — this skill carries only the ordering it needs to rank and resolve.

## Steps

1. **Resolve the target.** Use the PR number if given; else the current branch's
   PR (`gh pr view --json number -q .number`); else review the working diff.
   Capture the diff with `gh pr diff <n>` (or `git diff <base>...HEAD`).

2. **Dispatch dynamically.** Read the diff and seat only the angles whose surface
   it touches. You MAY summon an ad-hoc lens the diff demands (DB migration →
   data-integrity; public endpoint → API-compat; UI → accessibility). The seven
   standing angles live in `angles/` next to this file:
   `product · architect · engineer · qa · security · reliability · docs`.

3. **Run each seated angle in its OWN context.** Feed it (a) the contents of
   `angles/<angle>.md` and (b) the diff + changed-file list. It returns an
   `AngleResult` per `schema/finding.schema.json`.
   - **Claude Code:** spawn one `Task` subagent per angle (in parallel).
   - **Codex:** run one `codex exec` process per angle, **stdin closed**, e.g.
     `codex exec "<prompt>" --output-schema schema/finding.schema.json -o out.json -s read-only </dev/null`
     (parallel; fresh context each). Closing stdin (`</dev/null`) is required —
     `codex exec` blocks waiting on stdin otherwise.
   An angle whose surface is untouched returns `applicable:false` and is dropped.

4. **Synthesize (single context — the architect's seat).** This is the only stage
   that sees every angle's findings *and their proposed fixes* together, so it does
   the big-picture reasoning no isolated angle can: measure each fix's effect,
   reconcile fixes that conflict, and prefer one simpler solution over a stack of
   per-finding patches.
   - Dedupe findings by `path:line`.
   - **Is it wrong? (cross-check):** if another angle's perspective refutes a
     finding, drop it or downgrade it (mark `refuted`). Guards against
     plausible-but-wrong findings.
   - **Measure each finding's effect and its fix's cost.** Effect/benefit comes
     from the finding's `detail` (what's wrong and why it matters) plus the boundary
     the change opens; cost comes from the **owning angle's** proposed fix — the
     domain expert's remedy (the code, abstraction, or files it adds). The architect
     weighs and reconciles the experts' fixes but does NOT author a fix in a domain
     it isn't expert in. If a finding arrives without its expert fix, don't invent
     one: leave it at the angle's stated severity (no cost-based downgrade) and note
     that the owning angle should propose a fix. Never downgrade for cost without an
     expert fix in hand.
   - **Reconcile fix-vs-fix conflicts:** where two findings' fixes pull against each
     other (one angle's guard is another's unpaid-for complexity; one's abstraction
     breaks another's simplicity), resolve by the value ladder, naming the winner
     and the reason (e.g. Correctness over Simplicity). Never surface both as equal.
   - **Over-engineering check:** when a finding or proposed fix adds scheduling,
     persistence, retries, locking, permissions hardening, abstraction,
     cross-platform support, recovery logic, or other machinery, compare it to the
     smallest design that satisfies the governing spec, acceptance criteria, or the
     change's stated intent. Machinery required by a real
     Safety/reliability/security boundary is not over-engineering. Where extra
     machinery guards a boundary the change does open but adds disproportionate
     complexity, surface the simpler solution and rank the overgrowth by its
     material cost. Where it guards a boundary the baseline never opened, the
     Polish-at-most clamp below governs. Every over-engineering finding MUST name
     the smaller working alternative and the risk it accepts.
   - **Prefer a simpler unifying solution:** when several findings share a root, or
     a simpler structure would obviate them, propose that *one* change instead of
     stacking per-finding fixes — and downgrade the findings it resolves. A guard for
     a door the baseline never opened, or defense-in-depth whose cost exceeds the
     boundary it protects, MUST NOT reach `concern` (Polish at most). This NEVER
     touches a **Critical**: a real Safety-floor issue outranks Simplicity.
   - Rank Critical → Important → Polish.

5. **Verdict.**
   - Any **Critical** `security` or `reliability` finding ⇒ **block** (the Safety
     floor).
   - Else any Critical/Important ⇒ **concerns**.
   - Else ⇒ **ship it**.

6. **Render ONE report** — a section per seated angle (skipped angles noted as
   `_skipped — no governed surface changed_`; clean angles as `_no findings_`),
   findings as `` - `path:line` — title (Severity) `` then the detail + evidence,
   ending with the verdict line and `_Reviewed <sha>._`.

7. **Post (only in CI / when `--comment` is passed).** Write the report to a temp
   file and `gh pr comment <n> --body-file`. Otherwise print it. The agent never
   self-posts in any other path — the workflow owns the single post.
