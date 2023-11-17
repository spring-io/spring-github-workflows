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

The mentioned secrets must be passed explicitly since these reusable workflows might be in different GitHub org than target project.

The SNAPSHOT and Release workflows uses JFrog CLI to publish artifacts into Artifactory.

## Build SNAPSHOT and Pull Request Workflows

The [spring-gradle-pull-request-build.yml](.github/workflows/spring-gradle-pull-request-build.yml) and [spring-maven-pull-request-build.yml](.github/workflows/spring-maven-pull-request-build.yml) are straight forward, single job reusable workflows.
They perform Gradle `check` task and Maven `verify` goal, respectively.
The caller workflow is as simple as follows.

#### Gradle Pull Request caller workflow:
https://github.com/artembilan/spring-messaging-build-tools/blob/decee2963c926c34f1f52bf373a3bc1dc09f1724/samples/pr-build-gradle.yml#L1-L10

#### Maven Pull Request caller workflow:
https://github.com/artembilan/spring-messaging-build-tools/blob/decee2963c926c34f1f52bf373a3bc1dc09f1724/samples/pr-build-maven.yml#L1-L10

You can add more branches to react for pull request events.

The SNAPSHOT workflows ([spring-artifactory-gradle-snapshot.yml](.github/workflows/spring-artifactory-gradle-snapshot.yml) and [spring-artifactory-maven-snapshot.yml](.github/workflows/spring-artifactory-maven-snapshot.yml), respectively) are also that simple.
They use JFrog CLI action to be able to publish artifacts into `libs-snapshot-local` repository.
The Gradle workflow can be supplied with Gradle Enterprise secrets.

#### Gradle SNAPSHOT caller workflow:
https://github.com/artembilan/spring-messaging-build-tools/blob/decee2963c926c34f1f52bf373a3bc1dc09f1724/samples/ci-snapshot-gradle.yml#L1-L18

#### Maven SNAPSHOT caller workflow:
https://github.com/artembilan/spring-messaging-build-tools/blob/decee2963c926c34f1f52bf373a3bc1dc09f1724/samples/ci-snapshot-maven.yml#L1-L13

## Release Workflow

The [spring-artifactory-release.yml](.github/workflows/spring-artifactory-release.yml) workflow is complex enough, has some branching jobs and makes some assumptions and expects particular conditions on your repository:

- The versioning schema must follow these rules: 3-digit-dotted number for `major`, `minor` and `patch` parts, snapshot is suffixed with `-SNAPSHOT`, milestones are with `-M{number}` and `-RC{number}` suffix, the GA release is without any suffix.
For example: `0.0.1-SNAPSHOT`, `1.0.0-M1`, `2.1.0-RC2`, `3.3.3`.
- GitHub Milestone titles must be exact as the version to release number.
For example: `1.0.0-M1`, `2.1.0-RC2`, `3.3.3`.
- GitHub Milestones must be scheduled: have a `Due on` date set.
Otherwise, release workflow will be cancelled with a warning that nothing to release for respective SNAPSHOT in a branch.

The logic of this release workflow:

- Take a SNAPSHOT version from a dispatched branch (Maven `help:evaluate -Dexpression="project.version"` and Gradle `gradle properties | grep "^version:"`, respectively)
- List GitHub milestones matching the candidate version and select the closest one by due on date
- Cancel workflow if no scheduled Milestone
- Call Maven or Gradle according to the project (essentially, it tests for `pom.xml` file presence) with the release version extracted from the previous job.
This job stages released artifacts using JFrog CLI into `libs-staging-local` repository on Spring Artifactory and commits `Next development version` to the branch we are releasing against
- The next job is to [verify staged artifacts](#verify-staged-artifacts)
- When verification is successful, next job promotes release from staging either to `libs-milestone-local` or `libs-release-local`(and Maven Central) according to the releasing version schema
- Then `[spring-finalize-release.yml](.github%2Fworkflows%2Fspring-finalize-release.yml)` job is executed, which generates release notes using [Spring Changelog Generator](https://github.com/spring-io/github-changelog-generator) excluding repository admins from `Contributors` section.
The `gh release create` command is performed on a tag for just released version.
And in the end the milestone is closed and specific Slack channel is notified about release.

#### Example of Release caller workflow:
https://github.com/artembilan/spring-messaging-build-tools/blob/decee2963c926c34f1f52bf373a3bc1dc09f1724/samples/release.yml#L1-L25

Such a workflow must be on every branch which is supposed to be released via GitHub actions.

The `buildToolArgs` parameter for this job means extra build tool arguments.
For example, the mentioned `dist` value is a Gradle task in the project.
Can be any Maven goal or other command line arguments.

## Verify Staged Artifacts

Then `staging` job uses the `releaseVersion` output from the previous `releaseVersion` job and dispatch the work to Gradle or Maven jobs according to the project build tool.
The `verify-staged` expects an optional `verifyStagedWorkflow` input (the `verify-staged-artifacts.yml`, by default) workflow supplied from the target project.
For example, [Spring Integration for AWS](https://github.com/spring-projects/spring-integration-aws) use `jfrog rt download` command to verify that released `spring-integration-aws-${{ inputs.releaseVersion }}.jar` is valid.
Other projects may check out their samples repository and setup release version to perform smoke tests against just staged artifacts.

Verify staged workflow sample:
./samples/verify-staged-artifacts.yml

## Gradle and Artifactory

Gradle projects must not manage `com.jfrog.artifactory` plugin anymore: the `jf gradlec` command sets up this plugin and respective tasks into a project using JFrog specific Gradle init script.
In addition, the [spring-artifactory-gradle-snapshot.yml](.github/workflows/spring-artifactory-gradle-snapshot.yml) and [spring-artifactory-gradle-release-staging.yml](.github/workflows/spring-artifactory-gradle-release-staging.yml) add `spring-project-init.gradle` script to provide an `artifactory` plugin settings for artifacts publications.
This script also adds a `nextDevelopmentVersion` task which is used when release has been staged and job is ready to push `Next development version` commit.
