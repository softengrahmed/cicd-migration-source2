# CI/CD Pipeline Migration Report: TeamCity to GitLab CI

## Overview
This repository currently contains TeamCity Kotlin DSL under `.teamcity/` and build entry points in `build.sh`, `build.ps1`, and the `build/` tooling. The requested migration target is GitLab CI, so the implementation below translates the TeamCity pipeline model into GitLab jobs while preserving build ordering, environment variables, artifact handling, and manual release controls as closely as GitLab CI allows.

### CI/CD configuration files found
- `.teamcity/settings.kts`: TeamCity pipeline definition, including build types, triggers, dependencies, artifacts, and status checks.
- `.teamcity/pom.xml`: TeamCity DSL build descriptor used to generate settings.
- `.teamcity/README.md`: local instructions for editing TeamCity DSL.
- `build.sh` / `build.ps1`: build bootstrap scripts.
- `README.md`: references CI status badges, but no GitLab CI configuration exists in the current codebase.

## Migration details
### Source pipeline structure
The TeamCity configuration defines these build types:
- `Version` / Kickoff Build: computes a single version number for downstream jobs.
- `DevelopWin`: builds Windows/Web/Android artifacts.
- `DevelopMac`: builds Mac/iOS/Linux artifacts.
- `TestWindows`: runs Windows unit tests.
- `TestMac`: runs Mac unit tests.
- `GenerateDocumentation`: runs docfx metadata/build.
- `PackagingWindows`: creates Windows installer artifacts.
- `PackageMacAndLinux`: creates Mac/Linux installer artifacts.

### Target pipeline structure in GitLab CI
The GitLab CI equivalent is implemented as a multi-job pipeline with the following mapping:
- `Version` -> `version` job using a dotenv artifact.
- `DevelopWin` -> `build:windows` job.
- `DevelopMac` -> `build:mac` job.
- `TestWindows` -> `test:windows` job.
- `TestMac` -> `test:mac` job.
- `GenerateDocumentation` -> `docs` job.
- `PackagingWindows` -> `package:windows` job.
- `PackageMacAndLinux` -> `package:mac-and-linux` job.

### Transformation logic
- **Triggers**: TeamCity VCS and finish-build triggers were flattened into GitLab `rules:` and `needs:` dependencies. GitLab does not have a direct finish-build trigger equivalent, so ordering is expressed through `needs:` and stage sequencing.
- **Build order**: preserved with `stages:` plus `needs:` so version runs first, then builds, then tests/docs, then packaging.
- **Environment setup**: TeamCity `env.GIT_BRANCH` and `env.BUILD_NUMBER` map to GitLab predefined variables such as `CI_COMMIT_REF_NAME` and `CI_PIPELINE_IID`.
- **Secrets**: TeamCity GitHub status tokens were not copied directly. In GitLab CI, these should be stored as masked/protected CI variables and referenced via `$VARIABLE_NAME`.
- **Artifacts**: TeamCity artifact rules were translated to GitLab `artifacts:paths`. Manual packaging jobs retain manual gating with `when: manual`.
- **Conditional logic**: TeamCity failure conditions based on test counts and build duration do not have a one-to-one equivalent; they require manual rewrite as shell/script assertions or quality gates if still required.

### Side-by-side comparison
| Feature | TeamCity | GitLab CI equivalent |
|---|---|---|
| Pipeline definition | Kotlin DSL in `.teamcity/settings.kts` | YAML in `.gitlab-ci.yml` |
| Versioning | Dedicated `Version` build type with build-number pattern | `version` job + dotenv artifact using pipeline variables |
| Build order | Snapshot dependencies between build types | `stages:` and `needs:` relationships |
| Trigger model | VCS trigger + finish-build triggers | Branch/MR `rules:` and job dependencies |
| Parallelism | Multiple independent build types can run after kickoff | Jobs can run in parallel within the same stage when dependencies allow |
| Cache handling | Not explicit in the TeamCity DSL shown | Not yet configured; should be added with GitLab `cache:` if needed |
| Artifacts | TeamCity artifact rules per build type | GitLab `artifacts:paths` per job |
| Secrets | TeamCity secure parameters and GitHub status tokens | GitLab masked/protected CI variables |
| Manual gating | `paused = true` on packaging jobs | `when: manual` on packaging jobs |
| Agent selection | TeamCity requirements like OS and agent name | GitLab runner tags such as `macos` and `windows` |

### Implemented GitLab CI file
A new `.gitlab-ci.yml` file was added to the branch `gitlab-cicd-migration`. It includes:
- a `version` job that restores .NET tools and exports version metadata,
- build jobs for Mac and Windows,
- test jobs for Mac and Windows,
- documentation generation,
- manual packaging jobs for Mac/Linux and Windows.

## Known issues & resolutions
1. **TeamCity finish-build triggers have no direct GitLab equivalent**
   - Resolution: model the dependency chain with `needs:` and stage ordering.
   - Justification: GitLab CI is job graph-driven rather than build-type-driven.

2. **TeamCity status publishing to GitHub cannot be migrated verbatim**
   - Resolution: omit the TeamCity GitHub status feature and rely on GitLab commit status / job status reporting.
   - Justification: GitLab integrates natively with its own commit pipeline status and does not need the TeamCity plugin.

3. **NAnt and Cake execution are environment-sensitive**
   - Resolution: preserve the command names in the GitLab jobs, but validate the runner images and tool availability before enabling the pipeline.
   - Justification: the current TeamCity setup assumes specific agent capabilities.

4. **OS-specific builds need matching GitLab runners**
   - Resolution: use runner tags like `macos` and `windows` and ensure toolchains are installed on those runners.
   - Justification: TeamCity agent requirements were tied to OS and agent name constraints.

5. **Build failure metric checks are not ported automatically**
   - Resolution: add explicit scripts if the team still wants threshold-based checks for test counts or build duration.
   - Justification: GitLab CI does not expose the same TeamCity failure-condition primitives.

## Recommendations
- Add a GitLab `cache:` section for NuGet, tool restore, and any build intermediates to reduce pipeline time.
- Use protected variables for any tokens, package credentials, or signing secrets.
- Split packaging into release-only rules if those jobs should not run on every branch or merge request.
- Validate runner parity with TeamCity agents before removing the old pipeline.
- Add `allow_failure:` only where TeamCity had non-blocking checks; otherwise keep failures strict.
- Consider publishing artifacts to GitLab Releases or package registries if the installers are consumed externally.
- Keep the version job as the single source of truth for versioning so downstream jobs stay deterministic.
