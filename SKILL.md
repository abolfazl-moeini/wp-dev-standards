---
name: wp-dev-standards
description: "House rules for WordPress plugin/theme development on this team. Use for Localization/i18n (always English gettext sources; __(), _e(), esc_html_e(), text domains, Domain Path), string escaping, and team coding conventions. Prefer this skill over domain skills when deciding how to write or review user-visible strings."
compatibility: "Targets WordPress 7.0+ (PHP 7.4.0+). Filesystem-based agent with bash + node."
---

# WordPress Dev Standards (House Rules)

Internal rules for plugins, themes, blocks, patterns, and admin UI.
Domain skills (`wp-plugin-development`, `wp-patterns`, `wp-rest-api`, …) still own architecture and APIs; **this skill wins on string/i18n and team conventions**.

## When to use

- Adding or changing any **user-visible string** (admin, front, REST, patterns, blocks, JS UI).
- Reviewing PRs for **missing translations**, non-English gettext sources, wrong text domain, or unescaped output.
- Scaffolding plugins/themes (**Text Domain** / **Domain Path**).
- Interactivity API labels or serializing translations to JS.
- User asks for “house rules”, “our WordPress standards”, localization, or i18n.

## Inputs required

- Plugin/theme **slug** and **text domain** (must match; lowercase; dashes, not underscores).
- Target surface: plugin, theme, block, pattern, ability/REST, or JS script handle.
- Target **display** locale (e.g. `fa_IR`) only when generating translation catalogs — **never** as the language of gettext source strings.

---

## 1) Localization (i18n) — always first

Full depth: **`references/localization.md`**.

### 1.0 Source language is always English (CRITICAL)

> **The text inside translation functions (`__()`, `_e()`, `esc_html_e()`, `_n()`, `_x()`, and all siblings in PHP or JS) MUST be written in English.**

> **No matter what language the plugin, theme, market, client, or UI targets (Persian, Arabic, German, …): source strings in code are English. The target language is applied later via translation files (`.po` / `.mo` / language packs / translate.wordpress.org) — never by putting non-English literals inside gettext calls.**

```php
// CORRECT — English source; locale comes from translations
esc_html_e( 'Save settings', 'my-plugin' );

// WRONG — never put the target language in the gettext string
esc_html_e( 'ذخیره تنظیمات', 'my-plugin' );
esc_html_e( 'Einstellungen speichern', 'my-plugin' );
```

**Hard house rule.** Do not “localize” by writing Farsi/Arabic/etc. inside `__()` / `_e()` / `esc_*`. Write decent English; ship a `.po` for the real locale.

### 1.1 Non-negotiable rules

0. **English source strings only** — §1.0.
1. **Always i18n** — every user-visible string uses a gettext function + project text domain.
2. **Always escape on output** — prefer `esc_html_*` / `esc_attr_*` over bare `__()` / `_e()` when printing HTML/attributes.
3. **One text domain per project** — equals the **slug**: lowercase, **dashes not underscores**, no spaces. Same domain in header, PHP, and JS.
4. **Literal domain strings** — never `__( '…', $domain )`; extractors need a string literal.
5. **Always pass the text domain** — even for core-like words (`Settings`).
6. **Placeholders, not interpolation** — `sprintf` / `printf` with `%s` / `%1$s`; no `"Hello $name"` and no concatenated sentence fragments.
7. **Messages translate; machine codes do not** — `WP_Error` messages use `__()`; codes stay stable English slugs.
8. **Patterns** — visible text: `esc_html_e` / `esc_html__` / `esc_attr_e`; registration `title` / `description`: `__()`.

### 1.2 Quick function map

