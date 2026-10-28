# SDK release → private runner-agent → web-content API Reference

> Current design and implementation status: 2026-10-27.
>
> The implementation source of truth is:
>
> ```text
> .github/workflows/update-sdk-api-reference.yaml
> ```
>
> This document describes why that workflow is structured this way, how SDK repositories trigger it,
> which parts have already been deployed, and which operational gaps remain.

## Goal

When a supported Milvus SDK publishes a release tag, the SDK repository sends a minimal
`repository_dispatch` event to the private orchestrator repository:

```text
yhmo/runner-agent
```

The private repository starts a GitHub Actions runner, validates and pins the exact SDK release,
installs the Milvus skills, runs opencode against a shallow checkout of the current
`milvus-io/web-content` master, and republishes a restricted patch as a personal-account PR. The
same workflow creates and immediately rebase-merges that PR.

The receiver supports all current Milvus SDK families:

```text
pymilvus
java
go
cpp
rust
csharp
node
```

It has two trigger modes:

```text
repository_dispatch  exact release tag from an SDK repository
workflow_dispatch    operator selects SDK + release line; runner resolves latest stable tag
```

## Responsibility and trust boundaries

```text
public SDK repository
  ├─ validates that the tag push happened in the canonical repository
  ├─ resolves the exact tag commit
  ├─ stores only PRIVATE_DISPATCH_TOKEN
  └─ calls repository_dispatch on yhmo/runner-agent

yhmo/runner-agent
  ├─ stores OPENCODE_* secrets
  ├─ stores WEB_CONTENT_PERSONAL_PAT
  ├─ validates SDK, source repository, release line, tag, and commit
  ├─ installs skills and runs opencode
  ├─ exports a release-line-restricted patch
  └─ creates and merges the web-content PR as the maintainer's personal account

milvus-io/web-content
  └─ stores no model credentials and runs no opencode; it only receives API Reference PRs
```

This split keeps the LLM key out of both the public SDK repositories and web-content. The workflow
that can read the model credentials exists only on the default branch of the private repository.

The boundary is not absolute. A malicious workflow merged to `runner-agent/main` could read its
secrets, and SDK/web-content/skills content can still attempt prompt injection. The design therefore
combines secret isolation, exact source pinning, patch restrictions, short-lived credentials, model
quota controls, and auditability.

## Supported SDK mapping

The receiver normalizes both short SDK names and the legacy full names used by the first four
senders:

| Receiver SDK | Legacy alias | Canonical source repository | Skill name | Normal release tags |
|---|---|---|---|---|
| `pymilvus` | `milvus-sdk-python` | `milvus-io/pymilvus` | `pymilvus` | `vX.Y.Z` |
| `java` | `milvus-sdk-java` | `milvus-io/milvus-sdk-java` | `milvus-sdk-java` | `vX.Y.Z` |
| `go` | `milvus-sdk-go` | `milvus-io/milvus` | `milvus-sdk-go` | `client/vX.Y.Z` |
| `cpp` | `milvus-sdk-cpp` | `milvus-io/milvus-sdk-cpp` | `milvus-sdk-cpp` | `vX.Y.Z` |
| `rust` | `milvus-sdk-rust` | `milvus-io/milvus-sdk-rust` | `milvus-sdk-rust` | `vX.Y.Z` |
| `csharp` | `milvus-sdk-csharp` | `milvus-io/milvus-sdk-csharp` | `milvus-sdk-csharp` | `vX.Y.Z`, including preview suffixes |
| `node` | `milvus-sdk-node` | `milvus-io/milvus-sdk-node` | `milvus-sdk-node` | `vX.Y.Z` |

Go is special because the Go SDK is maintained under `client/` in the Milvus server repository. Its
Go SDK release tags use the `client/vX.Y.Z` prefix; ordinary Milvus server `vX.Y.Z` tags must not be
treated as Go SDK releases.

## Release-line selection

Manual runs expose two choices:

```text
SDK:          pymilvus / java / go / cpp / rust / csharp / node
release_line: 3.1 / 3.0 / 2.6
```

There is deliberately no manual tag input. A manual run always resolves the newest stable tag within
the selected release line.

GitHub Actions `workflow_dispatch` choice values are static and cannot be populated dynamically from
another repository when the Run workflow form is opened. The design therefore uses:

