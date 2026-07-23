# Release Flow — GitHub Action

A small, dependency-free composite action that turns a single manual choice
(`patch` / `minor` / `major`) into a complete release:

1. **Bumps the version** in the right place for your language
   (`package.json` for Node, or **the git tags** otherwise — no file to manage).
2. **Resolves the release notes**: renames the unreleased
   `## NEXT RELEASE` section to the new version number and extracts its body.
3. **Commits and tags** that state — so the tag captures the finished notes.
4. **Publishes a GitHub Release** from the extracted body.
5. **Re-opens a fresh `## NEXT RELEASE` section** for the next cycle — so
   contributors only ever *fill in* a section, never add one.

> The placeholder is the whole point: at authoring time you don't yet know
> whether the next release is a patch, minor or major, so you write under a
> stable `## NEXT RELEASE` heading. The action stamps the real number at release
> time, once the bump is known.

## Quick start

```yaml
name: Release

on:
  workflow_dispatch:            # adds a "Run workflow" button in the Actions tab
    inputs:
      bump:
        description: 'Version bump type'
        required: true
        default: patch
        type: choice            # <-- this is what renders the patch/minor/major dropdown
        options:
          - patch
          - minor
          - major

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.PAT_TOKEN }}   # PAT so the pushed tag can trigger a publish workflow
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: npm }
      - run: npm ci && npm run build && npm test
      - uses: softwarity/release-flow@v1
        with:
          bump: ${{ inputs.bump }}           # the value picked in the dropdown
```

## Choosing patch / minor / major

The bump is **not** decided by the action — *you* pick it when you launch the
workflow. The `workflow_dispatch` + `type: choice` block above is what GitHub
turns into a dropdown:

> **Actions** tab → pick the *Release* workflow → **Run workflow** → choose
> `patch` / `minor` / `major` → **Run workflow**.

That choice arrives as `${{ inputs.bump }}` (equivalently
`${{ github.event.inputs.bump }}`) and is forwarded to the action's `bump`
input. The action just trusts that string — so you can trigger it any other way
too (a PR label, a `release` event, the API…) as long as you pass
`patch | minor | major` into `bump:`. The dropdown is simply the common case.

A full, copy-pasteable workflow is in [`examples/release.yml`](examples/release.yml).

## Your `RELEASE_NOTES.md`

```markdown
# Release Notes

## NEXT RELEASE

### Features

- The thing you just built

---

## 2.0.1

- The previous release
```

During the cycle, contributors edit the `## NEXT RELEASE` section. On release
with `bump: patch`, given the last version `2.0.1`, the action produces:

```markdown
# Release Notes

## NEXT RELEASE          <- fresh & empty: where the NEXT release is drafted

---

## 2.0.2                 <- was NEXT RELEASE; this is the v2.0.2 release

### Features

- The thing you just built

---

## 2.0.1
...
```

…and a GitHub Release **v2.0.2** whose body is the `### Features …` block — the
empty `## NEXT RELEASE` is **not** part of it: the body is sliced from the
`## 2.0.2` section *before* the placeholder is re-added.

