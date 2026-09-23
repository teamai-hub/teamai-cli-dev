[English](./README.md) | **中文**

# teamai-cli-dev — Dogfood 团队仓库

这是 [teamai-cli](https://github.com/Tencent/teamai-cli) 贡献者的 **dogfood 团队仓库**。
在开发 CLI 本身时，把本地 CLI 构建指向这个仓库，即可端到端跑通真实的
`init` → `pull` → `push` 流程。

这**不是**用来拷贝的模板。要初始化你自己的团队仓库，请从
[`teamai-hub/teamai-template`](https://github.com/teamai-hub/teamai-template) 开始。

## 用法

在你的 teamai-cli clone 里操作（不要用 `teamai init .`）：

```sh
npm run build && npm link
teamai init https://github.com/teamai-hub/teamai-cli-dev --scope project --role dev
teamai pull
git status   # .teamai/ 和各工具目录都不应有内容被暂存进 teamai-cli
```

- 请 init 上面这个**规范 hub 地址**，不要 init 个人 fork。
- 成员注册时出现 `Push failed (you can push manually later)` 在无写权限时是**预期行为**。
  本地配置仍会保存，`teamai pull` 仍可正常工作。

## 仓库里有什么

### 团队 skill

**暂无。** skill 是可执行指令，必须先过团队 review 才能在此分发。审核通过后会通过
正常的 `teamai push` 流程加入。

CLI 的**内置 skill** 与本仓库无关——它们随 `teamai-cli` npm 包一起发布，
并在每次 `init` / `pull` 时自动部署：

- `team-wiki-codebase`——始终部署。
- `teamai-share-learnings`——recall 开启时部署。

因此 `teamai pull` 后你会拥有上述内置 skill，而来自本仓库的团队 skill 为 **0**，
直到有过审的 skill 被加入。

### 访问权限

| 谁 | hub 仓库权限 | 计入团队 digest |
| --- | --- | --- |
| 贡献者 | 只读 | 否 |
| 协作者（提过几个 PR 后） | 写，`main` 受保护 | 是 |

`digest` / `dashboard` 从团队仓库读取 `stats/`、`sessions/`、`members/`。
这些文件通过 git 写入，因此**没有写权限 ⇒ 不进团队统计**。

## 许可证

[MIT](./LICENSE)。
