---
name: wp-dev-standards
description: "House rules for WordPress plugin/theme development on this team. Use for Localization/i18n (always English gettext sources; __(), _e(), esc_html_e(), text domains, Domain Path), shipping locales via wp i18n (make-pot, update-po, make-mo, make-json), JS wp.i18n / @wordpress/i18n, string escaping, and team coding conventions. Prefer this skill over domain skills when deciding how to write or review user-visible strings."
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
- User asks to **translate**, add a locale, or ship a non-English UI (catalog pipeline, §1.6).
- User asks for “house rules”, “our WordPress standards”, localization, or i18n.

## Inputs required

- Plugin/theme **slug** and **text domain** (must match; lowercase; dashes, not underscores).
- Target surface: plugin, theme, block, pattern, ability/REST, or JS script handle.
- Target **display** locale (e.g. `fa_IR`) only when generating translation catalogs — **never** as the language of gettext source strings.

---

## 1) Localization (i18n) — always first

Full depth: **`references/localization.md`**.

### 1.0 Source language is always English (CRITICAL)

> **The text inside translation functions (PHP `__()`, `_e()`, `esc_html_e()`, `_n()`, `_x()`, and siblings; JS `__()`, `_x()`, `_n()`, `_nx()`) MUST be written in English.**

> **No matter what language the plugin, theme, market, client, or UI targets (Persian, Arabic, German, …): source strings in code are English. The target language is applied later via translation catalogs (`.po` → `.mo` for PHP, `.po` → `.json` for JS; language packs / translate.wordpress.org) — never by putting non-English literals inside gettext calls.**

```php
// CORRECT — English source; locale comes from translations
esc_html_e( 'Save settings', 'my-plugin' );

// WRONG — never put the target language in the gettext string
esc_html_e( 'ذخیره تنظیمات', 'my-plugin' );
esc_html_e( 'Einstellungen speichern', 'my-plugin' );
```

**Hard house rule.** Do not “localize” by writing Farsi/Arabic/etc. inside `__()` / `_e()` / `esc_*`. Write decent English; ship a locale via §1.6.

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

- Themes: `Text Domain` in `style.css`; `load_theme_textdomain( 'theme-slug', get_template_directory() . '/languages' )` on **`init`** (WP 6.7+ JIT; not `plugins_loaded`, and not `after_setup_theme` if gettext runs before `init`).
- Plugin `languages/`: `{text-domain}-{locale}.mo` (e.g. `my-plugin-fa_IR.mo`). Theme `languages/`: `{locale}.mo` (e.g. `fa_IR.mo`). Global `wp-content/languages/{plugins,themes}/` uses `{text-domain}-{locale}.mo`.
- wordpress.org language packs take priority since WP 4.6+; still declare domain/path correctly. Off-directory or custom `.mo` still needs `load_*_textdomain` (or a `load_textdomain_mofile` filter).

### 1.4.1 Anti-Patterns: Early Translation & `_load_textdomain_just_in_time`

WordPress 6.7+ triggers `Notice: Function _load_textdomain_just_in_time was called incorrectly. Translation loading for the <domain> domain was triggered too early` whenever `__()`, `_e()`, `_x()`, `esc_*`, etc. are executed **before the `init` action**.

| Anti-Pattern (WRONG) | Why it breaks | Recommended Pattern (CORRECT) |
|---|---|---|
| Calling `__()` in default arrays/options before `init` (e.g. during `plugins_loaded`, database setup, or enum registration). | Fires before `init` → triggers `_load_textdomain_just_in_time` JIT notice. | Guard with `did_action('init')`: `did_action('init') ? __('Label', 'slug') : 'Label'`. |
| Memoizing/caching untranslated defaults before `init`. | Statically cached before `init` → requests after `init` permanently receive untranslated English strings. | Only write to cache if `did_action('init')` is true, and flush the cache on `init` priority 0/1. |
| Passing translated label to `wpdev_register_plugin_settings_link()` on `plugins_loaded`. | `plugins_loaded` runs before `init` → early domain load. | Leave label empty: `wpdev_register_plugin_settings_link( $file, $url )`. The framework translates it on `plugins.php` after `init`. |
| Hooking `load_plugin_textdomain()` to `plugins_loaded` or at file root. | Violates WP 6.7+ JIT lifecycle. | Hook `load_plugin_textdomain()` to `init` at priority `0`. |
| Class properties initialized with gettext expressions at class load. | Executes at PHP parse/require time before WP boots. | Initialize inside a method called on/after `init` or compute lazily. |

```php
// WRONG: Called during plugins_loaded / module boot
public static function default_statuses(): array {
    return [
        ['slug' => 'in_progress', 'label' => __( 'In progress', 'my-plugin' )],
    ];
}

// CORRECT: Defers gettext until init has fired
public static function default_statuses(): array {
    return [
        ['slug' => 'in_progress', 'label' => did_action( 'init' ) ? __( 'In progress', 'my-plugin' ) : 'In progress'],
    ];
}
```

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
- [ ] JS uses `__` / `_x` / `_n` / `_nx` / `sprintf` only — **no `_e` in JS**
- [ ] If a target locale was requested: POT + PO (`msgstr` filled) + MO exist
- [ ] If JS UI exists and a locale was requested: `wp-i18n` dep + `wp_set_script_translations` + JSON from `make-json`

