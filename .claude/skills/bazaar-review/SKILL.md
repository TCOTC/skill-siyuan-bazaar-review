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
- **Repository source** (from default branch) is the latest development code — check LICENSE, code quality, i18n files, and **compare against `package.zip` to detect if the shipped artifact is out of sync with the latest source**

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

## Shell Environment

All file operations in this workflow (`curl`, `unzip`, `grep`, `find`, `cat`, heredoc writes) assume a **Unix-like shell**. On Windows, follow this priority:

1. **Git Bash** (from [Git for Windows](https://gitforwindows.org/)) — **preferred**
2. **macOS / Linux bash** — native
3. **WSL bash** — only if Git Bash (or native bash) is unavailable

### Why Git Bash on Windows

- Provides `grep`, `curl`, `unzip`, `find`, `cygpath`, and bash heredoc — tools this skill relies on
- Maps Windows project paths to Unix-style paths via `cygpath -w`, so `explorer.exe` can open review files
- Keeps `reviews/...` in the project directory, so files persist and are easy to access from the editor
- **Do not mix shells**: WSL has its own `/tmp`, separate from Git Bash. Writing files in WSL and opening them via `\\wsl.localhost\...` is fragile (paths may fail; files disappear after WSL restarts). Stick to one shell for the entire review

### Detecting the shell

Before the first file operation on Windows, confirm Git Bash is available:

```bash
which bash
# Git for Windows: /usr/bin/bash or /bin/bash, often under C:/Program Files/Git/
# WSL: /bin/bash inside a wsl.exe invocation — avoid unless Git Bash is missing
```

Run workflow commands through Git Bash, e.g. `bash -lc 'curl ...'` using Git for Windows' `bash.exe`. If `gh` is not in Git Bash's `PATH`, run `gh` from PowerShell separately — but keep all `reviews/...` file I/O in Git Bash.

### Opening files and folders on Windows

From Git Bash, convert paths before calling `explorer.exe`:

```bash
explorer.exe "$(cygpath -w reviews/<PR>/<date>-<commit_short>-<release_id>)"
explorer.exe "$(cygpath -w reviews/<PR>/<date>-<commit_short>-<release_id>/review.md)"
```

Do **not** use `\\wsl.localhost\...` paths or `wsl` to open review artifacts.

## Review Directory

All per-review files go under `reviews/<PR>/<date>-<commit_short>-<release_id>/` at the project root. All shell commands assume the current working directory is the project root, so `reviews/...` paths are relative to the repository root — no absolute path needed.

The directory name uses three components:

- `date`: `YYYYMMDD` — the date the directory was created (e.g., `20260503`)
- `commit_short`: first 7 characters of the release tag commit SHA
- `release-id`: GitHub release ID

This structure ensures:
- **zip and source always together** in one directory per release
- **Old versions preserved** for comparison across review rounds

Each version directory holds:
- `package.zip` — the downloaded artifact and its extracted contents
- `repo-source/` — the full repository source code from the **default branch** (latest, not the release tag)
- `version.txt` — records the tag, commit hash, release id, asset id, default branch, and default branch commit
- `review.md` — the review report (see Step 9)

The reviews directory is inside the project, so it persists across sessions and is easy to access from the editor.

The skill provides two utility functions that apply to this directory:

**Open the current version directory:**
```bash
explorer.exe "$(cygpath -w reviews/<PR>/<date>-<commit_short>-<release_id>)" 2>/dev/null || open reviews/<PR>/<date>-<commit_short>-<release_id> 2>/dev/null || xdg-open reviews/<PR>/<date>-<commit_short>-<release_id>
```

**Clean up review files** (paths relative to project root — see Step 11):
```bash
# Remove a specific version's review files
rm -rf reviews/<PR>/<version-dir>
# Remove all versions for a specific PR
rm -rf reviews/<PR>/
# Remove all reviews
rm -rf reviews/
```

These utilities are invoked at specific points in the workflow (see below).

## Workflow

Run all steps below in **Git Bash** on Windows (see [Shell Environment](#shell-environment)). Use PowerShell only for `gh` if it is missing from Git Bash's `PATH`.

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

### Step 4: Get release info & default branch info

```bash
# Release info
gh api repos/<owner>/<name>/releases/latest --jq '{id: .id, tag: .tag_name, assets: [.assets[] | {name: .name, size: .size, id: .id}]}'
# Get the commit hash for the release tag
gh api "repos/<owner>/<name>/commits/<tag>" --jq '.sha'
# Get default branch name
gh api repos/<owner>/<name> --jq '.default_branch'
# Get the latest commit on default branch
gh api "repos/<owner>/<name>/branches/<default_branch>" --jq '{sha: .commit.sha}'
```

Record these values:
- `release_id`: the GitHub release `id`
- `tag`: the release tag name
- `commit_short`: first 7 characters of the release tag commit SHA
- `asset_id`: the `package.zip` asset `id`
- `default_branch`: the repo's default branch name (e.g., `main` or `master`)
- `default_commit`: the full SHA of the latest commit on the default branch
- Version directory path: `reviews/<PR>/<YYYYMMDD>-<commit_short>-<release_id>/` (Git Bash full path: `reviews/<PR>/<YYYYMMDD>-<commit_short>-<release_id>/`, use `date +%Y%m%d` for the date prefix — this is the date the directory is created, not the release date)

These are used to:
- **Same release id** → full cache hit
- **New release id, commit_short matches an existing directory** → same source code, zip was re-uploaded. Create new directory, download zip fresh, repo-source is fetched separately
- **New commit_short** → new source code, download everything fresh
- Old version directories are kept for comparison across review rounds

### Step 5: Inspect `package.zip` (the shipped artifact)

First, construct the version directory path and check the cache. All commands use `reviews/<PR>/<date>-<commit_short>-<release_id>/` relative to the project root — run all commands from the project root directory.

```bash
# Check if this exact release id already exists under any directory
ls -d reviews/<PR>/*-<release_id>/ 2>/dev/null
```

Cache hit logic:
- **Directory ending with `-<release_id>` exists** → full cache hit, reuse everything
- **No exact match, but a directory with the same `commit_short` exists** → source code unchanged but zip was re-uploaded. Create new version directory, download zip fresh, `repo-source/` will be fetched separately
- **No match at all** → new version, download everything fresh

If this is a new directory (not a full cache hit), create it (paths relative to project root):

```bash
mkdir -p reviews/<PR>/<date>-<commit_short>-<release_id>/
curl -sL "<package.zip download URL>" -o reviews/<PR>/<date>-<commit_short>-<release_id>/package.zip
```

Then extract and inspect (use `-d` target dir instead of `cd`):

```bash
unzip -qo reviews/<PR>/<date>-<commit_short>-<release_id>/package.zip -d reviews/<PR>/<date>-<commit_short>-<release_id>/
# Record the version info (reviews/ path is relative to project root, no heredoc)
echo "tag: <tag>  commit: <commit_short>  release_id: <release_id>  asset_id: <asset_id>" > reviews/<PR>/<date>-<commit_short>-<release_id>/version.txt
echo "default_branch: <default_branch>  default_commit: <default_commit>" >> reviews/<PR>/<date>-<commit_short>-<release_id>/version.txt
```

If the user wants to inspect the extracted files themselves, open the version directory (path relative to project root):

```bash
explorer.exe "$(cygpath -w reviews/<PR>/<date>-<commit_short>-<release_id>)" 2>/dev/null || open reviews/<PR>/<date>-<commit_short>-<release_id> 2>/dev/null || xdg-open reviews/<PR>/<date>-<commit_short>-<release_id>
```

Then inspect the extracted contents as text:

- **File list**: read zip entries with proper UTF-8 filename handling. Do NOT use `unzip -l` to list files — its terminal output may garble non-ASCII characters (Chinese, Japanese, etc.), causing false positives. Use Python's `zipfile` module instead, which handles UTF-8 encoded filenames correctly:
  ```bash
  python3 -c "import zipfile; f='reviews/<PR>/<date>-<commit_short>-<release_id>/package.zip'; [print(e) for e in zipfile.ZipFile(f).namelist()]"
  ```
  If `python3` is unavailable, `find` on the extracted files is also reliable (the OS filesystem stores the true name). Only fall back to `find . -type f` (run from inside the extracted directory).
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
- **Path separators**: all paths inside the zip must use `/`, not `\`. Check with Python zipfile for accuracy (terminal `unzip -l` may garble path character encoding):
  ```bash
  python3 -c "import zipfile; f='reviews/<PR>/<date>-<commit_short>-<release_id>/package.zip'; [print(e) for e in zipfile.ZipFile(f).namelist()]" | grep '\\\\'
  ```
  If any entry contains `\`, it's an issue — the `unzip` extraction warning is another signal. Only flag backslash paths, not forward-slash paths.
- **Unnecessary files**: look for `node_modules`, `.git`, unused i18n folders, leftover template files
- **i18n files**: if present in the `i18n/` directory, check that each file's content matches its locale language (no mixing). The framework auto-loads these files — no `i18n` field in plugin.json is needed. The set of i18n locales is independent from the `displayName`/`description`/`readme` locale keys in plugin.json

### Step 6: Inspect the repository source (from default branch)

Download the repository source code from the **default branch** (not the release tag). This is the latest development code. Compare it against what's in `package.zip` to detect if the shipped artifact is out of sync with the latest source.

#### Download repo source from default branch

```bash
# Check if repo-source already exists in this version directory
ls reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source/ 2>/dev/null | head -20

# If not, check if another directory has repo-source with the same default_commit
# (use grep to find matching version.txt without a loop — avoid shell variables)
grep -l "default_commit: <default_commit>" reviews/<PR>/*/version.txt 2>/dev/null | head -1
```

Cache strategy:
- **This version directory already has repo-source** → reuse
- **Another directory has repo-source with the same default_commit** → copy locally
  ```bash
  cp -r reviews/<PR>/<cached-dir>/repo-source/ reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source/
  ```
- **No cache hit** → download fresh. Check archive size first:

```bash
curl -sI "https://api.github.com/repos/<owner>/<name>/zipball/<default_branch>" | grep -i content-length | awk '{print $2}'
```

**If the archive is larger than 5MB**, ask the user to confirm before downloading:
> 仓库源码压缩包约 X MB，需要下载全部源代码吗？这将用于全面检查代码质量，并与 package.zip 对比。回复 y 确认下载，回复 n 则仅检查关键入口文件。

If confirmed (or if under 5MB), download and extract. All paths relative to project root — no shell variables:

```bash
mkdir -p reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source/
curl -sL "https://api.github.com/repos/<owner>/<name>/zipball/<default_branch>" -o reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source.zip
unzip -qo reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source.zip -d reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source/
rm reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source.zip
# The extracted directory will be something like .../repo-source/<owner-name-xxxxx>/
# Use a wildcard for the extracted root dir under repo-source/
```

#### Compare repo source against package.zip

After both the repo source and package.zip are available, cross-reference them:

- **Check if the repo source has been updated since the release**: compare file modification times or check git log between the release tag and the default branch HEAD. If there are new commits on the default branch after the release tag, note this in the review.

- **Compare key files** between `package.zip` contents and the repo source. Focus on:
  - `plugin.json` — are there newer fields in the repo source that aren't in the package?
  - `i18n/` files — any new locales added in source but not shipped?
  - Source files (`.ts`, `.svelte`) — any significant logic changes not reflected in the compiled `index.js`?

  Only flag an issue if there are **meaningful differences** — e.g., new manifest fields, new i18n translations, or significant code changes. Trivial differences (whitespace, comments, unrelated config files) should not be flagged.

  If everything is in sync, report this as Passed.

- **LICENSE**: Check the repo source for a LICENSE file (not bundled in package.zip)

#### Inspect the code structure

```bash
# Path to the repo-source root directory (relative to project root)
find reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source/*/ -type f -not -path '*/node_modules/*' -not -path '*/.git/*' 2>/dev/null | head -80
```

For plugins, focus code inspection on these areas:

- **Lifecycle methods**: Check `onload()`, `onunload()`, `uninstall()`. `uninstall()` (if present) must call `removeData()` for stored config. `removeData()` must NOT appear in `onunload()` — see lifecycle reference above.

- **Cleanup completeness** (verify in code — do NOT delegate to manual verification): Cross-reference every resource registered in `onload()` against what's cleaned up in `onunload()`. If cleanup is missing, report as Issue Found; if complete, report as Passed. The SiYuan framework auto-removes plugin docks, tabs, top bar icons, status bar items, SVG icons, CSS, and registered commands (via `addCommand()`) — do NOT flag these as missing. Focus on: EventBus listeners, global event listeners, IPC listeners (if the plugin uses `require("electron")`), timers, observers, and network connections. See `references/checklist.md` Section 5 for the full list.

- **Config management**: Look for `saveData`/`loadData`/`removeData` patterns across all source files. `loadData()` should be called once and cached; `saveData()` should not be in `onunload`.

- **Code standards**: Check styles (`.plugin-sample` remnants), path separators (must use `/`), custom attributes (`custom-` prefix), logging (only in lifecycle functions), constants (named over literals), and `window.location.reload()` (must use `/api/ui/reloadUI` instead). See `references/checklist.md` Section 5 for the complete list.

Use `tail -n +1 reviews/<PR>/<date>-<commit_short>-<release_id>/repo-source/*/src/**/*.ts` or similar globs to read related source files in batches.

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

**IMPORTANT**: Use bash heredoc (`cat > ... << 'REVIEW_EOF'`) in **Git Bash** (or native bash) to write this file. Use full literal file paths in the heredoc — no shell variables. The `reviews/` directory is in the project, so the file persists and is easy to access.

**If the developer speaks Chinese (determined in Step 3):** write the file in Chinese only.

**If the developer does NOT speak Chinese:** write the file with both Chinese and an English translation appended at the end, separated by a horizontal rule. The English section should mirror the Chinese content exactly.

Template for **Chinese-speaking** developer (Chinese only, no English, no separator):

```bash
cat > reviews/<PR>/<date>-<commit_short>-<release_id>/review.md << 'REVIEW_EOF'
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
cat > reviews/<PR>/<date>-<commit_short>-<release_id>/review.md << 'REVIEW_EOF'
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

After writing the file, open it with the default text editor. On Windows, run from **Git Bash** so `cygpath` resolves correctly:

```bash
explorer.exe "$(cygpath -w reviews/<PR>/<date>-<commit_short>-<release_id>/review.md)" 2>/dev/null || open reviews/<PR>/<date>-<commit_short>-<release_id>/review.md 2>/dev/null || xdg-open reviews/<PR>/<date>-<commit_short>-<release_id>/review.md
```

Tell the user the file path in the review output.

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

### Step 11: Clean up review files

**Security note**: All shell commands use `reviews/` paths relative to the project root — run them from the project root directory and do not use shell variables in paths.

After the review report is delivered and the manual verification items are noted, ask the user:

> 是否需要清理审查文件？
> 1. 清理本次版本的审查文件 (`reviews/<PR>/<date>-<commit_short>-<release_id>/`)
> 2. 清理此 PR 所有版本的审查文件 (`reviews/<PR>/`)
> 3. 清理所有审查文件 (`reviews/`)
> 4. 暂时保留

Execute the corresponding command (paths relative to project root):

```bash
# Option 1: Clean this version only
rm -rf reviews/<PR>/<date>-<commit_short>-<release_id>/

# Option 2: Clean all versions for this PR
rm -rf reviews/<PR>/

# Option 3: Clean all reviews
rm -rf reviews/

# Option 4: No action
```

### Step 12: After approval

When all issues are resolved and the PR is ready to merge, the final comment on the PR should be:

```
感谢你的贡献，思源有你更精彩！

Thank you for your contribution. SiYuan will be more wonderful with you!
```

Note the blank line between the Chinese and English lines.
