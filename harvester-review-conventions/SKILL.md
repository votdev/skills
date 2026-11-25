---
name: harvester-review-conventions
description: Review Harvester pull requests and code changes. The analysis is delivered in the language the user writes in; review comments are always written in English as GitHub Markdown, short and precise.
metadata:
  author: Volker Theile
  email: vtheile@suse.com
---

# Harvester Review

Use this skill when reviewing Harvester code (PRs, diffs, branches).

## Language

- **Analysis and explanations to the user: in the user's native language**, i.e. the language they write to you in, unless they ask for another one. Do not fall back to English just because the code and PR are English.
- **Review comments: always English**, formatted as GitHub Markdown, regardless of the language of the analysis.

## Constraints

- Assume the code under review is available locally (checked-out branch or worktree). If that looks off (e.g. the branch does not match the PR, the diff is empty, the base is unclear), **ask the user**.
- Review comments are **short and precise**: one to three sentences, no filler.
- **Do not get lost in code suggestions.** Describe the issue and the direction in words. Add a snippet only when it is tiny and clearly clearer than prose.
- Do not post anything to GitHub (`gh pr review`, `gh pr comment`) unless the user explicitly asks.
- Do not modify code unless asked.

## Comment style

- **Phrase findings as questions, not instructions.** Do not tell the author to change code ("please keep...", "should be..."). State what you observed and ask about intent or consequences, e.g. "Is this change intentional, and if so, should it be mentioned in the description and covered by a test?"
- **Be factual and neutral.** No accusatory tone, no flattery, no overstated severity. If unsure, say so in the question instead of asserting a bug.
- Offer the comment text to the user for review first; never post it automatically.
