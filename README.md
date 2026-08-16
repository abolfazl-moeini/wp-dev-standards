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

**Gettext source strings MUST be English.** Target languages (Persian, German, …) are applied via translation catalogs, never as literals inside `__()`, `_e()`, `esc_html_e()`, or JS `wp.i18n` / `@wordpress/i18n`.

Wrapping strings is i18n. Shipping a locale is the WP-CLI catalog pipeline: `make-pot` → create/`update-po` → fill `msgstr` → `make-mo` for PHP, and `make-json` for JS (JSON comes from the **same PO**, not from JS source). There is no `wp i18n make-po`. JS has `__`, `_x`, `_n`, `_nx`, `sprintf` — no `_e`.

See `SKILL.md` §1.0 / §1.6 and `references/localization.md` §3.3 / §11.
