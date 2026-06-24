# 设计文档：AgentBox PVC 持久化的 Agent 级可配置化

> 分支：`feat/agentbox-configurable-persistence`
> 基线：`origin/main` @ `f8ffe01`
> 作者任务：让 PVC 持久化从「Runtime 进程级全局开关」下沉为「每个 agent 可单独配置」，并在 Portal 前端 agent 创建/编辑页暴露开关。
> 本次范围：**仅 PVC 开关**。TTL、memory 等同类下沉留待后续（设计已为其预留扩展路径，见 §7）。

---

## 1. 背景与问题

### 1.1 GPU Cloud 的需求

GPU Cloud 复用 siclaw agent 内核，对外有多种产品形态的机器人（导购 / workbench / 诊断），由产品方在 **Portal 前端动态创建** agent。不同形态对持久化的需求相反：

- **导购 agent**：聊产品信息。关页面 / TTL 到 → session 应永久释放，用户回来新起 session。**不需要 PVC。**
- **诊断 agent**：多轮排障。用户要能回到旧 session 继续；pod 重启后旧 session 必须还在。**需要 PVC。**

因为 agent 是前端动态创建的（部署时未知有哪些 agent），无法用「一类一个 Runtime」的多实例方案预先分类。**架构上只能：单 Runtime + 配置随 agent 走。**

### 1.2 siclaw 现状：持久化是 Runtime 进程级全局开关

当前持久化决策在 Runtime 启动时由环境变量一次性固化，对该 Runtime spawn 的**所有** AgentBox pod 生效，无法按 agent 区分：

```
SICLAW_PERSISTENCE_ENABLED (env)
  └→ bootstrap-runtime.ts:167  new K8sSpawner({ persistence: enabled ? {...} : undefined })
       └→ K8sSpawner 实例上的 this.config.persistence（构造时固化）
            └→ spawn() 时所有 agent 套用同一个 this.config.persistence
                 (k8s-spawner.ts:202 / 251 / 305)
```

**目标**：把「是否挂 PVC」的决策从 `this.config.persistence`（spawner 实例级）改为按 `spawn(boxConfig)` 传入的 **per-agent** 标志决定。

---

## 2. 关键架构约束（动手前必须理解）

### 2.1 Runtime 不直接访问数据库

`server.ts:4` 明确："**All data access goes through Portal/Upstream adapter API.**" Runtime 通过 `FrontendWsClient` 的 WS RPC（phone-home）向 Portal 取数据，**不能直接 SQL 查 `agents` 表**。

因此 per-agent 的 `persistence_enabled` 标志，Runtime 必须**通过 RPC 向 Portal 取回**，再传给 spawner。

### 2.2 现成的两个模板（本设计的实现基础）

| 模板 | 位置 | 复用点 |
|---|---|---|
| **`is_production` 布尔字段** | `migrate.ts:40`、`agent-api.ts`(create/PUT)、`AgentSettings.tsx:165`(UI 复选框) | 端到端的 agent 级布尔开关全链路，PVC 开关照抄 |
| **`config.getModelBinding` RPC** | Runtime: `agent-model-binding.ts:34`；Portal handler: `adapter.ts:1850` | Runtime 在使用前通过 RPC 取 agent 配置的范式 |

> 这两个模板意味着：本任务不是探索性改造，而是**沿着已有模式各做一遍**，风险低。

### 2.3 per-agent 配置管道已存在，但无人填充

`AgentBoxConfig`（`gateway/agentbox/types.ts:13`）已支持按 agent 传 `env` 和 `resources`，`getOrCreate(agentId, config)` → `spawn(boxConfig)` 全程透传。**但所有 `getOrCreate` 调用点目前只传 `agentId`，第二参 config 全空**（`server.ts:218/320/344/355` 等）。我们要做的就是给这条管道接上数据源。

### 2.4 不变量（来自 CLAUDE.md，不可破坏）

- **PVC 必须 ReadWriteMany**（`siclaw-data-pvc.yaml` 注释："architectural and not parametrised"）。本设计不动 PVC 本身，只动「某 agent 是否挂载它」。
- **subPath 隔离粒度按 agent**：`agents/{safeAgentId}`（`k8s-spawner.ts:306`）。本设计保持不变。
- **claimName 属于基础设施级**（哪块 PVC），保持全局；**enabled 属于 agent 级**（该 agent 是否挂）。这是干净的职责拆分。
- `migrate.ts` 单 DDL 必须 MySQL + SQLite 双兼容：不得用 `TIMESTAMP(3)` / `ON UPDATE` / `JSON` 列；加列走 `safeAlterTable`。

