# Activation Logic Reference

Gate: this skill must NEVER activate on English-only projects. Two conditions must both hold.

## Condition A — Meaningful Russian content

```
total_cyrillic  = 0
cluster_found   = false
relevant_files  = []

for file in walk(project_root):
    if file matches .gitignore OR any of:
        {node_modules, .git, dist, build, out, .next, target, coverage, __pycache__}:
        skip

    ext = file.extension.lower()
    if ext in PRIMARY_LOCALE_EXT or ext in SECONDARY_CODE_EXT:
        relevant_files.append(file)
        content = read(file, maxBytes=512 * 1024)    # skip huge generated files
        total_cyrillic += count_matches(content, /\p{Script=Cyrillic}/u)
        if re.search(/\p{Script=Cyrillic}{5,}/u, content):
            cluster_found = true

activate_A = (total_cyrillic >= 10) and cluster_found
```

Where:

```
PRIMARY_LOCALE_EXT = {"json", "json5", "yml", "yaml", "po", "pot", "properties"}
SECONDARY_CODE_EXT = {"ts", "tsx", "js", "jsx", "mjs", "cjs",
                      "vue", "svelte",
                      "py", "rb", "php", "go", "rs", "kt", "swift", "java", "cs"}
```

## Condition B — Relevant file types present

Implied by `relevant_files >= 1` after Condition A scan. If no code or locale files of the supported types exist in the project, nothing to check — deactivate.

## Final decision

```
if activate_A and len(relevant_files) >= 1:
    ACTIVATE
else:
    DEACTIVATE with message "Not activated — no Russian content detected"
```

## Explicit invocation bypass

If the user invokes the skill by name in the prompt (e.g. "use russian-text-quality" or "run the Russian review skill"), honor that regardless of activation thresholds — the user may be about to add Russian content to an English project.

## Test cases

| Scenario | Expected | Reason |
|---|---|---|
| Pure English Next.js project, no Cyrillic anywhere | **DEACTIVATE** | `total_cyrillic = 0` |
| Project with a single 🇷🇺 emoji | **DEACTIVATE** | Emoji is not Cyrillic script |
| Project with one Russian comment "// TODO" + "// сделать позже" (14 chars Cyrillic in one 5+ cluster) | **ACTIVATE** | Both conditions met |
| Project with one isolated Russian word `тест` scattered across 3 files (12 chars total) | **ACTIVATE if any single cluster ≥5** | `тест` is 4 chars; if no single 5+ cluster exists, DEACTIVATE |
| Pure English project, user prompt explicitly invokes skill | **ACTIVATE** | Explicit invocation bypass |
| README in Russian, no code files | **DEACTIVATE** | `relevant_files = 0` (`.md` is not in the primary/secondary list) |
| ru.json with 200 entries + TypeScript source | **ACTIVATE** | Strong signal on both axes |

## Excluded paths (non-exhaustive)

Always skip, regardless of content:

```
**/node_modules/**
**/.git/**
**/.svn/**
**/.hg/**
**/dist/**
**/build/**
**/out/**
**/.next/**
**/.nuxt/**
**/target/**
**/coverage/**
**/__pycache__/**
**/*.min.js
**/*.min.css
**/*.lock
**/vendor/**
**/bower_components/**
```

Also respect `.gitignore` patterns if the project has one.

## File-size guard

Skip individual files larger than **512 KB**. Generated locale files can be huge and add no detection value beyond the first batch of keys. Emit an `Info` note if large files were skipped.

## Rationale for the thresholds

- **10 total Cyrillic chars**: two Russian words in one file is enough signal; one accidental emoji or name never reaches this.
- **≥5 consecutive Cyrillic chars**: rules out single names (`Дмитрий`, `Ivan`) used as data but still catches short Russian words like `товар` (5 chars) or longer.
- **Both must hold**: single stray clusters in an otherwise English codebase (e.g. a comment "// note: Лёша's ticket") do not trigger the full scan.
