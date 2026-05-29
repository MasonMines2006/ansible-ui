# Pipeline Health Check

Analyze the latest nightly Jenkins CI runs from Currents and produce a standup-ready pipeline health report.

**Required environment variable:** `CURRENTS_PROJECT_ID` — the Currents project ID. Must be set in your shell or in `.claude/settings.local.json` env config.

---

## 1. Parse Input

Read `CURRENTS_PROJECT_ID` from the environment. If not set, stop and display: "Set the `CURRENTS_PROJECT_ID` environment variable. See `.claude/SETUP.md` for instructions."

- If the user passed a version argument (e.g., `2.7`), set `VERSION_FILTER` to that value.
- If no argument was passed, analyze both versions (2.6 and 2.7).
- If the user passed a build type argument (`next`, `stable`, or `all`), set `BUILD_TYPE_FILTER` to that value.
- Default: `BUILD_TYPE_FILTER` is `all` (show both Next and Stable builds when available).
- If the user passed `--no-trend`, set `SKIP_TREND` to `true`. When set, skip all previous-day data fetching (Steps 2b, 3b, 4b) and omit the Delta column from the report. This halves the number of API calls.
- Default: `SKIP_TREND` is `false` (trend comparison is enabled).
- If the user passed `--no-history`, set `SKIP_HISTORY` to `true`. When set, skip historical failure classification (Step 5c) and omit the Age column from failures. This avoids the `currents-get-tests-performance` API call.
- Default: `SKIP_HISTORY` is `false` (historical classification is enabled).
- If the user passed `--signoff`, set `SIGNOFF_MODE` to `true`. If a JIRA ticket key follows (e.g., `--signoff AAP-54321`), set `JIRA_TICKET` to that value; otherwise set `JIRA_TICKET` to `null`.
- When `SIGNOFF_MODE` is true:
  - Force `BUILD_TYPE_FILTER` to `stable` (verdict is based on Stable builds only).
  - Force `SKIP_TREND` to `true` (reduce API calls for verdict focus).
  - Force `SKIP_HISTORY` to `false` (age classification is required for the verdict).
- Default: `SIGNOFF_MODE` is `false` (no verdict section in report).

---

## 2. Discover Runs

Use `currents-get-runs` with **projectId `${CURRENTS_PROJECT_ID}`** to find recent completed runs.

### Build types: Next vs Stable

Both **Next** and **Stable** (Product) builds report to the same Currents branch. They are distinguished by the ciBuildId pattern:

- **Next builds**: `AAP_{version}_Next-Product_Build_CI` (contains `_Next`)
- **Stable builds**: `AAP_{version}-Product_Build_CI` (no `_Next`)

The branch-to-version mapping:

| Currents Branch | Contains                     |
|-----------------|------------------------------|
| `stable-2.6`    | 2.6 Next + 2.6 Stable builds |
| `devel`          | 2.7 Next + 2.7 Stable builds |

**Note:** 2.5 builds are not available in this Currents project (`${CURRENTS_PROJECT_ID}`). If 2.5 is requested, note: "2.5 builds are not available in Currents."

Query each branch separately based on `VERSION_FILTER`:

For each branch, call `currents-get-runs` with:
- `projectId`: `${CURRENTS_PROJECT_ID}`
- `branches`: the branch name
- `completion_state`: `["COMPLETE", "TIMEOUT"]`
- `date_start`: 24 hours ago in ISO 8601 format
- `limit`: `20`

The returned runs will contain **both** Next and Stable builds mixed together. These are separated in Step 3 using ciBuildId parsing.

If no runs are found for a version, note it in the report: "No runs found for {version} in the last 24 hours."

### 2b. Discover Previous-Day Runs (skip if `SKIP_TREND` is true)

For each branch matching `VERSION_FILTER`, call `currents-get-runs` with:
- `projectId`: `${CURRENTS_PROJECT_ID}`
- `branches`: the branch name
- `completion_state`: `["COMPLETE", "TIMEOUT"]`
- `date_start`: 48 hours ago in ISO 8601 format
- `date_end`: 24 hours ago in ISO 8601 format
- `limit`: `20`

