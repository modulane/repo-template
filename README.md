# repo-template

The reference GitHub setup for every modulane repository. When a workflow or a setting changes in one repository, it changes here first, and the others follow this repository.

This repository is private and is not part of any release. It is also a GitHub template, so a new repository can start from it with **Use this template**.

## Access

This repository stays private, and only org owners can open it. The org base permission is "none", and members cannot create repositories, so only org owners can use the template too. Never give the `contributors` team or an outside collaborator access to it, and keep forking turned off.

On the GitHub Free plan, rulesets and branch protection only work on public repositories, so the rulesets below are not enforced here and `main` is unprotected. Changes still go through squash-merged pull requests. The rules apply as soon as the org moves to a paid plan.

## What it holds

| Path | What it is |
| --- | --- |
| `.github/workflows/ci.yml` | CI on every pull request, ending in the single **CI passed** job that the main ruleset requires. Replace the `checks` job with the repository's own checks. |
| `.github/workflows/release.yml` | On every push to main, runs CI and then semantic-release: a `vX.Y.Z` tag and a GitHub release. |
| `.github/workflows/pr-title.yml` | Checks that the pull request title is a lowercase Conventional Commit. |
| `.github/workflows/ai-disclosure.yml` | Fails until the pull request's **AI use** section says whether AI tools were used, and how. See [AI disclosure](#ai-disclosure). |
| `.github/workflows/pr-labels.yml` | Labels each pull request from its title (kind and `release:` label) and assigns its author. |
| `.github/workflows/labels.yml` | Syncs the repository labels from `.github/labels.json`. |
| `.github/workflows/zizmor.yml` | Audits workflow security on pull requests that change `.github/`. |
| `.github/dependabot.yml` | One grouped pull request a week for the pinned actions, with a 7-day cooldown and a `ci:` prefix. |
| `.github/labels.json` | The labels every repository uses. |
| `.github/CODEOWNERS`, `.github/PULL_REQUEST_TEMPLATE.md` | Review requests, and the pull request template with its **AI use** section and checklist. |
| `.releaserc.json` | semantic-release without npm publishing. |
| `.gitattributes`, `.editorconfig` | LF line endings in every checkout, Windows included, and the editor defaults. |
| `rulesets/main.json` | Ruleset for the default branch: no deletion or force push, signed commits, squash-merged pull requests only, and the **Conventional Commit title**, **CI passed** and **AI disclosure** checks are required. |
| `rulesets/maintainer-review.json` | Ruleset for the default branch: squash merges only and one approving review from a code owner. Only org owners can bypass it, and only through a pull request. |

Issue templates, the security policy and the code of conduct come from [modulane/.github](https://github.com/modulane/.github), so they are not repeated here.

## AI disclosure

Every pull request and every issue says whether AI tools were used, and the maintainer follows the same rule as contributors.

- **Pull requests:** the template's **AI use** section has two boxes, **No AI tool was used** and **AI tools were used**. The second needs each tool and what it did written under it. The **AI disclosure** check fails until exactly one box is ticked, and the main ruleset requires it, so a pull request without a disclosure cannot be merged. Editing the description runs the check again. Dependabot pull requests are exempt.
- **Issues:** every issue form has a required **AI use** question, with a box to name the tools.

The check is the [modulane/ai-disclosure](https://github.com/modulane/ai-disclosure) action, pinned to a commit SHA like every other action. It needs no token: it reads the description from the event and never runs code from the pull request. Comments and reviews cannot be checked, so the contributing guide asks for the same disclosure there.

## Rules every workflow follows

- Actions are pinned to commit SHAs, with a `# vX.Y.Z` comment. Dependabot keeps both up to date.
- Workflows start from `permissions: {}` or `contents: read`, and each job asks for what it needs, with a comment on every permission.
- `actions/checkout` always runs with `persist-credentials: false`.
- Untrusted event fields reach scripts through `env`, never through `${{ }}` inside a `run:` block.
- Every job has a `timeout-minutes`.
- `uses: $/.github/workflows/ci.yml` is GitHub's syntax for a workflow in the same repository. It is correct as it is.

## Starting a new repository

1. Create the repository from this template, or copy the files above.
2. Replace `README.md` with the repository's own, check that the MIT `LICENSE` fits, and keep the `rulesets/` folder only until step 7 has applied it: the other repositories do not carry it.
3. Add a `package.json` that pins Yarn in `packageManager` and has `semantic-release` in `devDependencies`, and commit `yarn.lock`. The release job installs semantic-release from the lockfile and never downloads it, so it fails without them. A package that uses npm instead follows [Publishing to npm](#publishing-to-npm).
4. In `release.yml`, replace `REPO-NAME` with the repository's name. Until then the release job never runs.
5. Replace the `checks` job in `ci.yml` and list each new job under `needs:` in `ci-passed`.
6. Apply the repository settings: squash merges only, the pull request title as the commit title and an empty commit body, and branches deleted after merge.

   ```bash
   gh api -X PATCH repos/modulane/NAME \
     -F allow_squash_merge=true -F allow_merge_commit=false -F allow_rebase_merge=false \
     -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=BLANK \
     -F delete_branch_on_merge=true
   ```

7. Apply the rulesets:

   ```bash
   gh api repos/modulane/NAME/rulesets --method POST --input rulesets/main.json
   gh api repos/modulane/NAME/rulesets --method POST --input rulesets/maintainer-review.json
   ```

   On the GitHub Free plan, rulesets only work on public repositories. A private repository gets them when it goes public, or when the org moves to a paid plan.

8. Run the **Labels** workflow once from the Actions tab to create the labels.
9. If the repository should be readable by contributors, give the `contributors` team read access to it. The org base permission is "none", so a new repository is closed to them until then.

## Publishing to npm

`release.yml` here only creates tags and GitHub releases, as in nuxt-core and nuxt-extras. A package published to npm, like [dev-workspace-cli](https://github.com/modulane/dev-workspace-cli/blob/main/.github/workflows/release.yml), adds:

- an `npm` environment on the release job, with the package URL,
- the `id-token: write` permission, for trusted publishing with provenance,
- a check that npm is 11.5.1 or newer, and `npm audit signatures` after the install,
- `@semantic-release/npm` in `.releaserc.json`.

## Monorepos

A repository with several packages that are released on their own, like [nuxt-extras](https://github.com/modulane/nuxt-extras), keeps this setup with three changes:

- **Releases:** `release.yml` runs [multi-semantic-release](https://github.com/anolilab/semantic-release/tree/main/packages/multi-semantic-release) instead of semantic-release. It releases each package from the commits that change its folder, with tags such as `@modulane/nuxt-ripple@0.2.0`. Set `"multi-release": { "ignorePrivate": false }` in the root `package.json` while the packages are private.
- **Titles:** `pr-title.yml` has a second step: a `feat:` or `fix:` title must name, as its scope, a package that the pull request changes (`fix(ripple): …`).
- **First releases:** semantic-release makes a package's first release 1.0.0. Before merging the pull request that adds a package, tag `main` with `@modulane/<package>@0.0.0` so it starts at 0.1.0.

## License

[MIT](LICENSE)
