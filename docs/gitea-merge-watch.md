# Gitea pull request watch and merge verification

This maintainer-verification record documents the Gitea provider path implemented by `bin/fm-pr-check.sh`, `bin/fm-pr-poll.sh`, and `bin/fm-pr-merge.sh`.

## Supported identity

A canonical Gitea pull request URL is `https://<host>/<owner>/<repository>/pulls/<number>`.

The host may include a decimal HTTP port for a self-hosted instance.

The provider stores the complete URL-derived host and `owner/repository` path in its private poll sidecar and reconstructs the URL before every poll.

`github.com` pull URLs and GitLab `/-/merge_requests/` URLs remain separate provider shapes.

## gitea-axi contract

Gitea operations use the installed `gitea-axi` CLI rather than a second REST implementation in Firstmate.

The CLI receives `--repo owner/repository` and `--host https://<host>` derived only from the canonical URL.

The CLI reads `GITEA_PAT` through its documented authentication resolution and Firstmate never prints, stores, or adds that value to task records or pull request text.

The view command is requested with `--json` so registration, status reads, and post-merge confirmation can validate named fields instead of parsing display text.

The merge command receives `--head-sha` from the live pre-merge view and `--method merge|rebase|squash`.

`gitea-axi pr merge` performs its own immediate head comparison and post-mutation confirmation, and Firstmate performs a second live merged-state read before reporting a landed outcome.

A failed or unconfirmed provider call leaves the merge poll armed and does not create a landed result.

The CLI's `GITEA_AXI_TIMEOUT_MS` setting remains the provider request timeout, while the watcher bounds the whole poll with `FM_CHECK_TIMEOUT`.

A poll stays silent on a missing CLI, timeout, authentication failure, malformed JSON, or other provider error because silence cannot be mistaken for a merge.

Registration and synchronous merge paths report those errors before claiming readiness or a landed result.

## No-CI policy

`gitea-axi pr checks --json` returns a commit SHA, summary, and checks array.

An empty checks array with the explicit `none (...)` summary is recorded as an explicit no-checks status and is accepted when its SHA equals the live pull request head.

A non-empty array must contain only `pass` checks at that same head.

A failing, pending, unknown, stale, or unreadable check result refuses a guarded merge rather than becoming a provider failure or an implicit approval.

## Verification

The installed provider was inspected on 2026-09-23.

```text
$ gitea-axi --version
gitea-axi 0.1.0

$ gitea-axi pr --help
gitea-axi pr <sub>
  list    [--state open|closed|all] [--limit N]
  view    <number>
  create  --title T --head BRANCH --base BRANCH [--body B|--body-file F|-]
  merge   <number> [--method merge|rebase|squash] [--head-sha SHA]
  review  <number> (--approve | --request-changes | --comment) [--body B]
  checks  <number>
  diff    <number>
```

The focused hermetic checks are `bash tests/fm-pr-check-security.test.sh` and `bash tests/fm-pr-merge.test.sh`.

The complete shell and documentation checks are `bin/fm-lint.sh` and `bin/fm-doc-audience-check.sh`.
