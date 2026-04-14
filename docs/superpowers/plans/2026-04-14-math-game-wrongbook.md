# 口算小达人错题集 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `study_game/division-game.html` 增加“错题集”自动收录与“错题模式”刷错题能力，数据持久化到 `localStorage`。

**Architecture:** 在现有单文件内新增 `wrongbook` 模式与一套错题集存储/抽题逻辑；在 `endGame()` 统一把本局错过的题写入错题集；错题模式从错题集加权抽题生成 `questions`，复用现有答题流程与结果页。

**Tech Stack:** 纯 HTML/CSS/Vanilla JS；`localStorage`；单页 DOM 操作。

---

## Scope Check

本计划只改动单文件 `study_game/division-game.html`（加一份 localStorage 数据），不引入新构建系统/依赖；不做后端同步。

## File Structure

**Modify:**
- `study_game/division-game.html`

**Create:**
- 无（除计划与 spec 文档）

## Task 1: 定义错题集数据模型与存储函数

**Files:**
- Modify: `study_game/division-game.html`（在 `<script>` 顶部常量区附近）

- [ ] **Step 1: 新增常量**

在现有：

```js
const TOTAL = 10;
const STORAGE_KEY = 'math_game_history';
```

下方新增：

```js
const WRONGBOOK_KEY = 'math_game_wrongbook_v1';
```

- [ ] **Step 2: 新增基础存取函数**

在 `shuffle()` 之后或 `generateQuestions()` 之前新增（保持工具函数集中）：

```js
function loadWrongBook() {
  try {
    const raw = localStorage.getItem(WRONGBOOK_KEY);
    const items = JSON.parse(raw || '[]');
    return Array.isArray(items) ? items : [];
  } catch {
    return [];
  }
}

function saveWrongBook(items) {
  localStorage.setItem(WRONGBOOK_KEY, JSON.stringify(items));
}

function wrongBookIdFromText(text, mode) {
  return `${mode}|${text}`;
}
```

- [ ] **Step 3: 运行页面做一次基本回归**

用浏览器打开 `study_game/division-game.html`，确保页面可加载、控制台无错误。

Expected: 无报错，原有模式可正常开始挑战。

- [ ] **Step 4: Commit（可选）**

如果你要提交：

```bash
git add study_game/division-game.html
git commit -m "feat: add wrongbook storage helpers"
```

## Task 2: 在 `endGame()` 基于本局 results 更新错题集

**Files:**
- Modify: `study_game/division-game.html`（`endGame()` 与相关辅助函数）

- [ ] **Step 1: 让 results 带上 mode 信息**

现有 `handleCorrect(q)` 写入：

```js
results.push({ question: q.text, correctAnswer: q.answer, wrongAttempts: currentWrongAttempts, time: qTime });
```

改为：

```js
results.push({
  question: q.text,
  correctAnswer: q.answer,
  wrongAttempts: currentWrongAttempts,
  time: qTime,
  mode: q.mode || gameMode,
  sourceMode: gameMode
});
```

并在出题生成时确保每个 `q` 都有 `mode` 字段（见 Task 3）。

- [ ] **Step 2: 新增“汇总本局错题”函数**

在 `saveHistory()` 之前新增：

```js
function upsertWrongBookFromResults(results) {
  const items = loadWrongBook();
  const byId = new Map(items.map(i => [i.id, i]));
  const now = Date.now();

  results.forEach(r => {
    if (!r || !r.question) return;
    if (!r.wrongAttempts || r.wrongAttempts <= 0) return;

    const mode = r.mode || 'mixed';
    const id = wrongBookIdFromText(r.question, mode);
    const existing = byId.get(id);

    const next = existing || {
      id,
      questionText: r.question,
      answer: r.correctAnswer,
      mode,
      sourceMode: r.sourceMode || mode,
      wrongGames: 0,
      wrongTotalAttempts: 0,
      lastWrongAt: 0,
      lastSeenAt: 0
    };

    next.answer = r.correctAnswer;
    next.mode = mode;
    next.sourceMode = r.sourceMode || next.sourceMode || mode;
    next.wrongGames += 1;
    next.wrongTotalAttempts += r.wrongAttempts;
    next.lastWrongAt = now;
    next.lastSeenAt = now;

    byId.set(id, next);
  });

  const merged = Array.from(byId.values());
  saveWrongBook(merged);
}
```

- [ ] **Step 3: 在 `endGame()` 调用更新函数**

在 `endGame()` 末尾（`saveHistory(elapsed, acc);` 前后均可），加入：

```js
upsertWrongBookFromResults(results);
```

- [ ] **Step 4: 手动验收**

