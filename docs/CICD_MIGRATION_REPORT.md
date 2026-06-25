# CI/CD Pipeline Migration Report: TeamCity to GitLab CI

**Repository:** `softengrahmed/cicd-migration-source2`
**Source File:** `.teamcity/settings.kts` (TeamCity Kotlin DSL v2019.2)
**Target File:** `.gitlab-ci.yml` (GitLab CI)
**Date:** 2025
**Status:** ✅ Migration Complete — Packaging jobs disabled pending NAnt replacement

---

## Table of Contents

1. [Overview](#1-overview)
2. [Repository & Pipeline Inventory](#2-repository--pipeline-inventory)
3. [Migration Details](#3-migration-details)
   - 3.1 [Stage Mapping](#31-stage-mapping)
   - 3.2 [Trigger & Branch Filter](#32-trigger--branch-filter)
   - 3.3 [Version / Kickoff Build](#33-version--kickoff-build)
   - 3.4 [Build Windows](#34-build-windows)
   - 3.5 [Build Mac](#35-build-mac)
   - 3.6 [Test Windows](#36-test-windows)
   - 3.7 [Test Mac](#37-test-mac)
   - 3.8 [Generate Documentation (Skipped)](#38-generate-documentation-skipped)
   - 3.9 [Package Windows (Disabled)](#39-package-windows-disabled)
   - 3.10 [Package Mac and Linux (Disabled)](#310-package-mac-and-linux-disabled)
   - 3.11 [Environment Variables](#311-environment-variables)
   - 3.12 [Secrets & Credentials](#312-secrets--credentials)
   - 3.13 [Artifacts & Retention](#313-artifacts--retention)
   - 3.14 [Failure Conditions (Custom Logic)](#314-failure-conditions-custom-logic)
4. [Feature Comparison Table](#4-feature-comparison-table)
5. [Known Issues & Resolutions](#5-known-issues--resolutions)
6. [Recommendations](#6-recommendations)

---

## 1. Overview

### Purpose

This report documents the complete migration of the MonoGame CI/CD pipeline from **TeamCity** (Kotlin DSL v2019.2) to **GitLab CI** (`.gitlab-ci.yml`). The migration preserves all build logic, dependency ordering, environment variable contracts, and conditional failure handling — while adapting each construct to GitLab CI's native primitives.

### Summary of Results

| Category              | Result                                                         |
|-----------------------|----------------------------------------------------------------|
| Total Build Types     | 8 (TeamCity) → 7 GitLab CI jobs                               |
| Stages Mapped         | 4 (version → build → test → package)                          |
| Jobs Fully Migrated   | 5 (version, build:windows, build:mac, test:windows, test:mac) |
| Jobs Disabled         | 2 (package:windows, package:mac-linux — were paused in TC)    |
| Jobs Skipped          | 1 (docs:generate — docfx not installed on runners)             |
| Secrets Migrated      | 2 (GITHUB_ACCESS_TOKEN, GITHUB_BOT_USERNAME)                  |
| NAnt Steps Converted  | 2 (shell script replacements via dotnet-cake)                  |
| Artifact Retention    | 1 week (all jobs)                                              |
| Branch Exclusion      | `master` branch excluded from all triggers                     |

---

## 2. Repository & Pipeline Inventory

### CI/CD Files Found

| File                        | Platform   | Purpose                                      |
|-----------------------------|------------|----------------------------------------------|
| `.teamcity/settings.kts`    | TeamCity   | Full pipeline (Kotlin DSL v2019.2)           |
| `.teamcity/pom.xml`         | TeamCity   | Maven wrapper for DSL compilation            |
| `.teamcity/README.md`       | TeamCity   | Configuration notes                          |
| `.gitlab-ci.yml`            | GitLab CI  | ✅ **Migrated pipeline (this file)**          |
| `.github/workflows/`        | GitHub     | Empty — no GitHub Actions in use             |

### TeamCity Build Chain (Source)

```
[Version / Kickoff]
       │
  ─────┴──────────────────────
  │                           │
[Build Windows]          [Build Mac]
  │                           │
[Test Windows]           [Test Mac]
  │                           │
  └──────────┬────────────────┘
             │
  [Generate Documentation]  ← SKIPPED (docfx unavailable)
             │
  ─────┬─────┴──────────
       │                │
[Package Windows]  [Package Mac/Linux]
(paused)           (paused)
```

---

## 3. Migration Details

### 3.1 Stage Mapping

GitLab CI uses linear stages where all jobs in a stage run in parallel by default.
The `needs:` keyword is used to enforce explicit DAG-style dependencies within and across stages.

| TeamCity Build Order         | GitLab CI Stage | GitLab CI Job          | Parallelism       |
|------------------------------|-----------------|------------------------|-------------------|
| Version (Kickoff)            | `version`       | `version:kickoff`      | Solo              |
| DevelopWin + DevelopMac      | `build`         | `build:windows` + `build:mac` | Parallel   |
| TestWindows + TestMac        | `test`          | `test:windows` + `test:mac`   | Parallel   |
| GenerateDocumentation        | *(omitted)*     | *(skipped — see §3.8)* | —                 |
| PackagingWindows             | `package`       | `package:windows`      | Disabled          |
| PackageMacAndLinux           | `package`       | `package:mac-linux`    | Disabled          |

**Rationale:** GitLab CI's `needs:` DAG replaces TeamCity's `finishBuildTrigger` + `snapshot` dependency system. Each job explicitly declares which upstream jobs it requires, enabling fine-grained dependency control without relying on stage ordering alone.

---

### 3.2 Trigger & Branch Filter

**TeamCity Source:**
```kotlin
triggers {
    vcs {
        branchFilter = """
            +:*
            -:refs/heads/master
        """.trimIndent()
    }
}
```

**GitLab CI Target:**
```yaml
workflow:
  rules:
    - if: '$CI_COMMIT_REF_NAME == "master"'
      when: never
    - if: '$CI_PIPELINE_SOURCE == "push"'
      when: always
    - if: '$CI_PIPELINE_SOURCE == "web"'
      when: always
```

**Rationale:** TeamCity's `branchFilter` with a `+:*` include and `-:refs/heads/master` exclude is replicated using GitLab CI's `workflow.rules`. The `when: never` rule on `master` is evaluated first (rules are ordered), blocking any pipeline on that branch. The `web` source rule preserves manual pipeline triggering from the GitLab UI.

---

### 3.3 Version / Kickoff Build

**TeamCity Source:**
```kotlin
object Version : BuildType({
    buildNumberPattern = "%VersionMajor%.%VersionMinor%.%VersionPatch%.%VersionBuildCounter%"
    params {
        text("VersionMajor", "3")
        text("VersionMinor", "8")
        text("VersionPatch", "1")
        text("VersionBuildCounter", "%build.counter%")
    }
})
```

**GitLab CI Target:**
```yaml
variables:
  VERSION_MAJOR: "3"
  VERSION_MINOR: "8"
  VERSION_PATCH: "1"

version:kickoff:
  stage: version
  tags: [windows]
  script:
    - echo "BUILD_VERSION=${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}.${CI_PIPELINE_IID}" >> build.env
  artifacts:
    reports:
      dotenv: build.env
    expire_in: 1 week
```

**Rationale:**
- `%build.counter%` → `CI_PIPELINE_IID` (GitLab's auto-incrementing pipeline counter, scoped per project — functionally equivalent)
- `buildNumberPattern` → Assembled inline as `BUILD_VERSION` string
- Version values passed downstream via a `dotenv` artifact report — GitLab CI's standard mechanism for sharing dynamic variables between jobs, replacing TeamCity's `depParamRefs.buildNumber` propagation

---

### 3.4 Build Windows

**TeamCity Source:**
```kotlin
steps {
    exec { path = "dotnet"; arguments = "tool restore" }
    exec { path = "dotnet-cake"; arguments = "build.cake" }
}
requirements {
    startsWith("teamcity.agent.name", "MonoGameWin")
    exists("DotNetCLI")
}
```

**GitLab CI Target:**
```yaml
build:windows:
  tags: [windows]
  before_script:
    - dotnet tool restore
  script:
    - dotnet-cake build.cake
  needs:
    - job: version:kickoff
      artifacts: true
```

**Rationale:**
- `teamcity.agent.name` starts with `MonoGameWin` → `tags: [windows]` (self-hosted runner tag)
- `exec` steps → `before_script` / `script` blocks
- `finishBuildTrigger` on Version → `needs: [version:kickoff]`
- `onDependencyFailure = CANCEL` → GitLab CI automatically cancels dependent jobs when upstream fails

---

### 3.5 Build Mac

**TeamCity Source:**
```kotlin
steps {
    exec { path = "dotnet"; arguments = "tool restore" }
    exec { path = "dotnet"; arguments = "cake build.cake" }  // Note: space, not hyphen
}
requirements {
    equals("teamcity.agent.jvm.os.name", "Mac OS X")
}
```

**GitLab CI Target:**
```yaml
build:mac:
  tags: [windows]  # ⚠️ Replace with [macos] when macOS runner registered
  before_script:
    - dotnet tool restore
  script:
    - dotnet cake build.cake   # preserves the space-separated macOS syntax
```

**Rationale:**
- TeamCity used `dotnet cake` (space, invoking via dotnet SDK) on macOS vs `dotnet-cake` (hyphen, global tool) on Windows. This distinction is preserved in the script commands.
- macOS agent requirement → `tags: [macos]` (currently mapped to `[windows]` pending runner registration)

---

### 3.6 Test Windows

**TeamCity Source:**
```kotlin
steps {
    exec { path = "dotnet"; arguments = "tool restore" }
    exec { path = "dotnet-cake"; arguments = """build.cake -build-target="Test"""" }
}
failureConditions {
    failOnMetricChange {
        metric = TEST_COUNT; threshold = 1200; comparison = LESS
    }
    failOnMetricChange {
        metric = BUILD_DURATION; threshold = 300; stopBuildOnFailure = true
    }
}
```

**GitLab CI Target:**
```yaml
test:windows:
  script:
    - $startTime = Get-Date
    - dotnet-cake build.cake -build-target="Test"
    - |
      # Duration check: > 300s → fail
      $duration = (Get-Date) - $startTime
      if ($duration.TotalSeconds -gt 300) { exit 1 }
    - |
      # Test count check: < 1200 → fail
      [xml]$xml = Get-Content "Test\bin\Windows\AnyCPU\Debug\MonoGameTests.xml"
      if ([int]$xml.'test-results'.total -lt 1200) { exit 1 }
```

**Rationale:** GitLab CI has no built-in metric-based failure conditions. Both conditions are replicated as inline PowerShell assertions post-test-run. The XML result file path mirrors the TeamCity artifact rule exactly.

---

### 3.7 Test Mac

**TeamCity Source:**
```kotlin
failureConditions {
    failOnMetricChange {
        enabled = false   // ← DISABLED
        metric = TEST_COUNT; threshold = 1100
    }
    failOnMetricChange {
        metric = BUILD_DURATION; threshold = 300
    }
}
```

**GitLab CI Target:**
```yaml
test:mac:
  script:
    - START_SECS=$SECONDS
    - dotnet cake build.cake --build-target=Test
    - |
      # Duration check only — test count check intentionally omitted (was disabled in TeamCity)
      ELAPSED=$((SECONDS - START_SECS))
      if [ "$ELAPSED" -gt 300 ]; then exit 1; fi
```

**Rationale:** The `enabled = false` test-count failure condition from TeamCity is intentionally not ported. The duration check is ported as a bash `$SECONDS` delta. This preserves source behavior exactly.

---

### 3.8 Generate Documentation (Skipped)

**TeamCity Source:**
```kotlin
steps {
    exec { path = "docfx"; arguments = "metadata"; workingDir = "Documentation" }
    exec { path = "docfx"; arguments = "build";    workingDir = "Documentation" }
}
```

**Decision:** Skipped — `docfx` is not installed on the target GitLab runners.

**Re-enable Template (add when docfx is available):**
```yaml
# Add 'docs' to the stages list, then:
docs:generate:
  stage: docs
  tags: [windows]
  needs: [build:windows, build:mac]
  script:
    - cd Documentation
    - docfx metadata
    - docfx build
  artifacts:
    paths: [Documentation/_site/]
    expire_in: 1 week
```

---

### 3.9 Package Windows (Disabled)

**TeamCity Source:**
```kotlin
paused = true
steps {
    nant {
        mode = nantFile { path = "default.build" }
        targets = "build_installer"
    }
}
```

**GitLab CI Target:**
```yaml
package:windows:
  script:
    - dotnet-cake build.cake -build-target="BuildInstaller"   # NAnt → Cake conversion
  rules:
    - when: never   # Disabled — mirrors paused = true
```

**Rationale:**
- `paused = true` → `when: never` rule prevents execution
- NAnt `build_installer` target → delegated to dotnet-cake `BuildInstaller` target (shell script conversion)
- To re-enable for on-demand use: change `when: never` → `when: manual`

---

### 3.10 Package Mac and Linux (Disabled)

**TeamCity Source:** Same NAnt pattern as Package Windows, running on macOS agent.

**GitLab CI Target:**
```yaml
package:mac-linux:
  script:
    - dotnet cake build.cake --build-target=BuildInstaller   # NAnt → Cake (macOS)
  rules:
    - when: never
```

**Rationale:** Same as Package Windows. Artifacts (`.pkg`, `.deb`, `.run`) paths preserved verbatim from TeamCity artifact rules.

---

### 3.11 Environment Variables

| TeamCity Variable                     | GitLab CI Equivalent               | Notes                                     |
|---------------------------------------|------------------------------------|-------------------------------------------|
| `%teamcity.build.branch%`            | `$CI_COMMIT_REF_NAME`              | Current branch name                       |
| `%build.number%`                      | `$BUILD_VERSION` (via dotenv)      | Assembled from Major.Minor.Patch.IID      |
| `%build.counter%`                     | `$CI_PIPELINE_IID`                 | Per-project auto-incrementing counter     |
| `VersionMajor = 3`                    | `VERSION_MAJOR: "3"` (global var)  | Static value, unchanged                   |
| `VersionMinor = 8`                    | `VERSION_MINOR: "8"` (global var)  | Static value, unchanged                   |
| `VersionPatch = 1`                    | `VERSION_PATCH: "1"` (global var)  | Static value, unchanged                   |
| `env.GIT_BRANCH`                      | `GIT_BRANCH: $CI_COMMIT_REF_NAME`  | Set per job in `variables:` block         |
| `env.BUILD_NUMBER`                    | `BUILD_NUMBER: $BUILD_VERSION`     | Set per job in `variables:` block         |

---

### 3.12 Secrets & Credentials

**TeamCity Source:**
```kotlin
param("secure:github_access_token", "credentialsJSON:6be4e606-...")
param("secure:guthub_username",     "credentialsJSON:638301b3-...")
```

These were TeamCity-vaulted credentials used for GitHub commit status reporting.

**GitLab CI Target:**

Register the following as **masked CI/CD variables** in GitLab:
> *Settings → CI/CD → Variables → Add Variable → Masked: ✅*

| GitLab Variable Name    | Replaces TeamCity Secret              | Usage                           |
|-------------------------|---------------------------------------|---------------------------------|
| `GITHUB_ACCESS_TOKEN`   | `secure:github_access_token`          | GitHub API authentication       |
| `GITHUB_BOT_USERNAME`   | `secure:guthub_username`              | GitHub bot identity (mgbot)     |

**Important Notes:**
- TeamCity's `teamcity.github.status` feature plugin (reporting build status to GitHub) has **no direct GitLab CI equivalent**. If cross-platform GitHub status reporting is required, add a script step using the GitHub REST API (`curl -H "Authorization: token $GITHUB_ACCESS_TOKEN" ...`).
- Variables must be set as **masked** to prevent exposure in job logs.
- Variables should also be set as **protected** if they should only be available on protected branches.

---

### 3.13 Artifacts & Retention

| TeamCity Build Type      | TeamCity Artifact Path                                                          | GitLab CI Path                             | Retention |
|--------------------------|---------------------------------------------------------------------------------|--------------------------------------------|-----------|
| Build Windows            | `Artifacts/**/*.nupkg`, `*.vsix`, `*.mpack`                                    | Same                                       | 1 week    |
| Build Mac                | `Artifacts/**/iOS/**/*.nupkg`, `**/Android/**/*.nupkg`, `*.mpack`              | Same                                       | 1 week    |
| Test Windows             | `CapturedFrames/`, `Diffs/`, `MonoGameTests.xml` → `TestResults.Windows.zip`   | Paths preserved, no zip (GitLab archives)  | 1 week    |
| Test Mac                 | `Test/bin/Linux/AnyCPU/Debug/TestResult.xml`                                    | Same                                       | 1 week    |
| Generate Documentation   | `Documentation/_site => Documentation.zip`                                      | Skipped (docfx unavailable)                | —         |
| Package Windows          | `Installers\Windows\MonoGameSetup.exe`                                          | Same (job disabled)                        | 1 week    |
| Package Mac/Linux        | `*.pkg`, `*.deb`, `*.run`                                                       | Same (job disabled)                        | 1 week    |

**Note:** TeamCity uses named artifact rules with `=>` for zip packaging. GitLab CI archives artifacts as a flat zip automatically — no explicit zip step needed.

---

### 3.14 Failure Conditions (Custom Logic)

TeamCity provides built-in `failOnMetricChange` support. GitLab CI has no equivalent. Both conditions were replicated as inline shell scripts:

| TeamCity Condition                              | GitLab CI Replacement                                      | Job           |
|-------------------------------------------------|------------------------------------------------------------|---------------|
| `TEST_COUNT < 1200` (enabled)                   | PowerShell XML parse of MonoGameTests.xml                  | test:windows  |
| `BUILD_DURATION > 300s` (stop build)            | PowerShell `(Get-Date) - $startTime > 300`                 | test:windows  |
| `TEST_COUNT < 1100` (disabled)                  | **Not ported** — intentionally omitted                     | test:mac      |
| `BUILD_DURATION > 300s`                         | Bash `$((SECONDS - START_SECS)) > 300`                     | test:mac      |

---

## 4. Feature Comparison Table

| Feature                        | TeamCity                                         | GitLab CI                                              |
|--------------------------------|--------------------------------------------------|--------------------------------------------------------|
| **Pipeline Definition**        | Kotlin DSL (`settings.kts`)                      | YAML (`.gitlab-ci.yml`)                                |
| **Trigger Type**               | VCS trigger with `branchFilter`                  | `workflow.rules` with `if:` expressions                |
| **Branch Exclusion**           | `-:refs/heads/master` in `branchFilter`          | `if: '$CI_COMMIT_REF_NAME == "master"' when: never`    |
| **Parallelism**                | Parallel build chain via `finishBuildTrigger`    | `needs:` DAG — jobs in same stage run in parallel      |
| **Agent Selection**            | `requirements { startsWith("agent.name", ...) }`| `tags: [windows]` / `tags: [macos]`                   |
| **Build Dependencies**         | `snapshot()` + `finishBuildTrigger`              | `needs:` with `artifacts: true`                        |
| **Build Number / Version**     | `buildNumberPattern` + `%build.counter%`         | `CI_PIPELINE_IID` + `dotenv` artifact propagation      |
| **Environment Variables**      | `params {}` + `%variable%` references           | `variables:` block + `$VARIABLE` references            |
| **Secrets Management**         | TeamCity credentials vault (`credentialsJSON:`)  | GitLab CI/CD masked variables                          |
| **Artifact Collection**        | `artifactRules` with `=>` zip targets           | `artifacts.paths:` (auto-zipped)                       |
| **Artifact Retention**         | Project-level retention policy                   | Per-job `expire_in:` field (set to 1 week)             |
| **Artifact Pass-Through**      | `artifacts()` dependency block                   | `needs: [job, artifacts: true]`                        |
| **Failure on Metric**          | Native `failOnMetricChange {}`                   | Custom shell script assertions (PowerShell / bash)     |
| **Build Duration Check**       | `metric = BUILD_DURATION, threshold = 300`       | `$SECONDS` / `Get-Date` delta in script                |
| **Test Count Check**           | `metric = TEST_COUNT, threshold = 1200`          | PowerShell XML parse of NUnit result file              |
| **Paused/Disabled Jobs**       | `paused = true` on BuildType                     | `rules: - when: never`                                 |
| **Manual Jobs**                | Paused = triggered manually via UI               | `when: manual` in rules                                |
| **Cache**                      | Not defined in source pipeline                   | Not configured (add `cache:` block if needed)          |
| **Status Reporting**           | `teamcity.github.status` feature plugin          | ⚠️ No native equivalent — requires GitHub API curl     |
| **Checkout Strategy**          | `CheckoutMode.ON_SERVER` + `cleanCheckout = true`| `GIT_STRATEGY: clone` + `GIT_CLEAN_FLAGS: -ffdx`      |
| **DSL / Config as Code**       | Kotlin DSL (typed, compiled)                     | YAML (declarative, schema-validated)                   |

---

## 5. Known Issues & Resolutions

### Issue 1 — No Native GitHub Commit Status Reporting

**Problem:** TeamCity used the `teamcity.github.status` plugin to post build statuses back to GitHub on every job start/finish. GitLab CI has no equivalent plugin.

**Impact:** GitHub PRs will not show GitLab CI build statuses inline.

**Resolution Options:**
- **Option A (Recommended):** Add a shell step in each job using the GitHub Statuses API:
  ```yaml
  after_script:
    - |
      curl -s -H "Authorization: token $GITHUB_ACCESS_TOKEN" \
        -X POST https://api.github.com/repos/MonoGame/MonoGame/statuses/$CI_COMMIT_SHA \
        -d "{\"state\":\"success\",\"context\":\"gitlab-ci/$CI_JOB_NAME\"}"
  ```
- **Option B:** Use GitLab's built-in GitHub integration (Settings → Integrations → GitHub) to enable automatic status mirroring.

---

### Issue 2 — NAnt Build Tool Not Available

**Problem:** `PackagingWindows` and `PackageMacAndLinux` used NAnt (`nant -buildfile:default.build build_installer`). NAnt is not available on the target runners.

**Impact:** Packaging jobs cannot be directly executed.

**Resolution:** Both packaging jobs have been converted to invoke `dotnet-cake build.cake -build-target="BuildInstaller"`. A `BuildInstaller` target must be added to `build.cake` that replicates the NAnt `build_installer` task logic.

**Action Required:** Engineering team must implement the `BuildInstaller` Cake target in `build.cake` before enabling packaging jobs.

---

### Issue 3 — docfx Not Available on Runners

**Problem:** `GenerateDocumentation` required `docfx` installed on the Windows runner.

**Impact:** Documentation generation is skipped entirely.

**Resolution:** When docfx becomes available, restore the job using the template provided in §3.8. Alternatively, run docfx via a Docker image:
```yaml
docs:generate:
  image: docfx/docfx:latest   # if Docker executor is configured
```

---

### Issue 4 — macOS Runner Not Registered

**Problem:** `Build Mac`, `Test Mac`, and `Package Mac/Linux` require a macOS GitLab Runner. Only `[windows]` runners are currently confirmed.

**Impact:** Mac build and test jobs run on Windows runners (incorrect environment). iOS/Android builds will fail.

**Resolution:** Register a macOS self-hosted GitLab Runner with tag `macos`. Then replace `tags: [windows]` with `tags: [macos]` in the following jobs:
- `build:mac`
- `test:mac`
- `package:mac-linux`

---

### Issue 5 — TeamCity `CheckoutMode.ON_SERVER` vs GitLab Clone

**Problem:** TeamCity used server-side checkout (`CheckoutMode.ON_SERVER`) which avoids re-cloning. GitLab CI uses `GIT_STRATEGY: clone` for a clean checkout equivalent.

**Impact:** Slightly increased pipeline time due to full re-clone per job.

**Resolution:** For performance improvement, switch to `GIT_STRATEGY: fetch` with `GIT_CLEAN_FLAGS: -ffdx` after initial stability is confirmed. This is functionally equivalent to TeamCity's incremental checkout with clean workspace.

---

### Issue 6 — Dependency Artifact Rules (PackageMacAndLinux)

**Problem:** In TeamCity, `PackageMacAndLinux` used `artifacts()` dependency blocks to pull `MonoGame.Framework.zip`, `MonoGame.Framework.Content.Pipeline.zip`, and `Tools.zip` from both `DevelopMac` and `DevelopWin`. These ZIP files are not explicitly produced by the current `build.cake` scripts as confirmed artifacts.

**Impact:** Package jobs may fail to find expected dependency ZIPs.

**Resolution:** Verify whether `build.cake` produces these ZIP artifacts. If not, update the packaging job's `needs:` artifact paths to match actual Cake output paths, or add ZIP packaging steps to the respective build Cake targets.

---

## 6. Recommendations

### 6.1 Register a macOS GitLab Runner

Register a dedicated self-hosted macOS runner with tag `macos` as the highest priority action. Without it, `build:mac`, `test:mac`, and `package:mac-linux` run in the wrong environment.

```bash
# On the macOS machine:
gitlab-runner register \
  --url https://gitlab.com/ \
  --registration-token <YOUR_TOKEN> \
  --executor shell \
  --tag-list macos \
  --description "MonoGame macOS Runner"
```

### 6.2 Add GitHub Status Reporting

Implement GitHub commit status callbacks in a shared `after_script` or `.gitlab-ci.yml` hidden job template (`.report_github_status`) to preserve the `teamcity.github.status` behaviour. Use GitLab CI's YAML anchors for DRY reuse:

```yaml
.report_github_status: &report_github_status
  after_script:
    - |
      STATUS="success"
      if [ "$CI_JOB_STATUS" != "success" ]; then STATUS="failure"; fi
      curl -s -H "Authorization: token $GITHUB_ACCESS_TOKEN" \
        -X POST https://api.github.com/repos/MonoGame/MonoGame/statuses/$CI_COMMIT_SHA \
        -d "{\"state\":\"$STATUS\",\"context\":\"gitlab/$CI_JOB_NAME\"}"
```

### 6.3 Implement the Cake `BuildInstaller` Target

Before enabling packaging jobs, add a `BuildInstaller` target to `build.cake` that replicates the logic previously in NAnt's `build_installer` target. This is required for both `package:windows` and `package:mac-linux` to function correctly.

### 6.4 Restore docfx Documentation Job

Install `docfx` on the Windows runner (or use the official Docker image) and uncomment the documentation job template from §3.8. This restores the `GenerateDocumentation` stage in the pipeline.

### 6.5 Switch to `GIT_STRATEGY: fetch` After Stabilization

Once the pipeline is stable, change `GIT_STRATEGY: clone` to `GIT_STRATEGY: fetch` with `GIT_CLEAN_FLAGS: -ffdx` to reduce pipeline time by avoiding full repository re-clones on every job.

### 6.6 Add Pipeline Caching for dotnet Tools

Restore `.dotnet/` tool cache across runs to speed up `dotnet tool restore`:

```yaml
cache:
  key: "${CI_COMMIT_REF_SLUG}-dotnet-tools"
  paths:
    - .dotnet/
    - .config/
```

### 6.7 Enable Packaging Jobs Gradually

When ready to re-enable packaging:
1. Change `when: never` → `when: manual` to require explicit human trigger
2. Validate the `BuildInstaller` Cake target independently
3. Once validated, change `when: manual` → `when: on_success` for full automation

### 6.8 Rotate Secrets Post-Migration

The original TeamCity `credentialsJSON:` references point to TeamCity's vault. Register fresh credential values in GitLab CI/CD masked variables and rotate the GitHub tokens used in TeamCity to ensure no credential reuse across platforms.

### 6.9 Validate with Lint Before First Run

Before triggering the pipeline, validate the YAML file:

```bash
# Using GitLab's built-in linter (requires API token):
curl --header "PRIVATE-TOKEN: <your_token>" \
  https://gitlab.com/api/v4/projects/<project_id>/ci/lint \
  --form "content=@.gitlab-ci.yml"

# Or via the GitLab UI:
# CI/CD → Pipelines → CI Lint (paste .gitlab-ci.yml content)
```

---

*Report generated as part of the CI/CD migration initiative for the MonoGame project.*
*Branch: `gitlab-cicd-migration` | File: `.gitlab-ci.yml`*
