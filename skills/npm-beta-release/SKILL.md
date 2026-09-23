---
name: npm-beta-release
description: >-
  仅向 public npm 发布 teamai-cli 的 beta 预发布，且不触碰内部 tnpm、也不移动任何 registry 的 latest。
  用户提到发 npm beta、只发公开包、不要影响 tnpm、不要 bump latest、基于 main 打预发布、
  publish beta to npm only、npm-only prerelease、GitHub Actions tag 发布时必须使用本 skill。
  不要与 tnpm-beta-release（只发工蜂/tnpm）混淆。
---

# 仅发布 npm beta（不影响 tnpm / latest）

teamai-cli 是**同码双包**：GitHub tag 会打 public npm（`teamai-cli`），工蜂 tag 会打内部 tnpm（`@tencent/teamai-cli`）。要只发公开 beta，关键是 **tag 只推 GitHub，绝不推工蜂**。

`.github/workflows/release.yml` 对带 `-beta.` 的版本执行 `npm publish --tag beta`，**不会**改 npm `latest`。CLI 自动更新读的是 `latest`。

只发内部包用 `tnpm-beta-release`，不要用本 skill。

## 何时用

- 基于已合入的 `main`（或指定 commit）给公开用户装一版试用
- 明确要求：不影响 tnpm / 工蜂 Coding CI、不改 `latest`
- 不要用本 skill 发正式版（无 `-beta` / `-rc` 的版本会打到 `latest`）

## 发布前先给用户看（必须）

动手改版本、打 tag 之前，用**刚 fetch 过的**当前事实列出：

1. **建议版本号**（见下方怎么选）
2. **本次发布的改动 diff**：相对**上个稳定版**（= 当前 npm `latest` 对应的 tag，如 `v0.25.0`）到本次要打的 commit（默认 `<github-remote>/main` tip），按类型（feat / fix / docs）归类的 commit 摘要 + 文件变更统计，一句话点出主要变化（见下方怎么算）
3. **将执行的步骤**（fetch → worktree → bump → 验证 → 只推 GitHub tag）
4. **隔离保证**：不 push 工蜂 remote；版本是 prerelease 所以 dist-tag 是 `beta` 不是 `latest`
5. 等用户确认后再改版本、打 tag、push

### 怎么算改动 diff

上个稳定版取 npm `latest`（`npm view teamai-cli dist-tags`），对应 tag `v<latest>`：

```bash
git log --no-merges --pretty='%h %s' v<latest>..<github-remote>/main
git diff --stat v<latest>..<github-remote>/main | tail -1
```

把 commit 按 `feat` / `fix` / `docs` 前缀归类呈现，末尾给一句话总结主要变化。用户指定了 commit / PR head 时，diff 的右端换成该对象。

## 双流水线（为什么不能 `git push --tags`）

| 端 | 触发 | 包名 | 配置 |
|---|---|---|---|
| GitHub Actions | push `v*.*.*`（含 `v0.23.0-beta.1`） | `teamai-cli` → npmjs.org | `.github/workflows/release.yml` |
| Coding CI / 智研 | 工蜂 tag，且 `QCI_TRIGGER_TYPE=TRIGGER_TAG` | 构建期改名为 `@tencent/teamai-cli` → tnpm | `.coding-ci.yaml` |

工蜂 tag 正则是 `/^v\d+\.\d+\.\d+/`（**没有** `$`），因此 `v0.23.0-beta.1` 也会触发内部发布。

GitHub 的 `v*.*.*` 同样匹配预发 tag。所以：

- 只 push：`git push <github-remote> refs/tags/vX.Y.Z-beta.N:refs/tags/vX.Y.Z-beta.N`
- **不要** `git push --tags`，不要 push 到工蜂（`git.woa.com/teamai/teamai-cli`）

本仓库 remote **以 `git remote -v` 为准**，不要写死 `origin`：

- 有的 clone：`origin` = GitHub `Tencent/teamai-cli`，`tgit` = 工蜂
- 有的 clone：`origin` = 工蜂，`tencent` = GitHub

先认出哪个 remote 指向 `github.com/Tencent/teamai-cli`（下称 `<github-remote>`），哪个指向 `git.woa.com`（下称 `<tgit-remote>`）。

## 选版本号

先 **fetch 最新代码和 tag**，再查占用：

```bash
git fetch <github-remote> main --tags
git log -1 --oneline <github-remote>/main

npm view teamai-cli dist-tags versions --json --registry https://registry.npmjs.org
npm view @tencent/teamai-cli dist-tags versions --json --registry http://r.tnpm.oa.com
git ls-remote --tags <github-remote> 'v0.*.0-beta.*'
git ls-remote --tags <tgit-remote> 'v0.*.0-beta.*'
```

