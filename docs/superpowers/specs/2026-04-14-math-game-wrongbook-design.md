---
title: 口算小达人：错题集与错题模式（设计）
date: 2026-04-14
status: approved-by-user
---

## 背景与目标

当前 `study_game/division-game.html` 支持加减/乘/除/混合四种模式、难度选择、10 题一局的练习、以及局级历史记录（`math_game_history`）。

本设计新增两项能力：

- **错题集（Wrong Book）**：用户在任意普通模式中，一局里某题只要出现过错误尝试，就自动收录到错题集中，并长期保留统计（不自动移出）。
- **错题模式（Wrong Book Mode）**：从错题集中抽题出题，让用户专门刷错题；错题列表可查看、清空（可选：单题删除）。

## 非目标（YAGNI）

- 不做跨设备同步（只用浏览器 `localStorage`）。
- 不引入复杂账号体系/后端。
- 不做“自动掌握移出”或“连续正确出库”（用户明确选择长期保留统计）。

## 术语与约束

- **一局**：固定 10 题（常量 `TOTAL=10`）。
- **错题收录规则（已确认）**：每局内，只要某题的 `wrongAttempts > 0`，则该题计入错题集一次（`wrongGames += 1`），并累计该题本局的错误次数。
- **题目唯一性**：以“规范化题目文本”作为唯一 key。默认规则：不同顺序视为不同题（例如 `3 × 4` 与 `4 × 3` 是两道题），保持与当前出题文本一致，避免意外合并。
- **混合模式处理（已确认）**：混合模式出现的题也进入错题集，并在列表中标注来源模式。

## 用户体验（UX）

### 首页入口

- 在模式卡片区新增一个模式卡：`📕 错题集`（`data-mode="wrongbook"`）。
- 保留难度按钮：在错题模式中依然可选难度，但默认只用于 UI 统一（错题题目本身来自错题集；难度可用于过滤错题或忽略——本次实现默认 **忽略难度过滤**，避免把错题“筛没了”）。
- 在首页按钮区可增加一个 `📕 错题集` 按钮（与 `📊 记录`并列），用于直接查看错题列表（可与错题模式共用同一入口）。

### 错题列表页

- 展示字段：
  - 题目（文本）
  - 正确答案
  - 来源模式（addsub/multiply/divide/mixed）
  - `错过局数`（wrongGames）
  - `累计错次数`（wrongTotalAttempts）
  - 最近错题时间（lastWrongAt）
- 操作：
  - 清空错题集
  - 关闭/返回首页
  -（可选）单题删除

### 错题模式出题

- 当错题集为空：提示“还没有错题，先去挑战一局吧”并提供返回首页按钮。
- 当错题集非空：每局从错题集中抽取最多 10 题（不足则少于 10 题，或允许重复出题——本次实现默认 **不足则重复抽题**，保证仍然是 10 题一局）。
- 抽题策略（默认）：
  - **加权随机**：更常错、且更久没出现的题优先。
  - 近似权重：\( w = 1 + wrongGames + 0.2 \cdot wrongTotalAttempts + freshnessBoost \)
  - `freshnessBoost`：按 `now - lastSeenAt` 增加（越久没做越高）。

## 数据结构与存储

### 错题集存储

- `localStorage` key：`math_game_wrongbook_v1`
- value：JSON 数组（方便渲染与排序），每项为：

```json
{
  "id": "divide|12 ÷ 3",
  "questionText": "12 ÷ 3",
  "answer": 4,
  "mode": "divide",
  "wrongGames": 3,
  "wrongTotalAttempts": 7,
  "lastWrongAt": 1713090000000,
  "lastSeenAt": 1713091000000
}
```

说明：

- `id`：`<mode>|<questionText>`（混合模式题的 `mode` 字段保留为其实际运算类型，而不是 `mixed`，以便分类；同时可额外存 `sourceMode` 用于标注“来自混合模式”）。
- 时间使用毫秒时间戳（`Date.now()`）。

### 局级历史与错题集的关系

- 局级历史仍然写入 `math_game_history`（现有逻辑不变）。
- 错题集增量更新发生在 `endGame()` 之后、`saveHistory()` 前后均可；推荐在 `endGame()` 末尾统一处理一次，避免 `handleWrong()` 多次写入。

## 代码改动点（高层）

主要修改单文件：`study_game/division-game.html`

- **新增模式**：`wrongbook`
- **新增存储 key 与读写函数**：
  - `WRONGBOOK_KEY`
  - `loadWrongBook() / saveWrongBook(items)`
  - `upsertWrongBookFromResults(results, meta)`
  - `pickWrongBookQuestions(items)`
- **更新 `startGame()`**：
  - 当 `gameMode === 'wrongbook'` 时，`questions = pickWrongBookQuestions(loadWrongBook())`
- **更新 `endGame()`**：
  - 基于 `results` 中 `wrongAttempts > 0` 的题，更新错题集统计
- **新增 UI**：
  - 模式卡新增错题集入口
  - 新增错题列表页（结构可复用历史页样式）

## 测试与验收（手动）

- 做一局任意普通模式：故意错 1-2 题（同一题错多次），完成后进入错题集列表能看到题目被收录：
  - `wrongGames` 增加 1
  - `wrongTotalAttempts` 增加本局该题错误次数
- 同一题在另一局再次做错：统计继续累加，`lastWrongAt` 更新。
- 进入错题模式：
  - 错题集为空时有提示
  - 错题集不为空时，题目来自错题集并可完成一局
- 清空错题集：列表为空，错题模式提示为空。

