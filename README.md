# GitHub Actions Workflows for Spring Projects

This project containers reusable GitHub Actions Workflows to build Spring projects with Gradle or Maven.
The workflows are designed for specific tasks and can be reused individually or in combinations in the target projects.

To use these workflows in your project a set of organization secrets must be granted to the repository:
```
GH_ACTIONS_REPO_TOKEN
GRADLE_ENTERPRISE_CACHE_USER
GRADLE_ENTERPRISE_CACHE_PASSWORD
GRADLE_ENTERPRISE_SECRET_ACCESS_KEY
JF_ARTIFACTORY_SPRING
SPRING_RELEASE_SLACK_WEBHOOK_URL
OSSRH_URL
OSSRH_S01_TOKEN_USERNAME
OSSRH_S01_TOKEN_PASSWORD
OSSRH_STAGING_PROFILE_NAME
GPG_PASSPHRASE
GPG_PRIVATE_KEY
```

The Gradle Enterprise secrets are optional: not used by Maven and Gradle project might not be enrolled for the service.  
The `SPRING_RELEASE_SLACK_WEBHOOK_URL` secret is also optional: probably you don't want to notify Slack about your release.

The mentioned secrets must be passed explicitly since these reusable workflows might be in different GitHub org than target project.

The SNAPSHOT and Release workflows uses JFrog CLI to publish artifacts into Artifactory.

## Build SNAPSHOT and Pull Request Workflows

The [spring-gradle-pull-request-build.yml](.github/workflows/spring-gradle-pull-request-build.yml) and [spring-maven-pull-request-build.yml](.github/workflows/spring-maven-pull-request-build.yml) are straight forward, single job reusable workflows.
They perform Gradle `check` task and Maven `verify` goal, respectively.
The caller workflow is as simple as follows.

#### Gradle Pull Request caller workflow:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/pr-build-gradle.yml#L1-L10

#### Maven Pull Request caller workflow:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/pr-build-maven.yml#L1-L10

You can add more branches to react for pull request events.

The SNAPSHOT workflows ([spring-artifactory-gradle-snapshot.yml](.github/workflows/spring-artifactory-gradle-snapshot.yml) and [spring-artifactory-maven-snapshot.yml](.github/workflows/spring-artifactory-maven-snapshot.yml), respectively) are also that simple.
They use JFrog CLI action to be able to publish artifacts into `libs-snapshot-local` repository.
The Gradle workflow can be supplied with Gradle Enterprise secrets.

#### Gradle SNAPSHOT caller workflow:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/ci-snapshot-gradle.yml#L1-L18

#### Maven SNAPSHOT caller workflow:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/ci-snapshot-maven.yml#L1-L13

## Release Workflow

The [spring-artifactory-gradle-release.yml](.github/workflows/spring-artifactory-gradle-release.yml) (and therefore [spring-artifactory-maven-release.yml](.github/workflows/spring-artifactory-maven-release.yml)) workflow is complex enough, has some branching jobs and makes some assumptions and expects particular conditions on your repository:

- The versioning schema must follow these rules: 3-digit-dotted number for `major`, `minor` and `patch` parts, snapshot is suffixed with `-SNAPSHOT`, milestones are with `-M{number}` and `-RC{number}` suffix, the GA release is without any suffix.
For example: `0.0.1-SNAPSHOT`, `1.0.0-M1`, `2.1.0-RC2`, `3.3.3`.
- GitHub Milestone titles must be exact as the version to release number.
For example: `1.0.0-M1`, `2.1.0-RC2`, `3.3.3`.
- GitHub Milestones must be scheduled: have a `Due on` date set.
Otherwise, release workflow will be cancelled with a warning that nothing to release for respective SNAPSHOT in a branch.

The logic of this release workflow:

