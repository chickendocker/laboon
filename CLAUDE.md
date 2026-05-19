# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Laboon is a monorepo designed to build Docker images for 3rd party applications that lack official Docker repositories. It uses GitHub Actions to automate multi-architecture (amd64/arm64) Docker image builds.

## Architecture

### Configuration-Driven Build System

The repository uses a declarative configuration approach with `laboon.yaml` as the central configuration file:

- **laboon.yaml**: Defines build configurations for multiple applications. Each config specifies:
  - `name`: Application identifier (used in workflows and image naming)
  - `repository`: Source git repository to build from
  - `dockerfile`: Path to Dockerfile in the source repo
  - `matrix_include`: Platform-specific runners (linux/amd64 on ubuntu-latest, linux/arm64 on ubuntu-24.04-arm)

### Workflow Architecture

The build system consists of a three-stage GitHub Actions workflow (`.github/workflows/build-image.yml`):

1. **Setup Job**:
   - Parses `laboon.yaml` to extract configuration for the requested app
   - Converts git SSH URLs to repository paths
   - Generates dynamic build matrix for subsequent jobs

2. **Build Job**:
   - Runs in parallel across multiple architectures using matrix strategy
   - Each platform builds independently on architecture-specific runners
   - Builds are pushed by digest (not by tag) to support multi-arch manifests
   - Uses registry layer caching per architecture for faster rebuilds

3. **Merge Job**:
   - Downloads all architecture-specific digests
   - Creates a multi-architecture manifest combining all platform images
   - Tags the final image with the git reference provided as input
   - Final images are published to `ghcr.io/chickendocker/<app-name>:<git-ref>`

### Multi-Architecture Build Strategy

The workflow uses a "build-by-digest" approach:
- Each architecture builds separately and pushes an image digest
- Digests are passed between jobs via GitHub Actions artifacts
- The merge job combines digests into a single manifest that supports multiple architectures
- This allows users to pull a single tag that automatically selects the correct architecture

## Common Commands

### Triggering Builds

Builds are triggered manually via workflow_dispatch:

```bash
# Trigger via GitHub CLI
gh workflow run build-image.yml \
  -f name=<app-name-from-laboon-yaml> \
  -f git_ref=<branch-tag-or-commit>

# Example:
gh workflow run build-image.yml \
  -f name=memogram \
  -f git_ref=main
```

### Configuration Management

When adding a new application to build:

1. Add entry to `laboon.yaml`:
```yaml
build_configs:
- name: myapp
  repository: git@github.com:org/repo.git
  dockerfile: Dockerfile
  matrix_include:
  - platform: linux/amd64
    runner: ubuntu-latest
  - platform: linux/arm64
    runner: ubuntu-24.04-arm
```

2. Trigger the workflow with the new app name

### Inspecting Built Images

```bash
# Pull and inspect a built image
docker pull ghcr.io/chickendocker/<app-name>:<tag>
docker buildx imagetools inspect ghcr.io/chickendocker/<app-name>:<tag>
```

## Repository Structure

```
.
├── laboon.yaml              # Central configuration for all builds
├── .github/
│   └── workflows/
│       └── build-image.yml  # Main workflow for building apps
└── multi-arch-sample.yaml   # Sample/reference configuration
```

Note: This repository does not contain application source code. It only contains build configurations and workflows. Source code is pulled from repositories specified in `laboon.yaml` during the build process.
