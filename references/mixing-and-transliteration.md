# Mixing and Transliteration Detection Reference

Concrete detection algorithms and thresholds for **check #4 — Language mixing** and **check #5 — Transliterated identifiers**. Both are heuristic; check #4 emits `Warning`, check #5 emits `Info`.

---

## Check #4 — Language mixing in one file

Two sub-checks run per file.

### Sub-check 4A — Inconsistent comment language

```
INPUT: file (ts/tsx/js/jsx/vue/svelte/py/rb/php/go/rs/kt/swift/java/cs)

comments = extract_comments(file)        # per-language comment regex
# For .tsx/.jsx files, extract_comments MUST also walk JSX comment syntax {/* ... */}
# in addition to the normal // and /* */ forms. Both JSX and non-JSX comments count together.
# Filter: skip comments shorter than 10 characters (TODO, FIXME, etc.) — too short to classify
substantive = [c for c in comments if len(c.strip()) >= 10]
if len(substantive) < 4:
    return  # not enough data to judge

ru_count = 0
en_count = 0
for c in substantive:
    ru_chars = count_matches(c, /\p{Script=Cyrillic}/u)
    en_chars = count_matches(c, /[A-Za-z]/)
    # classification: dominant script for this comment
    if ru_chars >= 3 and ru_chars > en_chars:
        ru_count += 1
    elif en_chars >= 3 and en_chars > ru_chars:
        en_count += 1
    # else: uncategorized (numbers, symbols)

total = ru_count + en_count
if total < 4:
    return

minority = min(ru_count, en_count)
minority_ratio = minority / total

# Threshold: minority language must be at least 25% of classified comments to count as "mixing"
# AND absolute count must be at least 2 (one comment in another language is not "mixing")
if minority_ratio >= 0.25 and minority >= 2:
    emit(Warning,
         file=file,
         category="#4-comment-language-mixing",
         message=f"File has {ru_count} Russian and {en_count} English comments. "
                 f"Pick one language per file for consistency.")
```

#### Test cases

| File content | Expected |
|---|---|
| 10 comments, all Russian | No finding |
| 10 comments, all English | No finding |
| 10 comments, 3 Russian + 7 English | **Warning** (3/10 = 30%, ≥25%, minority≥2) |
| 10 comments, 1 Russian + 9 English | No finding (minority=1 < 2) |
| 4 Russian + 0 English | No finding |
| 2 Russian + 2 English (4 total) | **Warning** (exactly at threshold: 2/4 = 50%) |
| 3 short Russian + 3 short English, all < 10 chars | No finding (all filtered out) |

### Sub-check 4B — Inline Russian string literals outside i18n

```
INPUT: file (non-locale code file)

# Only run on files that are NOT in a locale path (avoid flagging the locale file itself)
if file.path matches PRIMARY_LOCALE_PATTERNS:
    return

string_literals = extract_string_literals(file)
russian_literals = []
for lit in string_literals:
    ru_chars = count_matches(lit.value, /\p{Script=Cyrillic}/u)
    if ru_chars >= 5:                      # at least 5 Cyrillic chars
        # skip literals that are inside an i18n call: t("..."), $t, <FormattedMessage id="...">
        if inside_i18n_call(lit):
            continue
        russian_literals.append(lit)

# Threshold: flag only if there are 3+ such literals in a single file
# (a single Russian comment-string for debugging is not worth a finding)
if len(russian_literals) >= 3:
    emit(Warning,
         file=file,
         category="#4-inline-russian-literals",
         message=f"File has {len(russian_literals)} hardcoded Russian string literals outside "
                 f"the i18n system. Extract to the locale file.",
         sample=russian_literals[:3])
```

#### Test cases

| File content | Expected |
|---|---|
| 5 hardcoded Russian strings in a .tsx file, none inside `t()` | **Warning** |
| 2 hardcoded Russian strings in a .tsx file | No finding (< 3 threshold) |
| 5 hardcoded Russian strings, all inside `t("...")` calls | No finding (all filtered) |
| 5 hardcoded Russian strings in `src/locales/ru.json` | No finding (locale file — skipped) |
| 1 hardcoded Russian string + 5 Russian comments | No finding on 4B (1 < 3) — 4A may fire |

---

## Check #5 — Transliterated identifiers (3-tier algorithm)

