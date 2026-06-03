---
description: Review a lesson PR — run integrity, accuracy, quality and quiz agents on each changed lesson, then post a consolidated feedback comment (with a copy-paste fix prompt) to the PR.
argument-hint: <PR number>   (omit to list open lesson PRs)
allowed-tools: [Bash, Read, Grep, Glob, Agent]
---

# Review Lesson PR

Run the Jupiter Academy lesson-review agents against the lessons changed in a pull request and post **one consolidated, self-updating comment per lesson** to the PR. The agents are read-only — this command never edits lesson files and never pushes to the PR. Its only write action is creating/updating a PR comment.

This repo (`jup-ag/academy-content`) holds the lesson source. Lessons are `lessons/<slug>.mdx`; the glossary is `glossary/terms.json`. Quizzes are NOT stored here — the quiz agent proposes one in the comment for the app maintainers.

PR number: **$ARGUMENTS**

> Compatibility note: `gh` may be old (2.4.x). Avoid newer flags — `gh pr diff --name-only` does NOT exist there; use the `pulls/<n>/files` API instead. Always resolve `REPO` dynamically.

## 0. Resolve the PR
- Set `REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)` first (should be `jup-ag/academy-content`).
- If `$ARGUMENTS` is empty, list open PRs and surface only those touching lessons, then stop and ask the user which one:
  ```bash
  for n in $(gh pr list --state open --json number -q '.[].number'); do
    if gh api "repos/$REPO/pulls/$n/files" --paginate -q '.[].filename' | grep -q '^lessons/.*\.mdx$'; then
      gh pr view "$n" --json number,title,headRefName -q '"#\(.number) \(.title) [\(.headRefName)]"'
    fi
  done
  ```
- Otherwise set `PR=$ARGUMENTS`. If `gh pr view "$PR"` fails (no such PR), STOP and tell the user — do not invent a review.

## 1. Find the changed lessons
```bash
gh api "repos/$REPO/pulls/$PR/files" --paginate -q '.[].filename'
```
Keep only paths matching `^lessons/.*\.mdx$`. These are the lessons to review (added OR modified). If none, post nothing — report "No lesson MDX files changed in PR #$PR" to the user and stop. Note each lesson's slug (filename without `.mdx`).

## 2. Get the PR's version of the files onto disk
The agents read files from disk, so the working tree must hold the **PR head** version.
- Ensure the working tree is clean: `git status --porcelain`. If dirty, STOP and tell the user to stash/commit first (do not touch their changes).
- Record the current branch: `ORIG=$(git rev-parse --abbrev-ref HEAD)`.
- Check out the PR: `gh pr checkout "$PR"`.
- **Restore `git checkout "$ORIG"` in step 5**, even if something fails.

## 3. Run the four agents (in parallel) per lesson
For EACH changed lesson, launch all four subagents concurrently (one `Agent` call each, in a single message so they run in parallel). Pass each agent the lesson path and slug.

| subagent_type | Prompt to give it |
|---|---|
| `lesson-integrity` | "Review the lesson at `lessons/<slug>.mdx` (slug `<slug>`). Run all integrity checks and return your findings block." |
| `lesson-accuracy`  | "Fact-check the lesson at `lessons/<slug>.mdx` (slug `<slug>`) against docs.jup.ag (start from https://docs.jup.ag/llms.txt). Return your findings block." |
| `lesson-quality`   | "Do an editorial + coherence review of the lesson at `lessons/<slug>.mdx` (slug `<slug>`). Return your findings block." |
| `lesson-quiz`      | "Propose a quiz for the lesson at `lessons/<slug>.mdx` (slug `<slug>`). Return your findings block." |

Each agent returns a ready-to-embed markdown block. Collect all four verbatim.