开启任意普通模式一局，故意让 1 题错误至少一次后再答对完成整局。

在控制台执行：

```js
JSON.parse(localStorage.getItem('math_game_wrongbook_v1') || '[]')
```

Expected: 能看到包含该题的条目，且 `wrongGames >= 1`、`wrongTotalAttempts >= 1`。

- [ ] **Step 5: Commit（可选）**

```bash
git add study_game/division-game.html
git commit -m "feat: collect wrong questions into wrongbook"
```

## Task 3: 给所有题目对象补齐 `mode` 字段

**Files:**
- Modify: `study_game/division-game.html`（`genAddSub/genMultiply/genDivide/genMixed`）

- [ ] **Step 1: 在各生成器 push 对象时增加 mode**

示例（加减）：

```js
pool.push({ mode: 'addsub', text: `${a} + ${b}`, display: ..., answer: ... });
```

乘法：

```js
pool.push({ mode: 'multiply', text: ..., display: ..., answer: ... });
```

除法：

```js
pool.push({ mode: 'divide', text: ..., display: ..., answer: ... });
```

混合：
- `genMixed()` 直接拼接各数组后返回，mode 会随题目对象携带。

- [ ] **Step 2: 回归**

做一局混合模式，结果页正常、历史记录正常。

- [ ] **Step 3: Commit（可选）**

```bash
git add study_game/division-game.html
git commit -m "refactor: annotate questions with mode"
```

## Task 4: 新增错题模式出题逻辑

**Files:**
- Modify: `study_game/division-game.html`

- [ ] **Step 1: 新增加权抽题函数**

在 `generateQuestions()` 之前新增：

```js
function pickWrongBookQuestions(items) {
  const now = Date.now();
  const list = Array.isArray(items) ? items : [];
  if (list.length === 0) return [];

  function weightOf(i) {
    const wrongGames = i.wrongGames || 0;
    const wrongTotalAttempts = i.wrongTotalAttempts || 0;
    const lastSeenAt = i.lastSeenAt || 0;
    const days = lastSeenAt ? (now - lastSeenAt) / (1000 * 3600 * 24) : 999;
    const freshnessBoost = Math.min(10, Math.max(0, days));
    return 1 + wrongGames + 0.2 * wrongTotalAttempts + freshnessBoost;
  }

  function pickOneWeighted() {
    const weights = list.map(weightOf);
    const sum = weights.reduce((a, b) => a + b, 0);
    let r = Math.random() * sum;
    for (let idx = 0; idx < list.length; idx++) {
      r -= weights[idx];
      if (r <= 0) return list[idx];
    }
    return list[list.length - 1];
  }

  const picked = [];
  for (let i = 0; i < TOTAL; i++) {
    const it = pickOneWeighted();
    picked.push({
      mode: it.mode,
      text: it.questionText,
      display: it.questionText
        .replace(' × ', '<span class="op"> × </span>')
        .replace(' ÷ ', '<span class="op"> ÷ </span>')
        .replace(' + ', '<span class="op"> + </span>')
        .replace(' − ', '<span class="op"> − </span>'),
      answer: it.answer
    });
  }
  return picked;
}
```

注：这里用字符串替换复用现有 `display` 风格；如果未来要更严谨，可把存储结构改为存 `display`。

- [ ] **Step 2: 更新 `startGame()` 以支持 wrongbook**

把：

```js
questions = generateQuestions();
```

改为：

```js
if (gameMode === 'wrongbook') {
  const items = loadWrongBook();
  questions = pickWrongBookQuestions(items);
} else {
  questions = generateQuestions();
}
```

并在错题集为空时做处理（见 Task 5 UI 提示）；这里先允许返回空数组，后续阻止开始。

- [ ] **Step 3: 回归**

错题集非空时进入 wrongbook 能正常出题并完成一局；错题集空时不应进入游戏区（在 Task 5 完成）。

- [ ] **Step 4: Commit（可选）**

```bash
git add study_game/division-game.html
git commit -m "feat: add wrongbook question picker"
```

## Task 5: UI 增加“错题集”入口与错题列表页

**Files:**
- Modify: `study_game/division-game.html`（HTML 与 CSS 与 JS）

- [ ] **Step 1: 首页模式卡新增错题集**

在 `.mode-grid` 内新增一张卡（样式复用）：

```html
<div class="mode-card" data-mode="wrongbook" onclick="selectMode('wrongbook')">
  <div class="mode-icon">📕</div>
  <div class="mode-name">错题集</div>
  <div class="mode-desc">专练做错的题</div>
</div>
```

- [ ] **Step 2: 增加 mode label 与 diff hint**

在 `modeLabels` 增加：

