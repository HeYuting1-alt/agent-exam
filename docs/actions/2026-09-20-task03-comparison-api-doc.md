# 行动文档：把定稿的对比接口写入 HTTP_API.md §10.4

## 状态与情况说明

- 状态：已完成（§10.4 正文）；**待补两条 400**，触发条件见文末“后续”节
- 来源请求：用户按任务 03 发出脚本，要求把 D 定稿的跨批次对比接口写入 `docs/interfaces/HTTP_API.md` 的 §10.3 之后成为 §10.4，并在 fork 上建分支、提 PR。用户给的脚本被截断：`python - <<'PY'` 没有结束标记、Python 字符串未收尾、也没有 `git add/commit/push`；其中粘贴的响应示例还停留在提案早期版本。因此本次按“查明真实状态再落笔”执行，不照抄脚本正文。
- 职责依据：任务 03 的任务 DRI 是 **B（Web 与 HTTP 模块）**，`HTTP_API.md` 由该模块维护；[D 的定稿提案](2026-09-20-d-comparison-api-proposal.md)第 5 节把“本文落入 `HTTP_API.md` §10.4”列为 **B** 的工作。本次即该项落笔。
- 当前事实（本次逐条核对，均以代码和测试为准）：
  - 对比端点**已实现并已注册**：`delivery/http/routes/jobs/report_comparisons.py`、`delivery/http/app.py:124`；契约测试 `tests/jobs/reporting/test_comparison_http.py` 有 3 个用例。`HTTP_API.md` 此前没有 §10.4，全库文档也没有描述该端点。
  - 真实 `totals` 形状是 **7 个整数**：`resolved/unresolved/infrastructure_error/incomplete/missing/decided/total`；其中 `decided = resolved + unresolved + infrastructure_error + incomplete`，`total = decided + missing`（`report_comparisons.py:48-111`）。测试断言 `body["totals"][0]["total"] == 2`。
  - 用户脚本与 D 提案 §2 示例里的 `"coverage": "6/6"` **不存在于实现**。提案 §6 决定 4 已明确“`totals` 提供 `decided` 与 `total` 两个整数，**不另设字符串覆盖率**”，实现与测试都按该决定；只有提案 §2 的示例与字段表仍留着旧的 `coverage`，属于失效描述。
  - 已实现的 400 族**当时**只有三种：空选择 `EMPTY_COMPARISON_SELECTION`、非 UUID `INVALID_REQUEST`、超过 20 个 `COMPARISON_LIMIT_EXCEEDED`（`test_comparison_http.py:115-129`、`errors.py:84-90`）。**“未知参数、重复参数返回 400”当时在对比端点上确实没有实现**，提案 §4 该行不能照写。（⚠️ **2026-09-20 更正**：本条初稿还写了“本仓没有严格 query 校验层”，**那是错的**——该机制早已存在于 `leaderboard/routes.py:53-57`、`jobs/routes.py:83-95`、`catalog.py:123-125` 三处，B 的搜索未递归进 `routes/` 子目录。对比端点当时缺这条校验，缺口为真；D 随后补实现，成为第 4 处。详见文末“后续”节。）
  - 去重按出现顺序（`dict.fromkeys`）、上限 20（`MAX_COMPARISON_JOBS`）、任一无权或不可读 Job 令整体失败且不指出是哪个、`internal_test` 按不存在处理（`service.py:57-73`）。
  - 行序为 `(repo, task_instance_id)` 升序（`matrix.py:94`）。
  - §2.1 清单自称“只列已经注册的 31 个 HTTP 端点”，而源码现有 **32** 个路由装饰器，第 32 个正是对比端点；Web 侧 `apps/web/src/` 没有任何 `comparisons` 调用（对比页 UI 尚未实现，属任务 03 后续）。
  - `GET /api/v1/reports/comparisons` 落在 §12 映射表已有的 `/reports` 行内，无需改动该表。
- 已确认决定：
  - 文档以**代码与测试**为准，不复制用户脚本/提案 §2 中不存在的 `coverage` 字段。
  - §10.4 只写已实现且可核验的行为，不写未实现的参数校验规则。（该决定的前提是“当时实现里没有这条校验”；D 于同日补实现后，§10.4 需要补写——**B 决定有意延后**，见文末“后续”节。）
  - 同步 §2.1 清单：增列该端点并把计数改为 32，Web 调用列如实标注“尚未接入（对比页待实现）”，避免同一文件内自相矛盾。
  - §15 变更记录追加本次条目，符合该文档既有惯例。
- 明确排除：不改任何产品代码、测试、路由或 schema；不改 D 的提案文档（属 D 维护，本次只报告其 §2 失效）；不实现对比页 UI；不改 §12、§13；不推送 `main`（已与上游一致，无内容可推）；不把上一任务未提交的改动带进本分支。

