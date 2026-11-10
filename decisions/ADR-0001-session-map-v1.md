# ADR-0001 会话↔agent 映射（session map v1）

- 状态：提议 v2（已吸收 ship 评审意见，待实现方确认）
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

## 改动 3：反向映射（支撑「每会话显式 agent_id」）

新增 `.aw/sessions/by-session/<key>.json`，`key = sha256(session_id)[:16]`：

```json
{ "session_id": "session-488f836f-ee40-4e2e-8ed1-e4066ebf123e",
  "agent_id": "tree", "source": "DSH_SESSION_ID",
  "updated_at": "2026-11-10T23:37:00+08:00" }
```

约束（来自 pi 侧评审）：

1. **文件名不用原始 `session_id`**（可能含分隔符或任意字符），用 `sha256(session_id)[:16]`，原始 id 存文件内；
2. **只写 `session_authoritative = true` 的真实 id**，`proc-*` 一律不写；
3. **必须有 GC**：按条数（建议 cap 500）或天数（建议 7 天）清理，避免无限增长；
4. **写失败只 warn**；消费方读不到反向映射时**回退到「唯一 cwd」规则**，不得直接拒绝或广播。

新增命令行为：`aw.py agent whoami --session <SID>` 在未显式给 `--agent` 时，用反向映射解析 agent_id；
解析不到则保持现有 `MISSING_AGENT_ID` 行为；`--json` 新增 `resolved_by: "session-map"`（其余字段不变）。

## 改动 4：MCP 侧标注身份不可用于映射

`aw_mcp.py`：

- 写会话文件时固定 `session_source: "mcp"`（或 `generated`），**不写 `.map.json` / `by-session/`**；
- 新增可选参数 `session`，透传时**必须拼成 `aw.py --session X <子命令>`**（`--session` 是全局参数，在子命令之前）；
- 工具名、参数名、返回结构不变。

身份来源优先级（两条都保留，不二选一）：

1. **env 透传优先**：客户端能在 MCP 配置里传 `AW_SESSION_ID` 时，直接拿到真实 id；
2. **map 反查兜底**：拿不到会话变量的客户端（如 DSH 桌面版，MCP 是 profile 级）走反向映射。

## 文档

`references/global/concurrency.md`（身份一节）+ `global/mcp.md` 增加契约：

> `.aw/sessions/<agent>.map.json` 是"会话↔agent"的唯一权威映射；
> `.aw/sessions/by-session/<sha256(session_id)[:16]>.json` 是反向映射，供会话从自身 session id 反查 agent id；
> `session_source` 为 `generated` / `mcp` 的记录只代表"某个进程用过这个身份"，不得用于会话定位；
> 映射是本机文件，不跨机同步。

## 验收标准

1. `python -m unittest discover -s tests -v` 0 失败；
2. **配置透传 `AW_SESSION_ID` 后**，pi/omp 实跑 `msg send` → 自唤醒（steer）行为不变，
   `session_source` 应为 `AW_SESSION_ID`；
3. 单 BA 无感：`agent whoami --json`、`msg inbox --json`、`doctor` 前后对比，除新增字段外一致；
4. 不覆盖：真实会话写过 map 后，连续 5 次 MCP 调用，`map.json` 的 `session_id` 不变，
   `session_authoritative` 仍为 `true`；
5. 多会话：同一 agent 在两个客户端各写一次 → `sessions` 含两条、打 warn、`session_id` 为最新；
6. 反向映射：`agent whoami --session <SID>` 解析出正确 agent_id，`resolved_by=session-map`；
   文件名是 sha256 前 16 位，且 `proc-*` 不产生反向映射文件；
7. GC：写满 cap 后旧条目被清理，总数不超上限；
8. 换会话自动改绑：新会话跑一条命令后 map 与 by-session 同时更新；
9. 无新增网络/Git 调用：`watch` / `msg` 路径 git 调用次数不变；
10. 失败软降级：`.aw/sessions/` 只读时 `claim list` 仍 exit 0，只打 warn。

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

## 评审修正（v2，来自 ship / pi 侧实测）

### 1. 验收 #2 在默认配置下不成立

omp 的 MCP 子进程**不继承** `PI_SESSION_ID`（`/proc/<pid>/environ` 里没有 `PI_*`）。
要在 MCP 侧拿到真实会话，必须在 `.mcp.json` 里显式透传：

```json
"env": { "AW_SESSION_ID": "${PI_SESSION_ID:-}" }
```

因此验收 #2 改写为：**"配置透传 `AW_SESSION_ID` 后"**，pi/omp 实跑 `msg send` → 自唤醒行为不变，`session_source` 为 `AW_SESSION_ID`。

### 2. `session_id` 不允许留空

`session_id` 还会进 claim 记录、`[session: ...]` 提交标签、审计 grep；留空会产出 `[session: ]`。

修正：**保留 `generated` 兜底值**（`proc-<pid>`），另加布尔字段 `session_authoritative`：

```json
{ "session_id": "proc-7300", "session_source": "generated", "session_authoritative": false }
```

消费方（插件/doctor/审计）只在 `session_authoritative == true` 时拿它做会话定位。

### 3. map 不能假设"一 agent 一活会话"

同一个 agent 在 omp 与 DSH 各开一个会话时，一 agent 一文件仍是 last-writer-wins，DSH 侧拿不到歧义信息，无法执行"拒绝唤醒"。

修正：map 内保存**最近 N 条**（N ≤ 8）会话记录：

```json
{
  "agent_id": "tree",
  "session_id": "session-488f...",
  "session_authoritative": true,
  "sessions": [
    { "session_id": "session-488f...", "source": "DSH_SESSION_ID", "at": "2026-11-10T23:37:00+08:00" },
    { "session_id": "session-bbbb...", "source": "PI_SESSION_ID",  "at": "2026-11-10T23:20:00+08:00" }
  ]
}
```

- `session_id` = 最新一条（保持简单读取）；
- 写入时若 TTL（建议 120 分钟，与 stale 阈值一致）内已存在**不同** id，追加并打 warn，不覆盖历史；
- 消费方看到 TTL 内多条不同 id → **拒绝唤醒并记日志**（歧义），由人确认谁该收消息。

### 4. 实现约束（两条）

- `--session` 是**全局参数**，MCP 侧拼接时必须放在子命令**之前**（`aw.py --session X msg inbox`），否则 argparse 报错；
- 60s 节流的 `stat` 只放在 `agent whoami` / `agent bind` / `claim` 这类低频命令上，**不要加进 `msg` 热路径**（每轮都要跑）。

