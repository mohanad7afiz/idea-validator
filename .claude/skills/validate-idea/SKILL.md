---
name: validate-idea
description: Validate a single business idea from an Excel workbook by running it through MiroFish multi-agent simulation, returning a rich prediction report.
user-invokable: true
args:
  - name: workbook
    description: "Path to the Excel workbook (.xlsx) containing ideas in the 17-column format"
    required: true
  - name: idea
    description: "The idea name to validate (case-insensitive substring match)"
    required: true
---

# validate-idea

Validates a single business idea by orchestrating a MiroFish prediction run. You read an Excel workbook, fuzzy-match the idea name, generate a brief + query, call `mirofish run` via Bash, and present the resulting report.

## How to invoke

```
/validate-idea --workbook <path-to-workbook.xlsx> <idea-name>
```

Example:
```
/validate-idea --workbook examples/sample-ideas.xlsx Slack Reminder Bot
```

## Workflow — execute these steps in order

### Step 1: Verify prerequisites

Before doing anything, check all four prerequisites. If any fail, print the diagnostic and STOP — do not continue.

```bash
# Check 1: mirofish CLI is on PATH
if ! command -v mirofish >/dev/null 2>&1; then
  echo "❌ mirofish-cli is not installed or not on PATH."
  echo "   Install it: https://github.com/mohanad7afiz/mirofish-cli"
  exit 1
fi

# Check 2: MiroFish backend is reachable at :5001
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5001/api/graph/project/list 2>/dev/null || echo "000")
if [ "$HTTP_CODE" = "000" ]; then
  echo "❌ MiroFish backend is not running at http://localhost:5001"
  echo "   Start it: cd ~/tools/mirofish && docker compose up -d"
  exit 1
fi

# Check 3: Python has openpyxl
if ! python3 -c "import openpyxl" 2>/dev/null; then
  echo "❌ Python module 'openpyxl' not installed."
  echo "   Install it: pip3 install openpyxl"
  exit 1
fi

# Check 4: Workbook file exists
WORKBOOK="$1"  # passed from the user's --workbook argument
if [ ! -f "$WORKBOOK" ]; then
  echo "❌ Workbook not found: $WORKBOOK"
  exit 1
fi
```

### Step 2: Read the workbook

Use a Python one-liner to dump all idea names + full row data as JSON. Skip any sheet with "scorecard" in the name. Only include sheets that have an "Idea Name" header column.

```bash
IDEAS_JSON=$(python3 <<'PY'
import openpyxl
import json
import sys

workbook_path = "WORKBOOK_PATH_HERE"  # replace with actual path
wb = openpyxl.load_workbook(workbook_path, read_only=True)
ideas = []
for sheet_name in wb.sheetnames:
    if "scorecard" in sheet_name.lower():
        continue
    ws = wb[sheet_name]
    rows = list(ws.iter_rows(values_only=True))
    if not rows:
        continue
    headers = list(rows[0])
    if "Idea Name" not in headers:
        continue
    name_col = headers.index("Idea Name")
    for row in rows[1:]:
        if row[name_col] is None:
            continue
        row_dict = {h: v for h, v in zip(headers, row) if h is not None}
        ideas.append({"sheet": sheet_name, "name": row[name_col], "row": row_dict})
print(json.dumps(ideas))
PY
)
```

### Step 3: Fuzzy-match the idea name

Using the IDEAS_JSON from Step 2, find ideas whose name contains the user's query (case-insensitive substring match).

```bash
QUERY="the user's idea argument"
python3 <<PY
import json, difflib, sys
ideas = json.loads('''$IDEAS_JSON''')
query = "$QUERY".lower()
matches = [i for i in ideas if query in i["name"].lower()]

if not matches:
    # No matches — show 5 closest by difflib
    all_names = [i["name"] for i in ideas]
    close = difflib.get_close_matches(query, all_names, n=5, cutoff=0.3)
    print("MATCH_STATUS=none")
    print(f"CLOSE_MATCHES={json.dumps(close)}")
elif len(matches) == 1:
    print("MATCH_STATUS=one")
    print(f"MATCHED_IDEA={json.dumps(matches[0])}")
else:
    print("MATCH_STATUS=multiple")
    print(f"CANDIDATES={json.dumps(matches)}")
PY
```