| Need | Function | Notes |
|------|----------|--------|
| Return translated string | `__( 'Text', 'text-domain' )` | **Text = English** |
| Echo translated string | `_e( 'Text', 'text-domain' )` | Prefer escaped helpers for HTML |
| Return + HTML-escape | `esc_html__( 'Text', 'text-domain' )` | Body text |
| Echo + HTML-escape | `esc_html_e( 'Text', 'text-domain' )` | **Default for visible HTML** |
| Context + HTML-escape | `esc_html_x( 'Post', 'noun', 'text-domain' )` | |
| Return + attr-escape | `esc_attr__( 'Text', 'text-domain' )` | |
| Echo + attr-escape | `esc_attr_e( 'Text', 'text-domain' )` | `alt`, `placeholder`, titles |
| Context + attr-escape | `esc_attr_x( … )` | |
| Plural | `_n( $single, $plural, $n, 'text-domain' )` | Prefer `'%s item'` over `'One item'`; escape when printing |
| Context | `_x( 'Post', 'noun', 'text-domain' )` / `_ex()` | |
| Plural + context | `_nx( … )` | |
| Numbers / dates | `number_format_i18n()` / `wp_date()` | Locale-aware formatting (`date_i18n()` is legacy) |
| URL | `esc_url( $url )` | Not gettext |

### 1.3 Minimal examples

```php
// Visible HTML — English source only
<h2><?php esc_html_e( 'Our Services', 'my-plugin' ); ?></h2>

return new WP_Error(
    'my_plugin_missing_order_id', // code: NOT translated
    __( 'An order ID is required.', 'my-plugin' ) // English message source
);

<img alt="<?php esc_attr_e( 'Team photo', 'my-plugin' ); ?>" />

printf(
    /* translators: %s: Name of a city. */
    esc_html__( 'Your city is %s.', 'my-plugin' ),
    esc_html( $city )
);

printf(
    /* translators: 1: City name 2: ZIP code. */
    esc_html__( 'Your city is %1$s, and your zip code is %2$s.', 'my-plugin' ),
    esc_html( $city ),
    esc_html( $zipcode )
);
```

```php
// WRONG: raw text
<h2>Our Services</h2>

// WRONG: non-English gettext source
esc_html_e( 'خدمات ما', 'my-plugin' );

// WRONG: translated but not escaped for HTML
_e( 'Our Services', 'my-plugin' );

// WRONG: variable inside gettext template
_e( "Your city is $city.", 'my-plugin' );

// WRONG: variable text domain
__( 'Save', $text_domain );

// WRONG: translated error code
return new WP_Error( __( 'my_plugin_missing_id', 'my-plugin' ), '...' );

// WRONG: sentence concatenation
__( 'Your city is ', 'my-plugin' ) . $city . __( '.', 'my-plugin' );
```

### 1.4 Text domain setup

```php
/**
 * Plugin Name: My Plugin
 * Text Domain: my-plugin
 * Domain Path: /languages
 */
```

```php
add_action( 'init', function () {
    load_plugin_textdomain(
        'my-plugin',
        false,
        dirname( plugin_basename( __FILE__ ) ) . '/languages'
    );
} );
```

- Themes: `Text Domain` in `style.css`; `load_theme_textdomain( 'theme-slug', get_template_directory() . '/languages' )` on `after_setup_theme`.
- Plugin/theme language-dir MO files: prefer `{text-domain}-{locale}.mo` (e.g. `my-plugin-fa_IR.mo`). Global dirs under `wp-content/languages/{plugins,themes}/` use the same pattern.
- wordpress.org language packs take priority since WP 4.6+; still declare domain/path correctly. Off-directory or custom `.mo` still needs `load_*_textdomain` (or a `load_textdomain_mofile` filter).

### 1.5 Checklist (before merge)

- [ ] **Every gettext string is English**
- [ ] Every user-visible string uses gettext + correct domain
- [ ] HTML → `esc_html_e` / `esc_html__` / `esc_html_x` (not bare `_e`)
- [ ] Attributes → `esc_attr_e` / `esc_attr__` / `esc_attr_x`
- [ ] URLs → `esc_url()`; not stuffed into gettext unless intentionally localizable
- [ ] Plurals → `_n` / `_nx`; context → `_x` / `_ex` / `_nx`
- [ ] Placeholders → `sprintf`/`printf` + `/* translators: ... */` immediately above the call
- [ ] No leading/trailing whitespace; no empty gettext strings
- [ ] Minimal HTML inside translated strings; full sentences preferred
- [ ] `WP_Error` codes literal; messages English gettext
- [ ] Text domain = slug, literal, same in header + PHP + JS
- [ ] Pattern/block registration titles and descriptions translated

