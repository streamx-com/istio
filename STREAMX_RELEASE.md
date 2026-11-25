# Release a patched Istio Pilot image

## Run the release workflow

1. In the `streamx-com/istio` fork, open **Actions → Release patched Istio images → Run workflow**. Select `master` branch containing the [release workflow](.github/workflows/release-patched-version.yaml).
2. Enter the exact stable upstream tag, such as `1.32.0`, in the **tag** field.
3. Check that the run passed the ambient gateway tests and published the Pilot image. Verify the image before updating the deployment.

```bash
version=1.32.0
docker buildx imagetools inspect "ghcr.io/streamx-com/istio/pilot:${version}-streamx"
```

The workflow uses `patches/<version>` when that branch exists. Otherwise it cherry-picks the patch from the highest version below the requested tag. For example, a `1.32.1` build can reuse `patches/1.32.0`. The run log records the branch and commit it selected. Each patch branch contains one commit directly on top of its matching upstream tag.

## Resolve a cherry-pick conflict

Create an exact patch branch from the upstream tag and port the commit identified in the run log. The commands below use `1.32.1` as an example target and assume `origin` points to the StreamX fork.

```bash
version=1.32.1
previous_patch=1.32.0

git fetch https://github.com/istio/istio.git "refs/tags/$version:refs/tags/$version"
git switch -c "patches/$version" "refs/tags/$version"
git fetch --no-tags origin "refs/heads/patches/$previous_patch"
git cherry-pick -x FETCH_HEAD
```

Resolve the files listed by `git status`, preserving the gateway label filter and its tests against the new upstream code. Stage the resolved files and run `git cherry-pick --continue`. Use `git cherry-pick --abort` to abandon the attempt.

Run the same test suite as CI, using the Go version specified in `go.mod`.

```bash
go test ./pilot/pkg/serviceregistry/ambient
git log --oneline "refs/tags/$version..HEAD"
```

Once the tests pass and the log shows one patch commit, push the branch with `git push -u origin "patches/$version"` and dispatch the workflow again for the same tag. If CI reports that no eligible patch exists, create the exact branch from the upstream tag and port the StreamX filter into one commit before testing and pushing.

## Investigate failed tests

- Find the failing test in the **Test ambient gateway selection** step and reproduce it locally. If the workflow used a fallback patch, create the exact branch using the commands above. If it used an exact patch, check out that branch.
- Fix the regression on the exact branch and rerun the full ambient package suite. If the same test fails on the clean upstream tag, investigate that upstream failure before releasing. The workflow requires the suite to pass.
- Keep the branch at one commit by amending the patch after staging the fix. If the branch already exists on the fork, coordinate the update, fetch its current tip, and use a lease when pushing the amended commit. Dispatch a fresh run after the branch update.

The workflow pushes the release Git tag before the image build. If the build or upload fails after tagging, rerun with the same patch commit. If the patch needs to change after the release tag exists, coordinate with the release maintainer before changing the tag.
