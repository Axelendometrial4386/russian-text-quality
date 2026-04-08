# Contributing

Thanks for wanting to improve `russian-text-quality`. This document defines the rules for adding new content. The rules exist because the skill's credibility comes from sourced linguistic claims and evidence-backed examples — and because every false positive we ship erodes trust with the zero-noise-on-English-projects goal.

**Rule zero: no invented content.** Every new dictionary entry, trigger word, or example must trace to a real source (linguistic reference, real code, published article). "I've seen this" is not a source.

---

## What you can contribute

1. New transliteration-dictionary entries (`references/transliteration-dict.yml`) — the seed list is deliberately small; there's room for domain-specific additions.
2. New concatenation trigger words (`references/concatenation-triggers.md`) — the closed trigger list for check #3.
3. New terminology synonym groups (`.claude/russian-glossary.example.yml` and the corresponding auto-drift list).
4. New real-world examples (`references/examples.md`) — always welcome, especially for categories where evidence is thin.
5. Detection-algorithm improvements to `references/plural-rules.md` or `references/mixing-and-transliteration.md`.
6. Bug fixes and documentation clarifications.
7. Language expansion to other Slavic languages (Ukrainian, Belarusian, Polish, Czech, Serbian) is **out of scope for v1**. Open a discussion issue first — we want to finish Russian before fragmenting effort.

## What you should NOT contribute

- Typography checks (« », em-dash, NBSP) — out of scope. Use [talkstream/ru-text](https://github.com/talkstream/ru-text).
- Spelling — use [yaspeller](https://github.com/hcodes/yaspeller).
- ё/е restoration — use [eyo](https://github.com/e2yo/eyo).
- MT/wooden UI copy detection — also covered by ru-text.
- Any check that requires a morphology engine (e.g. pymorphy3) at runtime. This is being tracked as a future upgrade path; the v1 skill is intentionally zero-runtime.

---

## Rules for each contribution type

### Adding a transliteration dictionary entry (`transliteration-dict.yml`)

Each new entry needs:

1. **A transliterated token that developers actually use.** Evidence of use: a GitHub search hit, a Habr discussion link, a Stack Overflow question, or a real-world codebase example (private code: screenshot the identifier and describe context, strip sensitive info).
2. **A suggested English equivalent** that is unambiguous. If the suggestion is ambiguous (e.g. `zakaz` could map to `order` or `contract`), pick the most common domain and note the ambiguity in a comment.
3. **No loanwords already accepted as English identifiers** — `admin`, `data`, `type`, `status`, `object`, `array` etc. must not be added (they are in the dict but flagged as "already English" and excluded from triggering).
4. **No pure Latin transliterations of single letters** (`ya`, `yu`, `yo` alone) — too short, would false-positive on English words.
5. A one-line PR description: what token, where it came from, why it's worth flagging.

### Adding a concatenation trigger word (`concatenation-triggers.md`)

Each new trigger needs:

1. **A link to a linguistic reference** (gramota.ru, Rosenthal handbook, orfogrammka.ru) confirming that the word governs a non-nominative case or otherwise creates a grammatical constraint on its object.
2. **A concrete bug example** — either a real line of code or a realistic constructed one, labeled as such.
3. **A false-positive sanity check:** grep the new trigger word across ≥5 publicly available Russian open-source projects (easy to find: search GitHub for `language:TypeScript` + the trigger word as a literal). If the grep produces more than 20% obvious non-bugs, either refine the trigger (require a following preposition/capitalization) or reject it.
4. Update the exclusion rules in `concatenation-triggers.md` if the new trigger needs a specific exclusion.
5. A one-line PR description summarizing the change.

### Adding an example to `references/examples.md`

Each new example needs:

1. **A public source link.** GitHub permalink, Habr article, published bug report, company engineering blog, open-source issue tracker. No screenshots without provenance.
2. **A clear mapping to one of the 5 checks.** If the example straddles multiple checks, pick the primary one and mention the others in a sentence.
3. **Before / Why wrong / Fix** structure. Match the format of existing examples.
4. If the example is **paraphrased** from a source because the source describes the bug in prose rather than code, mark it `[paraphrased from source]` explicitly.
5. Update the count-per-category table at the bottom of `examples.md`.

### Adding a synonym group or glossary entry

1. **Evidence of drift in the wild** — a Habr article, an open-source issue where maintainers argued about two Russian translations, a company style guide that bans one form over another. Link it.
2. The canonical form must be defensible: either the form used by a large Russian company's public style guide (Yandex, Kaspersky, VK) or one you can defend on a linguistic/usability basis in the PR description.
3. Do not add religious/political preferences as "correct" forms. Stick to demonstrably ambiguous UI terminology.

### Improving a detection algorithm

1. **Run the skill against a fixture** with planted bugs (contribute a fixture under `tests/fixtures/` if none exists yet — v0.1 did verification via ad-hoc fixtures not committed to the repo) and confirm your change does not regress the known-good detection cases.
2. **Add new test cases** to the `Test cases` table in the relevant reference file for any new behavior.
3. **Respect severity caps** declared in `plural-rules.md` and `mixing-and-transliteration.md`. Do not promote a heuristic check to Error.
4. If your change adds a new code pattern to scan for, document it with a regex in pseudo-code (we keep detection specs as pseudo-code so they're language-implementation-agnostic).

---

## Before you open a PR

- [ ] All new claims about Russian grammar/punctuation have authoritative source links (gramota.ru, orfogrammka.ru, Rosenthal, CLDR, etc.).
- [ ] All new examples have real source links.
- [ ] All new triggers/dict entries have passed a false-positive sanity check.
- [ ] You have read the affected reference file in full and your change integrates cleanly.
- [ ] You have NOT modified `SKILL.md` unless your change legitimately affects the skill's primary contract (activation, checks, severity, output format). Most contributions touch reference files only.
- [ ] You have not added typography, spelling, or ё/е checks (these are explicit non-goals — see `SKILL.md` and `README.md`).

## PR description template

```
## Summary
One or two sentences.

## Type
- [ ] Dictionary entry
- [ ] Concatenation trigger
- [ ] Example
- [ ] Glossary / synonym group
- [ ] Detection algorithm improvement
- [ ] Bug fix
- [ ] Documentation

## Source
<link>

## False-positive check (if applicable)
Describe what you grep'd and what you found.

## Test cases added
<file and cases, or "none needed">
```

---

## Non-Russian-speaking contributors

You are welcome. If you can read code and follow the linked primary sources, you can contribute structural improvements (detection algorithms, test cases, bug fixes, file organization) without needing to judge Russian grammar yourself.

For contributions that DO require Russian linguistic judgment (dictionary entries, triggers, glossary terms), either (a) ask a Russian-speaking reviewer in the PR, or (b) limit your contribution to structure/code and cite a source a Russian speaker can verify.

---

## Code of conduct

Be respectful. This repo exists to make Russian-language software a little less broken. Disagreements about linguistic judgment are expected — resolve them with sources, not volume.

## License

By contributing, you agree that your contributions will be licensed under the MIT License (see `LICENSE`).
