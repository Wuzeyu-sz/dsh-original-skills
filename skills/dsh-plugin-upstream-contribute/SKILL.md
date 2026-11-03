---
name: dsh-plugin-upstream-contribute
description: 把本机对第三方 DSH 插件（如 omdsh-dev/dsh-mnemon）的改动回提上游：先读上游 CONTRIBUTING(.zh-CN) 判定能否直接 PR（lib/ 等构建产物不得手改、全新能力必须先提 Issue、PR 须基于最新 main 且过 verify），再列 .github/ISSUE_TEMPLATE 解出字段 id 与双语 label，用 3~4 个同义关键词查重并逐个读 Issue 正文确认是否真被覆盖，然后按模板写中英双语正文用 gh issue create --body-file 提交，最后 gh issue view 回读 number/state/url/author 核验收口。当用户说「把改动回提上游」「能不能直接提 PR」「给上游作者提个 Issue」「这个需求上游是不是已经有人提过」「按上游模板提」时使用。
whenToUse: 用户有一份对第三方 DSH 插件仓库的本地改动（含改在 lib/ 等构建产物里的补丁），要判断能否直接开 PR、要走 Issue 先行、要按上游 Issue 模板提一条中英双语 Issue、要先查重避免重复提交，或要回读已提交 Issue 的编号与状态时使用。
---

# 第三方 DSH 插件改动回提上游

目标：用上游自己的规则判定通道（PR 还是 Issue），用上游自己的模板写正文，用多关键词查重避免重复提交，最后给出机器可核的编号与链接。

## 前置

- 需要 `gh` 已登录；Git Bash 下 `gh` 常不在 PATH：

```bash
export PATH="/c/Program Files/GitHub CLI:$PATH"
gh auth status 2>&1 | sed -n '1,4p'
```

确认输出是 `✓ Logged in to github.com account <你的登录名>` 且 `Active account: true`——后面 `author.login` 要拿它比对。

- 先锁定上游 repo slug（示例：`omdsh-dev/dsh-mnemon`）与本地实际改了哪些文件。

## Steps

1. 探上游仓库概况（默认分支、许可证、活跃度），确认你瞄准的是主源而不是镜像/分叉：

```bash
gh api repos/<owner>/<repo> -q '.default_branch, .license.spdx_id, .open_issues_count, .pushed_at' 2>&1 | head -6
```

2. 读上游贡献指南原文，别凭印象判断。先看存在性与大小，再读正文：

```bash
gh api repos/<owner>/<repo>/contents/CONTRIBUTING.md -q '.size'
gh api repos/<owner>/<repo>/contents/CONTRIBUTING.zh-CN.md -q .content 2>/dev/null | base64 -d | head -60
```

英文文件同样用 `contents/CONTRIBUTING.md -q .content | base64 -d` 读。

3. 把指南落成「能否直接 PR」判定表，至少覆盖三项硬规则：
   - `lib/` 是构建生成物，不进入版本库，**不得手工编辑**；
   - 全新能力必须先提 Issue，获得维护者确认后再开始实现；
   - PR 要求：基于最新 `main`、`pnpm run verify`、定向回归 + 真实证据、Conventional Commits、**禁 emoji**。

   字节级改 `lib/index.js` / `lib/client.js` 的补丁命中第 1、3 条 → **不能直接 PR，走 Issue 先行**。若改动落在 `plugins/*/src` 且上游同文件与本地「改动前」字节一致，才允许按指南走 PR（本机默认见 `~/.dsh/AGENTS.md` 第 3 条：`contribute_upstream.py --mode issue`，只有字节一致才 `--mode pr`）。

4. 列出上游全部 Issue 模板，决定用哪一个：

```bash
gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE -q '.[] | .name + " " + .type' 2>&1 | head
```

   功能请求/问题类改动通常用 `standard_issue.yml`；明确 bug 才用 `bug_report.yml`。

5. 解出模板的字段 `id` 与双语 `label`（正文必须按这些 id 组织，否则维护者要你重填）：

```bash
gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE/standard_issue.yml -q .content 2>/dev/null | base64 -d 2>/dev/null | grep -E "^\s+(label|id):" | head -30
```

   已知的一组必填 id（以实际读出的为准）：`duplicate-check`、`area`、`issue-type`、`summary`、`expected-outcome`。同时读出 `area` / `issue-type` 的可选值，写正文时按原文措辞填。

6. 查重第一遍——**3~4 个同义关键词批量搜**（中文词 + 英文词 + 术语变体；本次 publish/发布/one-click/evolution 各命中不同编号）：

```bash
for q in "publish" "发布" "one-click" "evolution"; do echo "== $q"; gh api "search/issues?q=repo:<owner>/<repo>+$q&per_page=5" -q '.total_count, (.items[]?|.number|tostring)+" "+(.title)' 2>&1 | head -6; done
```

   `total_count` 是大数（几百上千），真正要看的是 items 里的编号与标题。

7. 查重第二遍——用 `gh issue list --search` 复核候选（搜索命中常含标题无关项，必须剔除）：