```text
static SDK/release-line choices
  -> receiver resolves the exact latest tag at run start
  -> GitHub API pins the exact commit
  -> skill resolver checks out that exact tag
```

The current release-line choices are:

```text
3.1
3.0
2.6
```

`3.1` is preconfigured for future use. Selecting a line that a particular SDK has not published
fails during validation with an error such as:

```text
no stable release tag found for 3.1
```

The run does not fall back to another release line and does not start opencode.

### Exact-tag mode

`repository_dispatch` events continue to carry the exact released tag:

```json
{
  "event_type": "sdk-release",
  "client_payload": {
    "sdk": "rust",
    "tag": "v3.0.3",
    "commit": "26631bf3a24b691df2207c76bc180d23fae91619",
    "source_repo": "milvus-io/milvus-sdk-rust"
  }
}
```

For an automatic dispatch, the receiver derives the release line from the tag's major and minor
version. It does not need a separate `release_line` payload field. After validating the exact tag
and commit, the receiver also resolves the latest stable tag in that line. An incoming automatic tag
older than that line latest is skipped successfully; it does not roll the shared documentation line
backward.

## Receiver workflow

The receiver lives at:

```text
.github/workflows/update-sdk-api-reference.yaml
```

### 1. Validate and pin the target

For every run, the receiver:

1. Normalizes the SDK name.
2. Maps it to exactly one canonical source repository.
3. Rejects malformed tag names.
4. Derives the release line from the tag.
5. For manual runs, resolves the latest stable tag in the selected line.
6. Queries GitHub for the tag's actual commit SHA.
7. Compares an automatic dispatch's supplied commit with that SHA.
8. For automatic dispatches, resolves the selected line's latest stable tag and rejects stale work
   as a successful skip.
9. Publishes the selected `SDK`, `TAG`, `COMMIT`, `SOURCE_REPO`, skill name, release line, and
   documentation directory to subsequent steps.

Manual latest-tag resolution uses `git ls-remote --tags`, not model judgment. It accepts only stable
release forms:

```text
Go:    client/vMAJOR.MINOR.PATCH
Other: vMAJOR.MINOR.PATCH
```

It excludes prerelease, beta, RC, and development suffixes and chooses the largest semantic version
within the selected major/minor line.

Automatic exact-tag events use the same line-latest calculation as a monotonic guard. For example,
if a repository's stable line latest is `v3.0.2`, a later dispatch for `v3.0.1` exits successfully
without running opencode:

```text
Skipped stale <sdk> v3.0.1; current 3.0 line latest is v3.0.2.
```

An equal or newer stable tag proceeds. A prerelease with the same numeric version as the stable
line latest is also considered stale, while a prerelease with a higher numeric patch than the known
stable latest may proceed.

### 2. Install language toolchain and skills

The runner installs the required language toolchain conditionally:

| SDK | Additional setup |
|---|---|
| Java | Temurin 8 |
| Rust | stable Rust |
| PyMilvus | Python 3.11 |
| Go | stable Go |
| C# | .NET 8.x |
| C++ / Node | use the preinstalled runner toolchain; Node is also installed for opencode |

It clones `yhmo/milvus-skills` over public HTTPS and executes:

```bash
./install.sh
```

This symlinks skills into opencode's skill directory. Because the repository is public, no SSH
deploy key or `MILVUS_SKILLS_DEPLOY_KEY` secret is required.

The current implementation still clones the mutable default branch. Pinning `milvus-skills` to a
reviewed commit or tag remains a production hardening task.

### 3. Resolve the exact SDK checkout

The workflow does not manually clone the SDK repository. It invokes the selected skill's resolver:

```bash
python3 \
  "$HOME/.opencode/skills/${SKILL_NAME}/scripts/resolve_repo.py" \
  --sdk "$SDK" \
  --branch "$TAG"
```

The helper automatically performs a depth-1 clone of the canonical repository when no valid local
checkout exists. The receiver requires all of the following:

```text
resolver.repository == selected canonical repository
resolver.commit     == pinned tag commit
```

The resolver's returned path is exported as `SDK_REPO_DIR`, and the prompt refers to that exact
checkout.

### 4. Clone web-content and run opencode

The first web-content clone is always the current tip of master:

