# idea-validator

**Validate business ideas through multi-agent simulation. Runs locally. Requires Claude Code.**

Give it an Excel workbook of business ideas and the name of one idea you're considering. It feeds the idea through [MiroFish](https://github.com/666ghj/MiroFish) — a local multi-agent prediction engine — and returns a rich narrative report covering: would early adopters sign up, top objections, price sensitivity, likely failure modes, and positioning angles.

## The angle

This tool deliberately ships as a [Claude Code](https://claude.ai/code) skill, not a Python CLI or library. The entire product is a single markdown file (`.claude/skills/validate-idea/SKILL.md`) that Claude Code reads and follows.

Why: the design is the hard part. The skill encodes ~200 lines of orchestration logic — prereq checks, workbook parsing, fuzzy matching, brief generation, MiroFish invocation, report presentation — as instructions Claude Code executes using its own tools (Bash, Python one-liners). There is no code to install, no package to maintain, no SDK to version.

If you want this as a shell command, use [`mirofish-cli`](https://github.com/mohanad7afiz/mirofish-cli) directly. This project's value is the orchestration, and the orchestration lives in the skill.

## Prerequisites

1. **[Claude Code](https://claude.ai/code)** — the runtime. The skill only works inside Claude Code.
2. **Python 3.11+** with `openpyxl`:
   ```
   pip3 install openpyxl
   ```
3. **[mirofish-cli](https://github.com/mohanad7afiz/mirofish-cli)** installed and on PATH.
4. **MiroFish backend running locally at `:5001`**:
   ```
   cd ~/tools/mirofish && docker compose up -d
   ```
   You need MiroFish installed; see [its README](https://github.com/666ghj/MiroFish) for setup.
5. **An LLM API key** configured in MiroFish's `.env` (qwen-plus is the recommended default — cheap, fast).

## Quick start

```bash
git clone https://github.com/mohanad7afiz/idea-validator
cd idea-validator
# Open this directory in Claude Code.
# The skill auto-loads from .claude/skills/
```

Then in Claude Code:

```
/validate-idea --workbook examples/sample-ideas.xlsx "Slack Reminder Bot"
```

You'll see:
1. Prereq checks (mirofish, backend, openpyxl, workbook file)
2. Idea match confirmation
3. A MiroFish run kicked off in the background (5-15 min)
4. When done, the full prediction report printed inline

## Workbook format

The skill expects your workbook to follow a 17-column format. The sample workbook at `examples/sample-ideas.xlsx` demonstrates it.

| Column | Description |
|---|---|
| Idea Name | Short name — used for matching and the run slug |
| Description | 1-3 sentences: what the product does |
| Target Audience | Who it's for |
| Revenue Model | How it makes money |
| Pricing Range | Price points |
| Monthly Income Potential | Realistic MRR range |
| Setup Cost | Upfront cost to build |
| Time to Build | Calendar time estimate |
| Maintenance Level | Low/Medium/High |
| Pros | Why this could work |
| Cons | Why it might not |
| Competitors | Who else does this |
| Market Demand | Low/Medium/High |
| Difficulty | Low/Medium/High |
| Proven Examples | Others making money from this pattern |
| Business Setup Required | Legal/corporate requirements |
| Why This Problem Hurts | The pain the product solves |

Sheets containing "scorecard" in the name are skipped. You can have multiple sheets (one per category).

## How it works

1. You invoke `/validate-idea --workbook <path> <idea-name>` in Claude Code
2. The skill verifies mirofish-cli is on PATH and the MiroFish backend is running
3. The skill reads the workbook via `openpyxl`, collects all idea names across non-scorecard sheets
4. The skill fuzzy-matches your query against those names (substring + `difflib`)
5. The skill writes a markdown brief of the matched idea's 17 fields + a fixed prediction query to a temp directory
6. The skill calls `mirofish run` in the background (5-15 minutes)
7. When complete, the skill reads `report.md` and presents it inline

## Cost and time

- **Cost per validation:** ~$0.40 on qwen-plus (default 20 rounds × ~10 personas × 2 LLM calls × $0.002)
- **Time per validation:** 5-15 minutes
- **Cost gate:** mirofish-cli automatically gates anything above $5 or 80 rounds

## Generating your own sample workbook

If you want a different sample:

```bash
cd examples
pip install openpyxl
python generate_sample.py
```

This regenerates `sample-ideas.xlsx` from the data in `generate_sample.py`.

## What this doesn't do

- ❌ No batch validation — one idea per invocation
- ❌ No write-back to the workbook
- ❌ No caching — every invocation is a fresh run
- ❌ No auto-install of mirofish-cli or MiroFish
- ❌ No Python CLI fallback — Claude Code is required

## License

MIT — see [LICENSE](./LICENSE).

## Design doc

The full design reasoning is at [`docs/plans/2026-04-08-idea-validator-design.md`](docs/plans/2026-04-08-idea-validator-design.md) if you want to understand the decisions.
