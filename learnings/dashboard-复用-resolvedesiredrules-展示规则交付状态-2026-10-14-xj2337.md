---
title: "Dashboard 复用 resolveDesiredRules 展示规则交付状态"
author: WLH55
date: 2026-10-14
tags: [typescript, api, pattern, best-practice, testing]
confidence: 0.85
source_mr: "https://github.com/Tencent/teamai-cli/pull/1113"
---

## 背景

Issue #1028 要给 `teamai dashboard` 加一个 Resources 页面，让用户看清团队仓库里的每条规则在当前工作区会不会被 `pull` 交付，以及没交付的原因。规则交付由几条通道共同决定：

- 根规则：所有人都会收到；
- 命名空间规则：只有对应的知识命名空间（knowledge namespace）处于活跃状态才交付；
- 标签订阅：可能把某些规则排除掉；
- 同名覆盖：活跃命名空间里有同名规则时，会取代对应的根规则。

如果 Dashboard 自己重写一套判定逻辑，迟早会和 `pull` 的实际行为不一致，页面上显示"已交付"，`pull` 却没下发。另外，user scope 和未分配（unassigned）分组根本不拉团队仓库，返回空列表的话，用户分不清是"没有规则"还是"这里不适用"。

## 解决方案

1. **交付结果只来自一个地方**：新增 `GET /api/resources/rules`，"是否交付"直接调用 `pull` 用的 `resolveDesiredRules` 得到，不在 Dashboard 里另写判定，这样页面显示的交付状态和 `pull` 的实际结果一致。
2. **按流水线顺序找出被哪条通道排除**：对没交付的规则，再跑一次命名空间过滤和标签过滤，只用来说明是哪条通道把它排除的。检查顺序有讲究：被取代的根规则其实已经通过了命名空间和标签两道过滤，是在最后一步才被同名命名空间规则覆盖的，所以要**先判断 `replaced`**，再判断命名空间未激活和标签排除，否则原因会标错。
3. **只返回元数据**：接口返回名称、命名空间、描述和交付状态，不返回规则正文和 env 值，避免通过本地 HTTP 接口泄露内容。
4. **不适用的情况单独标出来**：user scope 和 unassigned 分组返回 `{"configured": false, "rules": []}`，和"配置了但没有规则"区分开。
5. **拆成小块交付**：这次只做规则列表，详情接口 `GET /api/resources/rules/:name` 留给后面的 skills 页面一起做（那边带抽屉的设计稿），distribution 页面也放到后续。
6. **验证**：
   - 新增单测 `dashboard-resources.test.ts`，覆盖根规则、命名空间、标签、同名取代四种情况；
   - 用真实 CLI 做端到端验证：`npm run build` 后，在沙箱 HOME 里手工搭一个含 4 条规则的团队仓库，运行 `node dist/index.js dashboard --port 3799`，再用 `curl` 检查项目工作区和 `?workspace=user` 两种返回；
   - 中英文 `docs/guide/dashboard.md` 同步更新。

## 经验总结

- **展示层不要重写业务判定**：Dashboard、doctor、status 这类"展示状态"的功能，要直接调用执行路径上的同一个函数（这里是 `resolveDesiredRules`），否则展示结果会和实际行为不一致。
- **"交付了没有"和"为什么没交付"分开算**：交付与否以复用的函数为准；原因可以单独再算一遍，但必须按真实流水线的顺序检查。后面阶段会推翻前面阶段的结果（比如同名覆盖），所以要先查后面的阶段。
- **"不适用"要明确返回**：不适用的范围返回 `configured: false`，不要只给一个空数组，前端和用户都更容易看懂。
- **本地 Dashboard 接口默认只给元数据**：规则正文、env 值等敏感内容不要放进列表接口，确实需要时再单独设计详情接口并评估。
- **大需求拆成纵向小块**：一个页面加一个接口先合进去，详情和其他页面等依赖的设计确定了再做，这样 PR 小、好 review。
- **本地全量测试失败要先和干净的 main 对比**：Windows 上 vitest RPC 超时、符号链接权限这类失败，如果在干净的 `main` 上也一样失败，就不是本次改动引起的。PR 里写明这一点，以 ubuntu/macOS 的 CI 结果为准，再附一次真实 CLI 的端到端记录。

## 相关 Skills

- teamai