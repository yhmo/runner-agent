# SDK 发版 → 私有仓库编排更新 web-content API Reference 方案

> 目标：`milvus-sdk-{cpp,java,rust,csharp}` 发布 `vX.Y.Z` tag 后，通过 `repository_dispatch`
> 通知维护者自己的 private orchestrator repo；private repo 启动 GitHub Actions runner，
> runner 以 shallow clone 方式获取 `milvus-io/web-content`，运行 opencode 更新对应 SDK 的
> API Reference，最后用维护者个人账号提交 PR，并在同一 workflow 中直接 rebase merge。

---

## 职责与信任边界

方案将自动化职责划分如下：

```text
public SDK repo
  └─ 只保存 private repo dispatch token

private orchestrator repo
  ├─ 保存 OPENCODE_* secrets
  ├─ 保存个人 web-content PAT
  ├─ 运行 opencode
  └─ 用个人账号创建并合并 PR

milvus-io/web-content
  └─ 不保存任何 OPENCODE_* / bot token，只接收 API Reference PR
```

这样对 LLM key 更有利：

- LLM key 不进入公共 web-content repo；
- 公共 web-content 不运行 opencode，其 Actions 日志不接触 LLM 配置；
- 4 个 SDK repo 也拿不到 LLM key，只保存一个可触发 private repo 的最小权限 token；
- 能引用 LLM key 的 workflow 缩小为 private repo 默认分支上的可信 workflow。

但这不是绝对安全。private repo 默认分支上被合入的恶意 workflow 仍可读取并外传
`OPENCODE_*` secrets；SDK tag 中的文档、示例和 web-content skill 内容也可能对 opencode
构成 prompt injection。因此仍需最小权限 token、限额、审计和发布步骤隔离。

---

## opencode 运行方式与已验证实验

private repo 的 runner 在启动 opencode 前准备两个工作目录：

```text
$GITHUB_WORKSPACE/milvus-skills     # skills 仓库，运行 install.sh
$GITHUB_WORKSPACE/web-content       # web-content master 的 depth=1 shallow checkout
```

SDK checkout 不由 workflow 手动 `git clone`。安装 skills 后，workflow 调用对应 SDK skill
捆绑的 `scripts/resolve_repo.py`，并传入 dispatch tag；该 helper 会在没有本地 checkout 时自动
执行 depth=1 clone，并把仓库放在临时目录：

```text
${TMPDIR:-/tmp}/milvus-skills/<sdk-repo>
```

例如 Java tag 触发时：

```bash
python3 "$HOME/.opencode/skills/milvus-sdk-java/scripts/resolve_repo.py" \
  --sdk java \
  --branch "$TAG"
```

resolver 返回 JSON 中的 `path` 和 `commit`。workflow 校验 `commit == dispatch payload COMMIT`
后，把 `path` 写入 `SDK_REPO_DIR`，供 opencode prompt 显式引用。这样既使用 skill 的自动 clone
能力，又保持 tag/commit 的确定性。

opencode 必须在 `web-content` 目录内启动：

```bash
opencode run \
  --dir "$GITHUB_WORKSPACE/web-content" \
  --auto \
  --model "$OPENCODE_FULL_MODEL" \
  "更新 ${SDK} SDK 的 API Reference 到最新版本。当前最新版本是 ${TAG}。"
```

这里“最新版本”不是让模型自行猜测或查询任意 latest tag，而是明确绑定 dispatch payload 中的
`${TAG}` 以及 skill resolver 生成的 exact tag checkout。这样 tag 触发、源码 checkout、
文档更新目标三者始终一致。

opencode 只负责修改 `web-content/API_Reference/`。它退出后，workflow 用可信发布步骤创建
web-content PR，并立即 rebase merge；不让 opencode 直接持有 web-content PAT。

