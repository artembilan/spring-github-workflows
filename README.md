# GitHub Actions Workflows for Spring Projects
---
**IMPORTANT**
This repository is fully migrated to the https://github.com/spring-io/spring-github-workflows.
So, the reference to the reusable workflows must be changed from an `artembilan` account to the `spring-io` organization, e.g.:
```
name: CI SNAPSHOT

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  build_snapshot:
    uses: spring-io/spring-github-workflows/.github/workflows/spring-artifactory-maven-snapshot.yml@main
    secrets:
      JF_ARTIFACTORY_SPRING: ${{ secrets.JF_ARTIFACTORY_SPRING }}
```
---
