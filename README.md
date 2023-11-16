# GitHub Actions Workflows for Spring Projects

This project containers reusable GitHub Actions Workflows to build Spring projects with Gradle or Maven.
SNAPSHOT and Release workflows uses JFrog CLI to publish artifacts into Artifactory.

The `spring-artifactory-release.yml` must be used like this in the target workflow:
```
name: Release

on:
  workflow_dispatch:

run-name: Release current version for branch ${{ github.ref_name }}

jobs:
  release:
    uses: artembilan/spring-messaging-build-tools/.github/workflows/spring-artifactory-release.yml@main
    with:
      buildToolArgs: dist
      verifyStagedWorkflow: verify-staged-artifacts.yml
    secrets:
      GH_ACTIONS_REPO_TOKEN: ${{ secrets.GH_ACTIONS_REPO_TOKEN }}
      GRADLE_ENTERPRISE_CACHE_USER: ${{ secrets.GRADLE_ENTERPRISE_CACHE_USER }}
      GRADLE_ENTERPRISE_CACHE_PASSWORD: ${{ secrets.GRADLE_ENTERPRISE_CACHE_PASSWORD }}
      GRADLE_ENTERPRISE_SECRET_ACCESS_KEY: ${{ secrets.GRADLE_ENTERPRISE_SECRET_ACCESS_KEY }}
      JF_ARTIFACTORY_SPRING: ${{ secrets.JF_ARTIFACTORY_SPRING }}
      SPRING_RELEASE_SLACK_WEBHOOK_URL: ${{ secrets.SPRING_RELEASE_SLACK_WEBHOOK_URL }}
      OSSRH_URL: ${{ secrets.OSSRH_URL }}
      OSSRH_S01_TOKEN_USERNAME: ${{ secrets.OSSRH_S01_TOKEN_USERNAME }}
      OSSRH_S01_TOKEN_PASSWORD: ${{ secrets.OSSRH_S01_TOKEN_PASSWORD }}
      OSSRH_STAGING_PROFILE_NAME: ${{ secrets.OSSRH_STAGING_PROFILE_NAME }}
      GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
```
on every branch which is supposed to be released via GitHub actions.

The `buildToolArgs` parameter for this job means extra build tool arguments.
For example, the mentioned `dist` value is a Gradle task in the project.
Can be any Maven goal or other command line arguments.

The mentioned secrets must be passed explicitly since these reusable workflows might be in different GitHub org than target project. 

When a release workflow is dispatched, the `releaseVersion` job determines the milestone to release against the current SNAPSHOT version in the branch.
If no milestone scheduled, the whole workflow is cancelled.
Then `staging` job uses the `releaseVersion` output from the previous `releaseVersion` job and dispatch the work to Gradle or Maven jobs according to the project build tool.
The `verify-staged` expects an optional `verifyStagedWorkflow` input (the `verify-staged-artifacts.yml`, by default) workflow supplied from the target project.
For example, [Spring Integration for AWS](https://github.com/spring-projects/spring-integration-aws) use `jfrog rt download` command to verify that released `spring-integration-aws.jar` is valid.
Other projects may check out their samples repository and setup release version to perform smoke tests against just staged artifacts.

The next job in the workflow is to promote release from staging either to `libs-milestone-local` or `libs-release-local`(and Maven Central) according to the releasing version schema.

After promotion, the `finalize` job is executed, which generates release notes using [Spring Changelog Generator](https://github.com/spring-io/github-changelog-generator) excluding repository admins from `Contributors` section.
Then the `gh release create` command is performed on a tag for just released version.
And in the end the milestone is closed and specific Slack channel is notified about release.

Gradle projects must not manage `com.jfrog.artifactory` plugin anymore: the `jf gradlec` command sets up this plugin and respective tasks into a project using JFrog specific Gradle init script.
In addition, the `spring-artifactory-gradle-snapshot.yml` and `spring-artifactory-gradle-release-staging.yml` add `spring-project-init.gradle` script to provide an `artifactory` plugin settings for artifacts publications.
This script also adds a `nextDevelopmentVersion` task which is used when release has been staged and job is ready to push `Next development version` commit.