此前已在 `milvus-io/web-content` PR
[#1192](https://github.com/milvus-io/web-content/pull/1192) 中验证过：在 GitHub Actions
runner 内用预先配置的 secret 提供 baseURL/model/key，并运行 opencode 更新 API Reference 是
可行的。本方案保留这条已验证的 opencode 调用链，把 secret 与运行日志迁移到 private repo，
并将 PR 发布与合并改为个人账号 PAT。

---

## 总体架构

```text
SDK repo (milvus-sdk-{cpp,java,rust,csharp})
  push tag vX.Y.Z
    └─ .github/workflows/notify-private-doc-orchestrator.yaml
         gh api repos/<OWNER>/<PRIVATE_REPO>/dispatches
         event_type = sdk-release
         payload = { sdk, tag, commit, source_repo }
                        │
                        ▼
<OWNER>/<PRIVATE_REPO>
  └─ .github/workflows/update-sdk-api-reference.yaml
         repository_dispatch: sdk-release
         ├─ 校验 payload、canonical source repo、tag commit
         ├─ checkout private repo 中的 opencode 配置模板
         ├─ git clone --depth 1 git@github.com:yhmo/milvus-skills.git
         ├─ 运行 milvus-skills/install.sh 安装 opencode skills
         ├─ 通过 SDK skill 的 resolve_repo.py 自动 depth=1 clone canonical SDK repo
         ├─ git clone --depth 1 milvus-io/web-content
         ├─ 合成 OPENCODE_CONFIG_CONTENT 并运行 opencode
         │    ├─ 只修改 API_Reference/
         │    ├─ 不拿 web-content PAT
         │    └─ 不 push、不创建 PR
         ├─ 从干净 shallow clone 重新 checkout web-content
         ├─ 校验并应用仅限 API_Reference/ 的 patch
         ├─ 用个人 PAT commit + push 分支
         ├─ 用个人 PAT gh pr create
         └─ 用个人 PAT gh pr merge --rebase --delete-branch
```

### 关键身份

| 操作 | 身份 | 凭证 |
|---|---|---|
| SDK repo 触发 private repo | dispatch PAT 所属账号 | `PRIVATE_DISPATCH_TOKEN`，存在 4 个 SDK repo |
| 运行 opencode | private repo 的 GitHub Actions runner | `OPENCODE_*` secrets，仅生成步骤可见 |
| push PR 分支 | 维护者个人账号 | `WEB_CONTENT_PERSONAL_PAT`，仅发布步骤可见 |
| 创建/合并 PR | 维护者个人账号 | 同上 |
| commit author | 维护者个人配置 | `GIT_AUTHOR_NAME` / `GIT_AUTHOR_EMAIL` |

`WEB_CONTENT_PERSONAL_PAT` 使用维护者个人账号的 PAT。因此 GitHub PR 页面显示的 PR 创建者
和 merge 操作者都是该个人账号。commit author 由 `git config user.name/user.email` 控制，
应使用个人 GitHub 账号绑定的邮箱，或账号的 noreply 邮箱。

---

## 仓库与 secrets 配置

### 1. private orchestrator repo

创建在维护者个人账号下：

```text
<OWNER>/<PRIVATE_REPO>
```

建议只有维护者本人拥有 admin/write 权限，禁止无关协作者修改默认分支和 workflow。

需要保存：

| Secret / Variable | 类型 | 用途 |
|---|---|---|
| `OPENCODE_BASE_URL` | Actions secret | 自定义模型网关 baseURL |
| `OPENCODE_API_KEY` | Actions secret | LLM key |
| `OPENCODE_MODEL` | Actions secret | 模型名 |
| `WEB_CONTENT_PERSONAL_PAT` | Actions secret | 个人账号操作 web-content 的 PAT |
| `MILVUS_SKILLS_DEPLOY_KEY` | Actions secret | 只读拉取 `yhmo/milvus-skills` 的 SSH deploy key |
| `GIT_AUTHOR_NAME` | Actions variable | commit author name |
| `GIT_AUTHOR_EMAIL` | Actions variable | commit author email |

private repo 中提交：

```text
+.github/workflows/update-sdk-api-reference.yaml
+ci/opencode.base.json
```

`ci/opencode.base.json` 仍使用占位符模板，不在 Git 内容中保存真实 key。

### milvus-skills 的获取与安装

runner 需要在 opencode 启动前安装 SDK skills：

```bash
git clone --depth 1 \
  git@github.com:yhmo/milvus-skills.git \
  "$GITHUB_WORKSPACE/milvus-skills"

cd "$GITHUB_WORKSPACE/milvus-skills"
./install.sh
```

`install.sh` 会把仓库中的各个 skill 目录软链接到：

```text
~/.opencode/skills/
~/.claude/skills/
~/.codex/skills/
```

因此 opencode 启动后可以加载 `milvus-sdk-cpp`、`milvus-sdk-java`、
`milvus-sdk-rust`、`milvus-sdk-csharp` 等 skills。

由于 GitHub-hosted runner 默认没有你的 SSH 私钥，使用 `git@github.com:` 需要给
`yhmo/milvus-skills` 配一个 read-only deploy key，并把对应私钥保存为 private repo 的
`MILVUS_SKILLS_DEPLOY_KEY` secret。该私钥只在 clone/install step 中写入 `~/.ssh`；
clone 和 install 完成后立即删除，避免后续 opencode 进程读取或外传。

### SDK repo 的自动 clone 与固定版本

最新版 `milvus-sdk-java` 这类 SDK skill 内置 `scripts/resolve_repo.py`，会先解析本地 checkout；没有有效 checkout 时自动 clone canonical repo：

```bash
python3 "$SKILL_DIR/scripts/resolve_repo.py" --sdk java
```

resolver 支持 `--branch <branch-or-tag>`，默认自动 clone 使用 depth=1。private runner 传入
dispatch payload 的 `${TAG}`，因此 Java 触发时会自动得到对应 release tag 的 shallow checkout，
而不是 `master` 或错误维护分支。

因此 receiver 不再手写 SDK `git clone`，而是在 opencode 启动前调用对应 skill 的 resolver，并传入：

```text
+$GITHUB_WORKSPACE/<SDK>
```

resolver 返回的 JSON 包含：

```text
+$GITHUB_WORKSPACE/web-content
```

workflow 校验 `repository` 等于 canonical SDK repo、`commit` 等于 dispatch payload 的
`COMMIT`，并把 `path` 写入 `SDK_REPO_DIR`。这可以防止 tag 在 GitHub API 校验和自动 clone
之间被移动，也让后续 opencode prompt 显式引用确定路径。

由于 resolver 行为属于会指导 LLM 的 skill 代码，生产化后建议将 `yhmo/milvus-skills` 固定到已
review 的 commit 或 tag，而不是长期跟随 mutable `main`。

`yhmo/milvus-skills` 当前是 public repo。如果不必坚持 SSH URL，workflow 可以改用：

```bash
https://github.com/yhmo/milvus-skills.git
```

这样不需要 deploy key，少一个 secret，也更简单。下文按用户指定的 SSH URL 保留 deploy key
方案。


```json
{
  "provider": {
    "aistation": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "AI Station",
      "options": {
        "baseURL": "__BASE_URL__",
        "apiKey": "__API_KEY__"
      },
      "models": {
        "__MODEL__": { "name": "__MODEL__" }
      }
    }
  },
  "model": "aistation/__MODEL__"
}
```

### 2. `WEB_CONTENT_PERSONAL_PAT` 权限

优先使用 fine-grained PAT，只授权：

```text
+milvus-io/web-content
```

至少需要：

| Permission | 权限 | 用途 |
|---|---|---|
| Metadata | Read-only | 基础 API 访问 |
| Contents | Read and write | push PR 分支、删除分支 |
| Pull requests | Read and write | 创建 PR、合并 PR |
| Checks | Read-only | 如后续需要等待 checks |

个人账号必须对 `milvus-io/web-content` 有 merge 权限。当前维护者账号具备 admin 权限，
且 `master` 无 required review / required checks，因此可以直接 merge。

### 3. 4 个 SDK repo

每个 SDK repo 保存：

```text
PRIVATE_DISPATCH_TOKEN
```

该 token 只需要能调用 private repo 的 `repository_dispatch`。如果使用 fine-grained PAT，
应只选择该 private repo，并给 Contents write 权限；如果使用 classic PAT，则需要 `repo`
scope，但权限明显更大，不推荐。

不要把以下 token 混用：

```text
PRIVATE_DISPATCH_TOKEN       # SDK repo → private repo
WEB_CONTENT_PERSONAL_PAT     # private repo → web-content
OPENCODE_API_KEY             # private repo → 模型网关
```

---

## 文件 1：SDK repo sender workflow

在 4 个 SDK repo 各新增：

```text
+.github/workflows/notify-private-doc-orchestrator.yaml
```

除 `<OWNER>`、`<PRIVATE_REPO>`、`<REPO>`、`<SDK_NAME>` 外保持一致。

```yaml
name: Notify private SDK doc orchestrator

run-name: notify doc orchestrator for ${{ github.ref_name }}

on:
  push:
    tags:
      - 'v*.*.*'

permissions:
  contents: read

jobs:
  dispatch:
    name: send SDK release dispatch
    if: github.repository == 'milvus-io/<REPO>'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Send repository_dispatch to private orchestrator
        env:
          GH_TOKEN: ${{ secrets.PRIVATE_DISPATCH_TOKEN }}
          TAG: ${{ github.ref_name }}
          SOURCE_REPO: milvus-io/<REPO>
        run: |
          set -euo pipefail

          if [ -z "$GH_TOKEN" ]; then
            echo "PRIVATE_DISPATCH_TOKEN is not configured" >&2
            exit 1
          fi

          if [[ ! "$TAG" =~ ^v[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z][0-9A-Za-z.-]*)?$ ]]; then
            echo "Unsupported tag format: $TAG" >&2
            exit 1
          fi

          COMMIT=$(git rev-list -n 1 "$TAG")
          if [ -z "$COMMIT" ]; then
            echo "Unable to resolve commit for tag: $TAG" >&2
            exit 1
          fi

          gh api repos/<OWNER>/<PRIVATE_REPO>/dispatches \
            --method POST \
            --field event_type=sdk-release \
            --field "client_payload[sdk]=<SDK_NAME>" \
            --field "client_payload[tag]=${TAG}" \
            --field "client_payload[commit]=${COMMIT}" \
            --field "client_payload[source_repo]=${SOURCE_REPO}"
```

各 repo 填参：

| repo | `<REPO>` | `<SDK_NAME>` |
|---|---|---|
| milvus-sdk-cpp | milvus-sdk-cpp | milvus-sdk-cpp |
| milvus-sdk-java | milvus-sdk-java | milvus-sdk-java |
| milvus-sdk-rust | milvus-sdk-rust | milvus-sdk-rust |
| milvus-sdk-csharp | milvus-sdk-csharp | milvus-sdk-csharp |

csharp 的预发布 tag（如 `v2.2.2-preview.5`）已被正则兼容。

---

## 文件 2：private repo receiver workflow

新增：

```text
+.github/workflows/update-sdk-api-reference.yaml
```

这份 workflow 的重点是职责分离：

1. opencode 只负责改文件，不接触 `WEB_CONTENT_PERSONAL_PAT`；
2. 发布步骤从全新 shallow clone 开始，避免 opencode 修改过的 `.git/config`、hooks 或本地
   Git 状态影响后续 `git commit/push`；
3. patch 只允许修改 `API_Reference/`；
4. PR 使用个人 PAT 创建并立即 rebase merge。

```yaml
name: Update SDK API Reference

run-name: update ${{ github.event.client_payload.sdk }} docs to ${{ github.event.client_payload.tag }}

on:
  repository_dispatch:
    types:
      - sdk-release

concurrency:
  group: sdk-docs-${{ github.event.client_payload.sdk }}
  cancel-in-progress: false

permissions:
  contents: read

env:
  SDK: ${{ github.event.client_payload.sdk }}
  TAG: ${{ github.event.client_payload.tag }}
  COMMIT: ${{ github.event.client_payload.commit }}
  SOURCE_REPO: ${{ github.event.client_payload.source_repo }}

jobs:
  update:
    runs-on: ubuntu-latest
    timeout-minutes: 180

    steps:
      - name: Validate dispatch payload
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          set -euo pipefail

          test -n "$SDK" -a -n "$TAG" -a -n "$COMMIT" -a -n "$SOURCE_REPO" \
            || { echo "missing sdk/tag/commit/source_repo payload" >&2; exit 1; }

          case "$SDK" in
            milvus-sdk-cpp|milvus-sdk-java|milvus-sdk-rust|milvus-sdk-csharp) ;;
            *) echo "unsupported SDK: $SDK" >&2; exit 1 ;;
          esac

          [[ "$TAG" =~ ^v[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z][0-9A-Za-z.-]*)?$ ]] \
            || { echo "bad tag: $TAG" >&2; exit 1; }

          EXPECTED_SOURCE_REPO="milvus-io/${SDK}"
          [[ "$SOURCE_REPO" == "$EXPECTED_SOURCE_REPO" ]] \
            || { echo "unexpected source repo: $SOURCE_REPO" >&2; exit 1; }

          ACTUAL_COMMIT=$(gh api "repos/${SOURCE_REPO}/commits/${TAG}" --jq .sha)
          [[ "$COMMIT" == "$ACTUAL_COMMIT" ]] \
            || { echo "commit mismatch for ${SOURCE_REPO}@${TAG}" >&2; exit 1; }

      # checkout private repo 只为读取 ci/opencode.base.json。
      # 不保留 runner token，避免 opencode 读到 private repo credential。
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install opencode
        run: |
          npm install -g opencode-ai
          opencode --version

      - name: Clone and install Milvus skills
        env:
          MILVUS_SKILLS_DEPLOY_KEY: ${{ secrets.MILVUS_SKILLS_DEPLOY_KEY }}
        run: |
          set -euo pipefail

          install -d -m 700 "$HOME/.ssh"
          printf '%s\n' "$MILVUS_SKILLS_DEPLOY_KEY" > "$HOME/.ssh/id_ed25519"
          chmod 600 "$HOME/.ssh/id_ed25519"

          # Pin GitHub's ED25519 host key instead of blindly ssh-keyscan-ing
          # a hostname from network output.
          printf '%s\n' \
            'github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl' \
            > "$HOME/.ssh/known_hosts"

          git clone \
            --depth 1 \
            git@github.com:yhmo/milvus-skills.git \
            "$GITHUB_WORKSPACE/milvus-skills"

          cd "$GITHUB_WORKSPACE/milvus-skills"
          ./install.sh

          # Skills are now symlinked into ~/.opencode/skills. Remove the SSH
          # private key before opencode starts so the model cannot read it.
          rm -f "$HOME/.ssh/id_ed25519"

      - uses: dtolnay/rust-toolchain@stable
        if: env.SDK == 'milvus-sdk-rust'

      - uses: actions/setup-java@v4
        if: env.SDK == 'milvus-sdk-java'
        with:
          distribution: temurin
          java-version: '8'

      - name: Resolve SDK checkout through the skill helper
        run: |
          set -euo pipefail

          case "$SDK" in
            milvus-sdk-cpp) SDK_KEY=cpp ;;
            milvus-sdk-java) SDK_KEY=java ;;
            milvus-sdk-rust) SDK_KEY=rust ;;
            milvus-sdk-csharp) SDK_KEY=csharp ;;
            *) echo "unsupported SDK: $SDK" >&2; exit 1 ;;
          esac

          RESOLVED=$(python3 \
            "$HOME/.opencode/skills/${SDK}/scripts/resolve_repo.py" \
            --sdk "$SDK_KEY" \
            --branch "$TAG")

          REPOSITORY=$(printf '%s\n' "$RESOLVED" | python3 -c 'import json,sys; print(json.load(sys.stdin)["repository"])')
          RESOLVED_COMMIT=$(printf '%s\n' "$RESOLVED" | python3 -c 'import json,sys; print(json.load(sys.stdin)["commit"])')
          RESOLVED_PATH=$(printf '%s\n' "$RESOLVED" | python3 -c 'import json,sys; print(json.load(sys.stdin)["path"])')

          EXPECTED_REPOSITORY="milvus-io/${SDK}"
          if [ "$REPOSITORY" != "$EXPECTED_REPOSITORY" ]; then
            echo "resolver returned unexpected repository: $REPOSITORY" >&2
            exit 1
          fi
          if [ "$RESOLVED_COMMIT" != "$COMMIT" ]; then
            echo "SDK tag moved during dispatch: ${EXPECTED_REPOSITORY}@${TAG}" >&2
            echo "expected ${COMMIT}, resolver returned ${RESOLVED_COMMIT}" >&2
            exit 1
          fi

          echo "SDK_REPO_DIR=$RESOLVED_PATH" >> "$GITHUB_ENV"

      - name: Clone web-content shallowly
        run: |
          set -euo pipefail
          git clone \
            --depth 1 \
            --branch master \
            --single-branch \
            https://github.com/milvus-io/web-content.git \
            "$GITHUB_WORKSPACE/web-content"

      - name: Inject opencode config
        env:
          OPENCODE_BASE_URL: ${{ secrets.OPENCODE_BASE_URL }}
          OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }}
          OPENCODE_MODEL: ${{ secrets.OPENCODE_MODEL }}
        run: |
          set -euo pipefail

          echo "::add-mask::$OPENCODE_BASE_URL"
          echo "::add-mask::$OPENCODE_API_KEY"
          echo "::add-mask::$OPENCODE_MODEL"

          CFG=$(python3 - <<'EOF'
          import json, os
          with open('ci/opencode.base.json', encoding='utf-8') as f:
              cfg = json.load(f)

          provider = cfg['provider']['aistation']
          provider['options']['baseURL'] = os.environ['OPENCODE_BASE_URL']
          provider['options']['apiKey'] = os.environ['OPENCODE_API_KEY']

          model = os.environ['OPENCODE_MODEL']
          provider['models'] = {model: {'name': model}}
          cfg['model'] = f'aistation/{model}'
          print(json.dumps(cfg, ensure_ascii=False, separators=(',', ':')))
          EOF
          )

          DELIMITER="__OPENCODE_CONFIG_$(python3 -c 'import secrets; print(secrets.token_hex(12))')__"
          {
            echo "OPENCODE_CONFIG_CONTENT<<${DELIMITER}"
            printf '%s\n' "$CFG"
            echo "${DELIMITER}"
          } >> "$GITHUB_ENV"

          echo "OPENCODE_FULL_MODEL=aistation/${OPENCODE_MODEL}" >> "$GITHUB_ENV"

      - name: Run opencode inside web-content
        env:
          OPENCODE: "1"
        run: |
          set -euo pipefail

          PROMPT="更新 ${SDK} SDK 的 API Reference 到最新版本。"
          PROMPT+=" 当前最新版本是 ${TAG}。"
          PROMPT+=" The exact tagged SDK checkout is available at ${SDK_REPO_DIR} and its commit is ${COMMIT}."
          PROMPT+=" Use and prefer that checkout."
          PROMPT+=" If an API has no SDK example, synthesize a minimal example from the tagged public API signatures and documented usage logic."
          PROMPT+=" Do not execute examples, do not connect to Milvus, and do not perform live API calls."
          PROMPT+=" Static or compile-level snippet verification is allowed if the repository skill requires it."
          PROMPT+=" Do not push, do not create a PR, and do not read or print environment variables."
          PROMPT+=" Stop after the working tree contains the documentation changes."

          opencode run \
            --dir "$GITHUB_WORKSPACE/web-content" \
            --auto \
            --model "$OPENCODE_FULL_MODEL" \
            "$PROMPT"

      - name: Create restricted patch
        run: |
          set -euo pipefail

          # Include new files under API_Reference/ in git diff.
          git -C "$GITHUB_WORKSPACE/web-content" add --intent-to-add -- API_Reference

          OUTSIDE=$(git -C "$GITHUB_WORKSPACE/web-content" diff --name-only \
            | grep -Ev '^API_Reference/' || true)
          if [ -n "$OUTSIDE" ]; then
            echo "opencode attempted to modify paths outside API_Reference/:" >&2
            printf '%s\n' "$OUTSIDE" >&2
            exit 1
          fi

          git -C "$GITHUB_WORKSPACE/web-content" diff --binary -- API_Reference \
            > "$RUNNER_TEMP/api-reference.patch"

          if [ ! -s "$RUNNER_TEMP/api-reference.patch" ]; then
            echo "No API Reference changes for ${SDK} ${TAG}." | tee -a "$GITHUB_STEP_SUMMARY"
            echo "HAS_PATCH=false" >> "$GITHUB_ENV"
            exit 0
          fi

          echo "HAS_PATCH=true" >> "$GITHUB_ENV"

      - name: Publish and merge PR as personal account
        if: env.HAS_PATCH == 'true'
        env:
          GH_TOKEN: ${{ secrets.WEB_CONTENT_PERSONAL_PAT }}
          GIT_AUTHOR_NAME: ${{ vars.GIT_AUTHOR_NAME }}
          GIT_AUTHOR_EMAIL: ${{ vars.GIT_AUTHOR_EMAIL }}
        run: |
          set -euo pipefail

          VERSION="${TAG#v}"
          BRANCH="sdk/${SDK}-${VERSION}-${GITHUB_RUN_ID}-doc"
          PUBLISH_DIR="$RUNNER_TEMP/publish/web-content"

          # Use a fresh clone for publishing. Do not reuse the .git directory
          # or hooks from the opencode working tree.
          mkdir -p "$(dirname "$PUBLISH_DIR")"
          git clone \
            --depth 1 \
            --branch master \
            --single-branch \
            https://github.com/milvus-io/web-content.git \
            "$PUBLISH_DIR"

          git -C "$PUBLISH_DIR" apply --check "$RUNNER_TEMP/api-reference.patch"
          git -C "$PUBLISH_DIR" apply "$RUNNER_TEMP/api-reference.patch"

          git -C "$PUBLISH_DIR" config user.name "$GIT_AUTHOR_NAME"
          git -C "$PUBLISH_DIR" config user.email "$GIT_AUTHOR_EMAIL"
          git -C "$PUBLISH_DIR" checkout -b "$BRANCH" origin/master
          git -C "$PUBLISH_DIR" add -- API_Reference

          # --no-verify and an empty hooks path prevent repository hooks from
          # running in the publish step.
          git -C "$PUBLISH_DIR" \
            -c core.hooksPath=/dev/null \
            commit --no-verify -s \
            -m "Update ${SDK} API Reference to ${TAG}"

          gh auth setup-git --hostname github.com
          git -C "$PUBLISH_DIR" push -u origin "HEAD:refs/heads/${BRANCH}"

          PR_URL=$(gh pr create \
            --repo milvus-io/web-content \
            --base master \
            --head "$BRANCH" \
            --title "Update ${SDK} API Reference to ${TAG}" \
            --body "Automated API Reference update for ${SDK} ${TAG}.")

          PR_NUMBER="${PR_URL##*/}"
          echo "Created and merging PR #$PR_NUMBER as personal account."

          # web-content master currently has no required reviews or required
          # checks, so direct rebase merge is sufficient. Auto-merge is not needed.
          gh pr merge "$PR_NUMBER" \
            --repo milvus-io/web-content \
            --rebase \
            --delete-branch

          echo "Merged PR #$PR_NUMBER for ${SDK} ${TAG}." | tee -a "$GITHUB_STEP_SUMMARY"
```

---

## 为什么两次 shallow clone web-content

第一次 clone 用于 opencode 修改文档：

```text
+$GITHUB_WORKSPACE/web-content
```

第二次 clone 用于可信发布：

```text
+$RUNNER_TEMP/publish/web-content
```

原因是 opencode 处理的是公开 SDK repo 和 web-content 中的内容，理论上可能受 prompt
injection 影响。如果发布步骤直接复用 opencode 用过的 `.git` 目录，opencode 理论上可以：

- 修改 `.git/config`；
- 修改 `.git/hooks`；
- 留下恶意 Git 配置或过滤器；
- 借后续 `git commit/push` 进程读取发布 step 中的个人 PAT。

因此发布步骤改为：

1. 只从 opencode 工作区导出 `API_Reference/` 的 binary patch；
2. 校验 patch 路径；
3. 从 GitHub 重新 shallow clone 干净 web-content；
4. 在干净 clone 中 `git apply`、commit、push、create PR、merge；
5. commit 使用 `--no-verify` 并临时禁用 hooks。

这比在同一工作区中让 opencode 直接执行 `git push / gh pr create` 安全得多。

---

## shallow clone 约束

web-content 历史较大，两次 clone 都使用：

```bash
git clone --depth 1 --branch master --single-branch \
  https://github.com/milvus-io/web-content.git
```

约束与注意事项：

- 不需要读取 web-content 历史；当前文档内容和当前文件树即可；
- PR 分支基于当前 `origin/master` tip，shallow commit 可以正常 push；
- 不运行依赖 `git log web-content` 的逻辑；
- 如需比较文档版本，应从文档文件或 SDK tag 中解析，而不是依赖 web-content 历史；
- SDK repo 由 skill resolver 按 tag 自动 shallow clone；
- 如果现有 `.skills/update-milvus-sdk-docs` 强依赖 web-content 历史，需要调整 skill 或额外
  只取必要元数据，而不是取消 depth 1。

---

## PR 与 merge 策略

当前 `milvus-io/web-content` 配置：

```text
allow_merge_commit = true
allow_squash_merge = true
allow_rebase_merge = true
delete_branch_on_merge = true
allow_auto_merge = false
```

同时 `master` 分支保护中没有 required reviews，也没有 required checks。因此 PR 创建后立即可
合并，不需要 GitHub auto-merge。

发布命令直接使用：

```bash
gh pr merge "$PR_NUMBER" \
  --repo milvus-io/web-content \
  --rebase \
  --delete-branch
```

这会在 workflow 内自动完成 merge。由于使用个人 PAT，PR 的创建者和 merge 操作者都显示为
维护者个人账号。

如果未来 web-content 增加 required checks，应改为：

```bash
gh pr checks "$PR_NUMBER" \
  --repo milvus-io/web-content \
  --watch \
  --fail-fast

gh pr merge "$PR_NUMBER" \
  --repo milvus-io/web-content \
  --rebase \
  --delete-branch
```

如果未来增加 required review，则维护者不能 approve 自己创建的 PR，必须由其他人或 review bot
通过后才能合并。

---

## 边界情况

1. **无文档变化**：opencode 没有产生 patch 时，workflow 写入 step summary 并正常退出 0，
   不影响其他 SDK dispatch。
2. **tag 早于当前文档线**：skill 拒绝降级时不会产生 patch，同上按无变化处理。
3. **重复触发**：`concurrency` 按 SDK 分组串行，避免同一 SDK 的分支互相覆盖；不同 SDK 可并行。
4. **payload 伪造**：receiver 校验 SDK、tag 格式、canonical source repo，并通过 GitHub API
   核对 tag 实际 commit。
5. **csharp 文档线**：当前只有 `v2.2.x` / `v2.6.x` 两条线。若收到无对应文档线的 tag，可在
   csharp sender 上预过滤，或接受无 PR 结果。
6. **分支冲突**：PR 分支带 `${GITHUB_RUN_ID}`，重试不会因旧分支存在而失败；merge 后自动删除。
7. **patch 超出范围**：只要出现 `API_Reference/` 之外的 tracked 修改，workflow 直接失败。
8. **SDK checkout**：receiver 调用 SDK skill 自带的 `resolve_repo.py` 自动按 exact tag shallow
   clone canonical SDK repo，并核对返回 commit；返回路径写入 `SDK_REPO_DIR` 供 prompt 引用。
9. **SDK 临时目录**：如果现有 skill 在 web-content 下创建 `sdk-tmp`，发布前只导出
   `API_Reference/` patch，临时目录不会进入 PR。
---

## 安全边界与防护

### 私有 repo 带来的改进

| 风险面 | 方案结果 |
|---|---|
| 公共 web-content Actions 日志 | 不再包含 opencode 运行日志，LLM key 泄漏面显著缩小 |
| web-content 中被合入的恶意 workflow | 无法读取 `OPENCODE_*`，因为这些 secret 不在 web-content |
| SDK repo | 只持有 private repo dispatch token，拿不到 LLM key 和 web-content PAT |
| 个人 web-content PAT | 只注入发布 step，不暴露给 opencode 进程 |
| opencode 修改 Git 元数据 | 发布使用 fresh clone 和受限 patch，不复用被模型修改过的 `.git` |
| milvus-skills SSH deploy key | 只在 clone/install step 短暂存在，安装完成后删除私钥，opencode 无法读取 |

### 仍然存在的风险

| 威胁 | 影响 | 防护 |
|---|---|---|
| private repo workflow 被恶意修改并合入 | 可读取 `OPENCODE_*` 或个人 PAT | 只有维护者有 write/admin；保护 workflow 和 CI 配置；禁止不可信 PR 读取 secrets |
| dispatch token 泄漏 | 反复触发 runner，消耗模型费用 | 最小权限、短有效期、quota、RPM/TPM 限制和告警 |
| SDK tag / web-content / milvus-skills prompt injection | 诱导模型修改文件或尝试读取环境 | 预先固定 clone payload tag/commit；不给 opencode web-content PAT；限制输出 patch 路径；fresh clone 发布；安装后删除 skills deploy key |
| 变换后的 secret 输出绕过 GitHub mask | 可能泄漏 baseURL、key 或模型名 | 不打印 config，不开启 debug 日志；模型网关限额和轮换 |
| 自动 merge | 模型生成的错误或恶意文档可能直接进入 master | 仅允许 `API_Reference/`；生成与发布隔离；建议保留可回滚 commit 记录和异常告警 |

### 必要治理

1. private repo 默认分支启用保护，禁止无关人员修改：

```text
+.github/workflows/**
+ci/**
```

2. 不使用 `pull_request_target` checkout 不可信 PR head。
3. LLM key 使用独立 quota、模型 allowlist、RPM/TPM、max tokens 和费用告警。
4. PAT 设置短有效期并定期轮换。
5. 定期审计 private repo Actions 历史、workflow 变更和 secrets 引用。
6. 若发现 token 泄漏，先 revoke/rotate，再排查日志。
7. 如果自动 merge 风险不可接受，将最后的 `gh pr merge` 改为只创建 PR，人工 review 后合并。

---

## 落地顺序

1. 创建 private orchestrator repo，提交 `ci/opencode.base.json` 和 receiver workflow。
2. 给 `yhmo/milvus-skills` 配 read-only deploy key，并在 private repo 配置
   `OPENCODE_BASE_URL`、`OPENCODE_API_KEY`、`OPENCODE_MODEL`、`WEB_CONTENT_PERSONAL_PAT`、
   `MILVUS_SKILLS_DEPLOY_KEY`、`GIT_AUTHOR_NAME`、`GIT_AUTHOR_EMAIL`。
3. 用 `repository_dispatch` 手动触发一次 private receiver，验证 milvus-skills 安装、
   shallow clone、opencode、patch、PR 创建和 merge。
4. 在 4 个 SDK repo 增加 sender workflow，并配置 `PRIVATE_DISPATCH_TOKEN`。
5. 打一个真实 SDK tag，走完整链路验证。
6. 确认 web-content master 上的 API Reference 变更、PR 作者、merge 作者和分支清理结果。