- Take a SNAPSHOT version from a dispatched branch (Maven `help:evaluate -Dexpression="project.version"` and Gradle `gradle properties | grep "^version:"`, respectively).
The composite internal [extract-release-version](.github/actions/extract-release-version/action.yml) action is implemented for this goal 
- List GitHub milestones matching the candidate version and select the closest one by due on date
- Cancel workflow if no scheduled Milestone
- Call Maven or Gradle (according to the workflow choice for the project in the repository) with the release version extracted from the previous job.
This job stages released artifacts using JFrog CLI into `libs-staging-local` repository on Spring Artifactory and commits `Next development version` to the branch we are releasing against
- The next job is to [verify staged artifacts](#verify-staged-artifacts)
- When verification is successful, next job promotes release from staging either to `libs-milestone-local` or `libs-release-local`(and Maven Central) according to the releasing version schema
- Then [spring-finalize-release.yml](.github/workflows/spring-finalize-release.yml) job is executed, which generates release notes using [Spring Changelog Generator](https://github.com/spring-io/github-changelog-generator) excluding repository admins from `Contributors` section.
The `gh release create` command is performed on a tag for just released version.
And in the end the milestone is closed and specific Slack channel is notified about release (if `SPRING_RELEASE_SLACK_WEBHOOK_URL` secret is present in the repository).

#### Example of Release caller workflow:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/release.yml#L1-L25

Such a workflow must be on every branch which is supposed to be released via GitHub actions.

The `buildToolArgs` parameter for this job means extra build tool arguments.
For example, the mentioned `dist` value is a Gradle task in the project.
Can be any Maven goal or other command line arguments.

In the end you just need to go to the `Actions` tab on your project, press `Run workflow` on your release workflow and choose a branch from drop-down list to release currently scheduled Milestone against. 
Such a release workflow can also be scheduled (`cron`, fo example) against branches matrix.

#### Scheduler workflow example:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/schedule-releases.yml#L1-L19

> **Warning**
> The [spring-artifactory-gradle-release.yml](.github/workflows/spring-artifactory-gradle-release.yml) (and [spring-artifactory-maven-release.yml](.github/workflows/spring-artifactory-maven-release.yml)) already uses 3 of 4 levels of nested reusable workflows.
> Where the caller workflow is the last one.
> Therefore don't try to reuse your caller workflow. 

## Verify Staged Artifacts

The `verify-staged` job expects an optional `verifyStagedWorkflow` input (the `verify-staged-artifacts.yml`, by default) workflow supplied from the target project.
For example, [Spring Integration for AWS](https://github.com/spring-projects/spring-integration-aws) uses `jfrog rt download` command to verify that released `spring-integration-aws-${{ inputs.releaseVersion }}.jar` is valid.
Other projects may check out their samples repository and setup release version to perform smoke tests against just staged artifacts.

#### Verify staged workflow sample:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/verify-staged-artifacts.yml#L1-L28

## Backport GitHub Issue Workflow

The [spring-backport-issue.yml](.github/workflows/spring-backport-issue.yml) uses [Spring Backport Bot](https://github.com/spring-io/backport-bot) to create a back-port issue according to the event configured in a caller workflow.
See its documentation for labeling convention and respective GitHub events for calling this workflow.

#### Backport Issue caller workflow example:
https://github.com/artembilan/spring-github-workflows/blob/78b29123a17655f019d800690cc906d692f836a9/samples/backport-issue.yml#L1-L16

## Dependabot Support

If [Dependabot](https://github.com/dependabot) is enabled for repository, its config should set a label compatible with [Spring Changelog Generator](https://github.com/spring-io/github-changelog-generator).
Typically, it is `type: dependency-upgrade`.
It is also a good practice to group all the development dependencies into a single pull request from Dependabot.
This includes all the Gradle and Maven plugins and those dependencies which are used only for testing in the project.
This projects provides a [spring-merge-dependabot-pr.yml](.github/workflows/spring-merge-dependabot-pr.yml) reusable workflow to make modifications to the Dependabot pull requests.
However, there are some prerequisites to use this workflow in your project:
- Pull requests must be protected by some check to pass, usually a workflow to build the project with this pull request changes;
- The [auto-merge](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-auto-merge-for-pull-requests-in-your-repository) must be enabled in the repository;

The `spring-merge-dependabot-pr` workflow does these modifications to the Dependabot pull requests:
- Modify label from `dependency-upgrade` to the `task` for the development dependencies group update to skip them from release notes by Spring Changelog Generator;
- Adds a currently scheduled milestone to the pull request against a snapshot version extracted from the target branch;
- And if milestone is scheduled, the pull request is queued for auto-merging after required checks have passed.

#### Dependabot merge pull request workflow example:
https://github.com/artembilan/spring-github-workflows/blob/710bf1214450ffb9a4d3a1cfbe12755ed2d59edc/samples/merge-dependabot-pr.yml#L1-L14

## Gradle and Artifactory

Gradle projects must not manage `com.jfrog.artifactory` plugin anymore: the `jf gradlec` command sets up this plugin and respective tasks into a project using JFrog specific Gradle init script.
In addition, the [spring-artifactory-gradle-snapshot.yml](.github/workflows/spring-artifactory-gradle-snapshot.yml) and [spring-artifactory-gradle-release-staging.yml](.github/workflows/spring-artifactory-gradle-release-staging.yml) add `spring-artifactory-init.gradle` script to provide an `artifactory` plugin settings for artifacts publications.

The `next-dev-version-init.gradle` script also adds a `nextDevelopmentVersion` task which is used when release has been staged and job is ready to push `Next development version` commit.

See more information in the [Reusing Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows). 
