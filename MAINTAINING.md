# Maintaining han-review

How to change, release, roll out, and test `han-review`. Read this before
touching anything; several constraints here were learned the hard way (see
[Gotchas](#gotchas)).

## Architecture in one minute

`han-review` is a coding-agent-agnostic, multi-angle PR reviewer with **two homes**:

```
            handol-park/han-review  (THIS repo — single source of truth, public)
            ├── SKILL.md            orchestrator (host-agnostic)
            ├── angles/*.md         7 angle specs (the review content)
            ├── schema/*.json       finding + consolidated-review contracts
            ├── .github/workflows/review.yml   reusable CI workflow (on: workflow_call)
            ├── templates/caller.yml           stub each target repo installs
            └── bin/han-review-init            installs the stub into a repo
                       │
          ┌────────────┴─────────────┐
   LOCAL skill                    GitHub bot (CI)
   dotfiles consumes this repo    target repos add a tiny caller stub that
   as a git submodule at          `uses:` review.yml@v1; the workflow checks
   shared/claude/skills/han-review out THIS repo at runtime for the specs.
   (install.sh symlinks it into
   ~/.claude/skills + ~/.codex/skills)
```

**Single source of truth:** every angle is defined once, in `angles/`. Hosts
contribute only dispatch + transport, never review content. Do not fork angle
logic per host.

**Value system carve-out:** the authoritative value system lives in the
*consuming repo's* `AGENTS.md` (e.g. dotfiles → *What I Value*). This repo carries
ONLY the operational ladder string
(`Safety > Effectiveness > Correctness > Simplicity > Efficiency`) it needs to
rank/resolve. Never copy the full value definitions into this public repo.

## Make a change

| To change… | Edit | Notes |
|---|---|---|
| What an angle hunts / its severity rubric | `angles/<angle>.md` | Keep the section structure identical across angles (H1, Lens/Question blockquote, When to run, What you hunt, Severity rubric, Output). `security` + `reliability` are the **Safety floor** — Critical ⇒ `block`; the others are normal/advisory. |
| Add a new angle | new `angles/<name>.md` + list it in `SKILL.md` step 2 | Mirror an existing angle file. Decide if it's a Safety-floor (block) or advisory angle. |
| The finding contract | `schema/finding.schema.json` | **Strict-mode rules** (see Gotchas): every property MUST be in `required`; no `minimum`/`maximum`; `additionalProperties:false`. `refuted` is synthesis-only — keep it OUT of the schema. |
| Orchestration (dispatch, synthesis, verdict) | `SKILL.md` | Host-agnostic prose. The CI prompts in `review.yml` just say "run the orchestrator at `.han-review/SKILL.md`", so most logic changes only touch `SKILL.md`. |
| CI behavior (gating, posting, a backend) | `.github/workflows/review.yml` | See [Add a backend](#add-a-ci-backend). |
| The caller stub | `templates/caller.yml` | Must keep the `permissions:` block (see Gotchas). Bumping the stub means re-running `han-review-init` in each target repo + a PR there. |

## Release & versioning

Callers and the dotfiles submodule pin **`@v1`**. We treat `v1` like the official
actions do (`actions/checkout@v4`): **a moving pointer for the major version.**

```bash
# in this repo, after committing your change to main:
git push origin main
git tag -f v1 && git push -f origin v1        # backward-compatible release
# Breaking change to the caller contract instead? cut a new major:
#   git tag v2 && git push origin v2     (then bump caller stubs to @v2)
```

Then bump the **dotfiles** consumer so the local skill matches:

```bash
cd ~/dotfiles                                  # work in a worktree/branch, not main
git -C shared/claude/skills/han-review fetch origin
git -C shared/claude/skills/han-review checkout v1     # or the new SHA
git add shared/claude/skills/han-review
git commit -m "chore(claude): bump han-review submodule to v1 (<what changed>)"
# merge the branch, then: git submodule update --init --recursive
```

A change is **backward-compatible** (move `v1`) if it doesn't change the caller
stub's required shape or secrets. Changing the stub, the secrets it needs, or
removing a backend is **breaking** (cut `v2`).

## Roll out to a new repo

```bash
han-review-init /path/to/repo        # writes .github/workflows/han-review.yml
# then, in that repo:
#  1) add the backend secret: gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo OWNER/REPO
#     (or OPENAI_API_KEY / GEMINI_API_KEY)
#  2) commit the stub to the DEFAULT branch via a PR (issue_comment workflows
#     only fire from the default branch)
#  3) on any PR, comment:  @claude review   (or @codex review / @gemini-cli /review)
```

The host repo (`handol-park/han-review`) **must stay public** — reusable
workflows are callable cross-owner (e.g. `noegram/*` + `handol-park/*`) only when
public.

## Test a change before releasing

**Local (fast, no CI).** Run an angle in its own context against a fixture:

- *Claude:* spawn a subagent fed `angles/<angle>.md` + a diff; expect a
  schema-valid `AngleResult`.
- *Codex:* `codex exec "<spec + diff>" --output-schema schema/finding.schema.json -o out.json -s read-only </dev/null`
  (the `</dev/null` is mandatory — see Gotchas).

Validate output: `uv run --with jsonschema python3 -c "..."` against the schema.

**CI (full path).** Use a throwaway sandbox repo — never test on a live repo's main:

```bash
gh repo create handol-park/han-review-sandbox --private --clone
cd han-review-sandbox
han-review-init .                       # stub on main
gh secret set CLAUDE_CODE_OAUTH_TOKEN   # paste token
git add -A && git commit -m init && git push
# open a PR with a planted vuln, then: gh pr comment <n> --body "@claude review"
gh run watch  # expect: gate+review succeed, one consolidated BLOCK comment
gh repo delete handol-park/han-review-sandbox --yes   # clean up
```

A good regression fixture is a file with a hardcoded secret + SQL injection →
expect **Verdict: BLOCK** with two Critical `security` findings.

## Gotchas

Hard constraints discovered building this — violating them silently breaks things:

- **Codex `codex exec` hangs on stdin.** Always run it with `</dev/null`.
- **OpenAI strict structured output** (`codex --output-schema`, Claude
  `--json-schema`) requires EVERY property listed in `required` and rejects
  `minimum`/`maximum`. This is why `refuted` and `minimum` are absent from
  `schema/finding.schema.json`.
- **Reusable-workflow `startup_failure`s** we hit:
  - The caller's `GITHUB_TOKEN` is often read-only by default; a reusable
    workflow can't escalate beyond the caller. The caller stub therefore MUST
    carry its own `permissions:` block (`pull-requests: write`, `id-token: write`,
    …). Don't remove it.
  - Don't reference `needs` in a job-level `concurrency` or `env`; use
    `github.event.issue.number` instead.
- **GitHub secrets are write-only** — you can't copy a token between repos; each
  repo needs its own `gh secret set`.
- **macOS has no `timeout`** — use `gtimeout` (coreutils) in local scripts.
- **`issue_comment` workflows only run from the default branch** — a stub on a
  feature branch won't trigger.

## Layout reference

```
SKILL.md                      orchestrator (also the local skill body)
angles/{product,architect,engineer,qa,security,reliability,docs}.md
schema/finding.schema.json    one angle's result (AngleResult)
schema/review.schema.json     {review_markdown} — consolidated output for CI backends
.github/workflows/review.yml  reusable workflow (gate → backend → post one comment)
templates/caller.yml          per-repo stub (keep the permissions block)
bin/han-review-init           installs the stub into a target repo
```
