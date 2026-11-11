# repo-template

The reference GitHub setup for every modulane repository. When a workflow or a setting changes in one repository, it changes here first, and the others follow this repository.

This repository is not part of any release. It is a GitHub template, so a new repository can start from it with **Use this template**, and [modulane/repo-check](https://github.com/modulane/repo-check) compares every repository with it. See [Repo check](#repo-check).

## What it holds

| Path | What it is |
| --- | --- |
| `.github/workflows/ci.yml` | CI on every pull request, ending in the single **CI passed** job that the main ruleset requires. Replace the `checks` job with the repository's own checks. |
| `.github/workflows/release.yml` | On every push to main, runs CI and then semantic-release: a `vX.Y.Z` tag and a GitHub release. Its last job gives each feature or fix listed in a pull request's **Release notes** section its own entry in the release, with the issues it closes. See [Shared workflows](#shared-workflows). |
| `.github/workflows/pr-title.yml` | Checks that the pull request title is a lowercase Conventional Commit. See [Shared workflows](#shared-workflows). |
| `.github/workflows/ai-disclosure.yml` | Fails until the pull request's **AI use** section says whether AI tools were used, and how. See [AI disclosure](#ai-disclosure). |
| `.github/workflows/pr-labels.yml` | Labels each pull request from its title (kind and `release:` label) and assigns its author, unless it is a bot. |
| `.github/workflows/labels.yml` | Syncs the repository labels from `.github/labels.json`. |
| `.github/workflows/zizmor.yml` | Audits workflow security on pull requests that change `.github/`. |
| `.github/workflows/repo-check.yml` | Every Monday, compares the repository with this one and keeps a pull request open with what has drifted. See [Repo check](#repo-check). |
| `.github/dependabot.yml` | One grouped pull request a week for the pinned actions and shared workflows, with a `ci:` prefix and a 7-day cooldown, except for modulane's own releases. |
| `.github/labels.json` | The labels every repository uses. |
| `.github/CODEOWNERS`, `.github/PULL_REQUEST_TEMPLATE.md` | Review requests, and the pull request template with its **Release notes** and **AI use** sections and checklist. |
| `.releaserc.json` | semantic-release without npm publishing. |
| `.gitattributes`, `.editorconfig` | LF line endings in every checkout, Windows included, and the editor defaults. |
| `rulesets/main.json` | Ruleset for the default branch: no deletion or force push, signed commits, squash-merged pull requests only, and the **Pull request title / Conventional Commit title**, **CI passed** and **AI disclosure / AI use section** checks are required. |
| `rulesets/maintainer-review.json` | Ruleset for the default branch: squash merges only and one approving review from a code owner. Only org owners can bypass it, and only through a pull request. |
| `settings.json` | The repository settings: squash merges only, the pull request title as the commit title and an empty commit body, and branches deleted after merge. |

Issue templates, the security policy and the code of conduct come from [modulane/.github](https://github.com/modulane/.github), so they are not repeated here.

## AI disclosure

Every pull request and every issue says whether AI tools were used, and the maintainer follows the same rule as contributors.

- **Pull requests:** the template's **AI use** section has two boxes, **No AI tool was used** and **AI tools were used**. The second needs each tool and what it did written under it. The **AI disclosure / AI use section** check fails until exactly one box is ticked, and the main ruleset requires it, so a pull request without a disclosure cannot be merged. Editing the description runs the check again. Dependabot pull requests are exempt.
- **Issues:** every issue form has a required **AI use** question, with a box to name the tools.

The check is the [modulane/ai-disclosure](https://github.com/modulane/ai-disclosure) action, pinned to a commit SHA like every other action. It needs no token: it reads the description from the event and never runs code from the pull request. Comments and reviews cannot be checked, so the contributing guide asks for the same disclosure there.

## Shared workflows

`pr-title.yml`, `pr-labels.yml`, `ai-disclosure.yml`, `labels.yml`, `zizmor.yml` and `repo-check.yml` hold only a trigger and a call to the same workflow in [modulane/workflows](https://github.com/modulane/workflows), so a change made there reaches every repository through Dependabot. Change the workflow there, not here.

- Each call is pinned to the commit SHA of a modulane/workflows release and grants the permissions the called workflow needs.
- A check from a called workflow is named after both jobs, so the main ruleset requires **Pull request title / Conventional Commit title** and **AI disclosure / AI use section**. Keep the caller job names as they are.
- `ci.yml` and `release.yml` stay in each repository, because they run the repository's own checks and release.
- `release.yml` ends with a **Release notes** job that calls `release-notes.yml` there. It rebuilds the GitHub release entries from the **Release notes** section of each pull request: one entry per feature or fix that section lists, with the issues it closes. Every run rebuilds every release, older ones included, so a change to a pull request's notes reaches its release on the next run.

## Rules every workflow follows

- Actions are pinned to commit SHAs, with a `# vX.Y.Z` comment. Dependabot keeps both up to date.
- Workflows start from `permissions: {}` or `contents: read`, and each job asks for what it needs, with a comment on every permission.
- `actions/checkout` always runs with `persist-credentials: false`.
- Untrusted event fields reach scripts through `env`, never through `${{ }}` inside a `run:` block.
- Every job has a `timeout-minutes`.
- `uses: $/.github/workflows/ci.yml` is GitHub's syntax for a workflow in the same repository. It is correct as it is.

## Starting a new repository

1. Create the repository from this template, or copy the files above.
2. Replace `README.md` with the repository's own, check that the MIT `LICENSE` fits, and keep `settings.json` and the `rulesets/` folder only until steps 6 and 7 have applied them: the other repositories do not carry them.
3. Add a `package.json` that pins Yarn in `packageManager` and has `semantic-release` in `devDependencies`, and commit `yarn.lock`. The release job installs semantic-release from the lockfile and never downloads it, so it fails without them. A package that uses npm instead follows [Publishing to npm](#publishing-to-npm).
4. In `release.yml`, replace `REPO-NAME` with the repository's name. Until then the release job never runs.
5. Replace the `checks` job in `ci.yml` and list each new job under `needs:` in `ci-passed`.
6. Apply the repository settings from `settings.json`:

   ```bash
   gh api repos/modulane/NAME --method PATCH --input settings.json
   ```

7. Apply the rulesets:

   ```bash
   gh api repos/modulane/NAME/rulesets --method POST --input rulesets/main.json
   gh api repos/modulane/NAME/rulesets --method POST --input rulesets/maintainer-review.json
   ```

   On the GitHub Free plan, rulesets only work on public repositories. A private repository gets them when it goes public, or when the org moves to a paid plan.

8. Turn on the security settings: Dependabot alerts and security updates, secret scanning with push protection, and private vulnerability reporting, which the [security policy](https://github.com/modulane/.github/blob/main/SECURITY.md) relies on.

   ```bash
   gh api repos/modulane/NAME/vulnerability-alerts --method PUT
   gh api repos/modulane/NAME/automated-security-fixes --method PUT
   gh api repos/modulane/NAME/private-vulnerability-reporting --method PUT
   echo '{"security_and_analysis":{"secret_scanning":{"status":"enabled"},"secret_scanning_push_protection":{"status":"enabled"}}}' |
     gh api repos/modulane/NAME --method PATCH --input -
   ```

   Secret scanning is free for public repositories only, so a private repository turns it on when it goes public, along with the rulesets. Workflows from a first-time contributor's fork need approval in every repository already: that setting comes from the org.

9. Run the **Labels** workflow once from the Actions tab to create the labels.
10. Install the **repo-check** GitHub App on the repository, so the **Repo check** workflow can run there.
11. If the repository should be readable by contributors, give the `contributors` team read access to it. The org base permission is "none", so a new repository is closed to them until then.

## Repo check

[modulane/repo-check](https://github.com/modulane/repo-check) compares a repository with this one: the rulesets and settings, the **CI passed** job, the pinned actions, the shared workflows, the release setup, Dependabot, the labels and the copied files. `repo-check.yml` runs it every Monday through the shared workflow, which keeps one pull request open with what a file change can bring back, as Dependabot does for dependencies. A ruleset or a setting that has drifted fails the run instead, since no pull request can change it.

So the other repositories follow a change made here on their own: merge it here, and each of them gets a pull request the next Monday, or straight away by running **Repo check** from its Actions tab.

A repository that keeps a difference on purpose lists it in `.github/repo-check.yml`, with the reason:

```yaml
accept:
  - finding: 'files:template-box:*'
    reason: The checklist names the package a pull request releases.
```

`finding` is the id that repo-check prints next to each finding, and `*` matches any text. To check a repository by hand, before or after a change here, run `npx @modulane/repo-check --reference ../repo-template` in its clone.

## Publishing to npm

`release.yml` here only creates tags and GitHub releases, as in nuxt-core and nuxt-extras. A package published to npm, like [dev-workspace-cli](https://github.com/modulane/dev-workspace-cli/blob/main/.github/workflows/release.yml), adds:

- an `npm` environment on the release job, with the package URL,
- the `id-token: write` permission, for trusted publishing with provenance,
- a check that npm is 11.5.1 or newer, and `npm audit signatures` after the install,
- `@semantic-release/npm` in `.releaserc.json`.

## Monorepos

A repository with several packages that are released on their own, like [nuxt-extras](https://github.com/modulane/nuxt-extras), keeps this setup with three changes:

- **Releases:** `release.yml` runs [multi-semantic-release](https://github.com/anolilab/semantic-release/tree/main/packages/multi-semantic-release) instead of semantic-release. It releases each package from the commits that change its folder, with tags such as `@modulane/nuxt-ripple@0.2.0`. Set `"multi-release": { "ignorePrivate": false }` in the root `package.json` while the packages are private.
- **Titles:** `pr-title.yml` passes `package-scope: true` to the shared workflow: a `feat:` or `fix:` title must name, as its scope, a package that the pull request changes (`fix(ripple): …`).
- **First releases:** semantic-release makes a package's first release 1.0.0. Before merging the pull request that adds a package, tag `main` with `@modulane/<package>@0.0.0` so it starts at 0.1.0.

## License

[MIT](LICENSE)
