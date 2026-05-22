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

## Habits plugin — disabled (broken upstream)

**Status:** the three `habits` steps in `.github/workflows/metrics.yml` are
commented out as of 2026-05-22, and the habits image was removed from `README.md`.

**Symptom:** `metrics.plugin.habits.svg`, `.habits.facts.svg`, and
`.habits.charts.svg` all rendered `Unexpected error`.

**Root cause:** the habits plugin builds a commit list with
`.flatMap(({payload}) => payload.commits)` and then immediately destructures
`author` from each entry, with no null-guard:

```
TypeError: Cannot destructure property 'author' of 'undefined'
   at .../source/plugins/habits/index.mjs:51
```

GitHub's Events API returns `null`/`undefined` entries inside a PushEvent's
`commits` array — for deleted author accounts, force-pushed/orphaned commits, or
sparse API responses. The `patches` block that hits this runs unconditionally,
so all three habits SVGs fail identically regardless of the `facts`/`charts`
flags.

**Not fixable via workflow config.** `plugin_habits_skipped` only filters whole
events *by repository* (before the flatMap); it cannot remove a `null` nested
inside a kept event's `commits` array. The fix must be in the action's source.

**Upstream tracking:**
- PR (open): lowlighter/metrics#1807 — guard against null commit entries in
  PushEvent payload (adds `.filter(commit => commit)` before the destructure)
- PR (open): lowlighter/metrics#1769 — also improves habits error handling
- PRs (closed, unmerged): #1739, #1764, #1778 — earlier null-guard attempts

## Re-enabling either plugin

`metrics`' last release is **v3.34 (Sept 2023)** — no release in 2.5+ years, and
the fixes above are all still open/unmerged. Options, in rough order of effort:

1. Wait for an upstream fix to merge **and** be released, then un-comment the
   steps and bump the `lowlighter/metrics@` version.
2. Pin the affected steps to a reviewed commit SHA of a fork carrying the fixes
   — e.g. PR #1769 (`dkhokhlov/metrics`) patches *both* plugins. Tradeoff: runs
   third-party action code with `METRICS_TOKEN` (repo scope).
3. Fork `lowlighter/metrics` yourself, apply the small patches (a one-line
   null-guard for habits; the Projects-V2 query for achievements), pin to your
   own fork's SHA.
4. Drop the plugins permanently.
