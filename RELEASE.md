# Releasing the base14 build of chproxy

`base-14/chproxy` is base14's fork of [ContentSquare/chproxy](https://github.com/ContentSquare/chproxy).
Fork `master` is the source of the images we run. It carries changes that aren't in upstream yet,
e.g. configurable redis client timeouts ([ContentSquare/chproxy#597](https://github.com/ContentSquare/chproxy/pull/597)).

Releases are **manual**. Pushing to `master`, or pushing a tag, never publishes anything.

## What a release produces

| | |
|---|---|
| Image | `010526246885.dkr.ecr.ap-south-1.amazonaws.com/chproxy:<tag>` |
| Platform | `linux/arm64` only. Every target cluster is arm64 |
| Binary | static (`CGO_ENABLED=0`), with the tag, commit and build time in `chproxy -version` |
| Workflow | [`.github/workflows/base14-release.yml`](.github/workflows/base14-release.yml) |

The workflow runs `go test ./...`, builds the binary, then builds the image from the repo's `Dockerfile`
and pushes it to ECR. The `Dockerfile` copies in the prebuilt binary; it doesn't compile anything. Nothing else is published: there's no GitHub release, no Docker Hub image and no Helm chart.

## Tag format

```text
v<upstream version>-b14.<n>
```

- `<upstream version>` is the latest upstream release contained in fork `master`, e.g. `1.30.0`.
- `<n>` starts at `1` and increments for each base14 release on top of that upstream version.
  It resets to `1` when we sync to a newer upstream release.

Examples: `v1.30.0-b14.1`, `v1.30.0-b14.2`, `v1.31.0-b14.1`.

ECR tags are mutable, so never reuse a tag. Always bump `<n>`.

Find the last released tag:

```bash
aws ecr describe-images --region ap-south-1 --repository-name chproxy \
  --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags' --output text
```

## Steps

1. **Make sure `master` is green.** It must contain everything you want to ship.

   ```bash
   make test
   GOTOOLCHAIN=go1.24.4 go run github.com/golangci/golangci-lint/cmd/golangci-lint@v1.64.8 run
   ```

   (`make lint` needs golangci-lint v1, because the repo config predates v2. The command above pins it.)
   `TestReverseProxy_ServeHTTP1/queue_overflow_for_user` is flaky upstream too. Re-run it before
   treating it as a regression.

2. **Start the workflow** on `master`, using the GitHub UI or the CLI:

   - UI: *Actions → base14-release → Run workflow*, branch `master`, tag `v1.30.0-b14.1`.
   - CLI:

     ```bash
     gh workflow run base14-release.yml -R base-14/chproxy --ref master -f tag=v1.30.0-b14.1
     gh run watch -R base-14/chproxy
     ```

   The workflow refuses any ref other than `master` and any tag that doesn't match the format above.

3. **Optionally tag the commit** so the image can be traced back to it in git:

   ```bash
   git tag v1.30.0-b14.1 <commit-sha> && git push origin v1.30.0-b14.1
   ```

   Pushing this tag doesn't start the workflow. Upstream's `goreleaser` workflow ignores `-b14.` tags.

4. **Verify the image:**

   ```bash
   aws ecr get-login-password --region ap-south-1 \
     | docker login --username AWS --password-stdin 010526246885.dkr.ecr.ap-south-1.amazonaws.com
   docker run --rm --platform linux/arm64 \
     010526246885.dkr.ecr.ap-south-1.amazonaws.com/chproxy:v1.30.0-b14.1 -version
   ```

5. **Roll it out.** Bump the image tag in the chproxy Helm chart. The chart lives in its own repository
   and is published to `charts.b14.dev`.

## Syncing with upstream

```bash
git remote add upstream https://github.com/ContentSquare/chproxy.git  # once
git fetch upstream --tags
git merge upstream/master   # or a release tag, e.g. git merge v1.31.0
```

Use a merge, not a rebase. Fork `master` is published, and rebasing it onto upstream would rewrite its history
and need a force push. For our own `master`, keep pulling with `git pull --rebase` as usual.

Resolve conflicts in favour of upstream when one of our changes has since been merged upstream
(e.g. #597 if accepted). Then release with the new upstream version and `-b14.1`.
Keep base14-specific CI in `base14-*` files so syncing doesn't conflict with upstream's workflows.

## One-time setup

The workflow needs the following before its first run. This is tracked in Jira **B14-2392**.

- **base14-infra:**
  - `chproxy` added to `tf-infra/base14_infra/aws/ecr_repos.py`.
  - `repo:base-14@176211094/chproxy@1409866318:ref:refs/heads/master` added to `gha_tf_assume_policy`
    (`tf-infra/base14_infra/aws/terrafrom_gitops_policy.py`). The repo uses GitHub's immutable OIDC subject format.
- **Repo variables** on `base-14/chproxy`:
  - `BASE14_CDK_ASSUME_ROLE` = `arn:aws:iam::010526246885:role/cdktf-gha-gitops-role`
  - `BASE14_REGION` = `ap-south-1`
- **GitHub Actions** enabled on the fork, with the org's `default` runner group (`scout-hz-hel1-gha-runner`) allowing this repo.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | OIDC subject missing from the trust policy, or the workflow was run from a ref other than `master` |
| `repository with name 'chproxy' does not exist` | ECR repo not created yet (base14-infra) |
| Job stays queued | the runner group doesn't allow `base-14/chproxy` |
| `tag '…' does not match` | tag isn't `v<x.y.z>-b14.<n>` |
