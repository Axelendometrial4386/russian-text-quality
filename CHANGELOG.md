# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] — 2026-04-07

Initial public release.

### Added

- **Skill manifest** (`SKILL.md`) — activation logic, 5-check catalog, output format, workflow, explicit non-goals.
- **Check #1 — Plural structure.** Detection for i18next (v4 suffix convention), ICU MessageFormat (FormatJS / react-intl), and vue-i18n (pipe-separated). Includes Step 2.5: flags locale values containing `{{count}}`/`{count}`/`%{count}`/`{n}` placeholders when the key has no plural siblings, with an ICU-guard to avoid false positives inside `{count, plural, ...}` blocks. See `references/plural-rules.md`.
- **Check #2 — Terminology drift.** Auto-detection via normalized value comparison across locale files; optional user glossary at `.claude/russian-glossary.yml`. Simple suffix-stripping normalization (no morphology engine in v1). Template provided at `.claude/russian-glossary.example.yml`.
- **Check #3 — Concatenation case bug.** Heuristic regex detection over a closed trigger list of ~30 Russian prepositions, transitive verbs, and gender-coupling adjectives. Always `Warning` severity, never `Error`. See `references/concatenation-triggers.md`.
- **Check #4 — Language mixing.** Per-file comment-language ratio check (25% minority + minimum-2 threshold) plus inline-Russian-literal detection outside the i18n layer (≥3 literals threshold). See `references/mixing-and-transliteration.md`.
- **Check #5 — Transliterated identifiers.** Three-tier detection: dictionary hit + (phonotactic signal OR file-level Russian content). Always `Info` severity. Seed dictionary of ~130 entries at `references/transliteration-dict.yml`.
- **Activation guard.** Dual-condition trigger: ≥10 total Cyrillic characters + ≥1 five-character Cyrillic cluster across code and locale files. Zero noise on English-only projects. See `references/activation-logic.md`.
- **Examples** (`references/examples.md`) — 16 before/after cases sourced from real projects (Badoo, Kaspersky, VK, ua-hosting, VC.ru, GitHub). No invented examples. Weak categories flagged honestly.
- **README.md** — English-facing landing with install instructions, check overview, limitations, and acknowledgments.
- **CONTRIBUTING.md** — contribution rules with mandatory source-link and false-positive-check requirements.
- **LICENSE** — MIT.
- **Habr post draft** at `docs/habr-post-draft.md`, self-audited against the skill's own rules.

### Explicit non-goals in v0.1.0

These areas are covered by other tools and are not in scope for this skill:

- Russian typography (« », em-dash, non-breaking spaces) → delegated to [talkstream/ru-text](https://github.com/talkstream/ru-text).
- Russian spelling → delegated to [yaspeller](https://github.com/hcodes/yaspeller).
- ё/е restoration → delegated to [eyo](https://github.com/e2yo/eyo).
- UX writing quality, machine-translation artifacts → partially covered by ru-text; not duplicated here.
- Morphology-based analysis (pymorphy3 integration) → deferred to a future version.
- i18next v3 (`key_plural` format) → only v4 suffix conventions are supported.
- Languages other than Russian → out of scope.
- Automated test suite → deferred to v0.2 (v0.1 was verified via manual subagent dispatch against a buggy fixture).

### Known limitations

- **vue-i18n pipe detection** is heuristic. Prose with literal `|` can cause false positives. Findings are emitted as `Warning`, never `Error`.
- **Concatenation check #3** is a regex heuristic over a closed trigger list. Expected false-positive rate: ~10–15%. All findings are `Warning`, never `Error`.
- **Transliteration check #5** has thin public evidence — the dictionary is seeded from common Russian dev vocabulary, because live GitHub permalinks for transliterated identifiers are rare (the problem lives primarily in private corporate codebases). All findings are `Info`, never `Warning`.
- **Terminology normalization** uses simple suffix-stripping. Over-stemming is possible; recommend providing a glossary via `.claude/russian-glossary.yml` for precise control.
- **Nested ICU plurals** (plural inside select) may be missed by the v1 regex-based parser.
- **`.claude-plugin/plugin.json`** is intentionally NOT included in v0.1.0 — the plugin manifest schema should be verified against the current Claude Code plugin spec before submission to the Anthropic plugin marketplace.

[Unreleased]: https://github.com/Anic888/russian-text-quality/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/Anic888/russian-text-quality/releases/tag/v0.1.0