```bash
git clone --depth 1 --branch master --single-branch \
  https://github.com/milvus-io/web-content.git
```

The workflow combines `ci/opencode.base.json` with:

```text
OPENCODE_BASE_URL
OPENCODE_API_KEY
OPENCODE_MODEL
```

The resulting model configuration is passed through a masked GitHub environment value. opencode runs
inside the web-content checkout:

```bash
opencode run \
  --dir "$GITHUB_WORKSPACE/web-content" \
  --auto \
  --model "$OPENCODE_FULL_MODEL" \
  "$PROMPT"
```

The prompt explicitly identifies:

```text
SDK label
selected release line
exact tag
exact SDK checkout path and commit
single allowed documentation-line directory
```

For example, a Java 3.0 run may modify only:

```text
API_Reference/milvus-sdk-java/v3.0.x/
```

A Rust 2.6 run may modify only:

```text
API_Reference/milvus-sdk-rust/v2.6.x/
```

### 5. Export a restricted patch

After opencode exits, the receiver runs:

```bash
git add --intent-to-add -- API_Reference
```

It then rejects every changed path outside:

```text
API_Reference/<sdk-label>/v<release-line>.x/
```

This is stricter than the original design, which allowed any path under `API_Reference/`. The current
guard prevents the model from changing another SDK, another release line, repository metadata, or
any non-documentation file.

Only a binary patch for the selected documentation line is written to:

```text
$RUNNER_TEMP/api-reference.patch
```

If the patch is empty, the workflow records "No API Reference changes" and exits successfully.

### 6. Republish from a clean clone

The opencode working tree and its `.git` directory are never reused for publication. The trusted
publication step performs a second shallow clone of current `web-content/master`, then:

```text
git apply --check
git apply
git checkout -b <unique-branch>
git add <selected-doc-line-directory>
git commit --no-verify -s
git push
gh pr create
gh pr merge --rebase --delete-branch
```

The second clone protects the publication token from Git configuration or hooks modified in the
opencode workspace. `git apply --check` also causes a safe failure if web-content master changed
incompatible files between the generation and publication clones.

## SDK sender workflows

### Common behavior

Implemented senders live in the SDK repositories as:

```text
.github/workflows/notify-runner-agent.yaml
```

They have two triggers:

```yaml
on:
  push:
    tags:
      - 'v*.*.*'

  workflow_dispatch:
```

The tag trigger sends:

```text
event_type   = sdk-release
sdk          = short receiver SDK name
tag          = github.ref_name
commit       = github.sha
source_repo  = canonical source repository
```

The manual trigger accepts an existing exact tag and full commit SHA. It is used to validate the
dispatch token and receiver integration without creating or moving a release tag.

The receiver accepts an automatic exact tag only when it is current for its release line. Re-pushing
an older patch tag after a newer stable release produces a successful skip rather than downgrading
the shared `vX.Y.x` documentation directory.

The sender performs these checks:

1. The canonical repository guard passes.
2. `PRIVATE_DISPATCH_TOKEN` is configured.
3. The tag matches the SDK's expected format.
4. The commit is a full SHA.
5. The dispatch API call succeeds.

A repository-dispatch PAT is required because events generated by a job's default `GITHUB_TOKEN`
do not trigger a workflow in another repository.

### Important tag/commit rule

For a pushed tag, GitHub runs the workflow file present at the commit pointed to by that tag. A
sender workflow merged after an existing tag was cut does not retroactively trigger that old tag.

Future release tags must point to commits that contain `notify-runner-agent.yaml`.

### Implemented repository status

| SDK repository | Implemented release branches | Sender status |
|---|---|---|
| `milvus-io/milvus-sdk-java` | `master`, `3.0`, `2.6` | deployed and token configured |
| `milvus-io/milvus-sdk-cpp` | `master`, `2.6` | deployed and token configured |
| `milvus-io/milvus-sdk-rust` | `main`, `2.6` | deployed and token configured |
| `milvus-io/milvus-sdk-csharp` | `main`, `2.6` | deployed and token configured |
| `milvus-io/pymilvus` | none yet | pending sender workflow/token rollout |
| `milvus-io/milvus` for Go | none yet | pending `client/v*` sender workflow/token rollout |
| `milvus-io/milvus-sdk-node` | none yet | pending sender workflow/token rollout |