## 实施措施

1. 建分支 `task03/comparison-api-spec`。
2. 在 `HTTP_API.md` §10.3 之后新增 §10.4，字段以代码为准：五档 `ComparisonOutcome`、缺失语义、`decided`/`total`、授权与 404 收敛、错误码表。
3. `§2.1` 增列 `GET /api/v1/reports/comparisons`，计数 31 改为 32。
4. `§15` 变更记录追加 2026-09-20 条目。
5. 精确暂存这 3 个改动加本行动文档，检查暂存 diff 后提交。
6. 推送到 `origin`（个人 fork）并给出向上游提 PR 的链接；本机无 `gh`，PR 由用户在网页确认创建。

完成标准：`HTTP_API.md` 的 §10.4 与实现、测试三者字段完全一致；§2.1 计数与源码路由数一致；提交只含本次范围；分支已推送且能给出 PR 链接。

## 实际修改的文件树

```text
D:\agent-exam\
├─ docs\interfaces\HTTP_API.md
│  # 修改：新增 §10.4 跨批次对比报告；§2.1 增列端点并 31→32；§15 追加变更记录
└─ docs\actions\2026-09-20-task03-comparison-api-doc.md
   # 新增：本次落笔的请求、依据、执行与验证记录
```

本次不引入代码设计模式，也不新增顶层目录或接口：只把已实现契约写入既有 HTTP 文档。

## 修改后自验证方式

1. `git diff docs/interfaces/HTTP_API.md`：逐项对照 `report_comparisons.py` 的 DTO 与 `service.compare` 的分支，确认字段名、五个 `outcome` 取值、错误码和上限数字一致。
2. 在全文中检索 `coverage`：预期 §10.4 内不出现该字段；确认没有把不存在的字段写进契约。
3. 检索 `31 个`：预期计数已改为 32，且 §2.1 表内出现对比端点的行。
4. 与测试对照：`test_comparison_http.py` 断言的 `decided`/`total`/`resolved is None`/`report_path is None` 必须都能在 §10.4 找到对应描述。
5. `git show --stat HEAD` 与 `git diff --cached --name-only`：确认提交只含 `HTTP_API.md` 与本次行动文档。
6. 推送后比较本地分支与 `git ls-remote origin` 的分支哈希。
7. Markdown 结构检查：§10.4 位于 §10.3 与 §11 之间，标题层级与相邻小节一致。

## 自验证情况

- 字段对照：把 `report_comparisons.py` 的四层 DTO 与五档枚举拆成 20 个字段名，逐个在 §10.4 内检索，命中 **20/20**，无遗漏、无多余。
- 不存在的字段：§10.4 内检索 `coverage` 无命中。用户脚本与提案 §2 示例里的 `"coverage": "6/6"` 未写入——实现与测试用的是整数 `total`（测试断言 `body["totals"][0]["total"] == 2`）。
- 缺失语义：`matrix.py:114` 为“没有 Run”返回 `MatrixCellValue("missing", None, ...)`、`:116` 为“有 Run 但报告缺失”返回带 `run_id` 的 missing，与 §10.4 写的区分一致。
- 计数一致性：§2.1 已改为“32 个 HTTP 端点”，与源码 32 个路由装饰器一致；新增端点的表格行已就位（第 79 行）。
- 结构检查：章节顺序为 §10.3（第 640 行）→ §10.4（第 669 行）→ §11（第 750 行），标题层级与相邻小节一致。
- 提交范围：`git diff --cached --name-only` 只有 `docs/interfaces/HTTP_API.md` 与本次行动文档；上一任务未提交的 `HANDOFF.md` 改动与 `2026-09-20-fork-upstream-remote.md` 均未进入提交。提交 `6e9bdc1`，`--stat` 为 2 文件 +143/-1。
- 提交 diff 复核：已完整输出 `git diff --cached` 并逐行阅读后才提交。
- 推送核验：本地 `HEAD`、`task03/comparison-api-spec`、`origin/task03/comparison-api-spec` 三者同为 `6e9bdc1e590fe236b2b26723a7631a08e8281a65`。
- 脚本第 1 步：`main`、`origin/main`、`upstream/main` 三者同为 `beed93f`，没有内容可推，`git push origin main` 按空操作处理，未实际执行。
- 本次没有运行的行为检查：只改文档，未运行 `pytest`、`ruff` 或 `mypy`；§10.4 的行为依据是既有实现与 D 已记录的结果（`pytest tests/jobs/reporting` 16 passed 等），不是本次新跑。契约未改动，因此未新增运行；但“文档描述与运行行为一致”这一层只做到“代码 + 测试源码对照”，没有重跑测试。
- 未覆盖的限制：
  - 本机无 `gh`，无法用命令行创建 PR，只能给出网页链接，PR 由用户在网页确认创建。
  - §12 映射表未改：既有 `/reports`、`/leaderboard` 行已涵盖对比端点，无需新增。
  - D 的提案文档 §2 示例与字段表仍写着 `coverage`，与提案 §6 决定 4 和实现不符；该文档属 D 维护，本次只报告、未修改。
  - 提案 §4 声称“未知参数、重复参数返回 400”，但对比端点当时未做该校验（**注意：不是“本仓没有该机制”**，见上文更正），因此本次未写入 §10.4。是补实现还是改提案，当时留待 B/D 决定——**D 已答复：补实现**，见文末“后续”节。
  - §13 接口验证清单未新增对比接口条目，属可选项，本次未扩大范围。

