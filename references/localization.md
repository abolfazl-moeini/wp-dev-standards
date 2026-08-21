# Localization (i18n) — full house rules

Depth for **§1 Localization** in `SKILL.md`.

Extracted and adapted from:

- [Internationalization (Common APIs)](https://developer.wordpress.org/apis/internationalization/)
- [Internationalization Guidelines](https://developer.wordpress.org/apis/internationalization/internationalization-guidelines/)
- [How to Internationalize Your Plugin](https://developer.wordpress.org/plugins/internationalization/how-to-internationalize-your-plugin/)
- [Theme Internationalization](https://developer.wordpress.org/themes/functionality/internationalization/)
- [wp i18n (WP-CLI)](https://developer.wordpress.org/cli/commands/i18n/)
- [Block Editor internationalization](https://developer.wordpress.org/block-editor/how-to-guides/internationalization/)
- [`@wordpress/i18n`](https://developer.wordpress.org/block-editor/reference-guides/packages/packages-i18n/)
- In-repo: `wp-patterns`, `wp-abilities-api`, `wp-interactivity-api`, `wp-plugin-development`

---

## 0. Source language is always English (CRITICAL — house rule)

> **The text passed into translation functions (PHP `__()`, `_e()`, `esc_html_e()`, `_n()`, `_x()`, and siblings; JS `__()`, `_x()`, `_n()`, `_nx()`) MUST be English.**

> **The plugin or theme may target any market or UI language. That does not change the source. Write English inside gettext; apply the target language later with catalogs (`.po` → `.mo` for PHP, `.po` → `.json` for JS; language packs / translate.wordpress.org).**

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

If the user asks for a “Persian plugin,” still write **English** inside translation functions and ship a `fa_IR` catalog via §11 — do **not** put Persian inside `__()`.

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

**WordPress 6.7+ JIT (this skill targets 7.0+):** `load_plugin_textdomain()` only registers the path; files load just-in-time. Call it on `init` (priority `0` if widgets or other `init` priority-1 code uses gettext). **Do not** hook `plugins_loaded` — that triggers `_load_textdomain_just_in_time was called incorrectly`.

### 3.2 Theme — `load_theme_textdomain`

```php
add_action( 'init', function () {
    load_theme_textdomain( 'my-theme', get_template_directory() . '/languages' );
} );
```

Child themes: `load_child_theme_textdomain()` when the child owns its own strings. Same hook rule as plugins: `init` or later (WP 6.7+). `after_setup_theme` is too early if any gettext runs before `init`.

**MO naming**

| Location | Filename pattern |
|----------|------------------|
| Plugin `languages/` | `{text-domain}-{locale}.mo` (e.g. `my-plugin-fa_IR.mo`) |
| Theme `languages/` | `{locale}.mo` (e.g. `fa_IR.mo`) — **not** `{text-domain}-{locale}.mo` |
| `wp-content/languages/plugins/` | `{text-domain}-{locale}.mo` |
| `wp-content/languages/themes/` | `{text-domain}-{locale}.mo` |

### 3.3 JavaScript (WP 5.0+)

JS uses the **same English sources and the same PO** as PHP. Runtime files for JS are Jed JSON produced from that PO (`wp i18n make-json`, §11) — JSON is not generated from JS source.

**1. Register with `wp-i18n`**

```php
wp_register_script(
    'my-plugin-script',
    plugins_url( 'js/admin.js', __FILE__ ),
    array( 'wp-i18n' ),
    '1.0.0',
    true
);
```

Bundled/block code may `import { __, _x, _n, _nx, sprintf } from '@wordpress/i18n'` instead of the `wp.i18n` global. The script still needs the `wp-i18n` dependency (usually via `*.asset.php`).

**2. Call `wp_set_script_translations` after the script is registered**

```php
add_action( 'wp_enqueue_scripts', 'my_plugin_enqueue', 10 );
add_action( 'wp_enqueue_scripts', 'my_plugin_script_i18n', 20 );

function my_plugin_script_i18n() {
    wp_set_script_translations(
        'my-plugin-script',
        'my-plugin',
        plugin_dir_path( __FILE__ ) . 'languages'
    );
}
```

- Argument 1: the **registered handle** (auto-registered block scripts are often `{plugin-slug}-editor-script`).
- Argument 2: text domain (same literal as PHP).
- Argument 3: **filesystem** directory (`plugin_dir_path()`, trailing slash included). Required off wordpress.org. Do not pass `plugin_basename()`.
- wordpress.org language packs can supply JSON automatically; a third-arg custom path is for shipping your own files. Packs in `WP_LANG_DIR` still take priority when present.

**3. Wrap JS strings (English only)**

Official JS API — these mirror PHP. **There is no `_e`, `_ex`, or `esc_*` in `@wordpress/i18n` / `wp.i18n`.** Return the string; the view layer prints it.

| Need | JS |
|------|-----|
| Translate | `__( 'Save settings', 'my-plugin' )` |
| Context | `_x( 'Post', 'noun', 'my-plugin' )` |
| Plural | `_n( '%s item', '%s items', count, 'my-plugin' )` |
| Plural + context | `_nx( … )` |
| Placeholders | `sprintf( __( 'Hello %s', 'my-plugin' ), name )` |

```javascript
import { __, _x, _n, _nx, sprintf } from '@wordpress/i18n';

__( 'Hello World', 'my-plugin' );
_x( 'My block name', 'block title', 'my-plugin' );
sprintf( _n( '%s comment', '%s comments', n, 'my-plugin' ), n );
```

Classic (non-module) scripts:

```javascript
const { __, _x, _n, _nx, sprintf } = wp.i18n;
```

**4. JSON file names**

WordPress loads, in the directory from argument 3 (or the language pack dir):

- `{domain}-{locale}-{md5}.json` — `md5` of the **relative JS source path** (what `make-json` writes)
- or `{domain}-{locale}-{handle}.json` — if you rename to the script handle

One JSON file per JS source that had strings in the PO.

**5. Built bundles**

`make-pot` records source paths (`src/index.js`). The browser loads `build/index.js`. If those differ, pass a map:

```bash
wp i18n make-json languages --no-purge --use-map='{"src/index.js":"build/index.js"}'
```

**Do not** put UI strings in `wp_localize_script()`. That helper is only for non-translation data (REST URL, nonce, bootstrap config).

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
- [ ] JS: `wp-i18n` + `wp_set_script_translations` + English sources; no `_e` in JS
- [ ] Error codes not passed through `__()`
- [ ] If a locale was requested: POT + PO (`msgstr` filled) + MO; JSON if JS strings exist (§11)

---

## 10. Failure modes & Anti-patterns

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Site stays English | Domain / path / load mismatch or missing catalog | Align slug, header, `load_*`, MO names; run §11 |
| “Persian plugin” still shows English | Missing `fa_IR` catalog | Run §11; do **not** replace sources with Persian |
| JS still English | No JSON, wrong handle/path, missing `wp-i18n` dep | `make-json --no-purge` + `wp_set_script_translations` after register |
| `make-json` empty / wrong file | Source path ≠ runtime path | `--use-map`; check `#:` refs in the PO |
| Theme MO not loaded | Named `{domain}-{locale}.mo` in the theme dir | Rename to `{locale}.mo` |
| JIT notice / `_load_textdomain_just_in_time` | Calling `__()` / `_e()` or loading textdomain before `init` | Defer gettext until `did_action('init')`; hook `load_plugin_textdomain` on `init` priority 0 |
| Stale untranslated strings after `init` | Defaults cached/memoized during `plugins_loaded` or boot | Cache only after `init` (`did_action('init')`) and flush cache on `init` |
| Strings never appear in `.pot` | Variable domain or dynamic string build | Literal domain + static English templates |
| Broken word order in locale | Concatenated fragments | One sentence + numbered placeholders |
| Security flag on `_e` | Unescaped HTML output | `esc_html_e` / `esc_attr_e` |

### 10.1 Deep Dive: Preventing `_load_textdomain_just_in_time` Notices (WP 6.7+)

In WordPress 6.7+, translation loading was reworked into a Just-In-Time (JIT) architecture. If code triggers translation lookups for a textdomain before WordPress reaches the `init` action hook:
1. `_load_textdomain_just_in_time( $domain )` fails normalization and fires a `_doing_it_wrong` Notice.
2. The domain is marked loaded early, preventing proper locale catalog resolution.

#### Key Anti-Patterns to Avoid:
1. **Calling `__()` in default array values or option schema generators executed on `plugins_loaded`**:
   ```php
   // ANTI-PATTERN: Called during plugin/module registration
   public static function default_options(): array {
       return array(
           'status' => __( 'In progress', 'my-plugin' ), // JIT NOTICE TRIGGER!
       );
   }

   // CLEAN PATTERN: Guard gettext with did_action('init')
   public static function default_options(): array {
       return array(
           'status' => did_action( 'init' ) ? __( 'In progress', 'my-plugin' ) : 'In progress',
       );
   }
   ```

2. **Memoizing/caching untranslated defaults before `init`**:
   If a helper computes and caches defaults statically during `plugins_loaded` (e.g. for database migration or enum definition), the English string is cached in memory. Subsequent calls *after* `init` will return the cached English string instead of the translated locale string.
   **Solution**:
   - Only cache into `static $cache` when `did_action('init')` is truthy.
   - Hook a cache-flushing callback on `init` (e.g. `add_action( 'init', [MyOptions::class, 'flush_cache'], 1 );`).

3. **Passing translated strings into settings link helpers on `plugins_loaded`**:
   ```php
   // ANTI-PATTERN:
   add_action( 'plugins_loaded', function() {
       wpdev_register_plugin_settings_link( __FILE__, $url, __( 'Settings', 'my-plugin' ) );
   } );

   // CLEAN PATTERN: Omit 3rd argument; framework translates on plugins.php after init:
   add_action( 'plugins_loaded', function() {
       wpdev_register_plugin_settings_link( __FILE__, $url );
   } );
   ```

4. **Hooking `load_plugin_textdomain()` to `plugins_loaded`**:
   Hook `load_plugin_textdomain()` to `init` with priority `0`. Do NOT hook `plugins_loaded`.

5. **Static property initializations or class constructors executed at file load time**:
   Never run `__()` at the file top-level or in static property initializers. Initialize lazily inside methods called after `init`.

---

## 11. Catalog pipeline (ship a locale)

Wrapping strings is i18n. Shipping a locale is this pipeline. There is **no** `wp i18n make-po`.

**When to run:** the user asks to translate, add a locale (`fa_IR`, `de_DE`, …), or ship a non-English UI. Do not invent a catalog on every new string. If `languages/*.po` already exists and they asked to refresh translations, re-run from `make-pot`.

PO is the **single source of truth** for both PHP and JS. `make-mo` feeds PHP; `make-json` extracts JS strings from the same PO.

### 11.1 Commands

| Command | Role |
|---------|------|
| `wp i18n make-pot` | Scan PHP, JS, Blade, `block.json`, `theme.json` → POT (English `msgid`, empty `msgstr`) |
| `wp i18n update-po` | Merge a POT into existing PO files (like `msgmerge`) |
| `wp i18n make-mo` | PO → binary MO (PHP runtime). **Required.** Do not hand-edit `.mo` |
| `wp i18n make-json` | Extract JS strings from a **translated PO** → Jed 1.x JSON (one file per JS source) |
| `wp i18n make-php` | Optional PO → `.l10n.php` (WP 6.5+). Not a substitute for MO |

### 11.2 File names

| File | Pattern | Who consumes it |
|------|---------|-----------------|
| POT | `{domain}.pot` | Extractors / `update-po` |
| PO | `{domain}-{locale}.po` (plugin); theme copies often `{locale}.po` | Humans / this agent |
| MO | Plugin: `{domain}-{locale}.mo`. Theme dir: `{locale}.mo` | PHP (`load_*_textdomain`) |
| JSON | `{domain}-{locale}-{md5}.json` or `{domain}-{locale}-{handle}.json` | JS (`wp_set_script_translations`) |
| PHP (optional) | `{domain}-{locale}.l10n.php` | PHP 6.5+ loader |

### 11.3 Agent steps

1. Resolve **slug = text domain** and the **target locale**. Ask once if the locale is missing.
2. Ensure `languages/` exists and the header has `Domain Path: /languages`.
3. From the plugin/theme root:

   ```bash
   wp i18n make-pot . languages/my-plugin.pot --domain=my-plugin
   ```

4. Create or update the PO:
   - **New locale:** copy the POT to `languages/my-plugin-fa_IR.po`. Set header `Language: fa_IR` and a correct `Plural-Forms` for that locale (e.g. `fa_IR`: `nplurals=2; plural=(n > 1);` — `de_DE`: `nplurals=2; plural=(n != 1);`).
   - **Existing PO files:**

     ```bash
     wp i18n update-po languages/my-plugin.pot languages
     ```

5. **Fill `msgstr`.** Translate every empty or `fuzzy` entry into the target language. Leave `msgid` in English. Do not write that language back into PHP/JS. Do not invent extra `msgid`s.
6. Build PHP catalogs:

   ```bash
   wp i18n make-mo languages
   ```

7. If the PO has `#:` references to `.js` / `.jsx` / `.ts` / `.tsx` (or you registered a JS handle):

   ```bash
   wp i18n make-json languages --no-purge
   ```

   `--no-purge` keeps JS strings in the PO so later `update-po` still works. Add `--use-map` when the built file path differs from the source path recorded in the PO.
8. Optional only: `wp i18n make-php languages`. House default is still MO.
9. Wire loaders (§3): `load_plugin_textdomain` / `load_theme_textdomain` on `init`; `wp_set_script_translations` when JS exists.

Commit `.pot` and `.po` when the project tracks them. Commit generated `.mo` / `.json` when the project ships catalogs (off-directory plugins always should).

### 11.4 Flow

```
PHP + JS English gettext
        │
        ▼
wp i18n make-pot . languages/{domain}.pot
        │
        ▼
copy POT → {domain}-{locale}.po   or   wp i18n update-po
        │
        ▼
agent fills msgstr (target language; msgid stays English)
        │
        ├─ PHP ─► wp i18n make-mo languages
        │         load_plugin_textdomain / load_theme_textdomain
        │
        └─ JS ──► wp i18n make-json languages --no-purge
                  wp_set_script_translations( $handle, $domain, $path )
```

---

## 12. Upstream references

- https://developer.wordpress.org/apis/internationalization/
- https://developer.wordpress.org/apis/internationalization/internationalization-guidelines/
- https://developer.wordpress.org/plugins/internationalization/how-to-internationalize-your-plugin/
- https://developer.wordpress.org/themes/functionality/internationalization/
- https://developer.wordpress.org/cli/commands/i18n/
- https://developer.wordpress.org/cli/commands/i18n/make-pot/
- https://developer.wordpress.org/cli/commands/i18n/update-po/
- https://developer.wordpress.org/cli/commands/i18n/make-mo/
- https://developer.wordpress.org/cli/commands/i18n/make-json/
- https://developer.wordpress.org/reference/functions/load_plugin_textdomain/
- https://developer.wordpress.org/reference/functions/load_theme_textdomain/
- https://developer.wordpress.org/reference/functions/wp_set_script_translations/
- https://developer.wordpress.org/block-editor/how-to-guides/internationalization/
- https://developer.wordpress.org/block-editor/reference-guides/packages/packages-i18n/
- https://make.wordpress.org/polyglots/handbook/plugin-theme-authors-guide/
