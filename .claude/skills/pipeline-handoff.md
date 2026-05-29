# Sprint Handoff Report

Produce a sprint handoff report for the incoming rotation engineer. Aggregates pipeline health trends, known failures, and go/no-go decisions from the sprint period.

**Required environment variable:** `CURRENTS_PROJECT_ID` — the Currents project ID. Must be set in your shell or in `.claude/settings.local.json` env config.

---

## 1. Parse Input

Read `CURRENTS_PROJECT_ID` from the environment. If not set, stop and display: "Set the `CURRENTS_PROJECT_ID` environment variable. See `.claude/SETUP.md` for instructions."

- If the user passed a version argument (e.g., `2.7`), set `VERSION_FILTER` to that value.
- If no argument was passed, analyze both versions (2.6 and 2.7).
- If the user passed `--sprint-start <YYYY-MM-DD>`, set `SPRINT_START` to that date in ISO 8601 format.
- If not passed, default to 14 days ago in ISO 8601 format.
- Set `SPRINT_END` to today's date in ISO 8601 format.
- Compute `SPRINT_DAYS` as the number of days between `SPRINT_START` and `SPRINT_END`.

Branch mapping (same as pipeline-health):

| Version | Currents Branch |
|---------|----------------|
| 2.6     | `stable-2.6`   |
| 2.7     | `devel`        |

---

## 2. Fetch Sprint Data

Issue all independent API calls in a single parallel batch.

### Parallel Batch 1 — All of the following at once:

**2a. Project Insights (pipeline trend)**

For each branch matching `VERSION_FILTER`, call `currents-get-project-insights` with:
- `projectId`: `${CURRENTS_PROJECT_ID}`
- `date_start`: `SPRINT_START`
- `date_end`: `SPRINT_END`
- `branches`: the branch name
- `resolution`: `1d`

This returns a `timeline` array with daily buckets containing `runs`, `passed`, `failed`, `flaky`, `pending`, `skipped` counts.

**2b. Test Performance (known failures)**

For each branch matching `VERSION_FILTER`, call `currents-get-tests-performance` with:
- `projectId`: `${CURRENTS_PROJECT_ID}`
- `date_start`: `SPRINT_START`
- `date_end`: `SPRINT_END`
- `branches`: the branch name
- `order`: `failures`
- `dir`: `desc`
- `test_state`: `["failed"]`
- `limit`: `50`

**2c. Latest Runs (current snapshot)**

For each branch matching `VERSION_FILTER`, call `currents-get-runs` with:
- `projectId`: `${CURRENTS_PROJECT_ID}`
- `branches`: the branch name
- `completion_state`: `["COMPLETE", "TIMEOUT"]`
- `date_start`: 48 hours ago in ISO 8601
- `limit`: `20`

---

## 3. Parse Latest Runs for Current Snapshot

From the `currents-get-runs` results (Step 2c), parse the ciBuildId of each run to extract build metadata.

### ciBuildId Pattern

Same patterns as pipeline-health:

- **Next builds**: `AAP_{version}_Next-Product_Build_CI` (contains `_Next`)
- **Stable builds**: `AAP_{version}-Product_Build_CI` (no `_Next`)

Extract: build type (Next/Stable), version, topology, build number.

### Topology Display Names

| Topology | Display Name |
|----------|-------------|
| `rpm-b`  | RPM B       |
| `saas`   | SaaS        |
| `cont-b` | Container B |
| `ocp-a`  | OCP A       |
| `man-b`  | Managed B   |

### Deduplicate

If multiple runs exist for the same build type + version + topology, keep only the run with the highest build number.

### Fetch Run Details (Parallel Batch 2)

For each deduplicated run, call `currents-get-run-details` with the run's `runId`. Run these in parallel.

Compute per run:

| Metric     | Formula                   |
|------------|---------------------------|
| Total      | Total test count          |
| Passed     | Tests with status passed  |
| Failed     | Tests with status failed  |
| Pending    | Tests with status pending |
| Actionable | Total - Pending           |
| Pass Rate  | Passed / Actionable * 100 |

The **98% threshold** determines PASS/FAIL status.

---

## 4. Classify Known Failures

From the `currents-get-tests-performance` results (Step 2b), classify each failing test.

### Categorize as Visual or Integration

- **Visual**: spec file path contains `tests/visual/` OR error message contains `toHaveScreenshot` or `screenshot`
- **Integration**: everything else

### Classify Age

| Classification | Criteria |
|---------------|----------|
| **NEW** | Failure rate < 15% AND fewer than 3 historical failures, OR test not found in data |
| **RECURRING** | Failure rate between 15% and 85% |
| **CHRONIC** | Failure rate > 85% with 10+ executions |

Edge case: if a test has fewer than 5 executions and a high failure rate, classify as NEW.

### Cross-Reference with Open PRs

For the top 5 chronic/recurring failures (by failure count), attempt to find open PRs that address them:

```bash
gh pr list --repo ansible/ansible-ui --state open --search "<spec-file-name>" --json number,title,url --limit 3
```