Current deployed sender commits include:

```text
Java master: cbb6a010
Java 3.0:    dd533271
Java 2.6:    7ba23d38

C++ master: cd73007b
C++ 2.6:    7918bff9

Rust main: 6548bcb
Rust 2.6:  d6cd618

C# main: 9b1a350b
C# 2.6:  d92ceeee
```

For Java, the `3.0` branch is the current v3.0 stabilization line, while `master` is the future
development line. For C++, Rust, and C#, the current development branch represents the active 3.0
line. When a future 3.1 or 4.0 line forks, the sender workflow must be present on that new release
branch as well.

### Go sender requirement

The future Go sender in `milvus-io/milvus` must trigger only on:

```yaml
tags:
  - 'client/v*.*.*'
```

It must not trigger on ordinary Milvus server `vX.Y.Z` tags.

## Repository configuration

### `yhmo/runner-agent`

Required repository secrets:

```text
OPENCODE_BASE_URL
OPENCODE_API_KEY
OPENCODE_MODEL
WEB_CONTENT_PERSONAL_PAT
```

Required repository variables:

```text
GIT_AUTHOR_NAME
GIT_AUTHOR_EMAIL
```

Current Actions permissions:

```text
Actions enabled
Allowed actions: selected
GitHub-created actions: allowed
Specified action: dtolnay/rust-toolchain@stable
Verified creators: not allowed
Default workflow token: read-only
```

The selected-action pattern must include the action tag. Using only
`dtolnay/rust-toolchain` caused startup failures; the working value is:

```text
dtolnay/rust-toolchain@stable
```

### `WEB_CONTENT_PERSONAL_PAT`

The preferred credential is a fine-grained PAT limited to:

```text
milvus-io/web-content
```

Permissions:

```text
Metadata        Read-only
Contents        Read and write
Pull requests   Read and write
Checks          Read-only
```

A classic PAT with `repo` scope is currently working as a fallback, but it has a broader permission
surface and should be replaced with a fine-grained PAT when organizational policy allows it.

### SDK repository dispatch token

Every SDK sender repository stores:

```text
PRIVATE_DISPATCH_TOKEN
```

This token needs only enough permission on `yhmo/runner-agent` to call:

```text
POST /repos/yhmo/runner-agent/dispatches
```

The GitHub token's display name is irrelevant to Actions; the secret name read by the sender is
always `PRIVATE_DISPATCH_TOKEN`.

Do not reuse this token for model access or web-content publication.

## Concurrency and failure behavior

The receiver groups runs by the SDK identifier supplied by the trigger:

```yaml
concurrency:
  group: sdk-docs-${{ sdk }}
  cancel-in-progress: false
```

All deployed senders use the normalized short IDs (`java`, `cpp`, `rust`, and so on), so their
runs for the same SDK and all of that SDK's release lines are queued serially.

The receiver still accepts legacy full names such as `milvus-sdk-java` for compatibility. A raw
dispatch using a legacy name does not share the concurrency group with the normalized short name;
operators should use short names for manual/raw dispatches. Removing this compatibility alias or
normalizing the concurrency key is a possible future hardening change.

For normalized IDs:

```text
same SDK, including different release lines: queued serially
different SDKs: may run in parallel
```

Important edge cases:

1. **Unsupported release line**: manual validation fails before opencode starts.
2. **No tag in selected line**: manual validation fails; no fallback.
3. **No documentation changes**: empty patch exits successfully.
4. **Tag moved between sender and receiver**: supplied and GitHub API commits must match.
5. **Resolver returns another repository or commit**: run fails.
6. **Patch outside selected line**: run fails; no PR is created.
7. **web-content master changed incompatibly**: `git apply --check` fails safely.
8. **Old release tags**: do not retroactively trigger a newly added sender.
9. **Stale automatic tag**: if an exact tag is older than the latest stable tag in its release line,
   the run exits successfully with a summary and skips opencode/publication.
10. **New release line**: if a stable SDK tag exists, the current implementation may create the
   corresponding `vX.Y.x` documentation directory. If automatic creation is undesired, add an
   existence check for the selected documentation directory before opencode.
11. **Temporary SDK files**: only the selected documentation-line directory is exported into the
    publication patch.

## Validation history

The end-to-end receiver/publication path has been validated with real web-content PRs:

