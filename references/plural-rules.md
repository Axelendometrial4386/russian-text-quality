# Plural Rules Reference

This is the detection logic for **check #1 — Plural structure**. Core value feature. Read fully before implementing.

## CLDR categories for Russian

Russian has **four** plural categories under CLDR. Source: [Unicode CLDR language plural rules](https://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html).

| Category | Formula (for integer `i`) | Examples |
|---|---|---|
| `one` | `i % 10 == 1 && i % 100 != 11` | 1, 21, 31, 101 |
| `few` | `i % 10 in 2..4 && i % 100 not in 12..14` | 2, 3, 4, 22, 23, 24 |
| `many` | `i % 10 == 0` OR `i % 10 in 5..9` OR `i % 100 in 11..14` | 0, 5–20, 25, 11, 12, 13, 14 |
| `other` | fractional numbers only | 1.5, 2.5 (integers never map here in Russian) |

**Critical fact:** integers in Russian never map to `other`. If a locale file has only an `other` branch for a Russian plural message, it will produce a wrong form for every integer. This is the bug shape we detect.

Historical note from the CLDR team: the names `many` and `other` would ideally have been swapped for Russian, but are frozen for backward compatibility. Do not let the naming confuse detection.

## Framework 1 — i18next

### Reference file shape (v4 format)

```json
{
  "item_one":   "{{count}} товар",
  "item_few":   "{{count}} товара",
  "item_many":  "{{count}} товаров",
  "item_other": "{{count}} товара"
}
```

Or nested:
```yaml
item:
  one: "{{count}} товар"
  few: "{{count}} товара"
  many: "{{count}} товаров"
  other: "{{count}} товара"
```

Source: [i18next plural documentation](https://www.i18next.com/translation-function/plurals). Real-file reference with the identical YAML shape: [rails-i18n ru.yml](https://github.com/svenfuchs/rails-i18n/blob/master/rails/locale/ru.yml).

Note: i18next v3 used `key_plural` instead of `key_other`. **v1 of this skill supports v4 only.** If the repo uses v3, emit an `Info` finding noting the scope limitation and stop plural checks for that file.

### Detection algorithm

```
REQUIRED_RU = {"one", "few", "many", "other"}
PLURAL_SUFFIXES = {"zero", "one", "two", "few", "many", "other"}

INPUT: locale_file (ru*.json / ru*.yml), call_sites (ts/js/tsx/jsx source files)

# Step 1 — find pluralized base keys in the locale file
flat = flatten_to_dot_paths(parse(locale_file))
pluralized_bases = {}
for key in flat:
    for suffix in PLURAL_SUFFIXES:
        sep = detect_plural_separator(locale_file) or "_"   # usually "_"
        if key.endswith(sep + suffix):
            base = strip_trailing(key, sep + suffix)
            pluralized_bases.setdefault(base, set()).add(suffix)

# Step 2 — locale-side completeness check
for base, present in pluralized_bases.items():
    missing = REQUIRED_RU - present
    if missing:
        emit(Error,
             file=locale_file,
             key=base,
             message=f"Russian plural key `{base}` missing: {sorted(missing)}",
             fix=scaffold_suffixes(base, missing, sep))

# Step 2.5 — locale-value interpolation-without-plurals detection
# If a locale value contains a count-interpolation placeholder but the key is NOT a pluralized
# base (has no known plural-suffix siblings), the key is mis-authored. This catches the bug
# shape where the developer wrote a single-form Russian string with {{count}} inside it.
COUNT_PLACEHOLDERS = [r'\{\{\s*count\s*\}\}',       # i18next mustache
                     r'\{\s*count\s*\}',            # ICU / format
                     r'%\{\s*count\s*\}',           # po/gettext style
                     r'\{n\}', r'\{\{n\}\}']         # short form "n"
for key, value in flat.items():
    if key in pluralized_bases:        # already a plural base, handled by Step 2
        continue
    # Skip keys that are themselves a plural suffix variant of some base
    if any(key.endswith(sep + s) for s in PLURAL_SUFFIXES):
        continue
    if not isinstance(value, str):
        continue
    # ICU guard: if the value is itself a `{x, plural, ...}` ICU block, do NOT flag it —
    # the ICU algorithm in Framework 2 handles completeness inside the block. Without
    # this guard, pure-ICU projects would false-positive on every valid plural.
    if re.search(r'\{\s*\w+\s*,\s*plural\s*,', value):
        continue
    if any(re.search(p, value) for p in COUNT_PLACEHOLDERS):
        emit(Error,
             file=locale_file, key=key,
             message=f"Key `{key}` value contains count placeholder but has no plural siblings "
                     f"(`{key}_one`/`_few`/`_many`/`_other`). Russian requires all four forms.",
             fix=scaffold_all(key, REQUIRED_RU, sep))

# Step 3 — call-site bug: t(key, {count}) but the key has no plural siblings at all
for call in scan(call_sites, /\bt\(\s*['"`]([^'"`]+)['"`]\s*,\s*\{[^}]*\bcount\s*:/):
    key = call.key
    if key in flat and key not in pluralized_bases:
        emit(Error,
             file=call.file, line=call.line,
             message=f"Key `{key}` called with count but has no plural forms in ru locale",
             fix=scaffold_all(key, REQUIRED_RU, sep))
```

### Test cases

| Input | Expected |
|---|---|
| `{"item_one":"...", "item_other":"..."}` in ru.json | **Error** (Step 2): missing `item_few`, `item_many` |
| All four `_one/_few/_many/_other` present | No finding |
| `{"notifications": "У вас {{count}} новых уведомлений"}` (count in value, no plural siblings, no call site) | **Error** (Step 2.5): value has `{{count}}` but key is not a plural base |
| `{"items": "{{count}} товаров"}` + `t('items', {count: 1})` in source | **Error** (Step 2.5 or Step 3): caught by both paths |
| `{"welcome": "Привет"}` + `t('welcome')` (no count anywhere) | No finding — not a plural key |
| `{"item_plural": "..."}` (i18next v3 legacy) | **Info**: v3 format unsupported in v1 — stop plural checks for that file |

### Known limitations

- Requires the literal `t(` function name at call sites; aliased/renamed imports (`const translate = t`) are missed.
- Cannot verify whether the **content** of each branch is linguistically correct Russian — only that the branch exists.
- Multi-namespace locale files (e.g. `common.json`, `errors.json`) must be scanned as a group; skill should load all `*.json` under the ru locale directory.

## Framework 2 — ICU MessageFormat (FormatJS / react-intl)

### Reference shape

```json
{
  "cart.items": "{count, plural, one {# товар} few {# товара} many {# товаров} other {# товара}}"
}
```

### Bug shape (syntactically valid ICU, grammatically wrong Russian)

```json
{
  "cart.items": "{count, plural, other {# товаров}}"
}
```

Source: [FormatJS ICU syntax documentation](https://formatjs.github.io/docs/core-concepts/icu-syntax/) — "other is required" but all other categories are optional. [ICU spec](https://unicode-org.github.io/icu/userguide/format_parse/messages/) confirms.

**FormatJS's own linter `@formatjs/cli lint` does NOT catch this.** It validates syntax, not Russian completeness. Confirmed via [FormatJS linter docs](https://formatjs.github.io/docs/tooling/linter/).

### Detection algorithm

```
INPUT: ru locale files containing ICU messages OR defineMessages({...}) blocks in source

# Regex for plural blocks (ICU, no nesting in v1):
PLURAL_RE = /\{\s*(\w+)\s*,\s*plural\s*,\s*((?:(?:=\d+|zero|one|two|few|many|other)\s*\{[^{}]*\}\s*)+)\}/g

for file in ru_locales + source_files_with_defineMessages:
    for match in PLURAL_RE.findall(file_content):
        selectors_raw = match.group(2)
        # Extract selector tokens (the words before each '{')
        selectors = set(token for token, _body in parse_icu_branches(selectors_raw))
        required = {"one", "few", "many", "other"}
        missing = required - selectors

        # Literal-case substitution: =1 substitutes for one, =0 substitutes for many (CLDR)
        if "=1" in selectors and "one" in missing: missing.discard("one")
        if "=0" in selectors and "many" in missing: missing.discard("many")

        if missing:
            emit(Error,
                 file=file, span=match.span(),
                 message=f"ICU Russian plural missing: {sorted(missing)}",
                 fix=insert_empty_branches(match, missing))
```

### Test cases

| Input | Expected |
|---|---|
| `{count, plural, other {# товаров}}` | **Error**: missing `one`, `few`, `many` |
| `{count, plural, one {# товар} other {# товаров}}` | **Error**: missing `few`, `many` |
| `{count, plural, one {} few {} many {} other {}}` | No finding |
| `{count, plural, =0 {нет} one {# товар} few {# товара} many {# товаров} other {# товара}}` | No finding |
| `{count, plural, =1 {один товар} few {# товара} many {# товаров} other {# товара}}` | No finding — `=1` substitutes for `one` |
| Plain string without `plural,` | Skip |

### Known limitations

- Nested plurals (plural inside select) need a real ICU AST parser. v1 regex-based detector may miss them — emit `Info` warning that nested blocks were skipped if detected.
- Escape-aware: `'{'` literal-escape sequences need handling in a full parser. v1 regex is approximate.

## Framework 3 — vue-i18n

### Reference shape

```json
{
  "car": "0 машин | {n} машина | {n} машины | {n} машин"
}
```

Four pipe-separated segments (0-form, one-form, few-form, many-form). Source: [vue-i18n pluralization docs](https://vue-i18n.intlify.dev/guide/essentials/pluralization).

Developer MUST ALSO configure `pluralizationRules.ru` in the VueI18n constructor. Without it, vue-i18n falls back to an English-style rule that is wrong for Russian.

### Detection algorithm

```
INPUT: ru locale files, optional i18n init files (main.ts, i18n.ts, plugins/i18n.ts)

# Pipe segments (ignore escaped \|)
def count_segments(value):
    parts = re.split(r'(?<!\\)\|', value)
    return len(parts)

for file in ru_locales:
    for key, value in flatten(parse(file)):
        if not isinstance(value, str): continue
        segs = count_segments(value)
        if segs >= 2:  # value looks like a plural (has a pipe)
            if segs < 4:
                emit(Warning,   # NOT Error — pipe-heuristic caveat accepted in v1
                     file=file, key=key,
                     message=f"vue-i18n Russian plural needs 4 pipe-separated forms, found {segs}",
                     fix="Add missing forms: 0-form | one | few | many")

# Config check
if any(ru_locale_file exists) and no `pluralizationRules.ru` found in init files:
    emit(Warning,
         message="vue-i18n has no built-in Russian plural rule. Configure pluralizationRules.ru.",
         link="https://vue-i18n.intlify.dev/guide/essentials/pluralization#custom-pluralization")
```

### Test cases

| Input | Expected |
|---|---|
| `"car": "машина"` (no pipes) | No finding — not a plural value |
| `"car": "{n} машина \| {n} машины"` (1 pipe, 2 segs) | **Warning**: only 2 forms, need 4 |
| `"car": "0 машин \| {n} машина \| {n} машины \| {n} машин"` | No finding |
| Locale has ru but app has no `pluralizationRules.ru` | **Warning**: missing custom rule |

### Known limitations

- Pipe-heuristic false positives on prose containing literal `|`. v1 emits `Warning` severity, not `Error`, to honor this caveat (accepted in Phase 2 review).
- Escaped pipes `\|` must be skipped when counting.
- vue-i18n v8 syntax was different; v1 targets v9+.

## Framework detection order

If `package.json` exists, detect dependencies in this order:
1. `i18next`, `react-i18next`, `next-i18next` → use Algorithm i18next
2. `@formatjs/*`, `react-intl` → use Algorithm ICU
3. `vue-i18n`, `@intlify/*` → use Algorithm vue-i18n
4. None → content sniff: ICU `{..., plural, ...}` → ICU; keys with `_one/_few/_many/_other` → i18next; pipe-separated → vue-i18n
5. Ambiguous → run all applicable, deduplicate findings.

## Severity matrix

| Framework | Severity | Rationale |
|---|---|---|
| i18next missing suffixes | **Error** | Deterministic, no false positive |
| ICU missing categories | **Error** | Deterministic, ICU spec-level |
| vue-i18n insufficient pipes | **Warning** | Pipe heuristic can false-positive on prose |
| vue-i18n missing pluralizationRules | **Warning** | Config check, no content false-positive risk |