```
INPUT: file (code file), transliteration_dict (references/transliteration-dict.yml),
       file_has_cyrillic: bool (true iff the same file or the project has Cyrillic content
                                per the activation scan)

# Compile phonotactic patterns once
PHONOTACTIC_PATTERNS = [
    r'shch',             # щ
    r'[bdgklmnprstvz]zh', # ж after a consonant (unusual in English)
    r'^ya[a-z]',         # я word-initial
    r'^yu[a-z]',         # ю
    r'^yo[a-z]',         # ё
    r'kh[aeiou]',        # х + vowel
    r'ts[a-z]{2,}'       # ц medial
]

# Load the dictionary as a set of lowercase keys
dict_keys = set(transliteration_dict.keys())        # lowercased at load time

identifiers = extract_identifiers(file)
# extract_identifiers returns: variable names, function names, class names,
# field names, parameter names, NOT string literals, NOT comments

for ident in identifiers:
    if not is_latin_only(ident):          # cyrillic identifiers are NOT flagged
        continue

    # split on camelCase boundaries and snake_case underscores
    parts = split_identifier_parts(ident)  # e.g. "kolichestvoTovarov" → ["kolichestvo", "tovarov"]
    parts_lower = [p.lower() for p in parts]

    # Tier 1: dictionary match on any part
    tier1 = any(p in dict_keys for p in parts_lower)

    # Tier 2: phonotactic signal on the full identifier
    tier2 = any(re.search(pat, ident.lower()) for pat in PHONOTACTIC_PATTERNS)

    # Tier 3: file (or project) has Russian content — comes from activation scan
    tier3 = file_has_cyrillic

    # Trigger condition: tier 1 MUST hit, AND (tier 2 OR tier 3) MUST hit.
    # - tier1 + tier2: strong phonotactic signal even if file is English
    # - tier1 + tier3: weaker, but the file context supports the dictionary hit
    # Tier 1 alone is not enough (could be coincidence: "tsar", "pizza")
    if tier1 and (tier2 or tier3):
        matched_part = next(p for p in parts_lower if p in dict_keys)
        suggestion = transliteration_dict[matched_part]
        emit(Info,
             file=file, line=ident.line,
             category="#5-transliteration",
             confidence=LOW,
             message=f"Identifier `{ident}` looks like transliterated Russian "
                     f"(dictionary hit on `{matched_part}`). "
                     f"Suggested English equivalent: `{suggestion}`. "
                     f"Or keep Cyrillic if your project convention uses Cyrillic identifiers.")
```

### Test cases

| Identifier | File has Cyrillic? | Expected |
|---|---|---|
| `polzovatel_id` | Yes | **Info** (tier1: `polzovatel` in dict; tier3: yes) |
| `kolichestvo_tovarov` | Yes | **Info** (tier1 on both parts; tier2: no; tier3: yes) |
| `tsar` | Yes | No finding (not in dict; phonotactic pattern requires medial `ts`) |
| `polzovatel_id` | No (pure English project) | No finding (tier3 false, tier2 false — only tier1 hits) |
| `zagruzka` | Yes | **Info** (tier1 hit, tier3 hit) |
| `userPolzovatel` | No | No finding (tier3 false) — unless `zh`/`shch`/`kh` cluster exists, which this doesn't |
| `poluchitShchenka` | No | **Info** (tier1: `poluchit`; tier2: `shch` cluster present) — fires even without file cyrillic |
| `count` | Yes | No finding — `count` is not in dict |
| `pizza` | Yes | No finding — not in dict, no phonotactic hit |
| `полз_идентификатор` (Cyrillic) | Yes | No finding — not Latin-only |

### Identifier-splitting rules

```
def split_identifier_parts(name):
    # Handle snake_case
    parts = name.split('_')
    # Handle camelCase within each underscore-part
    result = []
    for p in parts:
        # split on lower→upper transitions
        pieces = re.findall(r'[A-Z]?[a-z]+|[A-Z]+(?=[A-Z]|$)|\d+', p)
        result.extend(pieces)
    return [p for p in result if p]
```

### Compound-identifier note

A compound identifier where only *one* part hits the dictionary (e.g. `userKolichestvo`) still fires tier 1. This is intentional — the transliterated part is still a suggestion target. The suggestion message highlights the specific part that matched so the developer can rename just that piece.

### Nonsense-compound edge case

The fixture has `polzovatel_tovarov` ("user of products") which is semantically nonsensical and probably a typo. The skill cannot detect semantic nonsense in v1 — it will fire on `polzovatel` tier-1 and emit Info. That is acceptable behavior: the developer sees the flag and can decide whether to rename or fix the typo.

### False-positive avoidance

- **Never flag identifiers with Cyrillic characters directly.** Projects that use pure Cyrillic identifiers (`полз_идентификатор`) are a valid style and not in scope.
- **Do not flag loanwords already accepted as English.** `admin`, `video`, `photo`, `data`, `status`, `array`, `object`, `type` are tagged in the dictionary as "already English" and excluded from triggering (see the dict comments).
- **Do not flag in pure English projects** unless a phonotactic signal fires. Tier 3 (file has Cyrillic) is the soft guard; tier 2 (phonotactic) is the hard signal that catches `shch`/`zh` clusters regardless of file context.

---

## Summary of severity caps

| Check | Max severity | Notes |
|---|---|---|
| #4 comment mixing | Warning | Never promote to Error |
| #4 inline Russian literals | Warning | Never promote to Error |
| #5 transliteration | Info | Never promote to Warning or Error |

These caps are non-negotiable per the Phase 2 design decisions.