These calls can run **in parallel with the today-window calls** from Step 2 (all four discovery calls at once: today-2.6, today-2.7, prev-2.6, prev-2.7).

If no previous-day runs are found for a version, trend deltas for that version will show `---` in the report.

---

## 3. Parse Build Metadata

Each run has a `ciBuildId` that follows one of two patterns:

**Next builds:**
```
jenkins-AAPQA-AAP_{version}_Next-Product_Build_CI-tier1-{topology}.fresh-install.ui-playwright-{number}
```

**Stable (Product) builds:**
```
jenkins-AAPQA-AAP_{version}-Product_Build_CI-tier1-{topology}.fresh-install.ui-playwright-{number}
```

Extract from each `ciBuildId`:
- **Build type**: `Next` if the ciBuildId contains `_Next-Product`, otherwise `Stable`
- **Version**: the `{version}` segment (e.g., `2.6`, `2.7`)
- **Topology**: the `{topology}` segment (e.g., `rpm-b`, `saas`, `cont-b`, `ocp-a`, `man-b`)
- **Build number**: the trailing `{number}`

Map topologies to display names:

| Topology | Display Name |
|----------|-------------|
| `rpm-b`  | RPM B       |
| `saas`   | SaaS        |
| `cont-b` | Container B |
| `ocp-a`  | OCP A       |
| `man-b`  | Managed B   |

Group runs by **build type** (Next / Stable), then by **version**, then by **topology**.

If `BUILD_TYPE_FILTER` is `next`, discard all Stable runs. If `stable`, discard all Next runs. If `all`, keep both.

If a `ciBuildId` doesn't match the expected pattern, include it with topology "Unknown" and log a note.

### 3b. Parse Previous-Day Build Metadata (skip if `SKIP_TREND` is true)

Apply the same ciBuildId parsing from Step 3 to previous-day runs.

Group previous-day runs by build type, version, then topology — stored separately from today's runs (e.g., `prev_runs` vs `today_runs`).

**Deduplication**: If multiple runs exist for the same build type + version + topology in the previous-day window, keep only the run with the **highest build number** (most recent nightly).

---

## 4. Fetch Run Details

For each discovered run, call `currents-get-run-details` with the run's `runId`.

**Run these calls in parallel** where possible (use multiple tool calls in one message).

From each run's details, compute:

| Metric     | Formula                        |
|------------|--------------------------------|
| Total      | Total test count               |
| Passed     | Tests with status `passed`     |
| Failed     | Tests with status `failed`     |
| Pending    | Tests with status `pending`    |
| Flaky      | Tests marked as flaky          |
| Actionable | Total - Pending                |
| Pass Rate  | Passed / Actionable * 100      |

The **98% threshold** determines pass/fail status for each run.

### 4b. Fetch Previous-Day Run Details (skip if `SKIP_TREND` is true)

For each previous-day run (after deduplication in Step 3b), call `currents-get-run-details` with the run's `runId`.

Compute the same metrics as Step 4 (Total, Passed, Failed, Pending, Flaky, Actionable, Pass Rate).

**Parallelism**: These calls can run **in the same parallel batch** as today's detail calls from Step 4. Issue all detail calls (today + previous-day) together.

Do NOT fetch failure context (Step 5) for previous-day runs — only the pass rate is needed for trend comparison.

---

## 5. Fetch Failure Context

For runs that have failures, call `currents-get-context` to get failure details:

- `run_id`: the run's ID
- `format`: `"md"`
- `detail`: `"compact"`
- `limit`: `50`

**Run these calls in parallel** where possible.

### Categorize Each Failure

Classify every failed test as either **Visual** or **Integration**:

- **Visual**: The spec file path contains `tests/visual/` OR the error message contains `toHaveScreenshot` or `screenshot`
- **Integration**: Everything else

### Extract Error Themes

For each failure, note the error type:
- Timeout errors (`Timeout`, `exceeded`)
- Assertion errors (`toBeVisible`, `toContainText`, `toHaveText`, `toHaveScreenshot`)
- Navigation errors (`ERR_CONNECTION`, `page.goto`)
- Other (capture first line of error)

