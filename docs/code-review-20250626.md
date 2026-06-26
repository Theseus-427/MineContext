# MineContext 产品逻辑 Code Review

> 日期：2026-06-26  
> 范围：Todo 生命周期、每日摘要生成、热力图、前后端数据流

---

## CRITICAL（功能完全不可用或数据丢失）

### 1. 热力图数据范围硬编码为 2025 年

**文件：** `frontend/src/renderer/src/pages/home/components/heatmap/heatmap.tsx:79`

```typescript
const res = await window.dbAPI.getHeatmapData(
  dayjs('2025-01-01').valueOf(),
  dayjs('2025-12-31').valueOf()
)
```

无论用户选择哪个年份，数据始终查 2025 年。同时第 106-112 行 `yearList` 计算在 2026 年得出空数组（`diff = 2025 - 2026 = -1`，`Array.from({ length: 0 })` = `[]`），`selectedYear` 变为 `undefined`，年份下拉完全瘫痪。

**影响：** 热力图永远空白，或仅显示 2025 年的数据。2026 年及以后的数据不可见。

---

### 2. 每日/每周摘要查询字符串不匹配 — 永远不会加载

**文件：**
- `frontend/src/renderer/src/hooks/use-home-info.ts:113,116`
- `opencontext/models/enums.py:103-108`

```typescript
// 前端传的是（use-home-info.ts）:
getVaultsByDocumentType('daily')   // ❌
getVaultsByDocumentType('weekly')  // ❌

// 数据库存的是（generation_report.py → enums.py）:
VaultType.DAILY_REPORT = "DailyReport"    // ✅
VaultType.WEEKLY_REPORT = "WeeklyReport"  // ✅
```

`'daily'` ≠ `'DailyReport'`，`'weekly'` ≠ `'WeeklyReport'`。SQL WHERE 条件匹配不到任何行。

**影响：** 主页上的 Daily Summary / Weekly Summary 区域永远为空，即使报告已成功生成并存入数据库。

---

### 3. `takeScreenshot` IPC 参数顺序颠倒

**文件：**
- `frontend/src/preload/index.ts:89-90`
- `frontend/src/main/ipc.ts:497-499`

```
Preload 发送:     (groupIntervalTime, sourceId)
IPC 接收为:       (sourceId=groupIntervalTime, batchTime=sourceId)
调用 service:     screenshotService.takeScreenshot(sourceId, batchTime)
                  → 时间戳变成 sourceId，真正的 sourceId 变成 batchTime
```

**影响：** 截图功能参数错乱，取图逻辑不可预期。

---

### 4. 首次启动永远不生成每日报告（冷启动 bug）

**文件：** `opencontext/managers/consumption_manager.py:186-222`

```python
# _get_last_report_time() 在无历史报告时返回 datetime.now()
last_report_time = datetime.now()           # line 189
self._last_report_date = last_report_time.date()  # = today

# check_and_generate_daily_report():
if self._last_report_date != today:  # today == today → False, 永远不生成
```

首次安装/清空数据后，`_get_last_report_time()` 返回当前时间（因为没有历史报告），导致 `_last_report_date` = 今天，而生成检查要求 `_last_report_date != today`。除非用户通过 Debug API 手动触发一次，否则系统永远不会自动生成第一份每日报告。

**影响：** 新用户永远看不到自动生成的每日摘要。

---

### 5. ToDoCard 点击热力图日期切换无效

**文件：** `frontend/src/renderer/src/pages/home/components/to-do-card/index.tsx:345`

```typescript
if (selectedDays) {
  fetchTasks(dayjs(selectedDays))  // 调用了但返回值被丢弃
}
```

`fetchTasks` 是 `useRequest.runAsync`，返回 Promise。但它的结果没有赋值到 `tasks` 状态（`useHomeInfo` 中 `tasks` 是独立的 `useState`）。

**影响：** 点击热力图上任何日期，Todo 列表不会更新，始终显示当天（初始加载时）的任务。

---

## HIGH（核心功能缺陷）

### 6. Todo 读写使用两套不同的数据库连接

**文件：** `frontend/src/main/services/DatabaseService.ts:234-327`

