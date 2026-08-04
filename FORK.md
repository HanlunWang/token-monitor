# FORK.md

Fork-specific guidance. `AGENTS.md` still governs everything else — this file only covers what differs because this repo is a personal fork.

## What this fork is

`HanlunWang/token-monitor` (`origin`), forked from `Javis603/token-monitor` (`upstream`). The fork exists so the app is built from audited source instead of downloaded binaries, and to carry a small local feature.

**Never push, open pull requests, or file issues against `upstream`.** Contributing changes back is explicitly not the goal. `upstream` is fetch-only.

Local commits are kept rebased on top of `upstream/main`, so history stays linear and each sync is a clean replay: this file, plus the one functional change — *"feat(home): add full limits view mode for the Home limits module"*. That commit adds a `homeLimitDisplayMode` setting (`compact` | `full`, default `compact`); in `full` mode the Home limits module renders the Limits-page rows by sharing `buildLimitProviderNodes()` between both surfaces, toggled from Settings → Home → limits module.

## Syncing with upstream

```bash
git fetch upstream
git rebase upstream/main
```

Then verify, rebuild, and `git push --force-with-lease origin main` (the rebase makes the fork's previous commit non-fast-forward, so a plain push is correctly rejected).

### A clean rebase is not evidence the feature still works

This is the part that matters. Upstream refactors `src/electron/renderer/app.js` aggressively — thousand-line diffs and repeated reshuffling of the limits and settings code are normal between releases. The local commit has so far always applied without a textual conflict, which proves only that no *edited lines* overlapped. It does not prove that upstream logic wasn't silently dropped, because the local change **extracts** `buildLimitProviderNodes()` out of `renderLimits()`: if upstream adds a branch to the original loop, git can replay the extraction cleanly while the new branch ends up in neither function.

So after every rebase, diff the local functions against upstream's own copies rather than eyeballing them:

```bash
git show upstream/main:src/electron/renderer/app.js > /tmp/up-app.js
# expect: identical loop bodies
awk '/^function renderLimits\(\)/,/^}/'            /tmp/up-app.js
awk '/^function buildLimitProviderNodes\(rows\)/,/^}/' src/electron/renderer/app.js
# expect: the diff shows ONLY the local additions, nothing of upstream's removed
diff <(awk '/^function renderHomeLimitProviderList\(\)/,/^}/' /tmp/up-app.js) \
     <(awk '/^function renderHomeLimitProviderList\(\)/,/^}/' src/electron/renderer/app.js)
diff <(awk '/^function renderHomeLimitModule\(\)/,/^}/' /tmp/up-app.js) \
     <(awk '/^function renderHomeLimitModule\(\)/,/^}/' src/electron/renderer/app.js)
```

Also confirm the setting survived end to end: `normalizeHomeLimitDisplayMode` plus the three `homeLimitDisplayMode` hooks in `src/electron/main.js` (defaults, saved-settings merge, save patch), the `settings.home.fullLimitView` key in all five locales of `i18n.js`, and the `.home-module-limits-full` rule in `styles.css`.

> `grep` has silently returned no matches on `app.js` at its current size even when the strings are present. When a check comes back empty, re-confirm with Node before concluding anything is missing:
> `node -e "console.log((require('fs').readFileSync('src/electron/renderer/app.js','utf8').match(/buildLimitProviderNodes/g)||[]).length)"`

### Verification

Run `npm ci` first whenever upstream moved `package.json` — `tokscale` is bumped often, and a stale `node_modules` builds an app against the wrong collector.

`npm run verify` (lint + test) plus `npm run sync:worker` (must leave the tree clean). One test failure is **pre-existing upstream breakage, not a regression**: `tests/shared/sessionUsageArchive.test.js` → *"archive day and month windows expire while all-time stays available"*. It depends on hardcoded date fixtures and fails on a pristine `upstream/main` checkout too. Confirm rather than assume, with a throwaway worktree:

```bash
git worktree add -q /tmp/up-check upstream/main
node --test /tmp/up-check/tests/shared/sessionUsageArchive.test.js
git worktree remove --force /tmp/up-check
```

## Packaging

There is no Apple Developer signing identity on this machine, so upstream's `forceCodeSigning` must be overridden and the bundle ad-hoc signed afterwards (electron-builder leaves it unsigned, which arm64 will not launch reliably):

```bash
npx electron-builder --mac --arm64 --dir -c.mac.forceCodeSigning=false -c.mac.identity=null --publish never
codesign --force --deep --sign - "dist/mac-arm64/Token Monitor.app"
codesign --verify --deep --strict "dist/mac-arm64/Token Monitor.app"
```

Install by replacing `/Applications/Token Monitor.app` with the freshly built bundle. Quit the running app first — a single-instance lock means the packaged app and `npm start` cannot run at the same time, and they share one `userData` directory, so settings carry across upgrades untouched.

Ad-hoc signing is fine for this machine only; the bundle will trip Gatekeeper on anyone else's.

## The in-app updater

Leave it alone. `publish` in `package.json` and `GITHUB_REPO` in `src/shared/appUpdater.js` both point at **upstream's** releases, so installing an offered update replaces this build with the official one and drops the local feature. Automatic updates default to off and the updater is inert when running from source; the supported upgrade path is the sync-and-rebuild flow above.