After categorizing failures, retain the mapping of **test title → list of (version, topology) pairs where it failed**. This mapping is needed for the Failure × Topology Matrix in Step 6.

---

## 5c. Classify Failure Age (skip if `SKIP_HISTORY` is true)

Use `currents-get-tests-performance` to determine whether each failure is new, recurring, or chronic.

### 5c.1 — Fetch 7-Day Test Performance

Call `currents-get-tests-performance` with:
- `projectId`: `${CURRENTS_PROJECT_ID}`
- `date_start`: 7 days ago in ISO 8601 format
- `date_end`: now in ISO 8601 format
- `branches`: the branches matching `VERSION_FILTER` (e.g., `["devel"]` for 2.7)
- `order`: `failures`
- `dir`: `desc`
- `test_state`: `["failed"]`
- `limit`: `50`

This call can run **in parallel with Steps 4/5** since it has no dependencies on run details.

If analyzing both versions, make one call per branch (in parallel).

### 5c.2 — Match Failures to Historical Data

For each test title that failed in today's runs (from Step 5), find its entry in the 7-day performance data by matching the test `signature` or `title + spec` combination.

### 5c.3 — Classify Each Failure

Assign an age classification based on the 7-day failure rate and execution count:

| Classification | Criteria | Meaning |
|---------------|----------|---------|
| **NEW** | Failure rate < 15% AND fewer than 3 historical failures, OR test not found in 7-day data | First appeared today or very recently — highest urgency, likely a real regression |
| **RECURRING** | Failure rate between 15% and 85% | Intermittent / flaky — worth stabilizing but not a new regression |
| **CHRONIC** | Failure rate > 85% with 10+ executions | Failing in nearly every run for days — known broken, not urgent unless blocking release |