| 操作 | 使用的连接 |
|------|-----------|
| `getTasks()` | `DB.getInstance(DB.dbName)` |
| `addTask()` | `DB.getInstance(DB.dbName)` |
| `toggleTaskStatus()` | `this.db` |
| `updateTask()` | `this.db` |
| `deleteTask()` | `this.db` |

两个独立的 `better-sqlite3` 连接实例。虽然指向同一个文件（WAL 模式下不会直接报错），但独立的事务状态和语句缓存可能导致读到过期数据。此外，Python 后端同时也在写同一个 SQLite 文件（第三个连接），加重了不一致风险。

**影响：** 可能存在事务隔离问题，尤其在前后端同时写入时。

---

### 7. 历史 Todo 被禁用 — LLM 无法感知已有任务

**文件：** `opencontext/context_consumption/generation/smart_todo_manager.py:67-68`

```python
# historical_todos = self._get_historical_todos()  # 被注释掉
historical_todos = []  # 永远为空
```

`_get_historical_todos()` 方法（第 170-178 行）完全可用，会从 SQLite 查询已有的 todo 数据。但调用被注释掉，LLM prompt 中的 `historical_todos` 变量永远是空数组。LLM 无法知道自己之前生成过什么任务。

**影响：** 语义相近但措辞不同的重复任务无法被 LLM 识别为重复（仅靠后续的向量相似度 0.85 阈值去重，效果有限）。

---

### 8. `TODO_GENERATED` 事件从未发布

**文件：**
- `opencontext/managers/event_manager.py:29` — enum 已定义
- `opencontext/context_consumption/generation/smart_todo_manager.py:123-128` — 返回值被丢弃
- `opencontext/managers/consumption_manager.py:342-344` — 调用后未发布事件

`EventType.TODO_GENERATED` 定义了但全代码库没有任何 `publish_event(TODO_GENERATED, ...)` 调用。而其他三个生成器（Tips、Activity、Daily Summary）都正确发布了事件。

**影响：** 前端事件流永远不会收到 Todo 生成通知，无法实时更新任务列表。

---

### 9. Debug 端点的自定义 prompt 接口全部不可用

**文件：** `opencontext/server/routes/debug.py:539,574,668`

```python
# 实际方法签名都是 (self, start_time: int, end_time: int)，但调用时传了不存在的参数:
generate_smart_tip(lookback_minutes=lookback_minutes or 15)          # TypeError
generate_todo_tasks(lookback_minutes=lookback_minutes or 30)          # TypeError
generate_realtime_activity_summary(minutes=lookback_minutes or 15)    # TypeError
```

**影响：** 这些 Debug API 端点运行时直接 `TypeError`，完全不可用。

---

### 10. Debug 手动生成报告端点丢弃用户提供的参数

**文件：** `opencontext/server/routes/debug.py:160-165`

```python
if start_time is None or end_time is None:
    start_time = int(time.time()) - 24 * 3600   # 无条件同时覆盖两个值
    end_time = int(time.time())
```

用户传了 `start_time` 但没传 `end_time` 时，已提供的 `start_time` 也会被默认值覆盖。

**影响：** 手动调试时无法只指定起始时间。

---

## MEDIUM（影响数据正确性或用户体验）

### 11. `WEEKLY_SUMMARY_GENERATED` 是"僵尸功能"

前端有完整的 UI 展示代码（`proactive-feed-card/index.tsx` 有专门的星星图标和 "Weekly Summary" 标题），后端 enum 也定义了 `WEEKLY_SUMMARY_GENERATED` 和 `VaultType.WEEKLY_REPORT`，但**没有任何代码生成周报**——没有定时器、没有生成逻辑、没有事件发布。仅有 `DAILY_SUMMARY_GENERATED` 是实际工作的。

---

### 12. 热力图 context 数量被无故除以 100

**文件：** `frontend/src/renderer/src/pages/home/components/heatmap/heatmap.tsx:88`

```typescript
count: conversations + vaults + context / 100 + todos,
```

context 的权重被稀释为原始的 1/100，无注释解释原因。此外第 98 行存在 copy-paste 错误：

```typescript
set(acc, 'origin', get(acc, 'totalContext', 0) + context)
// 读的是 totalContext，写的却是 origin —— 应该是 get(acc, 'origin', 0) + context
```

---

### 13. `HeatmapService` 中 API 请求 URL 不一致

