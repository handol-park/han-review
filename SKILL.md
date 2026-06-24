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

4. **Synthesize (single context).**
   - Dedupe findings by `path:line`.
   - **Adversarially cross-check:** if another angle's perspective refutes a
     finding, drop it or downgrade it (mark `refuted`). This guards against
     plausible-but-wrong findings.
   - **Resolve real conflicts by the value ladder**, naming the winner and the
     reason (e.g. architect "simpler" vs engineer "less correct" → Correctness
     wins). Never surface both as equal.
   - **Proportionality pass (non-Critical only):** the cross-check above filters
     findings that are *wrong*; this one filters findings that are *not worth it*.
     For any Important/Polish finding whose *fix* adds code, a new abstraction, or
     a new file, weigh that cost against the boundary the change actually opens. If
     the same diff's architect lens flags the fix as unpaid-for complexity, or it
     guards a door the baseline never opened, downgrade it to Polish/`ok`. Net an
     "add a guard" finding against its own complexity cost — real-but-disproportionate
     defense-in-depth MUST NOT reach `concern`. NEVER applies to Critical findings:
     a real Safety-floor issue outranks Simplicity and stays untouched.
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