Handle each outcome:
- **`none`**: Print "no idea found matching '<query>'" and list the 5 closest matches. Stop.
- **`one`**: Proceed to Step 4 with the single matched idea.
- **`multiple`**: Print a numbered list of candidate idea names. Ask the user to pick by number (use the AskUserQuestion tool). Use their pick as the matched idea.

### Step 4: Generate the brief

Create a temp directory and write `brief.md` containing all 17 fields of the matched idea's row, verbatim, as markdown.

```bash
TMPDIR=$(mktemp -d -t validate-idea-XXXXXX)
# Write brief.md from the matched row
python3 <<PY > "$TMPDIR/brief.md"
import json
idea = json.loads('''$MATCHED_IDEA''')
name = idea["name"]
row = idea["row"]
print(f"# {name}\n")
# Ordered columns — if a column is missing in the row, skip it
columns = [
    "Description", "Target Audience", "Revenue Model", "Pricing Range",
    "Monthly Income Potential", "Setup Cost", "Time to Build",
    "Maintenance Level", "Pros", "Cons", "Competitors", "Market Demand",
    "Difficulty", "Proven Examples", "Business Setup Required",
    "Why This Problem Hurts",
]
for col in columns:
    value = row.get(col)
    if value is None:
        continue
    print(f"## {col}\n{value}\n")
PY
```

### Step 5: Generate the query

Write `query.md` in the same tempdir, using a fixed template parameterized with the idea name and target audience.

```bash
python3 <<PY > "$TMPDIR/query.md"
import json
idea = json.loads('''$MATCHED_IDEA''')
name = idea["name"]
audience = idea["row"].get("Target Audience", "the target audience")
print(f'''Simulate how the target audience for "{name}" would react to this product
when it launches publicly. Specifically:

1. Would early adopters in "{audience}" sign up? Why or why not?
2. What are the top 3 objections you'd expect to hear?
3. What price range would they actually pay?
4. What's the most likely reason this product fails in its first 6 months?
5. What positioning angle would resonate most strongly with the target audience?

Be specific. Quote individual personas where useful.
''')
PY
```

### Step 6: Call `mirofish run`

Use the Bash tool with `run_in_background: true` because mirofish run takes 5–15 minutes. Construct a slug from the idea name.

```bash
# Slug: lowercase, alphanumeric + dashes, max 40 chars
SLUG=$(echo "$IDEA_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-\|-$//g' | cut -c1-40)

# Kick off the mirofish run
mirofish run "$SLUG" --docs "$TMPDIR" --query "$TMPDIR/query.md" --rounds 20 --yes
```

Important: use Claude Code's `Bash` tool with `run_in_background: true` so you can notify the user the run has started and check back later. The run takes 5-15 minutes.

While waiting, tell the user:
- The run has started
- The expected duration (5-15 min)
- The slug / run ID they can use to check status manually via `mirofish status <run-id>`

### Step 7: Read and present the report

When the background mirofish run completes (you'll get notified), read the report from stdout of that run. `mirofish run` prints a line like:

```
report: /Users/.../runs/2026-04-08-slug-xxxx/report.md
```

Extract the path and read the report:

```bash
REPORT_PATH="<path extracted from mirofish stdout>"
cat "$REPORT_PATH"
```

Present the report to the user inline. Also tell them:
- The run directory path (so they can re-read later)
- How to chat with the ReportAgent for follow-ups: `mirofish chat <run-id> "<question>"`
- How to interview individual simulated agents: `mirofish interview <run-id> <agent-name> "<question>"`

## Error handling

If any step fails, print the error and stop. Never silently retry or fall back. Specifically:

| Failure | Action |
|---|---|
| Prereq check fails (Step 1) | Print actionable diagnostic, stop |
| Workbook unreadable | Print error, stop |
| No matches found | Print closest matches, stop |
| `mirofish run` exits non-zero | Print stderr verbatim, stop |
| Cost gate refused (mirofish exit 4) | Print refusal, stop |
| Report file missing | Print run-dir path, tell user to investigate |

## Cost expectations

- Default 20 rounds × ~10 personas ≈ **$0.40 per validation** on qwen-plus
- Time: 5-15 minutes per validation
- mirofish-cli handles the cost gate if rounds > 80 or cost > $5

## What this skill does NOT do

- ❌ Does not write back to the workbook
- ❌ Does not validate multiple ideas in one call (one per invocation)
- ❌ Does not auto-start MiroFish backend — prereq check prints the start command
- ❌ Does not cache results — every invocation is a fresh MiroFish run
