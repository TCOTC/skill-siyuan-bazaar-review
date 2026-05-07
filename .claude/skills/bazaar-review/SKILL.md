---
name: bazaar-review
description: Review pull requests for the siyuan-note/bazaar community marketplace. Use this skill whenever the user asks to review a bazaar PR, check a marketplace submission, audit a plugin/theme/widget/template/icon package for the SiYuan Note marketplace, or mentions reviewing PRs in the siyuan-note/bazaar repository. Also trigger when the user references a specific bazaar PR number and wants it checked.
---

# Bazaar PR Review

Review a pull request in [siyuan-note/bazaar](https://github.com/siyuan-note/bazaar) — the SiYuan Note community marketplace.

## Overview

The review has two phases:

1. **Automated review** — inspect both the `package.zip` (what users download) and the repository source code (what developers write). Everything in this phase can be done without installing the package.
2. **Manual verification** — the reviewer must actually install and run the package in SiYuan to test functionality, UI, and cleanup behavior. The skill flags what needs manual testing.

The key distinction:
- **package.zip** is the shipped artifact — check file sizes, path separators, unnecessary files, manifest fields, README/icon/preview as they appear inside the zip
- **Repository** is the source — check LICENSE, code quality, i18n files, and that source matches what's in the zip

## Security Rules

**The marketplace package code must never be executed locally.** The package being reviewed is untrusted third-party code. Running it could compromise the reviewer's machine or the SiYuan instance.

### Prohibited (the skill must never do these)
- Executing any script or binary from the package: `node`, `npm install`, `npm run`, `python`, `bash` on package files
- Loading or installing the package into a SiYuan instance
- Running `eval`, `import`, or `require` on package code
- Any operation that would cause the package's code to be interpreted or executed

### Allowed (read-only inspection)
- Downloading and extracting `package.zip` to read its files as text
- Reading text files: manifest JSON, README, source code, i18n files, CSS
- Checking file metadata: sizes, names, directory structure, path separators
- All `gh api` / `gh` CLI operations
- `curl` to fetch file contents from GitHub raw URLs
- `find`, `ls`, `stat`, `cat`, `head` on extracted files to inspect them

### User-only actions
These are high-risk and only the user should perform them manually:
- Actually installing the package in SiYuan for functional testing
- Enabling/disabling/uninstalling the package to verify cleanup
- Any testing that involves running the package's JavaScript/TypeScript

## SiYuan Plugin Lifecycle Reference

Understanding the Plugin class lifecycle is essential for reviewing code quality. The lifecycle is defined in `app/src/plugin/index.ts` and orchestrated by `app/src/plugin/loader.ts` and `app/src/plugin/uninstall.ts`.

### Lifecycle Methods

| Method | When Called | Purpose |
|---|---|---|
| `onload()` | Plugin loaded/installed | Initialize: register event listeners, IPC handlers, commands, load config |
| `onLayoutReady()` | After layout finishes loading | Re-attach UI elements (top bar icons, docks, status bar items) |
| `onunload()` | **Both** disable and full uninstall | Release runtime resources: remove event listeners, IPC handlers, DOM elements |
| `uninstall()` | **Only** full uninstall (after `onunload()`) | Persistent cleanup: call `removeData()` to delete stored config |
| `onDataChanged()` | Plugin's storage data changed externally | Reload plugin to apply synced config changes |

Note: There is **no** `onuninstall()` method. The method is named `uninstall()`.

### Lifecycle Flow

```
Install:            onload() → onLayoutReady()
Disable:            onunload()
Reload / Update:    onunload() → onload() → onLayoutReady()
Full Uninstall:     onunload() → uninstall()
Data Synced:        onDataChanged() → internal reload
```

Key distinction:
- **`onunload()`**: fires on disable, reload, and uninstall — use only for **runtime** cleanup (event listeners, IPC, DOM)
- **`uninstall()`**: fires only on full uninstall — use for **persistent** cleanup (`removeData()`)

Do NOT put `removeData()` in `onunload()`, as that would delete user config on every plugin disable/update, losing settings.

The framework handles these automatically during uninstall: plugin tabs, top bar icons, status bar icons, docks, inline comments, SVG icons, CSS styles, toolbar updates, and registered commands (via `addCommand()`). Plugins only need to clean up their own resources.

## Temp Directory

All per-review temporary files go under `/tmp/bazaar-review/<PR>/<date>-<short-commit>-<release-id>/`. The directory name uses three components:

- `date`: `YYYYMMDD` — the date the directory was created (e.g., `20260503`)
- `short-commit`: first 7 characters of the commit hash
- `release-id`: GitHub release ID

This structure ensures:
- **zip and source always together** in one directory per release
- **Old versions preserved** for comparison across review rounds
- **Same commit, different release** → separate directories, but `repo-source/` is copied locally from the existing same-commit directory (fast, no re-download)

Each version directory holds:
- `package.zip` — the downloaded artifact and its extracted contents
- `repo-source/` — the full repository source code (downloaded or copied from same-commit directory)
- `version.txt` — records the tag, commit hash, release id, and asset id

The skill provides two utility functions that apply to this directory:

**Open the current version directory:**
```bash
# Open in file manager (Windows: Explorer, macOS: Finder, Linux: xdg-open)
explorer.exe "$(cygpath -w /tmp/bazaar-review/<PR>/<version-dir>)" 2>/dev/null || open /tmp/bazaar-review/<PR>/<version-dir> 2>/dev/null || xdg-open /tmp/bazaar-review/<PR>/<version-dir>
```

**Clean up temp files:**
```bash
# Remove a specific version's temp files
rm -rf /tmp/bazaar-review/<PR>/<version-dir>
# Remove all versions for a specific PR
rm -rf /tmp/bazaar-review/<PR>/
# Remove all review temp files
rm -rf /tmp/bazaar-review/
```

These utilities are invoked at specific points in the workflow (see below).

## Workflow

### Step 1: Get the PR

Ask the user for the PR number if they haven't provided it. Then fetch the PR:

```bash
gh api repos/siyuan-note/bazaar/pulls/<NUMBER> --jq '{title, body, state, user: .user.login, head: .head.label, base: .base.label, changed_files: .changed_files, html_url: .html_url}'
```

Also check which files were changed and what the CI bot commented:

```bash
gh api repos/siyuan-note/bazaar/pulls/<NUMBER>/files --jq '.[].filename'
gh api "repos/siyuan-note/bazaar/issues/<NUMBER>/comments?per_page=20" --jq '.[] | select(.user.login == "github-actions[bot]") | .body'
```

### Step 2: Identify the package type and repo

From the changed files, determine:
- **Which txt file** was modified (`plugins.txt`, `themes.txt`, `widgets.txt`, `templates.txt`, `icons.txt`) — this tells you the package type
- **Which repo** is being added/updated — the new line in the diff gives `owner/repo`

### Step 3: Detect developer's language

Determine whether the developer speaks Chinese, which decides the language used for the review report. Check these signals in order of reliability:

- **PR body text** — does it contain Chinese? (The PR title is formatted by the reviewer and is irrelevant.)
- **Repo description**: `gh api repos/<owner>/<name> --jq '.description'` — if it only contains Chinese, the developer likely speaks Chinese.
- **Developer's GitHub profile**: `gh api users/<username> --jq '.bio, .company, .location, .blog'` — any Chinese text in these fields?
- **README files** — if the package only has a Chinese README without an English counterpart (e.g., only `README_zh_CN.md` or a Chinese-only `README.md`), the developer speaks Chinese. Do NOT infer language from the presence of bilingual READMEs, since bazaar packages almost always include both regardless of the developer's language.

If any of the above signals indicate Chinese proficiency, write the review report in Chinese. Otherwise, use English.

### Step 4: Get release info

```bash
gh api repos/<owner>/<name>/releases/latest --jq '{id: .id, tag: .tag_name, assets: [.assets[] | {name: .name, size: .size, id: .id}]}'
# Get the commit hash for the release tag
gh api "repos/<owner>/<name>/commits/<tag>" --jq '.sha'
```

Record these values:
- `release_id`: the GitHub release `id`
- `tag`: the release tag name
- `commit_short`: first 7 characters of the commit SHA
- `asset_id`: the `package.zip` asset `id`
- `vdir`: the version directory name — `<YYYYMMDD>-<commit_short>-<release_id>` (use `date +%Y%m%d` for the date prefix — this is the date the directory is created, not the release date)

These are used to:
- **Same release id** → full cache hit
- **New release id, commit_short matches an existing directory** → same source code, zip was re-uploaded. Create new `vdir`, download zip fresh, but **copy `repo-source/` locally** from the existing same-commit directory
- **New commit_short** → new source code, download everything fresh
- Old version directories are kept for comparison across review rounds

### Step 5: Inspect `package.zip` (the shipped artifact)

First, construct the version directory name and check the cache:

```bash
VDIR="<date>-<commit_short>-<release_id>"
# Check if this exact release id already exists under any directory
ls -d /tmp/bazaar-review/<PR>/*-<release_id>/ 2>/dev/null
```

Cache hit logic:
- **Directory ending with `-<release_id>` exists** → full cache hit, reuse everything
- **No exact match, but a directory with the same `commit_short` exists** → source code unchanged but zip was re-uploaded. Create new `vdir`, download zip fresh, `repo-source/` will be copied in Step 6
- **No match at all** → new version, download everything fresh

If this is a new directory (not a full cache hit), create it:

```bash
mkdir -p /tmp/bazaar-review/<PR>/<vdir>/
curl -sL "<package.zip download URL>" -o /tmp/bazaar-review/<PR>/<vdir>/package.zip
```

Then extract and inspect:

```bash
cd /tmp/bazaar-review/<PR>/<vdir>/ && unzip -qo package.zip
# Record the version info
echo "tag: <tag>  commit: <commit_short>  release_id: <release_id>  asset_id: <asset_id>" > /tmp/bazaar-review/<PR>/<vdir>/version.txt
```

If the user wants to inspect the extracted files themselves, open the version directory:

```bash
explorer.exe "$(cygpath -w /tmp/bazaar-review/<PR>/<vdir>)" 2>/dev/null || open /tmp/bazaar-review/<PR>/<vdir> 2>/dev/null || xdg-open /tmp/bazaar-review/<PR>/<vdir>
```

Then inspect the extracted contents as text:

- **File list**: run `find . -type f` to see exactly what's bundled
- **Manifest JSON** (e.g., `plugin.json`): check all fields per the metadata checklist
- **`icon.png`**: actual file size in KB (`wc -c <file>` or `ls -lh`)
- **`preview.png`**: actual file size in KB
- **README files**: scan for relative-path links and embedded preview.png.
  Relative-path links (e.g., `[text](./doc.md)`, `![img](image.png)`) have undefined behavior — SiYuan may handle them differently in future versions, so they are not allowed.
  Run this to detect relative Markdown links (portable, works on macOS/Linux/Windows):
  ```bash
  grep -nE '\[[^]]+\]\([^)]+\)' README*.md | grep -v 'https\?://' | grep -v 'mailto:'
  ```
  This catches: `[text](README.md)`, `[text](./path/doc.md)`, `![alt](image.png)`, `[..](../file)`, `[text](#anchor)`, language-switch links like `[English](README.md)` / `[中文](README_zh_CN.md)`, etc.
  Any match is an issue — use absolute `https://` URLs instead (only `mailto:` links are exempt).
- **Path separators**: all paths inside the zip must use `/`, not `\`
- **Unnecessary files**: look for `node_modules`, `.git`, unused i18n folders, leftover template files
- **i18n files**: if present in the `i18n/` directory, check that each file's content matches its locale language (no mixing). The framework auto-loads these files — no `i18n` field in plugin.json is needed. The set of i18n locales is independent from the `displayName`/`description`/`readme` locale keys in plugin.json

### Step 6: Inspect the repository (the source code)

Instead of fetching individual files, download the entire repository source at the release tag so all source code is available for inspection.

Since the cache was checked in Step 5, decide the repo source strategy:

```bash
# Check if repo-source already exists in this vdir
ls /tmp/bazaar-review/<PR>/<vdir>/repo-source/ 2>/dev/null | head -20

# If not, check if another directory with the same commit_short exists
ls -d /tmp/bazaar-review/<PR>/*-<commit_short>-*/ 2>/dev/null
```

- **Full cache hit (this vdir already has repo-source)** → reuse
- **Same commit_short exists in another directory** → copy `repo-source/` locally from that directory (fast, avoids re-download):
  ```bash
  cp -r /tmp/bazaar-review/<PR>/<existing-dir>/repo-source/ /tmp/bazaar-review/<PR>/<vdir>/repo-source/
  ```
- **No same commit_short anywhere** → needs download. Check archive size first:

```bash
# Check repo archive size before downloading
curl -sI "https://api.github.com/repos/<owner>/<name>/zipball/<tag>" | grep -i content-length | awk '{print $2}'
```

**If the archive is larger than 5MB**, ask the user to confirm before downloading:
> 仓库源码压缩包约 X MB，需要下载全部源代码吗？这将用于全面检查代码质量。回复 y 确认下载，回复 n 则仅检查关键入口文件。

If confirmed (or if under 5MB), download and extract:

```bash
mkdir -p /tmp/bazaar-review/<PR>/<vdir>/repo-source/
curl -sL "https://api.github.com/repos/<owner>/<name>/zipball/<tag>" -o /tmp/bazaar-review/<PR>/<vdir>/repo-source.zip
unzip -qo /tmp/bazaar-review/<PR>/<vdir>/repo-source.zip -d /tmp/bazaar-review/<PR>/<vdir>/repo-source/
rm /tmp/bazaar-review/<PR>/<vdir>/repo-source.zip
# The extracted directory will be something like .../repo-source/<owner-name-xxxxx>/
# Find the actual extracted root:
REPO_ROOT=$(ls -d /tmp/bazaar-review/<PR>/<vdir>/repo-source/*/ | head -1)
```

Then inspect the code. Start with the overall structure:
```bash
find "$REPO_ROOT" -type f -not -path '*/node_modules/*' -not -path '*/.git/*' | head -80
```

For plugins, focus code inspection on these areas:

- **Lifecycle methods**: Check `onload()`, `onunload()`, `uninstall()`. `uninstall()` (if present) must call `removeData()` for stored config. `removeData()` must NOT appear in `onunload()` — see lifecycle reference above.

- **Cleanup completeness** (verify in code — do NOT delegate to manual verification): Cross-reference every resource registered in `onload()` against what's cleaned up in `onunload()`. If cleanup is missing, report as Issue Found; if complete, report as Passed. The SiYuan framework auto-removes plugin docks, tabs, top bar icons, status bar items, SVG icons, CSS, and registered commands (via `addCommand()`) — do NOT flag these as missing. Focus on: EventBus listeners, global event listeners, IPC listeners (if the plugin uses `require("electron")`), timers, observers, and network connections. See `references/checklist.md` Section 5 for the full list.

- **Config management**: Look for `saveData`/`loadData`/`removeData` patterns across all source files. `loadData()` should be called once and cached; `saveData()` should not be in `onunload`.

- **Code standards**: Check styles (`.plugin-sample` remnants), path separators (must use `/`), custom attributes (`custom-` prefix), logging (only in lifecycle functions), constants (named over literals), and `window.location.reload()` (must use `/api/ui/reloadUI` instead). See `references/checklist.md` Section 5 for the complete list.

Use `tail -n +1 /tmp/bazaar-review/<PR>/<vdir>/repo-source/<repo-root>/src/**/*.ts` or similar globs to read related source files in batches.

### Step 7: Check against standards

Go through the checklist in `references/checklist.md`. For each item, report: Pass / Issue found / Needs manual check.

The checklist is organized by category:
1. **Metadata** (manifest JSON fields)
2. **Icon & Preview** (file sizes, not using built-in icons)
3. **README** (relative links, language consistency, no embedded preview)
4. **LICENSE** (existence, year, copyright)
5. **Code quality** (plugin only — config management, cleanup, logs, i18n)
6. **Package zip** (naming, path separators, unnecessary files)
7. **Type-specific rules** (theme, widget, template, icon)

### Step 8: Report findings

Output the review in this format:

```
## Review for PR #<N>: <title>

### CI Status
<summary of what CI already checked>

### Findings from package.zip
#### Issues Found
- [ ] <issue with specific fix>
#### Passed
- item

### Findings from repository
#### Issues Found
- [ ] <issue with specific fix>
#### Passed
- item

### Needs Manual Verification
- [ ] Install and test <specific feature that requires runtime interaction>
- [ ] Verify UI layout on all declared frontends (desktop, mobile, browser-desktop)
- [ ] Verify package works correctly after reload/update cycle
- [ ] Verify no console errors during normal use
```

**What belongs in "Needs Manual Verification"**: Only items that genuinely cannot be verified from source code alone — runtime behavior, visual appearance, cross-platform compatibility, errors that only surface during execution. If it can be checked by reading the source code (e.g., whether `onunload()` removes event listeners, whether `uninstall()` calls `removeData()`), verify it in code review and report it as Passed or Issue Found instead.

Issue description rules:
- **Quote once, then fix**: Only quote the problematic value once — when it's the issue itself. Don't repeat it in the fix suggestion (e.g., don't say "change X to Y instead of X" — "X" was already quoted).
- **Locale suggestion matches the original value's language**: When suggesting a replacement for a locale-specific field, the suggestion must be in the same language as the current value — don't cross languages. The only exception is when the current value is already in the wrong language for its key (e.g., Chinese text in `en_US`), in which case suggest the correct language.
- **Be specific**: quote the exact field value, file path, and line number.

### Step 9: Save review to file

After the review report is complete, write the review conclusions to a file in the version directory.

**IMPORTANT**: Use bash heredoc (`cat > ... << 'REVIEW_EOF'`) to write this file. Do NOT use the Write tool — it writes to a different filesystem than bash's `/tmp/`, causing `explorer.exe` to fail to find the file and open the Documents folder instead.

**If the developer speaks Chinese (determined in Step 3):** write the file in Chinese only.

**If the developer does NOT speak Chinese:** write the file with both Chinese and an English translation appended at the end, separated by a horizontal rule. The English section should mirror the Chinese content exactly.

Template for **Chinese-speaking** developer (Chinese only, no English, no separator):

```bash
cat > /tmp/bazaar-review/<PR>/<vdir>/review.md << 'REVIEW_EOF'
## 初步审核

以下是 AI 的审核结论，请开发者确认并修复之后回复，然后维护者会进行人工审核。

### 版本: <tag> (commit: <commit_short>)

### 发现的问题
<list each issue with specific fix>

### 需要维护者人工验证
<list items that require manual installation/testing>

### 检查通过
<list each item that passed>
REVIEW_EOF
```

Template for **non-Chinese** developer (Chinese first, then `---` separator, then English):

```bash
cat > /tmp/bazaar-review/<PR>/<vdir>/review.md << 'REVIEW_EOF'
## 初步审核

以下是 AI 的审核结论，请开发者确认并修复之后回复，然后维护者会进行人工审核。

### 版本: <tag> (commit: <commit_short>)

### 发现的问题
<list each issue with specific fix>

### 需要维护者人工验证
<list items that require manual installation/testing>

### 检查通过
<list each item that passed>

---

## Preliminary Review

The following is the AI's review conclusion. Please confirm and fix the issues before replying, then the maintainer will conduct a manual review.

### Version: <tag> (commit: <commit_short>)

### Issues Found
<list each issue with specific fix>

### Needs Manual Verification
<list items that require manual installation/testing>

### Passed
<list each item that passed>
REVIEW_EOF
```

After writing the file, open it with the default text editor:

```bash
explorer.exe "$(cygpath -w /tmp/bazaar-review/<PR>/<vdir>/review.md)" 2>/dev/null || open /tmp/bazaar-review/<PR>/<vdir>/review.md 2>/dev/null || xdg-open /tmp/bazaar-review/<PR>/<vdir>/review.md
```

The Chinese section structure:
- **初步审核**：二级标题，紧跟一句说明
- **版本**：三级标题，标注 tag 和 commit
- **发现的问题**：列出每个问题及具体修复建议，以 `- [ ]` 格式
- **需要维护者人工验证**：列出需要实际安装测试的项目，以 `- [ ]` 格式
- **检查通过**：列出每个已通过的检查项，以 `- [x]` 格式

The English section mirrors the same structure with equivalent headings and content.

### Step 10: Manual verification

Remind the reviewer of items that require actual installation (only things that cannot be verified from source code):

- Functional testing of all declared features
- UI layout and styling checks across all declared frontends
- Browser environment compatibility (if frontends include `browser-desktop`)
- Any behavior that depends on SiYuan runtime state (e.g., data sync, multi-window)

### Step 11: Clean up temp files

After the review report is delivered and the manual verification items are noted, ask the user:

> 是否需要清理临时文件？
> 1. 清理本次版本的临时文件 (`/tmp/bazaar-review/<PR>/<vdir>/`)
> 2. 清理此 PR 所有版本的临时文件 (`/tmp/bazaar-review/<PR>/`)
> 3. 清理所有审查的临时文件 (`/tmp/bazaar-review/`)
> 4. 暂时保留

Execute the corresponding command:

```bash
# Option 1: Clean this version only
rm -rf /tmp/bazaar-review/<PR>/<vdir>/

# Option 2: Clean all versions for this PR
rm -rf /tmp/bazaar-review/<PR>/

# Option 3: Clean all reviews
rm -rf /tmp/bazaar-review/

# Option 4: No action
```

### Step 12: After approval

When all issues are resolved and the PR is ready to merge, the final comment on the PR should be:

```
感谢你的贡献，思源有你更精彩！

Thank you for your contribution. SiYuan will be more wonderful with you!
```

Note the blank line between the Chinese and English lines.
