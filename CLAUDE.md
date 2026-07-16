# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not** an application with source code to build/test — it's a library of *reusable* GitHub Actions workflows (`.github/workflows/*.yml`) and composite actions (`.github/actions/*/action.yml`) that Spring projects (Gradle or Maven) call via `uses: spring-io/spring-github-workflows/.github/workflows/<file>.yml@main`. There is no compiler, package manager, or test suite. "Development" means editing YAML and reasoning about GitHub Actions semantics.

`samples/*.yml` contains example caller workflows referenced from README.md (each has a permalinked line range in the README pointing at a specific commit — when a sample changes meaningfully, consider whether the README permalink should be updated too).

## Validating changes

There is no CI that lints or tests this repo's own workflows. When editing YAML:
- Check indentation and `${{ }}` expression syntax carefully — GitHub Actions fails at workflow-parse time with often-unhelpful errors.
- Reusable workflows (`on: workflow_call`) declare their contract via `inputs`/`secrets`/`outputs` at the top — keep these in sync with what the jobs below actually consume, and update `README.md` when adding/removing/renaming any of them.
- References to sibling reusable workflows within this repo use the local path form `uses: ./.github/workflows/x.yml` (works because caller and callee are checked out together at runtime for the top-level workflow); references to composite actions in this repo from *within* a reusable workflow use the full form `uses: spring-io/spring-github-workflows/.github/actions/x@main` (composite actions can't be referenced by local path from a called workflow). Follow whichever form the surrounding file already uses.
- External dependencies are pinned to a version tag (e.g. `spring-io/spring-release-actions/...@0.0.5`, `actions/checkout@v7`) except this repo's own actions, which are pinned to `@main` in source and get rewritten to a release tag only during this repo's own release process (see below) — never hand-edit those to a version tag.

## Releasing this repository itself

`.github/workflows/release-workflows.yml` is manually dispatched to cut a release of *this* repo (distinct from the Spring project release workflow below, which releases *consumer* projects). It computes the next integer tag from the latest `vN` tag, rewrites `@main` → `@vN` for local composite-action references across `.github/workflows/*.yml`, commits/tags/pushes, generates a GitHub release, then reverts the rewrite back to `@main` for continued development. Don't manually pin local action refs to a version — this script owns that.

## Architecture: the release pipeline

The most complex piece is the Gradle/Maven **release** workflow chain, invoked by a consumer project's own caller workflow (see `samples/release-with-gradle.yml`). It's a tree of nested `workflow_call` reusable workflows (already 3 levels deep from the caller — do not add another caller layer on top, see the README warning):

```
spring-artifactory-gradle-release.yml (or -maven-release.yml)
├─ spring-find-release-version.yml      → resolves the scheduled Milestone to release
│   └─ actions/spring-compute-milestone-to-release  (handles hotfix vs. normal versioning)
├─ spring-pre-release.yml               → cancels the run if the milestone has open issues or last SNAPSHOT build failed
├─ spring-artifactory-gradle-build.yml (composite action) / inline Maven build → stages artifacts to libs-staging-local
├─ (verify-staged job, inline)          → actions/spring-dispatch-workflow-and-wait dispatches the consumer's
│                                          verifyStagedWorkflow (default verify-staged-artifacts.yml) and polls until done
├─ spring-artifactory-promote-release.yml → `jfrog rt build-promote` to libs-milestone-local or libs-release-local
├─ spring-artifactory-deploy-to-maven-central.yml   (skipped if bundleName input given)
├─ spring-enterprise-release-bundle.yml             (used instead, when bundleName input given)
└─ spring-finalize-release.yml          → tags, bumps to next SNAPSHOT, pushes the commit + tag. Ends there.
```

Pushing that tag is a separate trigger, not a job in the tree above: `spring-post-release.yml` is a `push: tags`-triggered reusable workflow (its own caller, wired up once per consumer repo alongside the release caller — see README) that derives the milestone/hotfix from the pushed tag name and does the rest — changelog generation (spring-io/github-changelog-generator), GH release creation, milestone close, spring.io website update, chat announcement, next-milestone creation.
Because it's tag-triggered rather than job-graph-triggered, the same workflow fires identically whether `spring-finalize-release.yml` pushed the tag or something external did (e.g. a release train orchestrator) — **do not** re-add a direct job-graph call from the pipeline above to `spring-post-release.yml`, that would double-fire it for every self-release (the original tag push already triggers it once).

Key conventions this pipeline assumes about the *calling* project (documented in more detail in README.md):
- Version scheme: `major.minor.patch[-M{n}|-RC{n}]`, SNAPSHOT suffixed `-SNAPSHOT`.
- A GitHub Milestone must exist with a title exactly matching the version to release, and must have a due date set — undue/missing milestones cause the run to self-cancel with a warning rather than fail loudly (search for `gh run cancel` to find these guard points).
- Everything downstream keys off `needs.release-version.outputs.releaseVersion`/`hotfix` — trace a value's origin by following `needs.<job>.outputs.*` back to `spring-find-release-version.yml`.

`spring-cut-release-branch.yml` is the separate, simpler workflow that creates a new `release/<version>` branch + milestone from a ref (used when branching off a maintenance line, not part of the release chain above).

## Other independent workflows

These are standalone (no nesting) and can be reasoned about in isolation:
- `spring-gradle-pull-request-build.yml` / `spring-maven-pull-request-build.yml` — PR check (`gradlew check` / `mvnw verify`).
- `spring-artifactory-gradle-snapshot.yml` / `spring-artifactory-maven-snapshot.yml` — CI SNAPSHOT publish to Artifactory.
- `spring-merge-dependabot-pr.yml` — relabels Dependabot PRs (dependency-upgrade → task for dev-dependency groups), assigns the current milestone, and queues auto-merge; has special-case logic to reject Dependabot's incorrect `-SNAPSHOT`-skipping-GA upgrades (see the long comment in that file before touching this logic).
- `spring-cherry-pick.yml` — parses `Auto-cherry-pick to X.Y.x & A.B.x` out of a push's commit message and cherry-picks (`-x`) to each named branch, stripping the trigger phrase from the picked commit message.
- `spring-backport-issue.yml` — fires on commits containing `Fixes:`/`Closes:`, delegates to `spring-io/backport-bot`.
- `spring-announce-milestone-planning.yml` — posts to Google Chat when a milestone's `due_on` is set/changed.
- `spring-trigger-dependabot-updates.yml` — forces a Dependabot re-scan by toggling the executable bit on `dependabot.yml`.
- `spring-build-and-deploy-docs.yml` / `spring-dispatch-docs-build.yml` — Antora docs build + rsync publish + Cloudflare cache bust.
- `merge-dependabot-pr.yml` and `release-workflows.yml` under `.github/workflows/` (not called externally) are this *repository's own* CI, not part of the reusable library.

## Secrets contract

Every reusable workflow declares its own `secrets:` under `workflow_call` — treat that block as the authoritative list of what a caller must pass (usually via `secrets: inherit` in a nested call, or explicitly from repo/org secrets at the top level). README.md lists the full set of org-level secrets consumer projects are expected to provision (`GH_ACTIONS_REPO_TOKEN`, `JF_ARTIFACTORY_SPRING`, `ARTIFACTORY_USERNAME`/`PASSWORD`, `CENTRAL_TOKEN_*`, `GPG_*`, `DEVELOCITY_ACCESS_KEY`, `SPRING_RELEASE_CHAT_WEBHOOK_URL`) — several are optional and gate optional steps via `if:` (e.g. signing only runs if GPG secrets are present, chat announce only if the webhook secret is present). When adding a new optional integration, follow this same "declare as `required: false`, guard the step with `if:`" pattern rather than making it mandatory.