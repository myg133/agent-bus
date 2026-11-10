# ADR-0001 会话↔agent 映射（session map v1）

- 状态：提议（待 skill 维护者评审）
- 提出：BA `tree`（DSH / Windows / Michael-WorkStation）
- 日期：2026-11-10
- 影响面：`aw.py`、`aw_mcp.py`、`references/global/{concurrency,mcp}.md`

## 背景

DSH 侧已实现 Host 插件 `aw-wake`：读 `.aw/wake/<agent>.json`，调用 DSH 的 `agent.followup()` 唤醒空闲会话。

问题：插件只能按 **cwd** 匹配会话。同一 repo 下开多个 BA 会话（多 BA 单 repo）时无法区分，会把所有同 cwd 的会话一起唤醒。

需要一个**权威的"会话 ↔ agent"映射**，让任意客户端都能回答：`agent_id=tree` 当前对应哪个会话？

事实依据（已实测）：

- 会话内的 shell 能看到 `DSH_SESSION_ID`（DSH 给每次命令注入），且等于 DSH 的会话 id；
- MCP 服务进程是 Host 共享、早期启动，看不到任何会话变量，`aw.py` 解析退回 `proc-<pid>`；
- 因此 `.aw/sessions/<agent>.json` 会被 MCP 用 `proc-<pid>` 覆盖真实 id —— 这是当前 bug 的根源。

## 硬约束

1. **只增不改**：现有 CLI 参数、输出字段、退出码、JSON 结构一律保持；新增字段只能追加。
2. **对其它工具零影响**：skill 同时在 pi/omp、DSH 及更多客户端使用。不得改变 pi/omp 现有 steer/唤醒路径、不得改变 `session` 解析优先级、不得加入按客户端特判的分支。
3. **不新增网络操作**：`watch` / `msg` 路径的 git 调用次数保持不变（本机到 GitHub 单次 6–12 秒）。
4. **不新增常驻进程/线程**。
5. **失败软降级**：映射写入失败只记日志，`claim` / `msg` / `agent` 命令不得因此非零退出。
6. **不加全局锁**：沿用"一 agent 一文件"。
7. Python 3.8+，不引入第三方依赖。

## 改动 1：generated 身份不得覆盖真实会话身份

- `session_source` 为 `generated`（`proc-<pid>`）时：**只更新 `last_seen`**，绝不改 `session_id`；
- 无真实记录时，可写 `mcp_session_id: "proc-<pid>"`，`session_id` 留空；
- 真实来源（`DSH_SESSION_ID` / `PI_SESSION_ID` / `AW_SESSION_ID` / `--session`）正常写入，`session_source` 记录**具体变量名**。

## 改动 2：新增权威映射文件 `.aw/sessions/<agent>.map.json`

```json
{
  "agent_id": "tree",
  "session_id": "session-488f836f-ee40-4e2e-8ed1-e4066ebf123e",
  "session_source": "DSH_SESSION_ID",
  "host": "Michael-WorkStation",
  "cwd": "E:/Dev_tmp/agent-bus-test",
  "pid": 0,
  "updated_at": "2026-11-10T23:37:00+08:00",
  "map_version": 1
}
```

- 只在拿到真实会话 id 时写；
- **原子替换**（临时文件 + `os.replace`），避免半截 JSON；
- 每 agent 一个文件，禁止汇总单文件；
- `session_id` 未变时 60s 内不重复写；
- `agent whoami` / `agent bind` 必须刷新；其它命令顺手刷新即可；
- `pid` 固定 `0`（会话级身份，非短命进程）。

## 改动 3：MCP 侧标注身份不可用于映射

`aw_mcp.py`：

- 写会话文件时固定 `session_source: "mcp"`（或 `generated`），**不写 `.map.json`**；
- 新增可选参数 `session`（透传 `aw.py --session`），缺省行为不变；
- 工具名、参数名、返回结构不变。

## 文档

`references/global/concurrency.md`（身份一节）+ `global/mcp.md` 增加契约：

> `.aw/sessions/<agent>.map.json` 是"会话↔agent"的唯一权威映射；
> `session_source` 为 `generated` / `mcp` 的记录只代表"某个进程用过这个身份"，不得用于会话定位；
> 映射是本机文件，不跨机同步。

## 验收标准

1. `python -m unittest discover -s tests -v` 0 失败；
2. pi/omp 实跑 `msg send` → 自唤醒（steer）行为不变，`session_source` 应为 `PI_SESSION_ID`；
3. 单 BA 无感：`agent whoami --json`、`msg inbox --json`、`doctor` 前后对比，除新增字段外一致；
4. 不覆盖：真实会话写过 map 后，连续 5 次 MCP 调用，`map.json` 的 `session_id` 不变；
5. 多 BA：`tree` / `tree2` 两个会话各生成独立 map，互不干扰；
6. 换会话自动改绑：新会话跑一条命令后 map 自动更新；
7. 无新增网络/Git 调用：`watch` / `msg` 路径 git 调用次数不变；
8. 失败软降级：`.aw/sessions/` 只读时 `claim list` 仍 exit 0，只打 warn。

## 明确不做

- 不改 `session` 解析优先级顺序；
- 不做按客户端（DSH/pi/…）的特判分支；
- 不为写 map 增加 `git fetch` 或任何网络调用；
- 不引入后台线程/守护进程；
- 不改 `.aw/wake/` 格式（DSH 插件与 pi 扩展都在读）；
- 不改 `bus` 消息格式、不改 `claims` 锁语义。

## 消费方（DSH 插件侧）承诺

映射落地后，DSH 插件按以下优先级匹配会话，不需要再改 skill 侧格式：

1. `.aw/sessions/<agent>.map.json` 的 `session_id` 精确匹配；
2. 配置中的 `sessionId`（显式 pin）；
3. 仅当 cwd 命中**恰好一个**会话时退化为 cwd 匹配；
4. 命中多个或找不到 → **拒绝唤醒并记日志**，绝不广播。
