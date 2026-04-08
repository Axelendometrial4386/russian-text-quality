# Concatenation Triggers Reference

Detection logic for **check #3 — Concatenation case bug**. Heuristic; medium confidence; always `Warning`, never `Error`.

## Why this check exists

Russian nouns inflect for **six grammatical cases** (nominative, genitive, dative, accusative, instrumental, prepositional). Verbs and prepositions *govern* a specific case for their object — for example, `Удалить` requires the **accusative**. When a developer writes:

```ts
const label = `Удалить ${item.name}?`;
```

they interpolate `item.name` in its nominative (dictionary) form. For many Russian nouns this is ungrammatical — animate masculines, all plural animates, and feminines ending in `-а/-я` have a different accusative. The sentence comes out wrong for half the Russian lexicon.

Primary linguistic source: [gramota.ru — Падеж](https://gramota.ru/biblioteka/spravochniki/russkij-yazyk-kratkij-teoreticheskij-kurs-dlya-shkolnikov/padezh-kak-morfologicheskiy-priznak-sushchestvitelnykh).

## Closed trigger list

Detection fires when a Russian token from this list appears immediately before an interpolated variable or a concatenation point. All tokens are lowercased; matching is case-insensitive.

### Prepositions governing non-nominative cases

| Token | Governs | Example bug |
|---|---|---|
| `в` | accusative/prepositional | `"Перейти в " + section` |
| `на` | accusative/prepositional | `` `Открыть на ${page}` `` |
| `за` | accusative/instrumental | `"Платить за " + item` |
| `под` | accusative/instrumental | `"Положить под " + shelf` |
| `над` | instrumental | `"Работать над " + task` |
| `перед` | instrumental | `"Показать перед " + block` |
| `от` | genitive | `"Отменить от " + user` |
| `из` | genitive | `"Удалить из " + folder` |
| `к` | dative | `"Перейти к " + step` |
| `у` | genitive | `"Забрать у " + person` |
| `с` | genitive/instrumental | `"Начать с " + chapter` |
| `о` | prepositional | `"Сообщить о " + event` |
| `об` | prepositional | `"Вопрос об " + item` |
| `про` | accusative | `"Рассказать про " + topic` |
| `для` | genitive | `"Создать для " + user` |
| `без` | genitive | `"Работать без " + tool` |
| `до` | genitive | `"Продлить до " + date` |
| `после` | genitive | `"Запустить после " + step` |
| `между` | instrumental | `"Связать между " + items` |
| `по` | dative | `"Отсортировать по " + field` |

### Transitive verbs in imperative/infinitive forms (template-start)

These typically require the **accusative** for their object.

| Token | Example bug |
|---|---|
| `удалить` | `` `Удалить ${item}?` `` |
| `добавить` | `"Добавить " + item` |
| `открыть` | `"Открыть " + file` |
| `закрыть` | `"Закрыть " + window` |
| `показать` | `"Показать " + dialog` |
| `скрыть` | `"Скрыть " + panel` |
| `выбрать` | `"Выбрать " + option` |
| `изменить` | `"Изменить " + setting` |
| `сохранить` | `"Сохранить " + file` |
| `найти` | `"Найти " + query` |
| `создать` | `"Создать " + entity` |
| `редактировать` | `"Редактировать " + entity` |

### Gender-coupled adjectives (salutation trap)

These agree in **gender** with the following noun. Interpolating a name without knowing the user's gender gives the wrong form half the time.

| Token | Trap |
|---|---|
| `уважаемый` | fails for female users (`Уважаемая`) |
| `уважаемая` | fails for male users (`Уважаемый`) |
| `дорогой` | same |
| `дорогая` | same |

## Patterns to match

```
# Template literals (JS/TS)
TEMPLATE_LITERAL = /`[^`]*?\b(TRIGGER)\s*\$\{(\w+)\}[^`]*`/i

# String concatenation
CONCAT = /["'][^"']*?\b(TRIGGER)\s*["']\s*[+.]\s*(\w+)/i

# Python f-strings
F_STRING = /f["'][^"']*?\b(TRIGGER)\s*\{(\w+)\}[^"']*["']/i

# Python .format / %
FORMAT_PCT = /["'][^"']*?\b(TRIGGER)\s*(%s|\{(\w+)?\})[^"']*["']/i
```

Where `TRIGGER` is the closed list above.

## Exclusions

A match is **NOT** flagged if any of these apply:

- **Interpolated variable is clearly numeric**: variable name matches `/^(n|count|num|total|amount|size|length|index|idx|i|j|k|qty|quantity)$/i`. Numeric interpolation doesn't trigger case problems.
- **ICU MessageFormat wrapper detected**: if the match sits inside a `{x, plural, ...}` or `{x, select, ...}` block, trust the developer's ICU usage and skip.
- **The trigger word is itself inside a code identifier or URL**, not a Russian sentence. Heuristic: require at least one Cyrillic character adjacent to the trigger word (before or after, within 10 chars).
- **Proper noun after preposition**: if the variable name is `city`, `country`, `street`, or similar geographic, and the preposition is `в`/`на`, treat as lower-confidence — emit `Info` instead of `Warning`. Proper geographic nouns have idiomatic accusative forms.

## Emitting findings

```
emit(Warning,   # Never Error — heuristic
     confidence=MEDIUM,
     file=match.file, line=match.line,
     message=f"Russian case agreement: `{trigger}` governs a non-nominative case; "
             f"interpolated `{var}` will be inserted in dictionary form.",
     fix="Use a full template per grammatical case, or ICU `{var, select, ...}` with inflected forms.")
```

## False-positive budget

Accepted in v1: ~10–15% false positive rate (per Phase 2 decision). Mitigation: all findings are `Warning` severity, never auto-fixed, always explained so the developer can dismiss them in one read.

## Extending the trigger list

New triggers may be added to this file via PR. Each addition must:
1. Link to a linguistic reference (gramota.ru, Rosenthal handbook) confirming the case government.
2. Provide at least one concrete bug example (real or constructed).
3. Pass a quick false-positive check: run the updated list against 5 Russian open-source repos and confirm warning volume did not explode.
