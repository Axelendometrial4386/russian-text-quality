# russian-text-quality

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Release](https://img.shields.io/github/v/release/Anic888/russian-text-quality?include_prereleases&sort=semver)](https://github.com/Anic888/russian-text-quality/releases)
[![Stars](https://img.shields.io/github/stars/Anic888/russian-text-quality?style=flat)](https://github.com/Anic888/russian-text-quality/stargazers)
[![Issues](https://img.shields.io/github/issues/Anic888/russian-text-quality)](https://github.com/Anic888/russian-text-quality/issues)
[![Discussions](https://img.shields.io/github/discussions/Anic888/russian-text-quality)](https://github.com/Anic888/russian-text-quality/discussions)
[![Claude Code skill](https://img.shields.io/badge/Claude_Code-skill-8A2BE2)](https://docs.claude.com/en/docs/claude-code)
[![i18n](https://img.shields.io/badge/i18n-i18next_|_ICU_|_vue--i18n-blue)](#what-it-checks)
[![CLDR](https://img.shields.io/badge/CLDR-plural_rules-green)](https://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html)

A Claude Code skill that catches Russian-language bugs in code that general linters and LLMs consistently miss: broken pluralization in i18n locale files, case-agreement bugs in string concatenation, terminology drift, transliterated identifiers, and inconsistent language mixing.

**Code-aware, not prose-focused.** If you want a Russian prose/typography reviewer, install [talkstream/ru-text](https://github.com/talkstream/ru-text) alongside this skill — they complement each other.

---

## Who this is for

- Russian-speaking developers whose projects mix English code with Russian UI, docs, or comments.
- Teams reviewing PRs that touch Russian locale files and want structural checks before merge.
- Non-Russian-speaking developers using Claude Code on a codebase with Russian content — this skill explains every finding in English so you understand *why* the Russian is wrong.

---

## What it checks

| # | Check | Example of what it catches | Severity |
|---|---|---|---|
| 1 | **Plural structure** | `ru.json` has `items_one` and `items_other` but is missing `items_few`/`items_many`; ICU messages with only `other {...}` branch; vue-i18n values with fewer than 4 pipe segments | Error |
| 2 | **Terminology drift** | The same UI concept translates to "пользователь" in one file and "юзер" in another | Warning |
| 3 | **Concatenation case bug** | `` `Удалить ${item.name}?` `` — `Удалить` requires the accusative but `item.name` is nominative | Warning |
| 4 | **Language mixing** | One file with half Russian comments, half English; inline Russian string literals outside the i18n system | Warning |
| 5 | **Transliterated identifiers** | Variable names like `polzovatel`, `kolichestvo_tovarov`, `nazvanie_zakaza` | Info (low confidence) |

Full detection algorithms and test cases live in `references/plural-rules.md` and the other reference files.

## What it does NOT check

| Out of scope | Use instead |
|---|---|
| Typography (« », em-dash, NBSP) | [talkstream/ru-text](https://github.com/talkstream/ru-text) |
| Spelling | [yaspeller](https://github.com/hcodes/yaspeller) |
| ё/е restoration | [eyo](https://github.com/e2yo/eyo) |
| UX writing quality, wooden MT copy | [talkstream/ru-text](https://github.com/talkstream/ru-text) |
| Translation quality from a source language | N/A — not a translation memory |

Why the delegation: these areas are already well-covered by purpose-built tools. Duplicating them would fragment the ecosystem. This skill specializes in the code-aware checks that no existing tool handles.

---

## Install

### As a personal Claude Code skill

```bash
git clone https://github.com/Anic888/russian-text-quality.git ~/russian-text-quality
ln -s ~/russian-text-quality ~/.claude/skills/russian-text-quality
```

Or copy the directory:
```bash
cp -r russian-text-quality ~/.claude/skills/
```

Claude Code picks up skills in `~/.claude/skills/` automatically.

### Verifying the install

Open a project with Russian content and ask Claude Code:
> "Review this project for Russian text quality issues."

The skill should activate and produce a Markdown report. If the project has no Russian content, it will respond "Not activated — no Russian content detected" and stop.

### Optional: project-level glossary

Copy `.claude/russian-glossary.example.yml` to your repo at `.claude/russian-glossary.yml` and customize. The glossary overrides the default auto-drift detection for check #2.

---

## Activation — when this skill runs

The skill runs **only** when a project contains Russian content. Specifically, it requires:

1. At least **10 total Cyrillic characters** across code and locale files (node_modules, .git, dist, build, and .gitignore'd paths excluded).
2. At least one file containing **5 consecutive Cyrillic characters** (rules out single stray tokens).

Otherwise it stays silent. **Zero noise on English-only projects** is a hard design goal.

You can also invoke it explicitly by asking Claude Code to "use the russian-text-quality skill" — useful before adding Russian content to a previously English-only project.

---

## Example output

```
# Russian Text Review — checkout-service

## Summary
- Files scanned: 42
- Russian content in: 7 files
- Findings: 3 errors, 2 warnings, 1 info

## Findings

### Errors (must fix)
#### #1 Plural structure
- [HIGH] src/locales/ru.json:45 — key `cart.items` missing `_few`, `_many`
  - Why: Russian CLDR requires one/few/many/other. Missing `_few` breaks n=2, 3, 4.
  - Fix: scaffolded below. Review and apply.

- [HIGH] src/messages/ru.json:12 — ICU plural block missing `one`, `few`, `many`
  - Bad: `{count, plural, other {# товаров}}`
  - Good: `{count, plural, one {# товар} few {# товара} many {# товаров} other {# товара}}`

### Warnings
#### #2 Terminology drift
- [MEDIUM] `user` is translated as "пользователь" in ru.json and "юзер" in admin/ru.json.
  - Fix: pick one, lock via .claude/russian-glossary.yml.

### Info
#### #5 Transliterated identifiers
- [LOW] src/order.ts:12 — `kolichestvo_tovarov` looks like transliterated Russian.
  - Suggestion: `product_quantity` or keep Cyrillic if your project convention uses Cyrillic.

## External tools we recommend
- Typography / UX writing: talkstream/ru-text
- Spelling: yaspeller
- ё/е: eyo
```

---

## Limitations and honest caveats

- **i18next v3 unsupported.** The legacy `key_plural` suffix format is out of scope for v1. Only the v4 `_one/_few/_many/_other` format is recognized.
- **vue-i18n pipe detection is heuristic.** Pipes in prose can false-positive. Findings for vue-i18n are always `Warning` severity, never `Error`, to reflect this.
- **Concatenation check #3 is a regex heuristic.** Expect roughly 10–15% false positives. Mitigation: always `Warning` severity, never auto-fixed.
- **Terminology normalization uses suffix stripping, not full morphology.** A real [pymorphy3](https://pypi.org/project/pymorphy3/) integration is on the roadmap for a future version.
- **Transliteration check #5 has thin public evidence.** The dictionary is seeded from common Russian dev vocabulary because live GitHub permalinks for transliterated identifiers are rare — most of them live in private corporate codebases. Always `Info` severity.
- **Nested ICU plurals (plural inside select) may be missed** by the v1 regex-based parser.
- **Russian comments in obscure languages' file formats** are not scanned. Supported languages: TypeScript, JavaScript, TSX, JSX, Vue, Svelte, Python, Ruby, PHP, Go, Rust, Kotlin, Swift, Java, C#.

---

## Design principles

1. **Zero false positives on English-only projects.** Activation is gated on Cyrillic presence; non-Russian codebases never see findings from this skill.
2. **Every finding explains the why.** Non-Russian-speaking developers must understand *why* the Russian is wrong, not just that it is.
3. **Errors only when deterministic.** The skill reserves `Error` severity for structural bugs (check #1). All heuristics are `Warning` or `Info`.
4. **No auto-fix without approval.** Only check #1 offers auto-fixes, and only after the user explicitly approves the suggested diff.
5. **Delegation over duplication.** Areas already served by good tools are explicit non-goals — see the table above.

---

## Contributing

New trigger words for check #3, new dictionary entries for check #5, and new examples in `references/examples.md` are all welcome. Please see `CONTRIBUTING.md` for the rules. Short version:

- New triggers and dict entries need a linguistic source link (gramota.ru, Rosenthal reference, or similar).
- New examples need a source link — no invented examples.
- A PR that risks introducing false positives must include a before/after test run on at least 5 open-source Russian projects.

---

## Security

See `SECURITY.md` for how to report prompt-injection payloads, malicious dictionary entries, or impersonation attempts on companion tools.

## License

MIT. See `LICENSE`.

---

## Acknowledgments

- [talkstream/ru-text](https://github.com/talkstream/ru-text) — the existing Claude Code skill for Russian prose and typography. Install it alongside this one.
- [rails-i18n](https://github.com/svenfuchs/rails-i18n) — reference implementation for the CLDR-compliant Russian locale file shape.
- [gramota.ru](https://gramota.ru) and [orfogrammka.ru](https://orfogrammka.ru) — authoritative sources for Russian punctuation and morphology cited throughout the detection rules.
- [Unicode CLDR](https://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html) — source of truth for Russian plural categories.
- The Badoo, Kaspersky, VK, Yandex, and ua-hosting engineering blogs on Habr, whose localization war stories seeded the real-world example set in `references/examples.md`.