Edge cases:
- If a test has very few executions (< 5) and high failure rate, classify as **NEW** — it hasn't been running long enough to be chronic
- If `SKIP_HISTORY` is true, omit the classification entirely (don't guess)

### 5c.4 — Identify Newly Fixed Tests

From the 7-day performance data, identify tests that:
- Had failures in the first half of the 7-day window (days 1-4)
- Have **0 failures in today's runs**
- Had a failure rate > 50% historically

These are candidates for "Recently Fixed" — tests that were broken but appear to have been resolved. Note these for the report but don't over-report (only include tests with > 50% historical failure rate to filter out flakes that happened to pass today).

---

## 6. Cross-Run Analysis

### Consistent Failures

Identify test titles that appear as failures in **3 or more runs**. These are systemic issues, not flakes.

For each consistent failure, note:
- The test title
- Category (Visual / Integration)
- How many runs it failed in (e.g., "6/8 runs")
- The common error theme

### Version Comparison

If analyzing both versions, compare 2.7 failures against 2.6:
- **New in 2.7**: Test titles that fail in 2.7 runs but NOT in any 2.6 run — these are regressions on `devel`
- **Fixed in 2.7**: Test titles that fail in 2.6 but pass in all 2.7 runs

### Impact Analysis

Calculate what the pass rates would be if visual failures were excluded:
- For each run: `adjusted_pass_rate = passed / (actionable - visual_failures) * 100`
- This shows whether fixing visual tests alone would meet the 98% target

### Trend Comparison (skip if `SKIP_TREND` is true)

For each topology in today's runs, find its matching previous-day run by **build type + version + topology**.

Compute: `delta = today_pass_rate - prev_pass_rate`

Format the delta as:
- `+X.X%` if positive (improvement)
- `-X.X%` if negative (regression)
- `---` if no matching previous-day run exists

Flag any topology where delta is **worse than -2.0%** as a notable regression — this will be surfaced in Action Items.

### Failure × Topology Matrix

Build a cross-reference matrix of failures against topologies:

1. Collect all test titles that failed in **2 or more runs** (across any version/topology combination).
2. For each such failure, record which (version, topology) pairs it appeared in.
3. Sort failures by number of topologies hit (most widespread first).
4. **Limit to the top 20 failures** to keep the report scannable.
5. Include the failure category (V for Visual, I for Integration) as a column.

A failure that hits all topologies is almost certainly an infrastructure or test-code issue, not a product bug. A failure hitting only 1-2 topologies suggests a topology-specific product issue. Note this distinction in the Action Items.

### Impact Scoring

Score each failing **spec file** (not individual test) to produce a ranked priority list. Group all failing tests by their spec file path, then compute:

```
impact_score = failed_test_count × topologies_hit × urgency_multiplier
```

Where:
- `failed_test_count`: number of distinct test titles failing in this spec
- `topologies_hit`: number of distinct topologies where this spec has failures (1–5)
- `urgency_multiplier`: based on failure age classification from Step 5c:
  - **NEW**: `3.0` — new regressions get highest priority
  - **RECURRING**: `1.5` — flaky tests that are getting worse
  - **CHRONIC**: `1.0` — known issues, important but not urgent
  - If `SKIP_HISTORY` is true, use `1.0` for all (no age data available)

If a spec has mixed classifications (e.g., one NEW test and two CHRONIC tests), use the **highest urgency multiplier** for the spec — a spec with any new failure should be investigated first.

Sort specs by `impact_score` descending. This ranking drives the Action Items section in the report.

**Example:**

| Spec | Failed Tests | Topos | Age | Score |
|------|-------------|-------|-----|-------|
| overview.spec.ts | 2 | 3 | CHRONIC | 2 × 3 × 1.0 = 6.0 |
| event-persistence.spec.ts | 4 | 3 | RECURRING | 4 × 3 × 1.5 = 18.0 |
| new-feature.spec.ts | 1 | 2 | NEW | 1 × 2 × 3.0 = 6.0 |

In this example, event-persistence would rank first despite being recurring, because it has the most total failures to eliminate.

---

## 7. Generate Report

Output the report in this exact format:

```
## Pipeline Health — {YYYY-MM-DD}
```

### Summary

Generate a 2-4 sentence executive summary at the top of the report. It should cover:

1. **Overall health**: How many runs were analyzed, how many passed the 98% threshold, and whether any timed out.
2. **Trend**: Whether results improved, worsened, or held steady vs the previous day (skip if `SKIP_TREND` is true).
3. **Top blocker**: The single biggest contributor to failures (e.g., "Visual snapshot mismatches account for X% of all failures" or "RBAC 403 errors drive Y failures across Z topologies").

Example:

> **Summary:** Analyzed 10 runs across 2.6 and 2.7 — 0 of 10 met the 98% threshold and all timed out at 90 minutes. 2.7 Next pass rates improved slightly (+0.5pp avg) vs yesterday, while 2.6 Stable SaaS/Managed B remain degraded. Visual snapshot mismatches account for 38-47% of UI failures; fixing them would bring 2.7 Next to ~97.5-97.9%.

Keep the summary factual and concise — no action items here (those go at the bottom).

### Release Verdict (only if `SIGNOFF_MODE` is true)

Apply these rules to produce a GO / NO_GO verdict using the Stable build data computed in Steps 4-6. Skip this section entirely if `SIGNOFF_MODE` is false.

**Blocking rules (any triggers NO_GO):**

- Any Stable topology has a pass rate below 98%
- Any NEW integration failure appears on 2+ Stable topologies
- Any SERVER_ERROR or AUTH_RBAC error type on 2+ Stable topologies that is not CHRONIC

**Non-blocking (does NOT prevent GO):**

- CHRONIC failures (>85% failure rate with 10+ executions)
- Visual-only failures (screenshot mismatches)
- RECURRING failures (5-85% failure rate)

**Verdict:** `GO` if zero blockers, `NO_GO` if any blockers exist.

Output:

```
### Release Verdict: {version} — {GO / NO_GO}

**Threshold:** 98% pass rate, no NEW regressions
**Verdict:** {GO / NO_GO}
**Reason:** {one-line reason}

| Topology | Pass Rate | Status | Blocker |
|----------|-----------|--------|---------|
| SaaS     | 98.61%    | PASS   | —       |
| OCP A    | 97.45%    | FAIL   | Below 98% |
```

If `JIRA_TICKET` is set, after displaying the full report:
1. Post the Release Verdict section as a JIRA comment via `addCommentToJiraIssue` MCP with `cloudId: "redhat.atlassian.net"`, `issueIdOrKey: "{JIRA_TICKET}"`, `contentFormat: "markdown"`.
2. Prepend: "> **Note:** This verdict was generated by `/pipeline-health --signoff`."
3. If posting succeeds, confirm: "Verdict posted to {JIRA_TICKET}."

If `JIRA_TICKET` is null, after displaying the report say: "Dry run. To post to JIRA: `/pipeline-health {version} stable --signoff AAP-XXXXX`"

### Pass Rate Tables

Generate one section per **build type + version** combination that has runs. Use this heading format:

- `### 2.6 Next (stable-2.6)`
- `### 2.6 Stable (stable-2.6)`
- `### 2.7 Next (devel)`
- `### 2.7 Stable (devel)`

Order sections: 2.6 Stable → 2.6 Next → 2.7 Stable → 2.7 Next (stable before next, lower version first).

If a build type has no runs in the last 24 hours, note: "No {version} {type} runs found in the last 24 hours. Last run: {date}." and omit the table.

Each section contains:

```
### {version} {type} ({branch})

| Build Type   | Build # | Pass Rate        | Delta   | Failures (V/I) | Status |
|--------------|---------|------------------|---------|-----------------|--------|
| SaaS         | #39     | 98.61% (XXX/YYY)| +0.4%   | 6 (4V / 2I)    | PASS   |
| RPM B        | #42     | 97.45% (XXX/YYY)| -1.2%   | 10 (9V / 1I)   | FAIL   |
| ...          | ...     | ...              | ...     | ...             | ...    |

### Consistent Failures (across 3+ runs)

- **[INTEGRATION] {test title}** — {error theme} ({N}/{total} runs) `{AGE}`
- **[VISUAL] {test title}** — {error theme} ({N}/{total} runs) `{AGE}`

Where `{AGE}` is one of: `NEW`, `RECURRING`, `CHRONIC`. Omit the age tag if `SKIP_HISTORY` is true.
Sort: NEW first, then RECURRING, then CHRONIC. Within each group, sort by number of affected runs descending.

### Failure × Topology Matrix

_Failures appearing in 2+ runs, sorted by spread. Top 20 shown._

| Failure                          | Cat | SaaS | RPM | OCP | Cont | Man |
|----------------------------------|-----|------|-----|-----|------|-----|
| jobs: inventory sync timeout     | I   |  x   |  x  |  x  |  x   |  x  |
| automation-dashboard nav missing | I   |      |     |  x  |  x   |     |
| viewport mismatch screenshots    | V   |  x   |  x  |  x  |  x   |  x  |

### New in 2.7 (not failing in 2.6)

- {test title} — {error summary}

### Priority Ranking

_Specs ranked by impact score (failed tests × topologies × urgency). Fix from top to bottom for maximum impact._

| # | Spec | Tests | Topos | Age | Impact | Est. Failures Eliminated |
|---|------|-------|-------|-----|--------|--------------------------|
| 1 | {spec file} | {N} | {M} | {NEW/RECURRING/CHRONIC} | {score} | {N × M} |
| 2 | ... | ... | ... | ... | ... | ... |

Omit this section if `SKIP_HISTORY` is true (the Age and Impact columns require historical data; without them, this table adds no value over the Failure × Topology Matrix). The "Est. Failures Eliminated" column shows `failed_test_count × topologies_hit` — the raw number of test failures removed per nightly run if this spec is fully fixed.

### Recently Fixed (skip if `SKIP_HISTORY` is true)

Tests that had >50% failure rate in the past 7 days but passed in all of today's runs:

- **{test title}** — was failing {X}% over last 7 days, now passing

If none, show: "No recently fixed tests detected." If `SKIP_HISTORY` is true, omit this section entirely.

### Impact: If Visual Tests Were Fixed

| Version | Current Avg Pass Rate | Without Visual Failures |
|---------|-----------------------|-------------------------|
| 2.6     | XX.XX%                | XX.XX%                  |
| 2.7     | XX.XX%                | XX.XX%                  |

### Action Items

1. {Highest-impact action item}
2. {Second priority}
3. ...

### Daily Action Plan

_Ready-to-run commands based on today's failures, ordered by impact._

**Quick wins (auto-fixable):**
- `/pipeline-triage {spec} --fix` — {reason}
- ...

**Investigate first:**
- `/pipeline-triage {spec}` — {reason}
- ...

**Product issues (file Jira):**
- `{spec}` — {reason}
- ...
```

### Report Rules

- **Status column**: `PASS` if pass rate >= 98%, `FAIL` if below
- **V/I**: V = visual failure count, I = integration failure count
- **Build #**: The Jenkins build number extracted from the ciBuildId (the trailing `{number}`), prefixed with `#` (e.g., `#42`)
- **Pass Rate**: Show as percentage with the fraction in parentheses, e.g., `98.61% (461/474)`
- **Delta column**: Shows pass rate change vs previous day. Omit this column entirely if `SKIP_TREND` is true. Show `---` if no previous-day data exists for that topology. Append `!!` for deltas worse than -2.0% (e.g., `-3.1% !!`)
- **Consistent Failures**: Sort by number of affected runs (most widespread first)
- **Failure × Topology Matrix**: Use `x` for topologies where the failure occurred, blank for unaffected. If analyzing both versions, include topologies from both. Show "No cross-topology failures detected." if no failures appear in 2+ runs
- **Action Items**: Derive from the Priority Ranking table (Step 6). Each action item corresponds to the top-ranked specs by impact score — the spec with the highest score becomes action item #1. For each, state the spec name, how many failures it would eliminate, and its age classification. Additionally:
  - If any topology shows a delta regression worse than -2.0%, add: "Investigate {topology} regression: pass rate dropped {delta} vs previous day"
  - If a failure hits all topologies in the matrix, add: "Fix {test title} — affects all topologies, likely infrastructure or test-code issue"
  - If a failure hits only Kubernetes-based topologies (OCP, Container), note: "{test title} — Kubernetes-specific, check OCP/Container deployment differences"
- **Daily Action Plan**: Pre-classify each spec from the Priority Ranking into action buckets using data already computed in Steps 5-6. Apply the rules below in order (first match wins) to determine the bucket and whether `--fix` is appropriate. Use just the spec file name without path (e.g., `oauth-applications.spec.ts`). Limit to 8 items total across all buckets, prioritized by impact score.

  **Quick wins (auto-fixable)** — use `/pipeline-triage {spec} --fix`:
  - VISUAL error type (any age): `--fix` will update baselines
  - CHRONIC + ALL_TOPOLOGIES spread: `--fix` will quarantine
  - CHRONIC + SINGLE_TOPOLOGY spread: `--fix` will skip (topology-specific)
  - RECURRING + TIMEOUT error type: `--fix` will stabilize selectors/waits

  **Investigate first** — use `/pipeline-triage {spec}` (no `--fix`):
  - NEW + any spread: needs root cause analysis before acting
  - RECURRING + ASSERTION error type: may need manual judgment

  **Product issues (file Jira)** — no `/pipeline-triage` command, just note the spec:
  - AUTH_RBAC error type on 2+ topologies: permissions likely changed
  - SERVER_ERROR error type on 2+ topologies: backend returning errors
  - NEW + non-test changes on 2+ topologies: product regression

  For each item, include a short reason fragment (e.g., "CHRONIC visual, update baselines" or "NEW regression, correlates with PR #3184"). If `SKIP_HISTORY` is true, omit age-based classification and list all specs as "Investigate first" since we can't determine quick wins without history data.
- If only one version was requested, omit the other version's sections and the version comparison
- If there are no failures for a category, note "None" instead of an empty section
- If a run timed out, append "(timed out)" to its Build Type label
