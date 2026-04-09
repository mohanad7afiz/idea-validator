# idea-validator — Design Doc

**Date:** 2026-04-08
**Author:** Mohanad (with Claude Code brainstorm)
**Status:** Draft — awaiting review
**Related:**
- mirofish-cli design: `docs/plans/2026-04-08-mirofish-claude-code-integration-design.md`
- mirofish-cli plan: `docs/plans/2026-04-08-mirofish-claude-code-integration-plan.md`
- idea-vault workbook generator: `generate_workbook.py`

---

## 1. Problem

The idea-vault workbook ranks 63 business ideas by a static weighted feasibility score. The score answers "is this feasible?" but not "would real people actually adopt this?" Before committing build time to any idea (e.g. RepostEngine took weeks of design + 7 days of build), there is no lightweight way to pressure-test it against simulated market reactions.

We just shipped `mirofish-cli` Phase 1, which gives us a clean way to drive MiroFish multi-agent prediction simulations from the terminal or Claude Code. But raw MiroFish requires you to hand-write a brief and a query for each prediction. That friction kills the "just check this idea real quick" workflow that would actually be useful.

We want a tool that makes idea validation a one-line invocation: pick an idea by name from the workbook, get back a rich MiroFish prediction report.

## 2. Non-Goals

- **No batch validation across all 63 ideas.** Single idea per invocation. Batch mode is a separate, later project — and only worth building once on-demand validation has been used enough to know what to extract.
- **No write-back to the workbook.** The workbook stays the source of truth for ideas; predictions are separate artifacts. No new columns, no schema mutations.
- **No standalone Python CLI.** The deliverable is a Claude Code skill, not a `pip install`-able package. (See §4 — this is the angle.)
- **No idea-vault dependency.** The validator works against any workbook in the idea-vault 17-column format. Idea-vault is just one possible consumer.
- **No replacement for the Feasibility Scorecard.** The scorecard answers "can I build it?"; the validator answers "would they buy it?" Different questions. They coexist.
- **No tests** beyond a manual smoke procedure documented in the README. The skill is markdown instructions, not testable code.

## 3. Constraints & Context

- **mirofish-cli Phase 1 ships**: `mirofish run <topic> --docs <dir> --query <file>` is the workhorse this validator wraps. Available at https://github.com/mohanad7afiz/mirofish-cli (currently private; will become public when stable).
- **Workbook format**: 17 columns per idea, defined in `idea-vault/generate_workbook.py`. 12 idea sheets + 1 feasibility scorecard sheet (skipped during validation).
- **Cost per validation**: ~$0.40 at default 20 rounds × ~10 personas. The mirofish-cli cost gate still applies if you bump rounds higher.
- **Time per validation**: 5–15 minutes (graph build + sim prepare + sim run + report generation).
- **MiroFish backend**: Must be running locally at `:5001` via Docker. The validator does not auto-start it.
- **The angle (positioning)**: This project is **Claude Code-native open source**. Anyone can clone the repo, but the skill only works inside Claude Code. The "code" is a single SKILL.md of orchestration instructions. There is intentionally no Python CLI fallback. Why: the design is the hard part; the runtime is Claude Code itself.

## 4. Approach

**Build a small standalone repo at `~/Documents/personal/idea-validator/` whose primary deliverable is a single Claude Code skill (`.claude/skills/validate-idea/SKILL.md`). The skill instructs Claude Code to: read an Excel workbook, fuzzy-match an idea name, generate a brief + query, call `mirofish run` via Bash, and present the resulting report.**

### Why a standalone project (not added to mirofish-cli or idea-vault)

