# Contributing to Jupiter Academy Content

Thanks for helping improve Jupiter Academy! Here's how to contribute.

## Lessons

Lessons are MDX files in the `lessons/` directory.

### Frontmatter format

Every lesson must start with YAML frontmatter:

```yaml
---
title: "Your Lesson Title"
publishedAt: '2026-01-15'
image: ''
badges:
  - Beginner
  - 10min
---
```

**Required fields:**
- `title` - Clear, concise lesson title
- `publishedAt` - Date in `YYYY-MM-DD` format
- `badges` - Array of tags:
  - Difficulty: `Beginner`, `Intermediate`, `Advanced`
  - Reading time: `5min`, `10min`, `15min`, `20min`, `30min`
  - Topic (if relevant): `Wallet`, `Portfolio`, `Borrow`, `Multiply`

### Content guidelines

- Use `##` for main sections, `###` for subsections
- Keep explanations clear and beginner-friendly
- Use bullet lists and bold for key terms
- No quiz sections or test questions
- Links are welcome where helpful

### File naming

Use lowercase, hyphenated slugs: `what-is-crypto.mdx`, `create-your-first-wallet.mdx`

## Glossary

The glossary lives in `glossary/terms.json` as a JSON array:

```json
[
  {
    "term": "DeFi",
    "definition": "Decentralized Finance. Financial services built on blockchain technology that operate without traditional intermediaries like banks."
  }
]
```

- Keep definitions concise (1-2 sentences)
- Terms are sorted alphabetically — add new entries in the right position
- Mention Jupiter/Solana context where relevant

## Submitting a PR

1. Fork this repo
2. Create a branch: `git checkout -b add-lesson-name` or `fix-typo-lesson`
3. Make your changes
4. Open a Pull Request with a clear description

We'll review and merge promptly. Thank you!
