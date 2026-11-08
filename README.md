# repo-template

The reference GitHub setup for every modulane repository. When a workflow or a setting changes in one repository, it changes here first, and the others follow this repository.

This repository is private and is not part of any release. It is also a GitHub template, so a new repository can start from it with **Use this template**.

## What it holds

| Path | What it is |
| --- | --- |
| `.github/workflows/ci.yml` | CI on every pull request, ending in the single **CI passed** job that the main ruleset requires. Replace the `checks` job with the repository's own checks. |
| `.github/workflows/release.yml` | On every push to main, runs CI and then semantic-release: a `vX.Y.Z` tag and a GitHub release. |
| `.github/workflows/pr-title.yml` | Checks that the pull request title is a lowercase Conventional Commit. |
| `.github/workflows/pr-labels.yml` | Labels each pull request from its title (kind and `release:` label) and assigns its author. |
| `.github/workflows/labels.yml` | Syncs the repository labels from `.github/labels.json`. |
| `.github/workflows/zizmor.yml` | Audits workflow security on pull requests that change `.github/`. |
| `.github/dependabot.yml` | One grouped pull request a week for the pinned actions, with a 7-day cooldown and a `ci:` prefix. |
| `.github/labels.json` | The labels every repository uses. |
| `.github/CODEOWNERS`, `.github/PULL_REQUEST_TEMPLATE.md` | Review requests and the pull request checklist. |
| `.releaserc.json` | semantic-release without npm publishing. |
| `rulesets/main.json` | Ruleset for the default branch: no deletion or force push, pull requests only, and the **Conventional Commit title** and **CI passed** checks are required. |
| `rulesets/maintainer-review.json` | Ruleset for the default branch: squash merges only and one approving review from a code owner. Only org owners can bypass it, and only through a pull request. |

Issue templates, the security policy and the code of conduct come from [modulane/.github](https://github.com/modulane/.github), so they are not repeated here.

## Rules every workflow follows

- Actions are pinned to commit SHAs, with a `# vX.Y.Z` comment. Dependabot keeps both up to date.
- Workflows start from `permissions: {}` or `contents: read`, and each job asks for what it needs, with a comment on every permission.
- `actions/checkout` always runs with `persist-credentials: false`.
- Untrusted event fields reach scripts through `env`, never through `${{ }}` inside a `run:` block.
- Every job has a `timeout-minutes`.
- `uses: $/.github/workflows/ci.yml` is GitHub's syntax for a workflow in the same repository. It is correct as it is.

## Starting a new repository

1. Create the repository from this template, or copy the files above.
2. In `release.yml`, replace `REPO-NAME` with the repository's name. Until then the release job never runs.
3. Replace the `checks` job in `ci.yml` and list each new job under `needs:` in `ci-passed`.
4. Apply the repository settings: squash merges only, the pull request title as the commit title and an empty commit body, and branches deleted after merge.

   ```bash
   gh api -X PATCH repos/modulane/NAME \
     -F allow_squash_merge=true -F allow_merge_commit=false -F allow_rebase_merge=false \
     -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=BLANK \
     -F delete_branch_on_merge=true
   ```

5. Apply the rulesets:

   ```bash
   gh api repos/modulane/NAME/rulesets --method POST --input rulesets/main.json
   gh api repos/modulane/NAME/rulesets --method POST --input rulesets/maintainer-review.json
   ```

   On the GitHub Free plan, rulesets only work on public repositories. A private repository gets them when it goes public.

6. Run the **Labels** workflow once from the Actions tab to create the labels.
7. If the repository should be readable by contributors, give the `contributors` team read access to it. The org base permission is "none", so a new repository is closed to them until then.

## Publishing to npm

`release.yml` here only creates tags and GitHub releases, as in nuxt-core and nuxt-ripple. A package published to npm, like [dev-workspace-cli](https://github.com/modulane/dev-workspace-cli/blob/main/.github/workflows/release.yml), adds:

- an `npm` environment on the release job, with the package URL,
- the `id-token: write` permission, for trusted publishing with provenance,
- a check that npm is 11.5.1 or newer, and `npm audit signatures` after the install,
- `@semantic-release/npm` in `.releaserc.json`.

## License

[MIT](LICENSE)