---

## 3. 方案选型

### 3.1 配置存储：`agents` 表加一列（方案 1）

```sql
ALTER TABLE agents ADD COLUMN persistence_enabled TINYINT(1) NOT NULL DEFAULT 0;
```

- 照搬 `is_production`（`TINYINT(1)`）。**默认 0**（关闭）——与 siclaw 现状（`SICLAW_PERSISTENCE_ENABLED` 默认未设=关）语义一致，且对导购这类多数形态是安全默认。
- 不选 JSON 配置列（方案 2）：本次只有一个布尔，JSON 容器属于过早设计（YAGNI）。后续要加 TTL/memory 时，再 `safeAlterTable` 增列即可，成本很低（见 §7）。

### 3.2 配置传递：新增 `config.getAgentRuntimeConfig` RPC

仿 `config.getModelBinding`，让 Runtime 在 spawn 前取回该 agent 的部署配置。

> 命名用 `getAgentRuntimeConfig`（而非 `getPersistence`）是为后续 TTL/memory 留口——同一个 RPC 返回对象里加字段即可，不必每个开关一个 RPC。

---

## 4. 端到端改造链路

```
┌─ 前端 ────────────────────────────────────────────────┐
│ AgentSettings.tsx: BasicTab 加一个「会话持久化」开关     │
│   state persistenceEnabled ← 照抄 isProduction          │
│   保存时 body.persistence_enabled 进 create/PUT 请求    │
└───────────────────────┬───────────────────────────────┘
                        │ POST/PUT /api/v1/agents[/:id]
┌─ Portal API ──────────▼───────────────────────────────┐
│ agent-api.ts: create 的 INSERT、PUT 的 fields[] 各加    │
│   persistence_enabled 字段                              │
└───────────────────────┬───────────────────────────────┘
                        │ 写入
┌─ DB ──────────────────▼───────────────────────────────┐
│ migrate.ts: agents 表加列 + safeAlterTable 迁移         │
└───────────────────────┬───────────────────────────────┘
                        │ RPC: config.getAgentRuntimeConfig
┌─ Portal adapter ──────▼───────────────────────────────┐
│ adapter.ts: handlers.set("config.getAgentRuntimeConfig")│
│   SELECT persistence_enabled FROM agents WHERE id=?     │
└───────────────────────┬───────────────────────────────┘
                        │ frontendClient.request(...)
┌─ Runtime ─────────────▼───────────────────────────────┐
│ server.ts prompt 处理: getOrCreate(agentId) 前，先取    │
│   runtimeConfig，传入 getOrCreate(agentId, {persistence})│
│ (新增 resolveAgentRuntimeConfig，仿 agent-model-binding)│
└───────────────────────┬───────────────────────────────┘
                        │ getOrCreate → spawn(boxConfig)
┌─ K8sSpawner ──────────▼───────────────────────────────┐
│ spawn(): 用 boxConfig.persistence ?? this.config 决定   │
│   挂 PVC(subPath) 还是 emptyDir                         │
│ (k8s-spawner.ts: volume:251 / mount:305 / ensureDir:205)│
└────────────────────────────────────────────────────────┘
```

---

## 5. 逐文件改造点（基于 `f8ffe01` 行号）

### 5.1 DB —— `src/portal/migrate.ts`

1. `agents` 表 `CREATE TABLE`（~L32-46）加列 `persistence_enabled TINYINT(1) NOT NULL DEFAULT 0`。
2. 迁移区加：`await safeAlterTable(db, "agents", "persistence_enabled", "TINYINT(1) NOT NULL DEFAULT 0");`（参考现有 `model_routing` 的 safeAlterTable 调用）。
- **契约**：MySQL + SQLite 双兼容；`TINYINT(1)` 两边都支持，安全。
- **测试**：`npm test` 跑 `migrate-sqlite.test.ts` + `schema-invariants.test.ts`。

