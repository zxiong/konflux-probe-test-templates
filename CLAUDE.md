# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains Tekton PipelineRun templates for Konflux probe load tests. The templates define CI/CD pipelines for different component types and are organized by component name.

## Repository Structure

The repository is organized into component-specific directories:

- `kflux-rhel-p01-RPM/` - RHEL RPM package build templates
- `kfluxfedorap01-RPM/` - Fedora RPM package build templates
- `stone-prd-rh01-RPM/` - Stone production RHEL RPM templates
- `stone-prod-p02-RPM/` - Stone production P02 RPM templates
- `nodejs-devfile-sample-MultiArch/` - Multi-architecture Node.js container builds
- `nodejs-devfile-sample-SingleArch/` - Single-architecture Node.js container builds
- `nodejs-devfile-sample-test/` - Test Node.js container builds

Each directory contains two pipeline templates:
- `COMPONENT-pull-request.yaml` - Pipeline triggered on pull requests
- `COMPONENT-push.yaml` - Pipeline triggered on push events

## Pipeline Template Types

### RPM Package Pipelines

RPM-based components reference external pipeline definitions via Git resolver:
- Pipeline source: `https://gitlab.cee.redhat.com/rhel-on-konflux/rpmbuild-pipeline`
- Pipeline file: `pipeline/build-rhel-package.yaml`
- Key parameters: `package-name`, `git-url`, `revision`, `target-branch`, `ociStorage`

### Container Build Pipelines

Container-based components embed complete pipeline specifications inline:
- Multi-platform support: linux/arm64, linux/amd64, linux/s390x, linux/ppc64le
- Uses Tekton task bundles from `quay.io/konflux-ci/tekton-catalog/`
- Build strategy: OCI trusted artifacts (oci-ta)
- Key features: hermetic builds, prefetch dependencies, source image generation

## Template Variables

All templates use Tekton Pipelines as Code template variables:
- `{{ source_url }}` - Source repository URL
- `{{ revision }}` - Git commit SHA
- `{{ target_branch }}` - Target branch name
- `{{ pull_request_number }}` - PR number (pull-request templates only)
- `{{ git_auth_secret }}` - Git authentication secret name

Placeholders that must be customized per deployment:
- `APPLICATION` - Application name label
- `COMPONENT` - Component name label
- `NAMESPACE` - Kubernetes namespace

## Tekton Task Bundle Management

### Automated Updates with Renovate

The repository uses Renovate for automated dependency updates configured in `renovate.json`:
- Task bundles are tracked from `quay.io/konflux-ci/tekton-catalog/`
- Updates are grouped into single PRs under "Tekton Task Bundle Upgrades"
- Runs daily before 6am
- Updates both version tags and SHA256 digests
- **Automatically applies migration scripts** via `postUpgradeTasks`

### Automated Migration Workflow

When Renovate detects task bundle updates, it automatically:

1. **Updates bundle references** in all PipelineRun YAML files
2. **Executes migration script** (`.renovate/run-migrations.sh`) as a post-upgrade task
3. **Downloads migration scripts** from [konflux-ci/build-definitions](https://github.com/konflux-ci/build-definitions)
4. **Applies migrations** to pipeline files using task-specific migration scripts
5. **Commits all changes** (bundle updates + migrations) in a single PR

#### Migration Script Location

For each task upgrade, the migration runner checks for scripts at:
```
https://raw.githubusercontent.com/konflux-ci/build-definitions/main/task/{task-name}/{version}/migrations/{version}.sh
```

Examples:
- `task/clamav-scan/0.3/migrations/0.3.sh`
- `task/buildah-remote-oci-ta/0.6/migrations/0.6.sh`

#### What Migrations Do

Migration scripts modify pipeline definitions to:
- Add/remove/update task parameters
- Change task execution order
- Add/remove mandatory tasks
- Update task configurations for API changes
- Ensure compatibility with new task versions

All modifications are done in-place using `yq` commands and are idempotent.

### Manual Updates

Use the `update-tekton-task-bundles.sh` script to manually update task bundle references:

```bash
./update-tekton-task-bundles.sh <directory>/*.yaml
```

The script:
1. Extracts all bundle references with `resolver: bundles`
2. Queries latest digests using `skopeo inspect`
3. Updates bundle references to new digests
4. Requires `yq` (either python-yq or mikefarah/yq) and `skopeo`

To manually run migrations after manual updates:
```bash
bash .renovate/run-migrations.sh
```

Reference: https://konflux-ci.dev/docs/troubleshooting/builds/#manually-update-task-bundles

### Migration Dependencies

Required tools (install via `.renovate/setup-dependencies.sh`):
- `yq` - YAML processor (mikefarah/yq or python-yq)
- `curl` - For downloading migration scripts
- `bash` - Shell for executing migrations

## Container Pipeline Architecture

Multi-arch container pipelines follow this task flow:

1. **Initialization**: `init` task validates build requirements
2. **Clone**: `git-clone-oci-ta` clones source to OCI artifact
3. **Prefetch**: `prefetch-dependencies-oci-ta` fetches npm dependencies
4. **Build**: `buildah-remote-oci-ta` builds images per platform (matrix task)
5. **Index**: `build-image-index` combines platform images into manifest
6. **Source Image**: `source-build-oci-ta` creates source image (optional)
7. **Security Checks** (parallel after build-image-index):
   - `deprecated-image-check` - Deprecated base image detection
   - `clair-scan` - Vulnerability scanning (per-platform matrix)
   - `ecosystem-cert-preflight-checks` - Red Hat certification checks
   - `sast-snyk-check-oci-ta` - Snyk SAST analysis
   - `clamav-scan` - Malware scanning
   - `sast-coverity-check-oci-ta` - Coverity static analysis
   - `sast-shell-check-oci-ta` - Shell script linting
   - `sast-unicode-check-oci-ta` - Unicode attack detection
   - `rpms-signature-scan` - RPM signature validation
8. **Finalization**:
   - `apply-tags` - Apply image tags
   - `push-dockerfile-oci-ta` - Push Dockerfile to registry
   - `show-sbom` - Display SBOM (finally task)

## Important Details

### OCI Artifacts for Data Sharing

All container pipelines use OCI artifacts (trusted artifacts) instead of PVCs for sharing data between tasks:
- Source code: `$(params.output-image).git`
- Prefetched deps: `$(params.output-image).prefetch`
- This enables Enterprise Contract policy compliance

### Matrix Tasks

Multi-platform builds use Tekton matrix parameters to fan out builds:
- `build-images` matrix on `PLATFORM` parameter
- Security tasks (`clair-scan`, `ecosystem-cert-preflight-checks`, `clamav-scan`) matrix on platform parameters

### Conditional Task Execution

Tasks use `when` expressions for conditional execution:
- Most tasks: `$(tasks.init.results.build) == "true"`
- Security checks: `$(params.skip-checks) == "false"`
- Coverity: Requires `$(tasks.coverity-availability-check.results.STATUS) == "success"`

### Image Output Differences

- Pull request images: `quay.io/redhat-user-workloads-stage/NAMESPACE/COMPONENT:on-pr-{{revision}}`
- Pull requests expire after 5 days (`image-expires-after: 5d`)
- Push images: Use same URL pattern but without expiration