By default (`single-commit: true`) this is **one commit**, so the `v2.0.2` tag
includes the empty `## NEXT RELEASE` at the top. That's intentional — it's just
the (still-empty) drafting section for the next version, and it never reaches the
GitHub Release body or the published package. Want a tag with no placeholder at
all? Set `single-commit: false` (see [One commit per release](#one-commit-per-release-and-the-tag)).

If the notes file doesn't exist, the action creates it.

## Versioning convention

The **last published** version is the starting point; the action bumps from it.
For Node that's `npm version <bump> --no-git-tag-version` (which also keeps
`package-lock.json` in sync). With no manifest, the last published version is read
straight from the **git tags** — the highest `vX.Y.Z`, with moving pointers like
`v1` ignored, and `0.0.0` if there are none — so there is no version file to keep
in sync. A committed `.version` file is still available via `language: generic`.

## Language support

| `language` | Version source | Bump |
|------------|----------------|------|
| `node`     | `package.json` `"version"` | `npm version <bump>` |
| `tag`      | the highest `vX.Y.Z` **git tag** (no file) | semver math |
| `maven_docker` | git tag → written into `pom.xml` (`mvn versions:set`, run in Docker) | semver math |
| `generic`  | `.version` (or `version-file`) | semver math |
| `auto` (default) | `package.json`→`node`, `pom.xml`+`Dockerfile`→`maven_docker`, else `tag` | — |

`tag` and `maven_docker` read the version from the git tags, so check out with
`fetch-depth: 0`. **`maven_docker`** also needs Docker on the runner: it runs
`mvn versions:set` in the `maven-image` container (default `maven:3-eclipse-temurin`)
— no local Java/Maven needed, and real `mvn` updates **only** the project
`<version>`, never the `<parent>` or dependency versions (a regex would). It suits a
Maven project shipped as a Docker image, where the tag is the source of truth and the
pom version just needs to stay in sync. `Python` / `Rust` / `PHP` are on the roadmap.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `bump` | `patch` | `patch` \| `minor` \| `major` |
| `notes-file` | `RELEASE_NOTES.md` | Path to the notes markdown file |
| `placeholder` | `NEXT RELEASE` | Heading text of the unreleased section (no `## `) |
| `language` | `auto` | `auto` \| `node` \| `tag` \| `maven_docker` \| `generic` |
| `version-file` | `.version` | Version file for `language: generic` |
| `maven-image` | `maven:3-eclipse-temurin` | Docker image running `mvn versions:set` for `language: maven_docker` |
| `tag-prefix` | `v` | Prefix prepended to the version to form the tag |
| `create-release` | `true` | Create a GitHub Release from the notes |
| `release-draft` | `false` | Create the Release as a draft |
| `release-prerelease` | `false` | Mark the Release as a prerelease |
| `push` | `true` | Push the commits and the tag |
| `major-tag` | `false` | Also force-move the major tag (`v1`) to this release — for publishing reusable actions |
| `single-commit` | `true` | One commit per release (fold the placeholder in; the tag then includes the empty `## NEXT RELEASE`) |
| `commit-user-name` | `github-actions[bot]` | git `user.name` for the commits |
| `commit-user-email` | `github-actions[bot]@users.noreply.github.com` | git `user.email` |
| `token` | `${{ github.token }}` | Token used to create the Release (needs `contents: write`) |
| `dry-run` | `false` | Edit files but skip commit/tag/push/release |

## Outputs

| Output | Description |
|--------|-------------|
| `version` | New version, no prefix (e.g. `2.0.2`) |
| `previous-version` | Version before the bump |
| `tag` | Tag created, with prefix (e.g. `v2.0.2`) |
| `notes` | Extracted release-notes body |
| `release-url` | URL of the created GitHub Release |
| `notes-file-created` | `true` if the notes file was created this run |

## Permissions &amp; tokens

- The job needs `permissions: contents: write` (push commits/tags, create the Release).
- The **GitHub Release** is created with `token` (default `GITHUB_TOKEN`).
- The **git push** uses whatever credentials `actions/checkout` set up. If a
  pushed tag must trigger another workflow (e.g. an NPM publish on `push: tags`),
  check out with a **PAT** — pushes made with `GITHUB_TOKEN` do not trigger
  workflows.

## One commit per release (and the tag)

By default (`single-commit: true`) a release is **one commit**: the resolved
notes (`## X.Y.Z`) and a fresh empty `## NEXT RELEASE` are folded together, and
that commit is tagged — so contributors pull a single commit per release. The
trade-off: the `RELEASE_NOTES.md` *inside the tag* shows an empty `## NEXT
RELEASE` at the top. It's cosmetic — the GitHub Release body (extracted before)
and the published package are unaffected.

Set `single-commit: false` for a **pure tag** instead: the action tags the
`version + resolved notes` commit (no placeholder), then re-opens `## NEXT
RELEASE` in a *separate* follow-up commit carrying `[skip ci]`. The tag never
contains the placeholder, at the cost of two commits per release.

(Folding with `git commit --amend` after tagging is deliberately **not** used: it
would leave the tag pointing at a commit *off the branch*, breaking `git
describe` and GitHub's "N commits since this release".)

## Migrating an existing `RELEASE_NOTES.md`

If your top section is currently a *guessed* next number (e.g. `## 0.2.10`),
rename that heading once to `## NEXT RELEASE`. From then on the action fills it in.

## Publishing a reusable action

If the repo you're releasing **is itself a GitHub Action** (consumed as
`you/action@v1`), set `major-tag: true`. After tagging `vX.Y.Z`, the action also
force-moves the major tag (`vX`) to the same commit, so your `@v1` consumers get the
new release with no extra step — no `npm`, no PAT needed. release-flow uses this on
itself (see its own [`.github/workflows/release.yml`](.github/workflows/release.yml)).

## Local development

```bash
node scripts/test.mjs     # pure-logic smoke tests (notes parsing + semver)
```

Run the whole flow without side effects on any repo with:

```yaml
- uses: softwarity/release-flow@v1
  with: { bump: patch, dry-run: true }
```

## License

[Apache-2.0](LICENSE)