### 5.2 Portal API —— `src/portal/agent-api.ts`

1. `POST /api/v1/agents`（~L71）：INSERT 列表 + VALUES 加 `persistence_enabled`，取 `body.persistence_enabled ?? 0`。
2. `PUT /api/v1/agents/:id`（~L145）：`fields[]` 数组加 `"persistence_enabled"`（动态 SET 子句自动处理）。
3. `POST .../fork`（~L348）：fork 的 INSERT 也带上该列（保持复制语义一致）。
4. `GET /api/v1/agents/:id`（~L121）：`SELECT *` 已自动返回，无需改；确认前端能读到。

### 5.3 Portal adapter（RPC handler）—— `src/portal/adapter.ts`

新增 handler（仿 `config.getModelBinding` @ L1850）：
```js
handlers.set("config.getAgentRuntimeConfig", async (params) => {
  const db = getDb();
  const [rows] = await db.query(
    "SELECT persistence_enabled FROM agents WHERE id = ?",
    [params.agentId],
  ) as any;
  const persistenceEnabled = rows.length > 0 ? !!rows[0].persistence_enabled : false;
  return { persistenceEnabled };
});
```
- **契约**：endpoint 只读；默认 false（agent 不存在时安全降级为不持久化）。

### 5.4 Runtime 取配置 —— 新增 `src/gateway/agent-runtime-config.ts`

仿 `agent-model-binding.ts:34`：
```js
export async function resolveAgentRuntimeConfig(agentId, frontendClient) {
  try {
    const data = await frontendClient.request("config.getAgentRuntimeConfig", { agentId });
    return { persistenceEnabled: !!data?.persistenceEnabled };
  } catch (err) {
    console.error(`[agent-runtime-config] RPC error:`, err);
    return { persistenceEnabled: false }; // 取不到 → 安全降级为不持久化
  }
}
```

### 5.5 Runtime 调用点 —— `src/gateway/server.ts`

`prompt` 处理中 `getOrCreate(agentId)`（L218）前，先取配置并传入：
```js
const rtConfig = await resolveAgentRuntimeConfig(agentId, frontendClient);
const handle = await agentBoxManager.getOrCreate(agentId, { persistence: rtConfig.persistenceEnabled });
```
- ⚠️ 其它 `getOrCreate` 调用点（L320/344/355、channels、task-coordinator）：本次可保持现状（不传=用 spawner 默认）。**但需统一策略**——见 §6 风险点 R1。

### 5.6 配置管道 —— `src/gateway/agentbox/types.ts` + `manager.ts`

1. `AgentBoxConfig`（types.ts:13）加字段：`persistence?: boolean;`
2. `manager.ts` `getOrCreateK8s`（L96）：`spawn({ ...config, agentId, ... })` 已透传 `...config`，确认 `persistence` 跟着进去即可（多数情况无需改，验证 spread 覆盖）。

### 5.7 Spawner 决策 —— `src/gateway/agentbox/k8s-spawner.ts`

把三处 `this.config.persistence?.enabled` 改为「per-agent 优先，回退全局」：
```js
const persistenceEnabled = boxConfig.persistence ?? !!this.config.persistence?.enabled;
const claimName = this.config.persistence?.claimName ?? process.env.SICLAW_PERSISTENCE_CLAIM_NAME ?? "siclaw-data";
```
改动三处：
- L202-205：`ensureAgentDir` 的 gate
- L251-258：volume 定义（PVC vs emptyDir）
- L305-306：volumeMount 的 subPath

> claimName 仍来自全局（基础设施级，哪块 PVC）；只有 enabled 变成 per-agent。
> **边界情况**：若 `this.config.persistence` 整个为 undefined（Runtime 没开全局持久化，即没建 PVC），但某 agent `persistence=true` —— 此时挂载会失败（PVC 不存在）。需在 spawner 加保护：per-agent 想开但全局无 claimName 时，打警告并降级为 emptyDir（见 §6 风险点 R2）。

### 5.8 前端 —— `portal-web/src/components/AgentSettings.tsx`