**文件：** `frontend/src/main/services/HeatmapService.ts:240 vs 270`

- `getMonitoringData()` 使用相对路径 `/api/monitoring/data-stats-range`
- `getMonitoringTrendData()` 使用绝对路径 `http://127.0.0.1:${port}/api/monitoring/data-stats-range`

如果 axios 实例没有配置 `baseURL`，相对路径会解析到 Electron 渲染进程的 origin 而失败。

---

### 14. 开发模式下数据库使用相对路径

**文件：** `frontend/src/main/services/DatabaseService.ts:26-31`

```typescript
!app.isPackaged && is.dev ? 'backend' : app.getPath('userData')
```

开发模式下数据库路径是 `backend/persist/sqlite/app.db`（相对路径），随 CWD 变化。换个启动方式就可能"丢失"所有历史数据（实际是创建了新的数据库文件）。

---

### 15. Dayjs 对象被静默修改

**文件：** `frontend/src/renderer/src/hooks/use-home-info.ts:102`

```typescript
const endOfDay = day.add(1, 'day').startOf('day').toISOString()
```

Dayjs 的 `.add()` 会**修改原对象**（与 moment.js 不同）。调用者的 `day` 参数会被静默改为下一天。应使用 `dayjs(day).add(1, 'day')`（先克隆）。

---

### 16. Preload 类型声明缺少 6 个 dbAPI 方法

**文件：** `frontend/src/preload/index.d.ts`

缺少声明的方法：`addTask`、`updateTask`、`deleteTask`、`getVaultsByDocumentType`、`getLatestActivity`、`getHeatmapData`。Renderer 中调用这些方法完全没有 TypeScript 类型检查和自动补全。

---

### 17. 标记任务为未完成时错误设置了 `end_time`

**文件：** `frontend/src/main/services/ToDoService.ts:156-158`

```typescript
end_time: status === 1 && !endTime
  ? dayjs().toISOString()
  : endTime
    ? dayjs(endTime).toISOString()
    : null
```

当 `status=0`（标记未完成）且 `endTime` 有值时，中间分支触发，给未完成的任务设了一个 `end_time`。未完成任务不应有 `end_time`。

---

### 18. `selectedDays` 为 null 时热力图显示 "Invalid Date" 且导航崩溃

**文件：** `frontend/src/renderer/src/pages/home/components/heatmap/heatmap.tsx:138,147,206`

当 `selectedDays` 为 null 时（初始状态或点击"Back"返回年份视图后），`dayjs(null).format(...)` 输出 `"Invalid Date"` 字符串显示在 UI 上，日期导航按钮（左右箭头）也会因为 `dayjs(null).subtract(1, 'day')` 产生无效日期而崩溃。

---

## 产品设计层面问题（非代码 Bug，但影响用户体验）

### A. Todo 无跨天继承机制

首页只按 `start_time` 筛选当天任务（`DatabaseService.ts:235`）：
```sql
SELECT * FROM todo WHERE start_time >= ? AND start_time < ?
```

昨天未完成的 todo 保留在数据库中，但首页不显示。没有"逾期"视图、没有跨天继承、没有"查看所有未完成"的入口。热力图也只标注有**已完成** todo 的日期，所以如果昨天一个都没勾完，那个日期甚至无法点击跳转。

### B. 每日报告在早上 08:00 生成，非晚上

报告覆盖过去 24 小时的内容，在配置的时间点（默认 08:00）生成。如果 08:00 时 app 没有运行，报告就不会被触发。结合 Bug #4（冷启动），即使之后启动 app 也不会补生成。

---

## 症状对照表

| 你遇到的问题 | 直接原因 | 对应 Bug |
|-------------|---------|---------|
| 热力图空白 + 显示 2025 年 | 数据范围硬编码 2025，年份计算在 2026 年崩溃 | #1 |
| 首页 Todo 被"清空" | 前端只按 `start_time` 查当天数据，昨天的看不到 | 设计问题 A |
| 昨晚没有 summary | 每日报告早上 08:00 生成（非晚上）+ 首次使用有冷启动 bug | #4 |
| Daily Summary 框永远为空 | `'daily'` ≠ `'DailyReport'`，查询字符串不匹配 | #2 |
| 点热力图日期 Todo 不切换 | `fetchTasks` 返回值被丢弃 | #5 |
