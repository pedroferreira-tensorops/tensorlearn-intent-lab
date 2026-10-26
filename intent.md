# Intent

## Goal
A support coordinator needs to preview the tickets contained in a CSV file before
anyone approves a later import. Add a pure function that takes CSV text and returns
either the full list of tickets it would produce, in input order, or a single error
code explaining why the file is rejected. The preview never changes data: it is a
read-only check that lets a human see the outcome before deciding to import.

## Scope
In scope: a new module `src/import-tickets.mjs` exporting the preview function, and
a new test file `tests/import-tickets.test.mjs` covering every acceptance case below.
These two files are the only ones you may create or change.

Out of scope: the actual import, any UI, database, network access, file reading or
writing, CLI wrappers and new package dependencies (CSV parsing is written by hand
with Node built-ins). Do not edit `src/tickets.mjs`, its tests, `.github/`,
`CLAUDE.md`, `package.json` or this document. If the work seems to need anything
outside these two files, stop and ask instead of expanding scope.

## Constraints
- Export exactly one function, `previewTickets(csvText)`, from `src/import-tickets.mjs`.
  It returns a plain object (not a JSON string) and never throws for bad CSV content.
- Reuse the stable ticket representation in `src/tickets.mjs` (`ticket(title, priority)`)
  without modifying it. Note that `ticket()` only defaults `undefined` and does not
  trim priorities, so trim the priority first and turn a blank one into `normal`
  before calling it; otherwise a blank priority would wrongly fail.
- Header: a `title` column is required and a `priority` column is optional. Any other
  column name, a missing `title`, a duplicated name, or a completely empty input
  produces `unknown-column`. A malformed row also produces `unknown-column`: a field
  count that differs from the header, or a quote that is never closed.
- A single line break at the end of the input ends the last row; it does not start
  an extra empty row. Every acceptance example ends with one.
- Values are trimmed. An empty title gives `empty-title`. Priority must be `low`,
  `normal` or `urgent`; a missing column or blank value becomes `normal`; anything
  else gives `invalid-priority`.
- At most 100 data rows. Count rows right after the header check and before
  validating any value, so 101 rows give `row-limit` even if some rows are invalid.
- Honour standard CSV quoting: quoted fields may contain commas, and `""` inside a
  quoted field is a literal quote.
- Output is exactly `{"tickets":[...]}` on success or `{"error":"<code>"}` on failure.
  Never return partial results.
- Assumptions not stated in the brief, to confirm with me before relying on them:
  when several rows fail, the first failing row in input order decides the code;
  within a row, the title is checked before the priority.
- No side effects: no I/O, no globals, no mutation of the input.

## Plan
1. Before writing code, restate the scope and these rules back to me and ask for
   approval. Ask rather than guess about the open questions the brief does not
   answer: header case or surrounding whitespace (` title `), capitalised
   priorities (`Urgent`), CRLF line endings, blank lines, whitespace-only input,
   and a header with zero data rows.
2. Write `tests/import-tickets.test.mjs` from the acceptance cases below, first.
3. Implement a small CSV tokenizer handling quotes, then the header check, then the
   row-limit check, then per-row validation via `ticket()`, mapping its thrown
   messages to error codes.
4. Run the verification commands, fix failures, and show me the diff before any
   commit. Do not commit, push or open a PR unless I ask.

## Acceptance
Positive examples: an explicit valid priority is kept; a file with only a `title`
column defaults every ticket to `normal`; a blank priority also becomes `normal`;
a quoted title keeps its comma. Negative examples: a blank title, an unknown column,
an unsupported priority such as `high`, and 101 data rows are each rejected with one
error code and no tickets. The contract below is the objective source of truth; each
case must also appear as a test.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nReset my password\n",
      "expected": {"tickets":[{"title":"Reset my password","priority":"normal"}]}
    },
    {
      "csv": "title,priority\nVPN drops,\n",
      "expected": {"tickets":[{"title":"VPN drops","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Question about billing, October\",low\n",
      "expected": {"tickets":[{"title":"Question about billing, October","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,assignee\nPrinter issue,Sam\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,high\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title\nPrinter issue\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Verification
Run `npm test` (existing and new tests must pass) and `git diff --check` (no
whitespace errors), and confirm `git status` shows changes only in the two allowed
files. Report the command output to me. Ask for my approval before implementing,
before touching any other file, and before committing or pushing.