| SDK | Release | Result |
|---|---|---|
| Java | `v2.6.27` | real tag-push dispatch; PR [#1236](https://github.com/milvus-io/web-content/pull/1236), merged |
| Java | `v3.0.12` | real tag-push dispatch; PR [#1237](https://github.com/milvus-io/web-content/pull/1237), merged |
| Java | `v3.0.11` | PR [#1225](https://github.com/milvus-io/web-content/pull/1225), merged |
| Rust | `v3.0.3` | PR [#1233](https://github.com/milvus-io/web-content/pull/1233), merged |
| Rust | stale `v3.0.2` | successful skip; no opencode/publication |

Java and C# sender handshakes were also validated through their manual sender mode. Both dispatch
runs succeeded; the receiver runs were then cancelled before model use to avoid unnecessary cost.

The Java `v2.6.27` and `v3.0.12` runs are the first validation of the newly deployed sender's real
`push tag` path. Both sender runs completed in approximately seven seconds. Because both events used
the normalized `java` SDK ID, the receiver queued them serially under the Java concurrency group and
processed the 2.6 line before the 3.0 line.

The stale Rust `v3.0.2` dispatch was later sent while `v3.0.3` was the 3.0 line latest. The receiver
completed successfully, wrote the skip reason to the step summary, and skipped every model, patch,
and publication step.

## Operational notes

### Branch-push CI is not sender activity

Adding or merging a sender workflow can trigger each SDK repository's existing build workflows.
Those Actions may display the commit title:

```text
Notify runner-agent for SDK documentation updates
```

but their workflow name remains, for example, `Build and test`. They are not notifying
runner-agent. The actual sender workflow name is:

```text
Notify runner-agent SDK docs
```

and it runs only for matching tag pushes or manual validation dispatches.

### PR and merge assumptions

The workflow assumes current web-content policy:

```text
base branch: master
rebase merge allowed
no required reviews
no required checks
```

If web-content later adds required checks, insert:

```bash
gh pr checks "$PR_NUMBER" --watch --fail-fast
```

before `gh pr merge`. If required reviews are added, direct self-merge must be replaced by a review
or approval workflow.

## Remaining risks and hardening tasks

| Risk | Current mitigation | Remaining task |
|---|---|---|
| Malicious `runner-agent` workflow change | private repo and read-only workflow token | enable branch/rule protection for `main`; currently unavailable on this personal free-plan private repository |
| Mutable skills source | HTTPS clone of public `yhmo/milvus-skills` | pin to a reviewed commit/tag |
| Dispatch token abuse | minimum-purpose PAT and receiver validation | quota, RPM/TPM limits, expiry, rotation, and spend alerts |
| Prompt injection from SDK/web-content/skills | exact checkout, no publication token in model step, restricted patch, clean publication clone | retain model allowlist and audit generated changes |
| Automatic merge of generated content | one-commit personal PR and strict path guard | retain rollback procedure; move to PR-only mode if review becomes mandatory |
| Old action versions | working deployment | upgrade `actions/checkout`, `setup-node`, `setup-java`, and related actions in a controlled pass |
| Sender rollout incomplete | four SDK repositories deployed | deploy PyMilvus, Go, and Node senders |

## Rollout status and next steps

Already completed:

1. Private `yhmo/runner-agent` receiver deployed.
2. Model and web-content secrets configured.
3. Manual SDK/release-line trigger deployed.
4. Seven-SDK receiver support deployed.
5. Rust and Java end-to-end publication validated.
6. Rust, C++, Java, and C# senders deployed to all currently relevant 2.6/3.0 release branches.
7. Java sender tag-push behavior validated with both the `v2.6.27` and `v3.0.12` release tags.

Next:

1. Add the Python sender to `milvus-io/pymilvus`.
2. Add the Go sender to `milvus-io/milvus`, restricted to `client/v*.*.*` tags.
3. Add the Node sender to `milvus-io/milvus-sdk-node`.
4. Exercise future tag-push dispatches for Rust, C++, C#, and the remaining SDK senders after those
   repositories cut new release tags.
5. Pin `yhmo/milvus-skills`.
6. Move private-repo branch protection to a plan/organization that supports it.
7. Replace the classic web-content PAT with a fine-grained PAT when available.
