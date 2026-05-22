# TODO

Known issues with the GitHub profile metrics, with context and fixes.

## Achievements plugin — disabled (broken upstream)

**Status:** the two `achievements` steps in `.github/workflows/metrics.yml` are
commented out as of 2026-05-22.

**Symptom:** `metrics.plugin.achievements.svg` (detailed) and
`metrics.plugin.achievements.compact.svg` (compact) both rendered
`Unexpected error`.

**Root cause:** `lowlighter/metrics@v3.34`'s achievements plugin issues a
GraphQL query (`AchievementsDefault`) that still selects a **Projects (classic)**
field. GitHub has fully sunset Projects classic, so GitHub's GraphQL API now
rejects the *entire* query:

```
GraphqlResponseError: Request failed due to following response errors:
 - Projects (classic) is being deprecated in favor of the new Projects experience.
   at .../source/plugins/achievements/list/users.mjs:4
```

`detailed` and `compact` reuse the same query, so both fail. No workflow-config
option avoids it — the deprecated field is hard-coded in the action's source.

**Upstream tracking:**
- Issue: lowlighter/metrics#1706 — "Manager" Achievement Broken Due to Projects
  Classic Deprecation (open since 2025-04-03)
- PR (unmerged): lowlighter/metrics#1769 — migrate achievements to Projects V2 API

**Fix paths (pick one):**
1. Wait for an upstream fix to merge + release, then un-comment the steps and
   bump `lowlighter/metrics@` to the new version.
2. Pin the steps to a fork or commit SHA that includes the Projects-V2 fix from
   PR #1769 — e.g. `uses: <fork>/metrics@<sha>`.
3. Drop the achievements plugin permanently.

> `metrics`' last release is **v3.34 (Sept 2023)** — no release in 2.5+ years,
> so fix path 1 may not land soon.

## Habits plugin — mitigated (upstream bug)

**Status:** mitigated 2026-05-22 by adding `plugin_habits_skipped: elfensky/elfensky`
to the three `habits` steps in `.github/workflows/metrics.yml`.

**Symptom:** `metrics.plugin.habits.svg`, `.habits.facts.svg`, and
`.habits.charts.svg` all rendered `Unexpected error`.

**Root cause:** the habits plugin flat-maps commits out of recent push events and
runs `.filter(({author}) => …)` with no null-guard:

```
TypeError: Cannot destructure property 'author' of 'undefined'
   at .../source/plugins/habits/index.mjs:51
```

Squashing/force-pushing this repo's `main` left a push event in the GitHub
activity feed whose commits no longer resolve, so one entry is `undefined` and
the destructure throws. All three habits SVGs share the same event fetch, so all
three fail identically.

**Mitigation:** repository skipping runs *before* the crashing filter, so
`plugin_habits_skipped: elfensky/elfensky` removes this repo's events from the
habits analysis and avoids the bad entry. This is also correct on its own —
profile-repo README commits aren't meaningful "coding habits". The error would
self-heal anyway once the force-push event ages out of GitHub's ~90-day events
window.

**Upstream tracking:**
- PR (open): lowlighter/metrics#1807 — guard against null commit entries in
  PushEvent payload
- PRs (closed, unmerged): #1739, #1764, #1778 — earlier null-guard attempts

**Fix path:** once an upstream null-guard merges + releases, the
`plugin_habits_skipped` line can stay (still desirable) or be removed if you want
this repo counted toward habits.