If a matching open PR is found, record its number for the report. This is best-effort — no match means `—` in the Active PR column.

---

## 6. Generate Report

Output the report in this exact format:

```
## Sprint Handoff Report — {SPRINT_START} to {SPRINT_END}

**Sprint duration:** {SPRINT_DAYS} days
**Versions:** {VERSION_FILTER or "2.6, 2.7"}
```

### Sprint Summary

Generate a 2-3 sentence overview covering:
1. Overall pipeline health trajectory — did pass rates improve, degrade, or hold steady over the sprint?
2. The biggest issue or blocker during the sprint period.
3. Key accomplishment or resolution if any (e.g., "Visual baseline PR merged, eliminating N failures").

Derive the trend by comparing average pass rate in the first half of the sprint vs the second half (from project-insights timeline data).

### Pipeline Trend

For each version, output a daily pass rate table from the project-insights timeline data:

```
### Pipeline Trend ({SPRINT_DAYS} Days)

#### 2.7 (devel)

| Date   | Runs | Pass Rate | Failed | Flaky |
|--------|------|-----------|--------|-------|
| May 28 | 3    | 95.8%     | 19     | 4     |
| May 27 | 5    | 95.4%     | 21     | 5     |
| ...    | ...  | ...       | ...    | ...   |

**Trend:** Average pass rate moved from {first-half-avg}% (week 1) to {second-half-avg}% (week 2). {improving/degrading/stable}.
```

Compute pass rate per day: `passed / (passed + failed) * 100`. Omit days with 0 runs.

Trend direction: "improving" if second-half avg > first-half avg by > 0.5pp. "degrading" if lower by > 0.5pp. "stable" otherwise.

### Current Pipeline Snapshot

For each build type + version combination with runs in the last 48 hours, output a table. Order: 2.6 Stable, 2.6 Next, 2.7 Stable, 2.7 Next.

```
### Current Pipeline Snapshot

#### {version} {type} ({branch})

| Topology    | Pass Rate          | Failures (V/I) | Status |
|-------------|--------------------|-----------------|--------|
| SaaS        | 98.61% (461/474)   | 6 (4V / 2I)    | PASS   |
| OCP A       | 97.45% (380/390)   | 10 (9V / 1I)   | FAIL   |

_Threshold: 98%. Data from last 48 hours._
```

If no runs found for a build type + version, note: "No recent runs found."

### Known Failures & Open Issues

Split into Integration and Visual tables. Show top failures from the sprint period, sorted by failure count descending.

```
### Known Failures & Open Issues

#### Integration Failures

| # | Test | Spec | Failure Rate | Executions | Age | Active PR |
|---|------|------|-------------|------------|-----|-----------|
| 1 | jobs: inventory sync timeout | jobs.spec.ts | 100% | 18 | CHRONIC | — |
| 2 | ... | ... | ... | ... | ... | ... |

#### Visual Failures

| # | Test | Spec | Failure Rate | Executions | Age | Active PR |
|---|------|------|-------------|------------|-----|-----------|
| 1 | overview page screenshot | overview-visual.spec.ts | 100% | 18 | CHRONIC | — |
| 2 | ... | ... | ... | ... | ... | ... |

_{N} chronic, {M} recurring, {P} new failures across the sprint.
```

Limit to top 15 integration failures and top 10 visual failures to keep the report scannable. Show the spec file name without the full path (e.g., `jobs.spec.ts` not `tests/integration/.../jobs.spec.ts`).

### Action Items for Next Engineer

Generate a prioritized list derived from all the data collected:

```
### Action Items for Next Engineer

1. **Investigate {test/spec}** — {failure rate}% failure rate across the sprint, {age}. Use `/pipeline-triage {spec}`.
2. **Update visual baselines** — {N} chronic visual failures across all topologies. Run `/pipeline-triage {spec} --fix`.
3. **Monitor {version} trend** — pass rates {trending direction} over the last week. {specific concern}.
4. **{Any other item from upcoming milestones or sprint context}**
```

Rules for generating action items:
- Chronic failures come first — prioritize by failure count.
- Visual baseline updates are a single grouped item if there are multiple visual failures.
- Trend concerns only if pass rates are degrading (> 0.5pp drop week-over-week).
- Limit to 5-7 action items. More than that is not actionable.

---

## Report Rules

- **Status**: `PASS` if pass rate >= 98%, `FAIL` if below
- **V/I**: V = visual failure count, I = integration failure count
- **Pass Rate**: Show as percentage with fraction in parentheses, e.g., `98.61% (461/474)`
- **Topology display names**: RPM B, SaaS, Container B, OCP A, Managed B
- **Age tags**: `NEW`, `RECURRING`, `CHRONIC` — same thresholds as pipeline-health
- **Date format**: "May 22" in tables for readability, YYYY-MM-DD in headers
- **Active PR column**: PR number if found, `—` otherwise
- **Trend direction**: "improving" if second-half avg > first-half avg by > 0.5pp, "degrading" if lower by > 0.5pp, "stable" otherwise
- If only one version requested, omit the other version's sections
- Keep the report under ~200 lines for 5-minute readability
