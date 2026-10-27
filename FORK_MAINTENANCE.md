# Fork maintenance: upstream sync and upgrades

This repository is the `nodescout` fork of [verygoodplugins/whatsapp-mcp](https://github.com/verygoodplugins/whatsapp-mcp). The fork exists so the operator keeps a copy of the code they run, can change it, and decides when upstream changes reach their machine. Upstream's `README.md` describes the project itself.

## Branches and tags

- `main` is an exact mirror of upstream `main`. Nobody commits to it, and the sync workflow fast-forwards it from upstream.
- `nodescout/main` is the default branch. It is based on the upstream release currently in use, and it carries this file, the sync workflow, and any changes the fork makes. Other fork branches also take the `nodescout/` prefix.
- Upstream release tags (`vX.Y.Z`) are mirrored unchanged.
- Runtime tags have the form `nodescout-X.Y.Z`, where `X.Y.Z` is the upstream release the build is based on. The operator's machine builds and runs only `nodescout-*` tags. If the fork ever needs a second build of the same upstream release, the second tag takes a suffix chosen at that time.

## Upstream sync workflow

`.github/workflows/nodescout-upstream-sync.yml` runs daily at 06:17 UTC and can be run by hand from the Actions tab. Each run does three things in order:

1. It fetches upstream `main` and upstream tags, and lists the upstream tags this fork does not have yet.
2. For each new release tag (`vX.Y.Z`), it opens an issue titled "Upstream release vX.Y.Z is available" unless one already exists. That issue starts the upgrade process below.
3. It fast-forwards the fork's `main` to upstream `main` and pushes the new tags.

The workflow never changes `nodescout/main`. A run fails, and shows as failed in the Actions tab, when upstream rewrote `main` so the push is not a fast-forward, when upstream moved a tag this fork already has, or when the sync token has expired or lost access. Each of these needs a person to look at it.

### Sync token

GitHub refuses pushes made with the built-in `GITHUB_TOKEN` that change files under `.github/workflows/`, and upstream commits do change those files. The workflow therefore pushes with a fine-grained personal access token with these settings:

- The resource owner is `nodescout`, and repository access is limited to `nodescout/whatsapp-mcp`.
- The repository permissions are Contents (read and write) and Workflows (read and write). GitHub adds Metadata (read) automatically.
- The token is stored as the secret `UPSTREAM_SYNC_TOKEN` in the GitHub environment `upstream-sync`, whose deployment branch policy allows only `nodescout/main`.

The token lives in an environment because `main` carries upstream's workflow files, and pushes to `main` run those workflows in this repository. Any of them could read a repository-level secret. An environment secret restricted to `nodescout/main` is available only to jobs that run from `nodescout/main` and name the environment.

To renew the token, generate a new one with the same settings, replace the environment secret, and update the stored copy.

### Upstream workflows in this fork

Three upstream workflows are disabled here because they would act on this repository: `release-please.yml` (opens release pull requests and creates release tags on every push to `main`), `release.yml` (publishes release artifacts), and `dependabot-auto-merge.yml` (merges Dependabot pull requests). `ci.yml` and `security.yml` stay enabled. They build, test, and scan, and the default `GITHUB_TOKEN` in this repository is read-only.

GitHub enables a newly added workflow file by default. Each upgrade review checks upstream for new workflows and disables any that would act on this repository.

GitHub disables scheduled workflows in a public repository after 60 days with no repository activity. If that happens, re-enable the sync workflow from the Actions tab.

## Upgrade process

The process starts when a sync issue opens. `vOLD` is the upstream base of the current runtime tag, and `vNEW` is the new release.

1. Review the upstream changes with `git diff vOLD vNEW -- whatsapp-bridge whatsapp-mcp-server .github/workflows`. Check outbound network destinations, listening sockets and bind addresses, the bridge REST API's authentication, the webhook and transcription defaults, paths written to disk, environment variables, new or changed workflows, and changes to `go.mod`, `go.sum`, `pyproject.toml`, and `uv.lock`. Record the findings in the issue.
2. Merge the release into the runtime branch with `git switch nodescout/main` and `git merge vNEW`, resolving any conflicts with fork changes.
3. Tag and push with `git tag nodescout-NEW` and `git push origin nodescout/main nodescout-NEW`, where `NEW` is the release version without the `v`.
4. Rebuild and re-verify on the operator's machine by following `docs/WHATSAPP_PERSONAL_MCP.md` in the operator's Claude Code configuration repository. The re-verification includes checking the server's tool names against the permission deny list.
5. Close the issue with the new runtime tag and the verification result.
