---
name: debug-github-actions
description: >
  Debug failing GitHub Actions builds. Covers triage with the gh CLI,
  parsing large log files, downloading and analyzing build artifacts,
  extracting build report summaries, and identifying flaky tests via
  Surefire XML markers or Develocity.
---

# Debugging GitHub Actions Builds

Use this skill when investigating a failing GitHub Actions CI build,
given a run URL or a repo + run ID.

## 1. Quick Triage

```bash
gh run view <run-id> --repo <owner>/<repo>                    # overview + job list
gh run view <run-id> --repo <owner>/<repo> -v                 # show per-job steps
```

List only the failed jobs:

```bash
gh api "repos/<owner>/<repo>/actions/runs/<run-id>/jobs?per_page=100" \
  --jq '.jobs[] | select(.conclusion == "failure") | {name, conclusion, steps: [.steps[] | select(.conclusion == "failure") | .name]}'
```

## 2. Reading Failed Logs

`gh` has **no built-in log filter/grep**. Failed-job logs can exceed
50 000 lines and often get truncated by GitHub. Always save to a file
first, then search:

```bash
gh run view <run-id> --repo <owner>/<repo> --log-failed > /tmp/failed-logs.txt
```

If the output is truncated (cuts off mid-line without a clear ending),
download the full log for a specific job instead:

```bash
gh run view <run-id> --repo <owner>/<repo> --log --job <job-id> > /tmp/job-log.txt
```

### Useful grep patterns

```bash
# Maven build failure summary
grep -n "BUILD FAILURE\|Reactor Summary\|\[ERROR\].*FAILURE" /tmp/failed-logs.txt

# Surefire/Failsafe test failure counts
grep -n "Tests run:.*Failures: [1-9]\|Tests run:.*Errors: [1-9]" /tmp/failed-logs.txt

# Specific failing test names (Surefire output format)
grep -n "<<< FAILURE!\|<<< ERROR!" /tmp/failed-logs.txt

# Gradle task failures
grep -n "> Task .*FAILED\|BUILD FAILED" /tmp/failed-logs.txt

# Process exit codes
grep -n "Process completed with exit code\|exit code" /tmp/failed-logs.txt
```

For Maven builds, the **Reactor Summary** near the end of the log gives
a per-module pass/fail overview — search for it first to narrow down
which module failed before diving into individual test output.

## 3. Artifacts

### Listing and downloading

```bash
# List all artifacts
gh api "repos/<owner>/<repo>/actions/runs/<run-id>/artifacts" \
  --jq '.artifacts[] | {name, id, size_in_bytes}'

# Download a specific artifact by ID
gh api "repos/<owner>/<repo>/actions/artifacts/<artifact-id>/zip" > /tmp/artifact.zip
```

### Common artifact naming patterns

Not every project uses all of these — check what's actually present:

| Pattern | Content |
|---|---|
| `build-reports-*` | Surefire/Failsafe XML (`TEST-*.xml`) + `build-report.json` |
| `test-reports-*` | Test report XML for failed jobs specifically |
| `build-logs-*` | Build output log files |
| `build-scan-data-*` or `maven-build-scan-data-*` | Develocity scan data (indicates the project publishes to a Develocity instance) |
| `build-coverage-data-*` | JaCoCo coverage `.exec` files |

### Extracting artifacts

Build-report artifacts are often nested zips (a zip containing a zip):

```bash
unzip -o /tmp/artifact.zip -d /tmp/artifact-outer
unzip -o /tmp/artifact-outer/*.zip -d /tmp/artifact-content
```

## 4. Build Report Summary

Some projects (e.g. Quarkus) have a **"Build report"** job that
aggregates test results into a GitHub Job Summary. The job summary is
**not** accessible via the REST API (UI only), but the same report may
also be available as a **check run**.

### Try the check run first