```js
wrongbook: '错题集'
```

在 `diffHints` 增加（可简单写）：

```js
wrongbook: { easy: '按错题出题', medium: '按错题出题', hard: '按错题出题' }
```

- [ ] **Step 3: 新增错题列表页 HTML**

在历史记录区域后新增一个结构类似的 screen：

```html
<div class="history-screen hidden" id="wrongBookScreen">
  <h2>📕 错题集</h2>
  <div class="history-list" id="wrongBookList"></div>
  <div class="btn-group">
    <button class="home-btn" onclick="goHome()">🏠 返回首页</button>
    <button class="history-btn" onclick="startWrongBookGame()">📝 刷错题</button>
    <button class="clear-history-btn" onclick="clearWrongBook()">🗑 清空错题集</button>
  </div>
</div>
```

- [ ] **Step 4: 新增错题列表渲染与操作函数**

新增：

```js
function showWrongBook() {
  document.getElementById('modeScreen').classList.add('hidden');
  document.getElementById('historyScreen').classList.add('hidden');
  document.getElementById('resultScreen').classList.add('hidden');
  document.getElementById('gameArea').classList.add('hidden');
  document.getElementById('wrongBookScreen').classList.remove('hidden');

  const items = loadWrongBook();
  const listEl = document.getElementById('wrongBookList');
  if (items.length === 0) {
    listEl.innerHTML = '<div class="history-empty">还没有错题哦～<br>先去挑战一局吧！</div>';
    return;
  }

  const modeName = m => (modeLabels[m] || m);
  const sorted = [...items].sort((a, b) =>
    (b.wrongGames || 0) - (a.wrongGames || 0) ||
    (b.wrongTotalAttempts || 0) - (a.wrongTotalAttempts || 0) ||
    (b.lastWrongAt || 0) - (a.lastWrongAt || 0)
  );

  listEl.innerHTML = '';
  sorted.forEach(it => {
    const item = document.createElement('div');
    item.className = 'history-item';
    const last = it.lastWrongAt ? new Date(it.lastWrongAt).toLocaleString('zh-CN') : '-';
    item.innerHTML = `
      <div class="history-left">
        <div class="history-date">最近错题：${last}</div>
        <div class="history-info">
          ${it.questionText} = ${it.answer}
          <span class="history-badge ${it.mode}">${modeName(it.mode)}</span>
        </div>
      </div>
      <div class="history-right">
        <div class="history-acc low">${it.wrongGames || 0}局</div>
        <div class="history-time">错${it.wrongTotalAttempts || 0}次</div>
      </div>`;
    listEl.appendChild(item);
  });
}

function clearWrongBook() {
  if (confirm('确定要清空错题集吗？')) {
    localStorage.removeItem(WRONGBOOK_KEY);
    showWrongBook();
  }
}

function startWrongBookGame() {
  gameMode = 'wrongbook';
  startGame();
}
```

- [ ] **Step 5: 增加入口按钮**

在首页 action buttons 增加一个：

```html
<button class="history-btn" onclick="showWrongBook()">📕 错题集</button>
```

并确保 `goHome()` 同时隐藏 `wrongBookScreen`：

```js
document.getElementById('wrongBookScreen').classList.add('hidden');
```

- [ ] **Step 6: 阻止空错题集直接开始**

在 `startGame()` 的 wrongbook 分支，如果 `items.length === 0`：

- 弹窗提示：`alert('还没有错题哦～先去挑战一局吧！')`
- 跳转到 `goHome()` 或 `showWrongBook()`

- [ ] **Step 7: 手动验收**

1) 错题集为空：点击“错题集”能看到空态；点击“刷错题”会提示并不进入游戏。
2) 错题集非空：列表显示；“刷错题”进入错题模式；完成一局不影响历史记录写入。

- [ ] **Step 8: Commit（可选）**

```bash
git add study_game/division-game.html
git commit -m "feat: add wrongbook screen and mode entry"
```

## Task 6: 自检（占位扫描/一致性）

**Files:**
- Review: `docs/superpowers/specs/2026-04-14-math-game-wrongbook-design.md`
- Review: `docs/superpowers/plans/2026-04-14-math-game-wrongbook.md`

- [ ] **Step 1: 自检一致性**

- `WRONGBOOK_KEY` 名称一致
- `wrongbook` mode 字符串一致
- `results` 中字段名一致（`question/correctAnswer/wrongAttempts/mode/sourceMode`）

- [ ] **Step 2: 回归冒烟**

打开页面：
- 普通模式做一局
- 进入记录页
- 进入错题集页
- 刷错题一局

Expected: 无报错，所有页面切换正常。