```bash
gh issue list --repo <owner>/<repo> --state all --search "<关键词>" --limit 8 2>&1 | head -10
```

8. 逐个读候选正文，判定「是不是真被覆盖」（标题像 ≠ 覆盖，必须看正文）：

```bash
gh issue view <number> --repo <owner>/<repo> --json title,body,labels,state -q '.title, .state, (.labels|map(.name)|join(",")), .body' 2>&1 | head -40
```

   判定口径：正文描述的诉求与你要提的是同一件事才叫「已覆盖」→ **不重复提交**，记下编号 + `OPEN`/`CLOSED` + 标题；只是相邻领域则记为「相邻但不重复」，写进新 Issue 的查重段。

9. 按模板字段 id 写中英双语正文到文件（放工作区 `_pub/issue-<slug>.md`），每段用 `### 中文 / English` 双标题，逐段对齐 id：

```markdown
### 提交前查重 / Duplicate check
- [x] 已搜索 open 与 closed issue。相邻但不重复：#474（...）只覆盖 X，不涉及 Y；#451（CLOSED）是 Z。未发现「<你的诉求原句>」的请求。

### 涉及区域 / Affected area
Web UI 或对话交互 / Web UI or conversation interaction

### Issue 类型 / Issue type
功能请求 / Feature request

### 摘要 / Summary
...

### 预期结果 / Expected outcome
...
```

   正文只写产品级描述与复现/证据，**不带本机绝对路径、凭据、内网信息**；禁 emoji。

10. 提交（`--body-file` 而不是 `--body`，避免引号与换行被 shell 吃掉；`cd` 到正文文件所在目录）：

```bash
cd <正文文件目录> && gh issue create --repo <owner>/<repo> --title "[Issue]: <中文 / English 标题>" --body-file issue-<slug>.md
```

    输出形如 `https://github.com/<owner>/<repo>/issues/<number>`，记下 number。一次只对一个未被覆盖的诉求提交。

11. 回读核验（这一步才是「提交成功」的证据，不要拿创建时的输出当结论）：

```bash
gh issue view <number> --repo <owner>/<repo> --json number,title,state,labels,url,author -q '.number, .state, .url, .author.login, (.labels|map(.name)|join(","))' 2>&1 | head -6
date "+%Y-%m-%d %H:%M"
```

   要求：`state` = `OPEN`、`author.login` = 自己的登录名、`url` 可点开。

12. 收口表：一条诉求一行，写清「已被某 Issue 覆盖 → 不提」还是「已提交 → 编号 + 链接 + 状态 + 作者」，并给出下一步（等维护者确认后再按指南改 `src` 并跑 `pnpm run verify`）。

| 诉求 | 结论 | 证据 |
|---|---|---|
| 自动起草/评估/采纳 | 已被 #474（OPEN, enhancement）覆盖 · **不提** | `gh issue view 474` 正文含 draft/eval/adopt 全链 |
| 技能发布到插件市场 | 已提交 #536 · https://github.com/<owner>/<repo>/issues/536 · OPEN · <author> | `gh issue view 536` 回读 |

## Pitfalls

- **直接拿改过的 `lib/*.js` 提 PR 必被拒**：上游写死 `lib/` 是构建生成物、不得手工编辑。先 Issue，拿到确认再改 `src`。
- **只搜一个关键词会漏**：同义/中英/术语变体命中的编号完全不同（publish / 发布 / one-click / evolution 各自指向不同 Issue）。至少 3~4 个词，且两遍（`gh api search/issues` + `gh issue list --search`）。
- **标题相似不等于被覆盖**：必须 `gh issue view` 读正文；本次靠读 #474 正文才确认「一键处理」已被覆盖，从而不重复提交。
- **正则太窄提取不到字段**：`grep -n "label:\|id:"` 在本例无输出，改用 `grep -E "^\s+(label|id):"` 才拿到全部 id。
- **把 `total_count` 当命中列表**：search 返回 315 这类大数，真实条目在 `.items[]`，要用 items 的 number/title。
- **不核 author 就宣布成功**：多账号环境里 `gh auth status` 的 active account 决定作者；提交后必须回读 `author.login`。
- **未经用户确认扩大动作**：Issue 提交后不要顺手提 PR、改上游仓库或代维护者打 label；查重结论与「不提」的判断要在最终表里明说。
- **正文泄密**：Issue 是公开的，别贴本机路径、token、私有仓库名。

## Verification

- CONTRIBUTING 的硬规则被转成判定表并引用原文要点（构建产物 / 先 Issue / main+verify / Conventional Commits 禁 emoji）。
- 每个诉求都有查重记录：关键词列表 + 候选编号 + `OPEN`/`CLOSED` + 读正文后的覆盖判定。
- 被覆盖的诉求有明确「不提交」声明；未被覆盖的诉求只有一条，且正文段落与模板字段 id 一一对应。
- 新 Issue 的 `gh issue view` 回读同时给出 `number`、`state=OPEN`、`url`、`author.login`。
- 最终表里每条诉求都有结论（不提 / 已提交）与可核证据。