The [quarkusio/build-reporter](https://github.com/quarkusio/build-reporter)
library can create a check run named `Build summary for <sha>` whose
`output.summary` and `output.text` contain the full report. It exists
when:

- **quarkusio/quarkus** PR builds — created by the quarkus-bot GitHub
  App (not available on other repos).
- **Forks** using action-build-reporter — the action creates it
  directly.

Only present when there are **test failures**.

```bash
HEAD_SHA=$(gh api "repos/<owner>/<repo>/actions/runs/<run-id>" --jq '.head_sha')

# Find the check run
gh api "repos/<owner>/<repo>/commits/${HEAD_SHA}/check-runs" --paginate \
  --jq '.check_runs[] | select(.name | startswith("Build summary for ")) | {id, name}'

# Read it (summary = job table, text = detailed failures with stack traces)
gh api "repos/<owner>/<repo>/check-runs/<check-run-id>" --jq '.output.summary'
gh api "repos/<owner>/<repo>/check-runs/<check-run-id>" --jq '.output.text'
```

### Fallback: download and parse artifacts

If no check run exists, download `build-reports-*` artifacts (see
section 3) and parse them:

```bash
# build-report.json — module-level status and compilation errors
find /tmp/artifact-content -name "build-report.json"
cat /tmp/artifact-content/target/build-report.json \
  | python3 -c "import json,sys; [print(r['name'], r.get('error','')) for r in json.load(sys.stdin)['projectReports'] if r['status'] != 'SUCCESS']"

# Surefire/Failsafe XML — test failure details
grep -rl '<failure\|<error' /tmp/artifact-content --include="TEST-*.xml"
grep -A 5 '<failure\|<error' /tmp/artifact-content/**/TEST-*.xml
```

## 5. Identifying Flaky Tests

### Within-build flakiness (Surefire XML)

When Surefire is configured with `<rerunFailingTestsCount>`, tests that
fail then pass on retry are recorded with `<flakyFailure>` or
`<flakyError>` elements in the XML report. These tests passed overall
but are known to be unstable:

```bash
grep -rl 'flakyFailure\|flakyError' /tmp/artifact-content --include="TEST-*.xml"
```

### Cross-build flakiness (Develocity)

If the project publishes to a Develocity instance (indicated by
`build-scan-data-*` or `maven-build-scan-data-*` artifacts), you can
check test history for flakiness.

**Finding the Develocity instance URL**: search the workflow files for
`develocity-url`, `gradle-enterprise-url`, or `ge.` URLs:

```bash
gh api "repos/<owner>/<repo>/contents/.github/workflows" --jq '.[].name' \
  | xargs -I{} sh -c 'gh api "repos/<owner>/<repo>/contents/.github/workflows/{}" --jq ".content" | base64 -d | grep -il "develocity\|gradle.enterprise\|ge\." && echo "  ^ {}"'
```

Common instances: `ge.quarkus.io`, `develocity.commonhaus.dev`.

**Checking test history for a specific test**: the Develocity test
dashboard is publicly accessible (no API key needed). Link directly
to a test class or test method:

```
https://<instance>/scans/tests?tests.container=<fully.qualified.ClassName>
https://<instance>/scans/tests?tests.container=<fully.qualified.ClassName>&tests.test=<testMethodName>
```

For example:
```
https://ge.quarkus.io/scans/tests?tests.container=io.quarkus.it.keycloak.BearerTokenAuthorizationInGraalITCase&tests.test=testBearerTokenAuthenticationRequestFilter
```

This shows the pass/fail/flaky history of that test across all builds,
revealing whether a failure is a known flaky test or a genuine
regression.

The Develocity REST API (`/api/tests/containers`, `/api/tests/cases`)
provides programmatic access to this data but requires an access key
(Bearer token). It is not accessible anonymously.

## 6. Never Dismiss Failures Without Evidence

**Never claim a CI failure is "pre-existing", "unrelated to your
changes", or "likely flaky" without concrete evidence.** When you
encounter a failure you don't understand, the honest answer is "I don't
know what caused this" — not "it's probably pre-existing."

Before you may classify a failure as pre-existing or unrelated, you
**must** provide evidence from at least one of:

1. **A CI run on another branch showing the same failure.** Check
   recent runs on the base branch (e.g. `main`) or other unrelated PRs:

   ```bash
   # List recent CI runs on the base branch
   gh run list --repo <owner>/<repo> --branch main --limit 10 --json databaseId,conclusion \
     --jq '.[] | select(.conclusion == "failure") | .databaseId'

   # Download failed logs from a base-branch run and search for the same error
   gh run view <base-branch-run-id> --repo <owner>/<repo> --log-failed > /tmp/main-failed.txt
   grep -n "<pattern-from-the-failure>" /tmp/main-failed.txt
   ```

2. **Develocity test history** (see section 5) showing the test has
   been failing or flaky in recent builds on other branches:

   ```
   https://<instance>/scans/tests?tests.container=<fully.qualified.ClassName>&tests.test=<testMethodName>
   ```

If neither method produces evidence, **do not speculate**. Instead:

- Investigate the failure as if it were caused by the current changes.
- If you still cannot determine the cause, say so explicitly and ask
  for guidance. "I wasn't able to determine the root cause of this
  failure" is always better than an unfounded claim that it's
  pre-existing.