## 2026-09-20 后续：D 的回复、错误更正与本 PR 有意不含的内容

**D 的回复**：三项全部对齐。

- 确认对比接口在 `bd47925` 交付。
- **补了一件实现**：`cdcb4cf feat: reject unknown and duplicate comparison query params`（`_reject_foreign_params`，与排行榜等既有读端点同模式；新增第 4 个契约用例）。
- 确认 `ComparisonOutcome` 与 `MatrixCell` **保持独立、不收敛、不提升为跨模块公开接口**。
- 提案文档两处失真由 D 在 `c5e036d` 收口（§2 改 `decided + total`；§4 补实现+测试）。
- D 的交叉验证：`test_comparison_http.py -q` → **4 passed**；全量 2 failed / 453 passed / 36 skipped。

**B 的一处判断更正**：本文件初稿写的“本仓没有严格 query 校验层”是错的——搜索未递归进 `routes/` 子目录。准确事实：该机制早已存在于 `leaderboard/routes.py:53-57`、`jobs/routes.py:83-95`、`catalog.py:123-125` 三处，且都在会话检查之前执行；对比端点当时缺这条是**真实的缺口**，D 补的是第 4 处，不是新规范。

### 本 PR 有意不含的内容（2026-09-20 决定）

§10.4 **不含** D 在 `cdcb4cf` 新增的两条 400（未知 query 参数名、`job_ids` 重复出现 → `INVALID_REQUEST`）与“参数校验先于会话检查”的优先级说明。

- **原因**：该实现只存在于 `upstream/xinyue-modules`，尚未合入 `main`。写进 §10.4 会让本契约 PR 依赖 D 的分支合并，并让 `main` 上出现“有契约、无可读实现”的描述；不写则本 PR 全部内容都与 `main` 一致，可独立合并。
- **影响**：契约暂时不完整（不是错误）。对对比页 UI 无影响——UI 自行拼接 `job_ids`，不会触发这两条。
- **触发补写**：D 宣布 `xinyue-modules` 已合入 `main` 之后，在 §10.4 补两条 400 与优先级说明（可另开小 PR）。
- **唯一权威**：这条待办的完整记录以本文件为准；模块文档 `progress.md` 与实现侦察行动只放指针，不复制理由。契约正文保持纯净，不写分支与合并状态。

**本机环境与实测（2026-09-20）**：后端环境已恢复——`uv 0.12.17`（`python -m pip install --user uv`，直连 PyPI）+ uv 管理的 Python 3.13.15（本机原只有 3.14.5，不满足 `requires-python`），`uv sync --locked --no-python-downloads` 退出码 0 建立 `.venv`。§10.4 所描述的既有行为已实跑佐证：`test_comparison_http.py` → **3 passed**（本分支无 D 的第 4 个用例）、`tests/jobs/reporting` → 14 passed / 2 skipped、全量 → 2 failed / 404 passed / 84 skipped（2 个失败为缺 `framework/harbor`）。因此上文“本次没有运行的行为检查”这条限制**已部分解除**：字段与既有行为有实跑佐证；**两条 400 的行为仍未在本机验证**（不在本分支）。基线可移植性的说明见模块文档 `docs/architecture/modules/web-and-http/actions/03-comparison-api-impl.md` 第 5.3 节（随 `docs/web-http-module-scaffold` 分支合并后可用；此处不写链接，避免本分支单独合并时产生断链）。

**PR 状态**：分支 `task03/comparison-api-spec` 已推送到 `origin`（个人 fork），**PR 尚未创建**——本机 `gh` 未登录（浏览器授权在换取 token 时因直连超时失败）。创建链接见模块文档 `docs/architecture/modules/web-and-http/progress.md`（同上，随 `docs/web-http-module-scaffold` 分支合并后可用）。
