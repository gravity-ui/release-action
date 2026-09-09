# release-action

## What is it?

It is a [GitHub action](https://github.com/features/actions), that does the following things:

- when you push new commits into `main` branch (or any other branch you designate in `on.push.branches` field),
  a release PR is created which includes an automatically generated CHANGELOG.md and bumped NPM version, all this is
  done according to [conventional commits spec](https://www.conventionalcommits.org/en/v1.0.0/)
- in case any new code is merged into `main` while that release PR is still open, it will be either updated (if bump
  type stays the same) or recreated (if bump type changes)
- once the maintainer is ready to publish a new package version, he merges the release PR
- package unit tests are run (at least `"test": "exit 0"` should be defined in `package.json`)
- if they are successful, a new GitHub release is created and, by default, a new package version is published to NPM.

## How should I use it?

### Setting up

Create the file `.github/workflows/release.yml` at the root of your repo, providing at least the following inputs:

- `github-token` (who does create release PR)
- `publish`, optional - set to `false` to publish from a separate job. It is `true` by default.
- `npm-token` - who publishes the NPM package. It is required when the built-in publish step is enabled.
- `node-version`, optional - which node version to use for running unit tests.
- `node-version-file`, optional - which file containing node version to use for running unit tests. If node-version and node-version-file are both provided the action will use version from node-version.
- `default-branch`, optional - branch to open release PR against.
- `npm-dist-tag`, optional - if you want to release version of package with custom tag (e.g. alpha, beta, latest).
- `prerelease`, optional - if set, create releases that are pre-major or pre-release version marked as pre-release on GitHub.

The file looks roughly like, you can change target branch, tokens and node version.
```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: gravity-ui/release-action@v2
        with:
          github-token: ${{ secrets.GRAVITY_UI_BOT_GITHUB_TOKEN }}
          npm-token: ${{ secrets.GRAVITY_UI_BOT_NPM_TOKEN }}
```

### Publishing with npm trusted publishing

Set `publish: 'false'` to disable the built-in token-based publish step. The action exposes `release-created` and
`release-sha`; pass those outputs to the reusable `npm-publish.yml` workflow. It publishes the exact released commit
from a separate, environment-protected job without a long-lived npm token:

```yaml
permissions: {}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      created: ${{ steps.release_action.outputs['release-created'] }}
      sha: ${{ steps.release_action.outputs['release-sha'] }}
    steps:
      - name: Create GitHub release
        id: release_action
        # Pin the action and reusable workflow to the same reviewed commit.
        uses: gravity-ui/release-action@<FULL_COMMIT_SHA>
        with:
          github-token: ${{ secrets.GRAVITY_UI_BOT_GITHUB_TOKEN }}
          publish: 'false'

  publish:
    needs: release
    if: ${{ needs.release.outputs.created == 'true' }}
    permissions:
      contents: read
      id-token: write
    uses: gravity-ui/release-action/.github/workflows/npm-publish.yml@<FULL_COMMIT_SHA>
    with:
      release-sha: ${{ needs.release.outputs.sha }}
      environment-name: npm-publish
      npm-dist-tag: ${{ github.ref_name == 'main' && 'latest' || 'untagged' }}
```

The reusable workflow accepts these inputs:

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `release-sha` | yes | - | Full 40-character commit SHA returned by the release action. |
| `environment-name` | no | `npm-publish` | GitHub Environment used by the publish job. |
| `npm-dist-tag` | no | `latest` | Distribution tag passed to `npm publish`. |

Configure the package's [trusted publisher](https://docs.npmjs.com/trusted-publishers/) on npmjs.com with the
**caller repository** and the **caller workflow filename** (for example, `release.yml`), not `npm-publish.yml`. npm
validates the calling workflow when `workflow_call` is used. Set its Environment field to the same value passed as
`environment-name` and allow the trusted publisher to run `npm publish` (not only `npm stage publish`). All values
are case-sensitive. The package's `repository.url` in `package.json` must also match the caller GitHub repository.

The caller's `publish` job must explicitly grant `id-token: write`, even though the reusable workflow also declares
that permission: permissions can only be maintained or reduced across reusable workflow boundaries. Do not pass
`NODE_AUTH_TOKEN`; npm obtains a short-lived credential through OIDC. The reusable workflow intentionally accepts no
secrets, uses a GitHub-hosted runner and Node.js 24, verifies that npm is at least 11.5.1, and publishes only after
checking out and verifying the supplied full commit SHA.

Pin both references to the same full commit SHA. A mutable branch or tag can otherwise change code that runs with
OIDC permission without a corresponding change in the caller repository.

This reusable workflow is intentionally limited to public root packages that use npm, have a `package-lock.json`,
and can install dependencies without registry credentials. Use a repository-specific publish job for pnpm, private
packages, private dependencies, monorepo subdirectories, or staged publishing.

### Early development

The action encourages "moving fast and breaking things" by preventing breaking changes from bumping major version
in the early stages of project development (before you reach version 1.0.0). In other words, both the
'feat: something' commits and the commits with 'BREAKING CHANGE: something' footer bump a minor component
automatically. Once you consider your project stable enough, you should add `Release-As: 1.0.0` footer in one of
the commits and see the usual effect of the 'BREAKING CHANGE: something' footer on you major version.
