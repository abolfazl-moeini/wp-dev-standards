# wp-dev-standards

WordPress agent skill: team house rules for plugin/theme development, with Localization (i18n) first.

## Install

Copy or symlink this directory into your agent skills path (e.g. `.agents/skills/wp-dev-standards` or the equivalent for your host).

Required layout:

```
wp-dev-standards/
├── SKILL.md
└── references/
    └── localization.md
```

## Critical house rule

**Gettext source strings MUST be English.** Target languages (Persian, German, …) are applied via translation catalogs (`.po` / `.mo` / language packs), never as literals inside `__()`, `_e()`, `esc_html_e()`, etc.

See `SKILL.md` §1.0 and `references/localization.md`.