规则：

- 新版本必须在 **npm 不存在**，GitHub tag 也不存在
- 也要查 tnpm / 工蜂 tag：不要复用 tnpm 上**已经发过、且不是当前这棵树**的号（例如 tnpm 已有 `0.23.0-beta.0` 但指向别的 commit 时，npm 用 `0.23.0-beta.1`）。这样以后把**同一 tag** 推工蜂补发 tnpm 才不会撞车
- 不要复用 PR 分支里的旧 `package.json` 版本（常常已经发过）
- 不要发 **低于当前 npm `latest` 的** `0.22.0-beta.N`：`0.22.0` 已正式发布时，新功能预发用下一个 minor，例如 `0.23.0-beta.0`
- 预发标识必须是单词 `beta`（或 `rc`），以便 `release.yml` 解析 dist-tag；不要用 `0.23.0-1`

## 流程

### 1. 必须先拉最新

默认从 `<github-remote>/main` 的 **fetch 后 tip** 发，不要用本机可能过期的 `main`。

用户指定 commit / PR head 时，仍先 `git fetch`，再以该对象建 worktree。

### 2. 隔离 worktree，不要改功能分支 / 主工作区

从选定 commit 建 **detached** worktree。版本 bump 只存在于这条发布提交，不回写 `main` 或 feature 分支，也不把 release commit push 到任何 branch。

```bash
git worktree add --detach <path> <github-remote>/main
```

### 3. Bump 仅限 package 文件

```bash
npm version <chosen> --no-git-tag-version
```

只应改 `package.json` / `package-lock.json` 的 `version`。

### 4. 本地验证

```bash
npm ci --ignore-scripts
npx tsc --noEmit
npx vitest run
npm run build
# 确认构建产物版本，例如：
node dist/index.js --version
```

### 5. 确认后再 commit + annotated tag

```bash
git add package.json package-lock.json
git commit -m "chore(release): <version>"
git tag -a v<version> -m "v<version>"
```

### 6. 只把 tag 推到 GitHub

推送前再确认两侧都没有该 tag：

```bash
git ls-remote --tags <github-remote> refs/tags/v<version>
git ls-remote --tags <tgit-remote> refs/tags/v<version>
git push <github-remote> refs/tags/v<version>:refs/tags/v<version>
```

推完立刻再查一次：GitHub 有 tag，工蜂 **没有**。

不要在本机 `npm publish`。默认走 GitHub Actions。

### 7. 盯 GitHub Actions Release

```bash
gh run list --repo Tencent/teamai-cli --workflow release.yml --limit 5
gh run watch <run-id> --repo Tencent/teamai-cli --exit-status
```

工作流：`.github/workflows/release.yml`（job `Publish to public npm`）。`headBranch` 会是 tag 名（如 `v0.23.0-beta.1`）。成功后会建 **prerelease** 的 GitHub Release。

### 8. 发布后核对（四项都要看）

```bash
npm view teamai-cli dist-tags --json --registry https://registry.npmjs.org
npm view teamai-cli@<version> version --registry https://registry.npmjs.org
npm view @tencent/teamai-cli dist-tags --json --registry http://r.tnpm.oa.com
git ls-remote --tags <tgit-remote> refs/tags/v<version>
```

成功标准：

- npm `beta` = 新版本；npm `latest` **没变**
- tnpm 的 `latest` 和 `beta` **都没变**
- 工蜂 **没有**该 tag

安装：

```bash
npm i -g teamai-cli@beta
# 或钉死版本
npm i -g teamai-cli@<version>
```

以后若要补发内部 tnpm：把**同一个 tag** 推到 `<tgit-remote>`（改走 `tnpm-beta-release` 的推送与核对）。不要另起一个号，除非代码又变了。

## 不要做的事

- 把预发 tag 推到工蜂 / `<tgit-remote>`
- `git push --tags`（会把 tag 推到当前 remote 的全部 tags，且容易推错端）
- 发不带 `-beta`/`-rc` 的版本（会写成 npm `latest`，自动更新会吃到它）
- 在 PR 分支或本机 `main` 上直接 `npm version` 再 push 分支
- 为了发公开包去合无关 PR 或 cherry-pick
- 本机 `npm publish`

## 相关 skill

- `tnpm-beta-release`：只发内部 tnpm beta（tag 只推工蜂）
