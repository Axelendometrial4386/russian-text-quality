# Security Policy

## Supported versions

Security-relevant fixes target the latest released version on `main`. Older tags do not receive back-ported fixes.

| Version | Supported |
|---|---|
| 0.1.x   | ✅ latest |
| < 0.1.0 | ❌ |

## What counts as a security issue in this project?

This skill ships **no executable runtime code**. It is a set of Markdown and YAML instruction files that are loaded into a Claude Code agent's context. "Security" in the traditional sense (RCE, XSS, supply-chain attacks) does not apply.

However, the following DO count as security-relevant and I want to hear about them:

1. **Prompt-injection payloads embedded in the skill content.** If you find a place in `SKILL.md` or the reference files where untrusted input (e.g. a Russian string copied from an external source) could steer an agent into ignoring its safety rules, that is a security issue.
2. **Instructions that could cause an agent to leak sensitive data.** For example, if the skill tells the agent to email findings somewhere, or to upload locale files to a third-party service — that should not happen and reporting it is welcome.
3. **Malicious dictionary entries or examples.** If someone opens a PR adding a `transliteration-dict.yml` entry or an `examples.md` case that contains hidden prompt-injection, it must be caught. Reporting via security advisory is welcome.
4. **Typosquatting or impersonation of companion tools.** If a PR tries to redirect install instructions or the "External tools we recommend" list to a fake version of `talkstream/ru-text`, `yaspeller`, or `eyo`, that is a supply-chain attack vector and should be reported privately.

## How to report

**Do not open a public Issue for security problems.**

- **Preferred:** open a private GitHub security advisory via https://github.com/Anic888/russian-text-quality/security/advisories/new
- **Alternative:** open a Discussion marked as private if advisories are not available to you.

I will respond within seven days and either confirm, request clarification, or explain why the report does not count as a security issue. Please include:
- A concrete example of the payload or content
- A description of the expected vs. actual agent behavior
- Any references to the specific file and line

## Thanks

Reporters who find real issues will be credited in `CHANGELOG.md` unless they ask to remain anonymous.