- **Not added to mirofish-cli**: mirofish-cli is a generic wrapper around MiroFish. It must stay generic so future projects can consume it. Polluting it with workbook-specific logic narrows it to one use case.
- **Not added to idea-vault**: idea-vault is the "ideas database" — a workbook generator and content store. The validator is a *consumer* of that workbook, not part of it. They should be independently versionable. (Tomorrow, the validator could process someone else's ideas workbook, with no idea-vault involvement at all.)
- **As its own project**: clean separation of concerns, independent github visibility, clean clone-and-use experience for anyone who finds it.

### Why a Claude Code skill (not a Python CLI)

The user's positioning angle is "Claude Code-native open source." The skill IS the implementation. If someone wants to call this from cron or a script, they should use `mirofish run` directly. This project's value is the *orchestration logic* — the intelligence of "given an idea name and a workbook, do these 7 steps." That logic lives as markdown instructions Claude Code reads and follows.

This is a deliberate trade: portability is sacrificed for tightness of design and zero-code-to-maintain. The skill is the entire product.

## 5. Architecture

```
                      ┌────────────────────────────┐
                      │ idea-validator/ (clone)    │
                      │                            │
                      │ .claude/skills/            │
                      │   validate-idea/           │
                      │     SKILL.md  ◄─── Claude Code reads
                      │                  │         │
                      │ examples/        │         │
                      │   sample.xlsx    │         │
                      │                  │         │
                      │ README.md        │         │
                      └──────────────────┼─────────┘
                                         │
                                         ▼
              ┌──────────────────────────────────────────┐
              │ Claude Code (the runtime)                │
              │                                          │
              │ /validate-idea --workbook <path> "<idea>"│
              │       │                                  │
              │       ▼                                  │
              │  1. Verify prereqs (mirofish, python,    │
              │     openpyxl, MiroFish backend up)       │
              │  2. Bash: read workbook via openpyxl     │
              │  3. Fuzzy match idea name                │
              │  4. Generate brief.md + query.md         │
              │     into a temp directory                │
              │  5. Bash: mirofish run <slug>            │
              │     --docs <tmp> --query <tmp>/query.md  │
              │     --rounds 20 --yes                    │
              │     (run_in_background, 5-15 min)        │
              │  6. Read report.md from                  │
              │     ~/.local/share/mirofish-cli/runs/    │
              │  7. Present report inline + run-dir path │
              └──────────────────────────────────────────┘
                                         │
                                         ▼
                      ┌──────────────────────────┐
                      │ MiroFish backend :5001   │
                      │ (already installed,      │
                      │ Docker compose)          │
                      └──────────────────────────┘
```

### Components

**1. `SKILL.md`** — the entire product. ~200-300 lines of markdown including embedded Python one-liners. Sections:
- YAML frontmatter (`name`, `description`, `user-invokable: true`, `args`)
- Prerequisites check (with fail-fast diagnostics)
- Workbook reader (openpyxl one-liner)
- Fuzzy matcher (substring + difflib)
- Brief generator (17-field markdown)
- Query template (5 questions)
- mirofish invocation (run_in_background pattern)
- Report presentation

**2. `examples/sample-ideas.xlsx`** — a tiny workbook (3 generic ideas, not from real idea-vault) so anyone cloning can immediately try `/validate-idea --workbook examples/sample-ideas.xlsx "Sample Idea Name"` without needing their own workbook.

**3. `examples/generate_sample.py`** — the script that produces `sample-ideas.xlsx`. Committed alongside the .xlsx so anyone can regenerate it. Tiny — maybe 60 lines, just enough to produce 3 ideas in the 17-column format.

**4. `README.md`** — public-facing positioning + prerequisites + usage. The "marketing" of the project. Owns the Claude Code-native angle openly.

**5. `LICENSE`** — MIT. Standard.

**6. `.gitignore`** — `.DS_Store`, `__pycache__/`, `*.pyc`, anything else trivial.

## 6. The skill workflow in detail

### Step 1: Prerequisites check

The skill verifies before doing anything else:

| Check | Command | Failure message |
|---|---|---|
| `mirofish` on PATH | `which mirofish` | "mirofish-cli not installed. See https://github.com/mohanad7afiz/mirofish-cli" |
| MiroFish backend reachable | `curl -s -o /dev/null -w "%{http_code}" http://localhost:5001/api/graph/project/list` | "MiroFish backend not running. Start it: `cd ~/tools/mirofish && docker compose up -d`" |
| Python has openpyxl | `python3 -c "import openpyxl"` | "openpyxl not installed. Run: `pip3 install openpyxl`" |
| Workbook file exists | check `--workbook` arg | "workbook not found at <path>" |

If any check fails, print the diagnostic and stop. No partial workflow execution.

### Step 2: Read workbook

A Python one-liner via Bash:

```bash
python3 -c "
import openpyxl, json, sys
wb = openpyxl.load_workbook('$WORKBOOK_PATH', read_only=True)
ideas = []
for sheet in wb.sheetnames:
    if 'scorecard' in sheet.lower():
        continue
    ws = wb[sheet]
    headers = [c.value for c in ws[1]]
    if 'Idea Name' not in headers:
        continue
    name_col = headers.index('Idea Name')
    for row in ws.iter_rows(min_row=2, values_only=True):
        if row[name_col]:
            ideas.append({'sheet': sheet, 'name': row[name_col], 'row': dict(zip(headers, row))})
print(json.dumps(ideas))
"
```

Parses the JSON into a list of `{sheet, name, row}` dicts.

### Step 3: Fuzzy match

Substring (case-insensitive) match against `name` field. Then:
- **0 matches**: print "no idea found matching '<query>'. Closest matches:" + top 5 from `difflib.get_close_matches`. Exit.
- **1 match**: proceed.
- **>1 matches**: print numbered list, ask user to pick by number via stdin. Exit on ctrl-c.

### Step 4: Generate brief

Iterate the matched row's 17 columns and write a markdown brief:

```markdown
# <Idea Name>

## Description
<description>

## Target Audience
<target audience>

## Revenue Model
<revenue model>

…all 17 fields, in workbook order…
```

Save to `<tempdir>/brief.md`.

### Step 5: Generate query

Fixed template, parameterized with idea name + target audience:

```
Simulate how the target audience for "<Idea Name>" would react to this product
when it launches publicly. Specifically:

1. Would early adopters in "<Target Audience>" sign up? Why or why not?
2. What are the top 3 objections you'd expect to hear?
3. What price range would they actually pay?
4. What's the most likely reason this product fails in its first 6 months?
5. What positioning angle would resonate most strongly with the target audience?

Be specific. Quote individual personas where useful.
```

Save to `<tempdir>/query.md`.

### Step 6: Call MiroFish

Slug = the idea name slugified (lowercase, alphanumeric, dashes — let mirofish-cli's run_id generator handle the random suffix).

```bash
mirofish run "<slug>" --docs <tempdir> --query <tempdir>/query.md --rounds 20 --yes
```

The skill uses `run_in_background: true` because mirofish run takes 5-15 minutes. Claude Code is notified when complete.

### Step 7: Read and present the report

When mirofish run completes:
- Parse stdout for the line `report: <path>` and extract the path.
- Read `<path>` (the markdown report).
- Present to the user inline, plus tell them the run_id so they can `mirofish chat <run-id> "<follow-up question>"` later.

If mirofish run fails (non-zero exit), present its stderr verbatim and exit.

## 7. Workbook format contract

The validator assumes the workbook follows this format (matching `idea-vault/generate_workbook.py`):

| Sheet rules |
|---|
| Multiple sheets, one per category |
| Sheets with "Scorecard" in the name are skipped |
| Each idea sheet has a header row at row 1 |
| Headers must include `Idea Name` (case-sensitive) |
| Each subsequent row is one idea |

| Expected columns (17) |
|---|
| Idea Name |
| Description |
| Target Audience |
| Revenue Model |
| Pricing Range |
| Monthly Income Potential |
| Setup Cost |
| Time to Build |
| Maintenance Level |
| Pros |
| Cons |
| Competitors |
| Market Demand |
| Difficulty |
| Proven Examples |
| Business Setup Required |
| Why This Problem Hurts |

If a workbook doesn't have all 17 columns, the validator still works — it includes whatever columns are present in the brief. Missing columns are simply omitted.

## 8. Sample workbook (`examples/sample-ideas.xlsx`)

Generic, made-up ideas so the project is immediately tryable on clone. Three ideas:

1. **Slack Reminder Bot** — Generic productivity app
2. **AI Cat Photo Enhancer** — Generic consumer app
3. **API Status Dashboard** — Generic dev tool

None of these are from the real idea-vault. They exist to demonstrate the format and let new users `/validate-idea --workbook examples/sample-ideas.xlsx "Slack Reminder Bot"` on day one.

`examples/generate_sample.py` generates this file (~60 lines, same pattern as the real `generate_workbook.py` but tiny).

## 9. README structure

The README is the public-facing document that explains the project to anyone who finds it on GitHub. Sections:

1. **Headline + one-line pitch**: "Validate business ideas through multi-agent simulation. Runs locally. Requires Claude Code."
2. **The angle**: why this ships as a Claude Code skill, not a CLI
3. **Prerequisites**: Claude Code, Python + openpyxl, mirofish-cli, MiroFish backend, LLM API key
4. **Quick start**:
   - `git clone https://github.com/mohanad7afiz/idea-validator`
   - `cd idea-validator`
   - Open in Claude Code
   - `/validate-idea --workbook examples/sample-ideas.xlsx "Slack Reminder Bot"`
5. **Workbook format**: the 17-column contract above as a small table
6. **How it works**: a numbered 7-step explanation matching §6 above
7. **Cost expectations**: ~$0.40 per validation at default 20 rounds
8. **Why public**: "the design is the hard part — the runtime is Claude Code"
9. **License**: MIT

## 10. Error handling

| Failure mode | Skill behavior |
|---|---|
| `mirofish` not on PATH | Print install link, stop |
| Backend down | Print start command, stop (no auto-start) |
| `openpyxl` missing | Print pip install command, stop |
| Workbook file missing | Print path, stop |
| Workbook has no `Idea Name` column in any sheet | Print "no recognizable workbook format", stop |
| Idea name not found | Print closest matches, stop |
| Multiple matches | Numbered prompt, user picks |
| `mirofish run` fails | Print mirofish stderr verbatim, stop |
| `mirofish run` cost-gates and user declines | Print refusal, exit (mirofish-cli handles this) |
| Report file missing after successful run | Print run dir path, stop |

The skill **never** silently retries or substitutes a fallback. Every failure is loud and actionable.

## 11. Cost and time

- **Cost per validation**: ~$0.40 (20 rounds × ~10 personas × 2 calls × $0.002, qwen-plus baseline)
- **Time per validation**: 5–15 minutes
- **Cost gate**: handled by mirofish-cli automatically; if rounds > 80 or cost > $5, user must confirm

## 12. Project layout

```
~/Documents/personal/idea-validator/
├── .claude/
│   └── skills/
│       └── validate-idea/
│           └── SKILL.md
├── examples/
│   ├── generate_sample.py
│   └── sample-ideas.xlsx
├── docs/
│   └── plans/
│       └── 2026-04-08-idea-validator-design.md  (copy of this file, travels with the repo)
├── README.md
├── LICENSE
└── .gitignore
```

The design doc and the eventual implementation plan can either:
- (a) Live in this project's own `docs/plans/` (own project, own docs)
- (b) Live in idea-vault's `docs/plans/` (control center for all design docs)

Recommendation: **(a)** — the project is its own thing. The design doc travels with the repo. Only the implementation plan lives in idea-vault as a "scratch" doc that doesn't need to be public.

## 13. GitHub plan

Same flow as mirofish-cli:

1. `mkdir ~/Documents/personal/idea-validator && cd && git init -b main`
2. Build the files (skill, README, sample, license, .gitignore)
3. Commit on local main as a series of TDD-ish atomic commits (one per file group)
4. Create GH repo: `gh repo create idea-validator --public --description "Validate business ideas through multi-agent simulation. Claude Code-native."`
5. Branch local main → `feat/initial`
6. Create empty main with just initial readme commit
7. Push both branches
8. Open PR `feat/initial` → `main`
9. User reviews, merges via UI
10. Sync local main, delete merged feature branch

## 14. Open Questions

1. **Project name** — `idea-validator` is the working name. Alternatives: `mirofish-idea-validator` (clearer about what it wraps but uglier), `idea-prophet` (memorable but jokey), `validate-ideas` (verb-y). Recommendation: `idea-validator` for clarity.

2. **Visibility** — Public from day one (matches the angle). Mirofish-cli is currently private until stable; should idea-validator wait too? Recommendation: **public from day one**, because the angle (Claude Code-native open source) requires public visibility to mean anything.

3. **Sample workbook ideas** — generic ones the user can verify (Slack bot, cat enhancer, API dashboard). Open to suggestions if you have specific generic ideas you'd rather see.

4. **License** — MIT (default for OSS). Open to switch to Apache 2.0 or another if you have a preference.

5. **Should `examples/generate_sample.py` be runnable as `python examples/generate_sample.py` or behind a make target?** Recommendation: just runnable. Simple.

## 15. Success Criteria

- [ ] After implementation, cloning the repo + opening in Claude Code + invoking `/validate-idea --workbook examples/sample-ideas.xlsx "Slack Reminder Bot"` produces a MiroFish report
- [ ] The skill auto-loads from `.claude/skills/` when Claude Code opens the repo
- [ ] The README is clear enough that someone who has never seen MiroFish can set up prerequisites and use the validator
- [ ] mirofish-cli stays clean (no idea-validator-specific changes leak into it)
- [ ] idea-vault stays unchanged (no validator code lives there)
- [ ] The project is small (<10 files, <500 lines total including README)

---

**Next step:** Review this doc. When approved, we move to `writing-plans` to produce the implementation plan, then execute it inline (subagent fallback per saved memory).