### 1.6 Catalog pipeline (when a locale is requested)

Full depth: `references/localization.md` §3.3 (JS) and §11 (commands).

Run this when the user asks to translate, add a locale, or ship a non-English UI. Skip on ordinary string edits unless `languages/*.po` already exists and they asked to refresh.

JS and PHP share **one PO**. JSON is **not** generated from JS source. There is **no** `wp i18n make-po`.

1. Resolve domain (= slug) and locale (ask once if missing).
2. Ensure `languages/` and `Domain Path: /languages`.
3. `wp i18n make-pot . languages/{domain}.pot --domain={domain}`
4. New locale: copy POT → `{domain}-{locale}.po`, set `Language:` and `Plural-Forms`. Existing: `wp i18n update-po languages/{domain}.pot languages`
5. **Agent fills `msgstr`** in the target language. `msgid` stays English. Do not write that language into PHP/JS.
6. `wp i18n make-mo languages`
7. If the PO references JS/TS files: `wp i18n make-json languages --no-purge` (add `--use-map` when build path ≠ source path). Then `wp_set_script_translations( $handle, $domain, plugin_dir_path( __FILE__ ) . 'languages' )` after the script is registered.

JS wrap (English sources): `import { __, _x, _n, _nx, sprintf } from '@wordpress/i18n'` or `const { __, _x, _n, _nx, sprintf } = wp.i18n`. Register the script with dependency `wp-i18n`.

Optional: `wp i18n make-php languages` (`.l10n.php`, WP 6.5+). House default remains MO.

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
4. If a target locale was requested, run **§1.6** (do not put that locale in gettext).
5. Run §1.5 checklist.
6. Route architecture/API work to the matching domain skill; keep i18n decisions here.

## Verification

- Grep: no non-English literals inside gettext / `wp.i18n` calls.
- Grep JS: no `_e(` / `_ex(` as i18n helpers.
- If a locale was requested: `{domain}.pot`, `{domain}-{locale}.po` with filled `msgstr`, and matching `.mo` exist; JSON exists when the PO has JS `#:` refs.
- Load that locale and confirm UI language changes **without** editing PHP/JS sources.
- Grep: no raw user-facing strings without gettext; no translated error codes.
- Patterns: English gettext sources; no bare `_e()` in markup.

## Failure modes

| Symptom | Likely cause | Fix |
|---------|----------------|-----|
| Always English on fa/de site | Domain/path/load or missing catalog | Align domain + load + run §1.6 — **do not** put fa/de in `__()` |
| Locale requested but only English gettext | Pipeline not run | §1.6: POT → PO → fill `msgstr` → MO (and JSON if JS) |
| JS still English | Missing JSON / handle / path / `wp-i18n` dep | `make-json --no-purge` + `wp_set_script_translations` after register |
| Non-English inside gettext | Violates §1.0 | Rewrite to **English**; add real translation file |
| Security flags `_e` | Unescaped HTML | `esc_html_e` / `esc_attr_e` |
| Broken word order after translate | Concatenated fragments | One English sentence + numbered placeholders |
| Agent/client breaks on errors | Translated error **code** | Literal code + translated message |
| Strings missing from `.pot` | Variable domain or dynamic template | Literal domain + static English templates |

## Escalation

- Upstream: [I18n Common APIs](https://developer.wordpress.org/apis/internationalization/), [Guidelines](https://developer.wordpress.org/apis/internationalization/internationalization-guidelines/), [Plugin i18n](https://developer.wordpress.org/plugins/internationalization/how-to-internationalize-your-plugin/), [Theme i18n](https://developer.wordpress.org/themes/functionality/internationalization/).
- Catalogs: [wp i18n](https://developer.wordpress.org/cli/commands/i18n/) (`make-pot`, `update-po`, `make-mo`, `make-json`).
- JS: [`@wordpress/i18n`](https://developer.wordpress.org/block-editor/reference-guides/packages/packages-i18n/), [Block Editor i18n](https://developer.wordpress.org/block-editor/how-to-guides/internationalization/), [`wp_set_script_translations`](https://developer.wordpress.org/reference/functions/wp_set_script_translations/).
- Ask if the project intentionally uses multiple text domains (rare; must be documented).

## Sources

| Topic | Origin |
|-------|--------|
| **English-only gettext sources** | Team house rule (§1.0) |
| Domain, Domain Path, load_*, placeholders, plurals, escape helpers | WordPress Plugin / Theme / Common API handbooks |
| POT / PO / MO / JSON pipeline | [wp i18n](https://developer.wordpress.org/cli/commands/i18n/), Block Editor i18n guide |
| JS `__` / `_x` / `_n` / `_nx` / `sprintf` (no `_e`) | `@wordpress/i18n`, Common APIs i18n |
| `wp_set_script_translations` | WP 5.0+ script translations |
| Pattern i18n + escape | `skills/wp-patterns/` |
| Error message vs code | `skills/wp-abilities-api/references/error-code-vocabulary.md` |
| Interactivity labels | `skills/wp-interactivity-api/references/server-side-rendering.md` |
| Header detection | `skills/wp-plugin-development/scripts/detect_plugins.mjs` |
| Sanitize / escape baseline | `skills/wp-plugin-development/references/security.md` |
