# Review Checklist

Each item is tagged with **[zip]** (check against the extracted package.zip), **[repo]** (check against the repository source), or **[both]**.

## 1. Metadata (manifest JSON) — check in [zip]

Inspect the manifest from the extracted `package.zip`. Fields to check:

- [ ] **`disabledInPublish`** (plugins): Must be `true` if:
  - Data stored via `saveData()` in petal may contain sensitive information, OR
  - The plugin uses kernel write APIs without publish-specific handling
- [ ] **`funding`**: Delete if empty (no sub-fields) or if `custom` is the template default `["https://ld246.com/sponsor"]`
- [ ] **`keywords`**: Must NOT contain `"siyuan"` (don't reference the Siyuan brand name)
- [ ] **Redundant locale fields**: Delete any locale field whose value is identical to `"default"`
- [ ] **No `i18n` field**: The `i18n` field is not part of the plugin.json schema. The framework auto-loads i18n files from the `i18n/` directory and exposes them as `this.i18n`. If an `i18n` field appears in plugin.json, it should be removed — it's not needed for `this.i18n` to work
- [ ] **`frontends`**: Set to `["all"]` only if the package actually supports all frontends (desktop, browser-desktop, mobile)
- [ ] **`backends`**: Set to `["all"]` only if the package actually supports all backends (windows, linux, darwin, docker, android, ios)
- [ ] **`name`**: Must match the repository name exactly
- [ ] **`url`**: Must be the correct GitHub repository URL (`https://github.com/owner/repo`)
- [ ] **`readme`**: The filenames listed must match actual README files present in the zip
- [ ] **`displayName`**: Must NOT contain the text `Siyuan` (don't reference the brand name). The `description` field is exempt from this rule. The locale language should match the corresponding `readme` entry: if a locale's README is Chinese-only, `displayName` for that locale being Chinese is acceptable. Only flag when a locale has its own README file in a different language but `displayName` doesn't match (e.g., `readme.en_US` points to an English README but `displayName.en_US` is Chinese).

## 2. Icon & Preview — check in [zip]

- [ ] **`icon.png`**: File size must be < 20KB (dimensions 160x160 are recommended but not enforced)
- [ ] **`preview.png`**: File size must be < 200KB (dimensions 1024x768 are recommended but not enforced)
- [ ] **Icon choice**: Must NOT use SiYuan built-in icons — should use a custom SVG icon to avoid confusion with built-in features
- [ ] **README**: Must NOT embed `preview.png` in README (Siyuan displays it separately in the marketplace)

## 3. README Files — check in [zip]

- [ ] **Relative links**: No relative-path links. Use `grep -nE '\[[^]]+\]\([^)]+\)' README*.md | grep -v 'https\?://' | grep -v 'mailto:'` to detect them. Common patterns that are issues:
  - `[text](file.md)` — cross-file links (e.g., `[English](README.md)`, `[中文](README_zh_CN.md)`)
  - `[text](./path/doc.md)` or `[text](../file)` — relative directory links
  - `![alt](./image.png)` or `![alt](image.png)` — relative image links
  - `[text](#anchor)` — same-page anchor links
  - Any link whose destination does not start with `https://` or `http://` is an issue — only `mailto:` is exempt
  - Fixed links should use absolute URLs (GitHub raw, CDN, or stable image hosting)
- [ ] **Language consistency**: If separate language files exist (e.g., `README.md` + `README_zh_CN.md`), the content of each must match the filename's language
- [ ] **Filenames**: Must match the `readme` field in the manifest JSON
- [ ] **No preview**: Preview image should not be embedded in README content

## 4. LICENSE — check in [repo]

- [ ] **Exists**: Repository must contain a LICENSE file
- [ ] **Year**: Copyright year must be current/updated
- [ ] **Copyright holder**: Must be the actual author (not a template placeholder)
- [ ] **License type**: Must be an appropriate open source license

## 5. Code Quality (plugins) — check in [repo]

- [ ] **Config storage**: Use `saveData()` / `loadData()` for plugin configuration, unless the file genuinely needs to be stored outside the petal directory
- [ ] **Uninstall cleanup**: Must call `removeData()` in the `uninstall()` method — NOT in `onunload()`. The lifecycle is: disable/reload → only `onunload()` fires; full uninstall → `onunload()` fires first, then `uninstall()` fires. Putting `removeData()` in `onunload()` would delete config on every disable, losing user settings.
- [ ] **No save on unload/install**: Must NOT save data during `onunload` or `uninstall` (causes sync conflicts)
- [ ] **Lifecycle understanding**: `onunload()` is for releasing runtime resources (event bus listeners, timers, observers, global event listeners, network connections). `uninstall()` is for persistent cleanup (removing stored data). The framework auto-removes docks, tabs, top bar icons, status bar items, SVG icons, CSS, and registered commands — the plugin only needs to clean up what it manually created outside framework APIs. If the plugin confuses these responsibilities, flag it.
- [ ] **Constants**: Use named constants instead of hardcoded string literals
- [ ] **Avoid re-reading config**: Don't call `loadData()` (which internally calls `getFile`) repeatedly — read once, cache in a variable, write with `putFile` when needed
- [ ] **Logging**: Outside of lifecycle functions, only log on errors. `console.log` is acceptable within lifecycle functions (onload, onunload, etc.)
- [ ] **Batch saves**: Consecutive `saveConfig` calls should be merged into one
- [ ] **Write frequency**: Don't write files at high frequency during Siyuan's runtime (can cause sync issues)
- [ ] **Custom attributes**: Custom DOM attributes must use the `custom-` prefix (e.g., `custom-run-python-code`)
- [ ] **EventBus cleanup**: For every `this.eventBus.on()` registered, there must be a matching `this.eventBus.off()` in `onunload()`. The same function reference must be used — anonymous inline functions cannot be removed and are an issue.
- [ ] **Global event listener cleanup**: For every `addEventListener` on `window`/`document`/other globals, there must be a matching `removeEventListener` in `onunload()`
- [ ] **IPC cleanup**: If the plugin uses `require("electron")` to access `ipcRenderer`, every `ipcRenderer.on()` must have a matching `ipcRenderer.removeListener()` in `onunload()`
- [ ] **Timer cleanup**: Any `setTimeout`/`setInterval` calls must have corresponding `clearTimeout`/`clearInterval` in `onunload()`
- [ ] **Observer cleanup**: Any `MutationObserver`/`ResizeObserver`/`IntersectionObserver` instances must call `.disconnect()` in `onunload()`
- [ ] **Network cleanup**: Any WebSocket, EventSource, or persistent network connections must be closed in `onunload()`
- [ ] **Command registration**: Command keys must be in English; hotkey must use Siyuan format (e.g., `⌥⇧⌘A`), not raw key names like `Ctrl+Alt+C`. Commands registered via `addCommand()` are auto-cleaned by the framework — no manual cleanup needed in `onunload()`.
- [ ] **Template styles**: `index.scss` must not contain leftover template styles like `.plugin-sample` — only the plugin's own styles
- [ ] **No `window.location.reload()`**: Must NOT call `window.location.reload()` to reload the UI. Use `fetch('/api/ui/reloadUI')` instead. `window.location.reload()` causes a full browser reload and loses application state; `/api/ui/reloadUI` can automatically save SiYuan's layout information. Detect with `grep -rn 'location\.reload'` or `grep -rn 'window\.location'` on the source.
- [ ] **Path separators**: All file paths in code must use forward slash `/`, not backslash `\`

## 6. Package Zip — check in [zip]

- [ ] **Filename**: Must be exactly `package.zip`
- [ ] **Internal paths**: All file paths inside the zip must use forward slash `/`, not backslash `\`
- [ ] **No extra files**: Don't include unused i18n folders, node_modules, `.git`, leftover template code, or other unnecessary files
- [ ] **Build method**: Prefer webpack ZipPlugin or the official sample's `pnpm build`
- [ ] **Manifest in zip matches manifest in repo**: The manifest JSON inside the zip should be consistent with the one in the repository (same fields, same values)

## 7. i18n — check in [both]

- [ ] **No mixing**: Different languages must not be mixed within the same locale (proper nouns excepted)
- [ ] **Consistent messages**: If a locale is supported, UI messages shown to the user (errors, notifications) must be in that locale
- [ ] **Files bundled**: If the plugin uses `this.i18n` in code, the corresponding i18n JSON files must be present in the `i18n/` directory inside the zip. The framework auto-loads them — no `plugin.json` field is needed. If the plugin does not use i18n, don't bundle an `i18n/` directory

## 8. Theme-specific Rules

- [ ] **No built-in fonts** [zip]: Don't bundle fonts that are already built into SiYuan
- [ ] **Font changes documented** [zip]: If the theme changes interface fonts, mention this in the README
- [ ] **Uninstall restore** [repo]: Must restore all interface changes when the theme is uninstalled

## 9. Widget-specific Rules

- [ ] **No config in `data/widgets`** [repo]: This is the install directory and gets overwritten on widget update. Save config to `data/storage` or use `setLocalStorageVal` instead

## 10. Pre-existing CI Checks

The following are checked automatically by CI — no need to re-verify unless the CI result looks suspicious:
- Release exists with `package.zip` asset
- Required files exist in the release: `icon.png`, `preview.png`, `README.md`, manifest JSON
- Manifest has required fields: `name`, `version`, `author`, `url`
- `name` is a valid directory name, matches repo name, and is unique across all package types
- New themes do not contain `theme.js` (unless in the allowlist)
- `icon.png` dimensions (CI requires 160x160)
- `preview.png` dimensions (CI requires 1024x768)