照抄 `isProduction`（L165）：
1. `const [persistenceEnabled, setPersistenceEnabled] = useState(agent.persistence_enabled)`
2. L195 的 reset 逻辑加 `setPersistenceEnabled(agent.persistence_enabled)`
3. L251 保存 body 加 `persistence_enabled: persistenceEnabled`
4. `BasicTab`（L317）加一个开关 UI + 说明文案（如「开启后：会话与上下文在 pod 重启后保留，用户可回到历史会话继续。适合诊断类 agent；导购类建议关闭」）
5. agent 类型定义（L11 附近）加 `persistence_enabled: boolean`

---

## 6. 风险点与待确认项

| ID | 风险 | 处理 |
|---|---|---|
| **R1** | `getOrCreate` 有多个调用点（prompt / channels / task-coordinator）。若只改 prompt 路径，其它路径走 spawner 默认，行为不一致。 | 本次统一策略：**所有调用点都不传时回退全局默认**，prompt 路径显式传 per-agent。需在文档明确：channel/task 触发的 spawn 是否也要 per-agent？建议一并改（取数逻辑相同）。 |
| **R2** | per-agent 想开 PVC，但 Runtime 全局没建 PVC（无 claimName）→ 挂载失败 pod 起不来。 | spawner 加保护：`persistence=true` 但无可用 claimName 时 `console.warn` + 降级 emptyDir，不让 pod 挂死。 |
| **R3** | pod 复用：`spawn` 对已 Running 的同名 pod 直接复用（k8s-spawner.ts:107-110），不会因配置变更重建。改了 agent 的 PVC 开关后，旧 pod 仍用旧配置。 | 文档说明：配置变更在**下次 pod 重建后生效**（或提供手动 terminate）。本次不做热重建。 |
| **R4** | 关闭持久化后，emptyDir 在 pod 销毁时回收 JSONL；但 siclaw 无代码主动删 JSONL。若 agent 从「开」改「关」，已在 PVC 上的旧数据不会被清理。 | 本次范围不含数据清理。文档标注为已知行为，后续可加 GC。 |
| **R5** | 权限：create/PUT agent 是 admin-only（`requireAdmin`）。确认产品方配置 agent 的角色满足。 | 与 mentor 确认产品方账号权限模型。 |

---

## 7. 为后续扩展预留（不在本次实现）

同类的 runtime 级全局开关（性质相同，下沉方式一致），后续可沿本设计扩展：

| 开关 | 现状 | 扩展方式 |
|---|---|---|
| Session TTL | `SESSION_RELEASE_TTL_MS` 硬编码（session.ts:162） | agents 表加 `session_ttl_ms` 列 → 经 `boxConfig.env` 注入 pod → session.ts 读 env |
| 空闲自毁 | `IDLE_TIMEOUT_MS` 硬编码（http-server.ts:429） | 同上，注入 `SICLAW_IDLE_TIMEOUT_MS` |
| Memory | `SICLAW_MEMORY_ENABLED` 全局 env（config.ts:89） | agents 表加 `memory_enabled` 列 → 注入 `SICLAW_MEMORY_ENABLED` |

`config.getAgentRuntimeConfig` RPC 的返回对象统一承载这些字段，无需新增 RPC。届时 `agent-runtime-config.ts` 和 adapter handler 各扩字段即可。

---

## 8. 测试与验证

- **单元/集成**：`npm test`（重点 `migrate-sqlite.test.ts`、`schema-invariants.test.ts`、`k8s-spawner.test.ts`、`manager.test.ts`、`agent-api` 相关）。
- **类型**：`npx tsc --noEmit`。
- **集群手测**（远程开发机 → pod）：
  1. 建两个 agent：A `persistence_enabled=true`、B `false`。
  2. 各发 prompt → `kubectl get pod` 看 A 的 pod volume 是 `persistentVolumeClaim`、B 是 `emptyDir`（`kubectl describe pod`）。
  3. A：删 pod → 重发 prompt → 旧 session 上下文恢复；B：删 pod → 旧 session 丢失、新起 session。
  4. 前端：创建/编辑页开关能正确保存与回显。

---

## 9. 提交边界

- DB / API / adapter / Runtime / spawner / 前端 一个 PR 内闭环（端到端一个开关）。
- 不含 TTL/memory（§7 留待后续 PR）。
- 不含数据清理 GC（R4）。
- 文档：本设计文档 + 必要的代码注释；若改了持久化契约，更新 CLAUDE.md Change Impact Matrix 对应行。