---

## 2) Security baseline (strings & I/O)

- Sanitize/validate on input; escape on output.
- Explicit `$_POST` / `$_GET` keys + `wp_unslash()` then sanitize.
- Nonces **and** `current_user_can()`.
- SQL: `$wpdb->prepare()` only.
- Depth: `wp-plugin-development` → `references/security.md`.

---

## 3) Patterns, Interactivity, Abilities (i18n pointers)

- **Patterns** (`wp-patterns`): registration-time PHP only; `esc_html_e` / `esc_attr_e` for visible/attr text; translate `register_block_pattern()` title/description.
- **Interactivity** (`wp-interactivity-api`): English gettext in PHP for server HTML and `wp_interactivity_state()` labels; always pass text domain.
- **Abilities/REST** (`wp-abilities-api`): translate messages, never codes; translator comments for placeholders.

---

## Procedure (agent)

1. Resolve text domain from plugin/theme header (or ask once). Domain must equal slug.
2. Apply **§1** to every new/changed user-visible string — **English sources only**.
3. Prefer escaped helpers for HTML; bare `__()` for non-HTML contexts (arrays, errors, registration args).
4. Run §1.5 checklist.
5. Route architecture/API work to the matching domain skill; keep i18n decisions here.

## Verification

- Grep: no non-English literals inside gettext / `wp.i18n` calls.
- Load a locale (or test `.po`) and confirm UI language changes **without** editing PHP/JS sources.
- Grep: no raw user-facing strings without gettext; no translated error codes.
- Patterns: English gettext sources; no bare `_e()` in markup.

## Failure modes

| Symptom | Likely cause | Fix |
|---------|----------------|-----|
| Always English on fa/de site | Domain/path/load or missing catalog | Align domain + load + ship locale — **do not** put fa/de in `__()` |
| Non-English inside gettext | Violates §1.0 | Rewrite to **English**; add real translation file |
| Security flags `_e` | Unescaped HTML | `esc_html_e` / `esc_attr_e` |
| Broken word order after translate | Concatenated fragments | One English sentence + numbered placeholders |
| Agent/client breaks on errors | Translated error **code** | Literal code + translated message |
| Strings missing from `.pot` | Variable domain or dynamic template | Literal domain + static English templates |

## Escalation

- Upstream: [I18n Common APIs](https://developer.wordpress.org/apis/internationalization/), [Guidelines](https://developer.wordpress.org/apis/internationalization/internationalization-guidelines/), [Plugin i18n](https://developer.wordpress.org/plugins/internationalization/how-to-internationalize-your-plugin/), [Theme i18n](https://developer.wordpress.org/themes/functionality/internationalization/).
- JS: `@wordpress/i18n` / `wp.i18n` + `wp_set_script_translations`.
- Ask if the project intentionally uses multiple text domains (rare; must be documented).

## Sources

| Topic | Origin |
|-------|--------|
| **English-only gettext sources** | Team house rule (§1.0) |
| Domain, Domain Path, load_*, placeholders, plurals, escape helpers | WordPress Plugin / Theme / Common API handbooks |
| Pattern i18n + escape | `skills/wp-patterns/` |
| Error message vs code | `skills/wp-abilities-api/references/error-code-vocabulary.md` |
| Interactivity labels | `skills/wp-interactivity-api/references/server-side-rendering.md` |
| Header detection | `skills/wp-plugin-development/scripts/detect_plugins.mjs` |
| Sanitize / escape baseline | `skills/wp-plugin-development/references/security.md` |
