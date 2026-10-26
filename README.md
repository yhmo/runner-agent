# runner-agent

Private GitHub Actions orchestrator that updates Milvus SDK API Reference documentation after SDK releases.

See [DESIGN.md](DESIGN.md) for the full design, trust boundaries, workflow specifications, and rollout plan.

## Flow

1. A Milvus SDK repository sends a `repository_dispatch` event named `sdk-release`.
2. This repository validates the SDK, tag, source repository, and commit.
3. The runner installs skills from `yhmo/milvus-skills`.
4. The SDK skill resolver automatically clones the tagged SDK repository with depth 1.
5. The runner clones `milvus-io/web-content` with depth 1.
6. opencode updates only `API_Reference/` inside the web-content checkout.
7. The workflow exports a restricted patch and republishes it from a clean web-content clone.
8. The workflow creates and immediately rebase-merges a PR as the personal account that owns `WEB_CONTENT_PERSONAL_PAT`.

## Required repository configuration

Configure these Actions secrets:

- `OPENCODE_BASE_URL`
- `OPENCODE_API_KEY`
- `OPENCODE_MODEL`
- `WEB_CONTENT_PERSONAL_PAT`

Configure these Actions variables:

- `GIT_AUTHOR_NAME`
- `GIT_AUTHOR_EMAIL`

Configure Actions permissions under **Settings → Actions → General**:

- Allow GitHub-created actions
- Allow the specified action `dtolnay/rust-toolchain@stable`
- Keep the default workflow token permission read-only

`WEB_CONTENT_PERSONAL_PAT` must belong to the personal GitHub account and have access to `milvus-io/web-content`. A fine-grained PAT should grant:

- Contents: read and write
- Pull requests: read and write
- Checks: read-only
- Metadata: read-only

SDK repositories need a separate `PRIVATE_DISPATCH_TOKEN` capable of calling `repository_dispatch` on this private repository. Do not reuse that token for web-content writes or model access.

## Manual test

```bash
SDK=milvus-sdk-java
TAG=v3.0.11
COMMIT=$(gh api "repos/milvus-io/${SDK}/commits/${TAG}" --jq .sha)

gh api repos/yhmo/runner-agent/dispatches \
  --method POST \
  --input - <<EOF
{
  "event_type": "sdk-release",
  "client_payload": {
    "sdk": "${SDK}",
    "tag": "${TAG}",
    "commit": "${COMMIT}",
    "source_repo": "milvus-io/${SDK}"
  }
}
EOF
```

## Security notes

- `OPENCODE_*` secrets are exposed only to opencode configuration and generation steps.
- `WEB_CONTENT_PERSONAL_PAT` is exposed only to the trusted publication step, not to opencode.
- The opencode working tree is never reused for Git push. Changes are exported as an `API_Reference/`-only binary patch and applied to a fresh shallow clone.
- Git hooks are disabled during publication.
- Direct merge is used because `web-content/master` currently has no required reviews or checks.
