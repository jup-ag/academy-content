# Lesson-review agents

A set of read-only Claude Code agents that review **lesson PRs** in this content repo and post feedback (plus a copy-paste fix prompt) as a comment on the PR. They never edit lesson files and never push — the author stays in control.

## Run it

```
/review-lesson-pr <PR number>
```

(Omit the number to list open PRs that touch `lessons/`.)

The orchestrator command (`.claude/commands/review-lesson-pr.md`):
1. finds the lesson `.mdx` files changed in the PR,
2. checks out the PR branch locally (clean tree required; restores your branch after),
3. runs the four agents **in parallel** per lesson,
4. posts **one self-updating comment per lesson** — re-running edits the same comment in place (matched by a hidden `<!-- lesson-review:<slug> -->` marker).

## The agents

| Agent | Responsibility | Tools |
|---|---|---|
| `lesson-integrity` | Illustrations resolve (CDN 200 + alt text), `videoId` embed works, links not broken (internal `/lessons/<slug>` exist, external links 200), frontmatter valid, every `<Term id>` exists in `glossary/terms.json` | Bash, Read, Grep |
| `lesson-accuracy` | Cross-checks product claims/numbers/flows against `docs.jup.ag` (starts from `docs.jup.ag/llms.txt`) and `jup.ag`, with citations | WebFetch, WebSearch, Read, Bash |
| `lesson-quality` | Grammar/spelling/tone **and** structure/flow/pedagogy/convention-fit, calibrated to the lesson's difficulty badge | Read, Grep, Bash |
| `lesson-quiz` | Proposes a quiz (4-option `.quiz.json`) in the comment — quizzes are NOT stored in this repo, the proposal is for the app maintainers | Read, Bash |

The original asks → four agents: image + video + link + frontmatter + glossary checks share one deterministic pass (`integrity`); grammar + coherence share one read pass (`quality`).

## Repo layout this assumes

| Thing | Path |
|---|---|
| Lessons | `lessons/<slug>.mdx` |
| Glossary | `glossary/terms.json` |
| Quizzes | not in this repo — they live in the app repo (`academy.jup.ag` → `src/data/quizes/<slug>.quiz.json`) |

## Output contract

Each agent returns a single markdown block with a severity table (🔴 blocker · 🟡 suggestion · 🟢/PASS) and a `Fix instructions` list. The orchestrator merges those into the per-lesson comment and into the **Ready-to-use prompt for Claude Code** that the author copies to apply changes and update the PR.

## Requirements

- `gh` CLI authenticated with `jup-ag/academy-content` (used for diff + comments).
- Network access to `static.academy.jup.ag`, `docs.jup.ag`, and YouTube oEmbed (video checks).
- `node` available (for the deterministic glossary/quiz validators).

## Extending

Add a new check by dropping another `lesson-*.md` agent here (same output contract) and adding a row to the agent table in `.claude/commands/review-lesson-pr.md`.