If a PR changes many lessons, process them a few at a time (keep the 4 agents-per-lesson parallel; don't launch 40 agents at once).

## 4. Assemble & post the consolidated comment (one per lesson)
Build a single markdown comment per lesson with this exact skeleton. The first line is a hidden marker used to find-and-update the comment on re-runs — **do not change its format**:

```markdown
<!-- lesson-review:<slug> -->
## 📚 Lesson review — `<slug>.mdx`

> Automated review by the Academy lesson agents. Read-only — nothing was changed. Reviewed commit `<short sha>`.

**Summary:** <e.g. 🔴 2 blockers · 🟡 6 suggestions across 4 checks, or ✅ all checks pass>

<integrity block>

<accuracy block>

<quality block>

<quiz block>

---
### 🤖 Ready-to-use prompt for Claude Code
Copy this into Claude Code on the PR branch to apply the fixes:

\`\`\`
Apply the Jupiter Academy lesson review for lessons/<slug>.mdx.

Blockers (must fix):
<numbered list — every 🔴 fix instruction from the integrity/accuracy/quality agents>

Suggestions (apply if you agree):
<numbered list — every 🟡 fix instruction from the integrity/accuracy/quality agents>

Then commit and push to this PR branch. Do not invent facts — cite docs.jup.ag where the review did. Keep the Academy tone (educational, no financial advice).

(Quiz proposal in the comment above is for the app repo — not this content repo.)
\`\`\`

<sub>Re-run with <code>/review-lesson-pr <PR></code> after pushing — this comment updates in place.</sub>
```

Compose the two numbered lists by pulling the `Fix instructions` lines out of the integrity / accuracy / quality blocks (blockers = lines tied to 🔴 findings, suggestions = the rest). The quiz block is always a proposal and is shown above the prompt, not folded into the blocker/suggestion lists. If an agent returned PASS / `- none`, contribute nothing from it. If integrity+accuracy+quality all pass, set Summary to ✅ and replace the blocker/suggestion lists with a single line: `No fixes needed — all checks passed. (Quiz proposal above is optional, for the app repo.)`

### Post or update (self-updating, one sticky comment)
Write the assembled markdown to a temp file, wrap it as a JSON payload (so no `gh -F` magic substitution can corrupt the body), then find an existing comment carrying `<!-- lesson-review:<slug> -->` and PATCH it, else POST a new one:
```bash
BODY=$(mktemp); PAYLOAD=$(mktemp)
# ...write the assembled markdown into "$BODY" (e.g. via the Write tool or a heredoc)...
node -e 'const fs=require("fs");process.stdout.write(JSON.stringify({body:fs.readFileSync(process.argv[1],"utf8")}))' "$BODY" > "$PAYLOAD"
EXISTING=$(gh api "repos/$REPO/issues/$PR/comments" --paginate \
  -q ".[] | select(.body | contains(\"<!-- lesson-review:<slug> -->\")) | .id" | head -n1)
if [ -n "$EXISTING" ]; then
  gh api -X PATCH "repos/$REPO/issues/comments/$EXISTING" --input "$PAYLOAD"
else
  gh api -X POST "repos/$REPO/issues/$PR/comments" --input "$PAYLOAD"
fi
```
(`--input` sends a raw JSON request body, avoiding the `{repo}`/`true`/`null` magic conversion that `-F` applies. Use the real slug in the marker.)

## 5. Cleanup & report
- Restore the original branch: `git checkout "$ORIG"`.
- Print to the user: which lessons were reviewed, the per-lesson summary line, and the PR URL (`gh pr view "$PR" --json url -q .url`; comments are on the conversation tab).

## Rules
- Never edit lesson files and never push commits — the only write is the PR comment.
- Never fabricate findings; relay exactly what the agents returned. If an agent errored, say so in that section rather than dropping it silently.
- Always restore the user's original branch, even on failure.
- Keep one sticky comment per lesson; never spam new comments on re-runs (the marker-based update prevents this).
- No emojis beyond the severity markers and section icons defined above.
