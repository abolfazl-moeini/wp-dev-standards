# Localization (i18n) — full house rules

Depth for **§1 Localization** in `SKILL.md`.

Extracted and adapted from:

- [Internationalization (Common APIs)](https://developer.wordpress.org/apis/internationalization/)
- [Internationalization Guidelines](https://developer.wordpress.org/apis/internationalization/internationalization-guidelines/)
- [How to Internationalize Your Plugin](https://developer.wordpress.org/plugins/internationalization/how-to-internationalize-your-plugin/)
- [Theme Internationalization](https://developer.wordpress.org/themes/functionality/internationalization/)
- In-repo: `wp-patterns`, `wp-abilities-api`, `wp-interactivity-api`, `wp-plugin-development`

---

## 0. Source language is always English (CRITICAL — house rule)

> **The text passed into translation functions (`__()`, `_e()`, `esc_html_e()`, `_n()`, `_x()`, `esc_attr_e()`, and every sibling in PHP or JS) MUST be English.**

> **The plugin or theme may target any market or UI language. That does not change the source. Write English inside gettext; apply the target language later with translation files (`.po` / `.mo` / language packs / translate.wordpress.org).**

| Situation | What goes in `__()` / `esc_html_e()` | What the user sees in fa_IR / de_DE / … |
|-----------|--------------------------------------|------------------------------------------|
| Plugin for Iranian sites | English source strings | Persian from `.po` / language pack |
| Plugin for German shops | English source strings | German from translations |
| Internal admin-only tool | English source strings | Still translated via catalogs if needed |

```php
// CORRECT
esc_html_e( 'Save settings', 'my-plugin' );

// WRONG — never use target-language literals as gettext source
esc_html_e( 'ذخیره تنظیمات', 'my-plugin' );
esc_html_e( 'Einstellungen speichern', 'my-plugin' );
```

**Why this is mandatory**

- WordPress gettext tooling, GlotPress, and translate.wordpress.org assume a single source language (English) and translator-supplied locales.
- Non-English sources break shared catalogs, reuse, and standard contributor workflows.
- English sources + proper `.po` is the supported path for “plugin UI speaks Persian (or any locale).”

If the user asks for a “Persian plugin,” still write **English** inside translation functions and, if needed, add a `fa_IR` catalog — do **not** put Persian inside `__()`.

---

## 1. Concepts

| Term | Meaning |
|------|---------|
| **i18n** | Internationalization — code shape so strings *can* be translated |
| **l10n** | Localization — actual translations for a locale |
| **Text domain** | Unique ID for this plugin/theme’s strings (**= slug**) |
| **Domain Path** | Header path to translation files (e.g. `/languages`, leading slash) |
| **Gettext** | WordPress wrappers: `__()`, `_e()`, `_n()`, `_x()`, … |

Rules of thumb:

- **English source only** (see §0).
- **Same text domain** in header, PHP, and JS.
- **Domain argument is always a string literal** (never a variable).
- **Do not concatenate** sentence fragments; use `sprintf` / `printf` placeholders.
- **Escape for context** when printing (HTML vs attribute vs URL).

---

## 2. Text domain and Domain Path

### 2.1 Text domain rules

- Must match the plugin/theme **slug**.
- **Lowercase**, **dashes not underscores**, **no spaces** (`my-plugin`, not `my_plugin`).
- On wordpress.org: slug of `wordpress.org/plugins/<slug>` or `wordpress.org/themes/<slug>`.
- Declare in plugin header / theme `style.css` so metadata can be translated when inactive.
- Pass the same domain on every gettext call.
- Even strings that exist in core (e.g. `Settings`) need **your** domain — core wording can change.

```php
// WRONG — tools cannot extract a variable domain
__( 'Translate me.', $text_domain );
```

### 2.2 Domain Path

- Location of translations (including when the plugin is disabled).
- Files under `languages/` → `Domain Path: /languages` (**leading slash required**).
- Keep it when shipping local `.mo` files; may be optional for some wordpress.org projects.

### 2.3 Plugin header

```php
/**
 * Plugin Name: My Plugin
 * Text Domain: my-plugin
 * Domain Path: /languages
 */
```

### 2.4 Theme header (`style.css`)

```css
/*
Theme Name: My Theme
Text Domain: my-theme
Domain Path: /languages
*/
```

---

## 3. Loading translations

### 3.1 Plugin — `load_plugin_textdomain`

```php
add_action( 'init', 'my_plugin_load_textdomain' );

function my_plugin_load_textdomain() {
    load_plugin_textdomain(
        'my-plugin',
        false,
        dirname( plugin_basename( __FILE__ ) ) . '/languages'
    );
}
```

- Argument 1: text domain (matches header).
- Argument 3: path **relative to the plugins directory**, not an absolute filesystem path.

**WordPress 4.6+ / wordpress.org:** language packs from translate.wordpress.org take priority. Directory plugins may omit an explicit load if they rely only on those packs and set `Requires at least:` to 4.6+. Off-directory plugins, custom `.mo` files, or early strings still need `load_plugin_textdomain` or a `load_textdomain_mofile` filter.

### 3.2 Theme — `load_theme_textdomain`

```php
add_action( 'after_setup_theme', function () {
    load_theme_textdomain( 'my-theme', get_template_directory() . '/languages' );
} );
```

Child themes: `load_child_theme_textdomain()` when the child owns its own strings.

**MO naming**

| Location | Filename pattern |
|----------|------------------|
| Plugin `languages/` | `{text-domain}-{locale}.mo` (e.g. `my-plugin-fa_IR.mo`) |
| Theme `languages/` | Prefer `{text-domain}-{locale}.mo`; some themes still use `{locale}.mo` |
| `wp-content/languages/plugins/` | `{text-domain}-{locale}.mo` |
| `wp-content/languages/themes/` | `{text-domain}-{locale}.mo` |

### 3.3 JavaScript (WP 5.0+)

1. Register/enqueue the script with dependency `wp-i18n`.
2. Call `wp_set_script_translations( $handle, $domain, $path_optional )`.
3. In JS (English sources only):

```javascript
const { __, _x, _n, sprintf } = wp.i18n;

__( 'Hello World', 'my-plugin' );
_x( 'My block name', 'block title', 'my-plugin' );
```

Path argument to `wp_set_script_translations` is required when not hosted on wordpress.org.

Prefer `wp-i18n` over legacy `wp_localize_script()` bags for new UI strings. `wp_localize_script` remains fine for non-translation data (REST URL, nonce).

---

## 4. Core translation functions

### 4.1 Basic

| Function | Echo? | Escape? | Use |
|----------|-------|---------|-----|
| `__( $text, $domain )` | No | No | Return value; arrays; API args |
| `_e( $text, $domain )` | Yes | No | Prefer escaped helpers for HTML |
| `_x( $text, $context, $domain )` | No | No | Same word, different meaning |
| `_ex( $text, $context, $domain )` | Yes | No | Echo + context |

`$text` is always **English**.

### 4.2 Translate + escape (preferred in markup)

| Function | Role |
|----------|------|
| `esc_html__` / `esc_html_e` / `esc_html_x` | HTML body text |
| `esc_attr__` / `esc_attr_e` / `esc_attr_x` | HTML attributes |

Escaping limits damage if a translation catalog is compromised (handbook recommendation).

### 4.3 Plurals

```php
printf(
    esc_html(
        _n(
            '%s comment',
            '%s comments',
            $comment_count,
            'my-plugin'
        )
    ),
    number_format_i18n( $comment_count )
);
```

- Singular form is still a template (`'%s item'`), **not** `'One item'`, because other languages use singular for counts other than 1.
- `$count` selects the form and is usually printed into the string.
- Prefer `_n()` over English-only `if ( 1 === $count )` for normal plurals.

**Deferred plurals:** `_n_noop()` / `_nx_noop()`, then `translate_nooped_plural()` when the count is known.

### 4.4 Context (disambiguation)

```php
_x( 'Post', 'noun', 'my-plugin' );
_x( 'Post', 'verb', 'my-plugin' );
```

### 4.5 Date and number helpers

- `number_format_i18n( $number )` — locale-aware numbers.
- `wp_date( $format, $timestamp )` — preferred locale-aware dates (WP 5.3+; this skill targets WP 7.0+).
- `date_i18n()` — legacy; avoid in new code.

If the **format string** itself is user-facing (e.g. `g:i:s a`), wrap it in gettext with a translators comment.

---

## 5. Variables, placeholders, translator comments

### 5.1 Never put variables inside the gettext string

```php
// WRONG — extractors see "$city"; runtime sees "London"; translation misses
_e( "Your city is $city.", 'my-plugin' );

// CORRECT — English template + runtime placeholder
printf(
    /* translators: %s: Name of a city. */
    esc_html__( 'Your city is %s.', 'my-plugin' ),
    esc_html( $city )
);
```

### 5.2 Multiple placeholders — numbered + single quotes

```php
printf(
    /* translators: 1: Name of a city 2: ZIP code. */
    __( 'Your city is %1$s, and your zip code is %2$s.', 'my-plugin' ),
    $city,
    $zipcode
);
```

Translators can reorder `%1$s` / `%2$s`. Prefer single-quoted PHP strings so `$s` is not interpolated.

### 5.3 Translator comments

- Must start with `translators:` (lowercase).
- Must be the **last PHP comment immediately before** the gettext call.

```php
/* translators: draft saved date format, see https://www.php.net/date */
$saved_date_format = __( 'g:i:s a', 'my-plugin' );
```

---

## 6. Best practices for writing strings (handbook)

All source strings in **English**, then:

1. **Decent English** — minimize slang and obscure abbreviations.
2. **Entire sentences** — word order differs by language.
3. **Split at paragraphs** — related sentences OK; not a whole page in one string.
4. **No leading/trailing whitespace** inside the gettext phrase.
5. **Assume length can double** after translation (layout).
6. **Avoid wrapping tags** in the string when possible.
7. **No unnecessary HTML** inside translated strings.
8. **Don’t put URLs in gettext** unless a localized URL is intentional.
9. **Placeholders for dynamic values** — position may change per language.
10. **Phrases, not word glue** — one sentence with `%s`, not concatenation of fragments.
11. **Reuse identical strings** — avoid near-duplicates (`Posts` vs `Posts:`) when one form works.
12. **No empty string** in gettext (reserved). If truly needed, add context.
13. **Avoid `\r` (CR)** in translatable strings; use `\n` if a newline is required.

---

## 7. Domain-specific patterns (sibling skills)

### 7.1 Block patterns (`wp-patterns`)

- Every user-visible string → gettext + domain; **English source**.
- Prefer `esc_html_e` / `esc_html__` / `esc_attr_e` / `esc_attr__`.
- Manual registration: translate `title` and `description` with `__()`.
- Anti-patterns: bare `_e()`; raw HTML text; non-English gettext source.

### 7.2 Abilities / `WP_Error` (`wp-abilities-api`)

- **Messages:** `__()` / `sprintf( __( … ) )` with English source.
- **Codes:** never translate (`my_plugin_missing_order_id` stays literal).
- Ability `label` / `description`: English gettext sources.

### 7.3 Interactivity API (`wp-interactivity-api`)

```php
wp_interactivity_state( 'myPlugin', array(
    'labels' => array(
        'add'    => __( 'Add Item', 'my-plugin' ),
        'remove' => __( 'Remove Item', 'my-plugin' ),
        'empty'  => __( 'No items found', 'my-plugin' ),
    ),
) );
```

Always include the text domain; English sources only.

---

## 8. What not to put through gettext

| Do not gettext | Why |
|----------------|-----|
| Option keys, CPT/taxonomy slugs | Machine IDs |
| REST namespaces/routes | API contract |
| CSS classes, block names | Code |
| `WP_Error` codes | Client/agent matching |
| File paths, nonces, capability names | Not prose |
| Empty string | Gettext reserved |
| Target-language UI as the *source* | Violates §0 — use English source + `.po` |

Hardcoding non-English UI **without** gettext is also wrong for shippable plugins; use English gettext + translations.

---

## 9. Review checklist

- [ ] **All gettext sources are English**
- [ ] Text Domain + Domain Path headers correct (domain = slug, dashes)
- [ ] Domain is a string literal on every call
- [ ] `load_plugin_textdomain` / `load_theme_textdomain` when required
- [ ] HTML → `esc_html_*`; attributes → `esc_attr_*`
- [ ] Plurals → `_n` / `_nx`; context → `_x` / `_ex`
- [ ] Placeholders + `/* translators: … */` comments
- [ ] No sentence concatenation; no variables inside gettext templates
- [ ] No unnecessary HTML/URLs/empty strings in gettext
- [ ] JS: `wp-i18n` + `wp_set_script_translations` + English sources
- [ ] Error codes not passed through `__()`

---

## 10. Failure modes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Site stays English | Domain / path / load mismatch or missing catalog | Align slug, header, `load_*`, MO names; ship locale |
| “Persian plugin” still shows English | Missing `fa_IR` catalog | Add translations; do **not** replace sources with Persian |
| Strings never appear in `.pot` | Variable domain or dynamic string build | Literal domain + static English templates |
| Broken word order in locale | Concatenated fragments | One sentence + numbered placeholders |
| Security flag on `_e` | Unescaped HTML output | `esc_html_e` / `esc_attr_e` |

---

## 11. Catalog generation (optional)

```bash
# From plugin root (WP-CLI)
wp i18n make-pot . languages/my-plugin.pot --domain=my-plugin
```

Do not hand-edit `.mo` files. Commit `.pot` when the project tracks it.

---

## 12. Upstream references

- https://developer.wordpress.org/apis/internationalization/
- https://developer.wordpress.org/apis/internationalization/internationalization-guidelines/
- https://developer.wordpress.org/plugins/internationalization/how-to-internationalize-your-plugin/
- https://developer.wordpress.org/themes/functionality/internationalization/
- https://developer.wordpress.org/block-editor/reference-guides/packages/packages-i18n/
- https://make.wordpress.org/polyglots/handbook/plugin-theme-authors-guide/
