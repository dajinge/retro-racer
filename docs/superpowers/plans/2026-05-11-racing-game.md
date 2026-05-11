# 2D 怀旧赛车游戏 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个单文件 HTML5 Canvas 2D 俯视角怀旧赛车游戏，包含 3 张地图、漂移物理、赛道边界碰撞。

**Architecture:** 单个 `index.html` 文件，内嵌 CSS + JS。游戏循环由 `requestAnimationFrame` 驱动，每帧依次处理输入→物理更新→边界检测→渲染。赛道由路点数组定义中心线，用 `ctx.lineWidth` 绘制路面。

**Tech Stack:** HTML5 Canvas + 原生 JavaScript，零外部依赖

---

## 文件结构

```
racing/
└── index.html          # 全部代码所在（~600 行）
```

---

### Task 1: 创建 HTML 骨架 + Canvas 画布 + 全局常量

**Files:**
- Create: `index.html`

- [ ] **Step 1: 写入 HTML 骨架**

创建 `index.html`，包含完整 HTML 结构、CSS 样式和 Canvas 初始化：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>怀旧赛车</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #1a1a2e;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    font-family: 'Courier New', monospace;
    color: #ccc;
  }
  #toolbar {
    padding: 10px 0;
    display: flex;
    gap: 8px;
    align-items: center;
  }
  #toolbar button {
    padding: 6px 14px;
    font-family: inherit;
    font-size: 14px;
    background: #333;
    color: #ccc;
    border: 1px solid #555;
    cursor: pointer;
    border-radius: 3px;
  }
  #toolbar button.active {
    background: #556;
    border-color: #88a;
    color: #fff;
  }
  #map-name {
    margin-left: 10px;
    font-size: 14px;
    color: #aaa;
  }
  #hint {
    padding: 6px 0 10px;
    font-size: 12px;
    color: #666;
  }
  canvas {
    border: 1px solid #333;
    display: block;
  }
</style>
</head>
<body>

<div id="toolbar">
  <button id="btn-map1" class="active">地图1 · 新手村</button>
  <button id="btn-map2">地图2 · 标准赛道</button>
  <button id="btn-map3">地图3 · 进阶挑战</button>
</div>

<canvas id="game"></canvas>

<div id="hint">WASD / 方向键 操控 &nbsp;|&nbsp; 按住方向键过弯自动漂移</div>

<script>
// ==== 常量 ====
const ACCEL = 400;           // 加速 (px/s²)
const REV_ACCEL = 300;       // 后退加速
const MAX_SPEED = 350;       // 最高速度 (px/s)
const MAX_REV_SPEED = 120;   // 最高倒车速度
const FRICTION = 0.97;       // 惯性摩擦系数 (per frame @60fps → ~0.98实际用dt计算)
const TURN_RATE = 3.5;       // 基础转向率 (rad/s)
const DRIFT_FORCE = 60;      // 漂移侧向力
const DRIFT_SPEED_LOSS = 0.995;  // 漂移速度衰减
const DRIFT_RECOVERY = 0.90;     // 松键回正速率
const BOUNDARY_BOUNCE = 0.4;     // 边界回弹系数

// ==== 全局状态 ====
let canvas, ctx;
let currentTrack = 0;
let car = { x: 0, y: 0, angle: 0, speed: 0, driftOffset: 0 };
let input = { forward: false, backward: false, left: false, right: false };
let tracks = [];

// ==== 初始化 ====
function init() {
  canvas = document.getElementById('game');
  ctx = canvas.getContext('2d');
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);
  defineTracks();
  resetCar();
  bindInput();
  bindUI();
  requestAnimationFrame(gameLoop);
}

function resizeCanvas() {
  const t = tracks[currentTrack];
  if (t) {
    canvas.width = t.canvasW;
    canvas.height = t.canvasH;
  } else {
    canvas.width = 800;
    canvas.height = 600;
  }
}
</script>
</body>
</html>
```

- [ ] **Step 2: 验证文件可打开**

Run: `ls -la index.html`
Expected: 文件存在

---

### Task 2: 定义 3 张地图的赛道数据

**Files:**
- Modify: `index.html` — 在 `defineTracks()` 函数体位置插入赛道定义

- [ ] **Step 1: 替换 `defineTracks()` 占位为完整赛道数据**

找到 `function defineTracks() {` 行，替换为：

```javascript
function defineTracks() {
  // ===== 地图 1: 新手村 — 椭圆环形 =====
  const cx1 = 400, cy1 = 300, rx1 = 260, ry1 = 190;
  const ovalWps = [];
  for (let i = 0; i < 24; i++) {
    const a = (i / 24) * Math.PI * 2;
    ovalWps.push({ x: cx1 + Math.cos(a) * rx1, y: cy1 + Math.sin(a) * ry1 });
  }

  tracks[0] = {
    name: '地图1 · 新手村 — 椭圆环形',
    canvasW: 800, canvasH: 600,
    waypoints: ovalWps,
    width: 90,
    carStart: { x: cx1 + rx1 * 0.6, y: cy1, angle: Math.PI * 0.5 }
  };

  // ===== 地图 2: 标准赛道 — 不规则环形 =====
  tracks[1] = {
    name: '地图2 · 标准赛道 — 多弯环形',
    canvasW: 900, canvasH: 700,
    waypoints: [
      {x:700,y:200}, {x:750,y:220}, {x:770,y:280}, {x:760,y:340},  // 右长直道
      {x:740,y:390}, {x:700,y:430}, {x:640,y:460},                 // 下坡弯
      {x:570,y:470}, {x:500,y:460}, {x:440,y:430},                 // U 弯底部
      {x:390,y:390}, {x:360,y:350}, {x:340,y:310},                 // 左转上坡
      {x:310,y:260}, {x:270,y:220}, {x:220,y:200},                 // 左上直道
      {x:170,y:190}, {x:140,y:210}, {x:130,y:260}, {x:150,y:320},  // 左上弯
      {x:180,y:370}, {x:220,y:410}, {x:270,y:430},                 // 左 U 弯
      {x:330,y:440}, {x:390,y:440}, {x:440,y:420},                 // U 弯回
      {x:500,y:390}, {x:540,y:340}, {x:550,y:280},                 // 右上弯
      {x:570,y:220}, {x:600,y:190}, {x:650,y:190}                  // 回直道
    ],
    width: 70,
    carStart: { x: 680, y: 200, angle: Math.PI * 0.5 }
  };

  // ===== 地图 3: 进阶挑战 — S 弯 + 发卡弯 =====
  tracks[2] = {
    name: '地图3 · 进阶挑战 — S弯+发卡弯',
    canvasW: 900, canvasH: 600,
    waypoints: [
      {x:650,y:100}, {x:720,y:110}, {x:770,y:150}, {x:790,y:210},  // 右上直道
      {x:780,y:270},                                                // 右弯
      {x:750,y:310}, {x:700,y:330}, {x:640,y:320},                 // S 弯第一段
      {x:580,y:290}, {x:530,y:250}, {x:500,y:200},                 // S 弯第二段
      {x:490,y:150},                                                // 急弯入口
      {x:450,y:120}, {x:390,y:110}, {x:330,y:120},                 // 发卡弯 1 外侧
      {x:280,y:150}, {x:250,y:200}, {x:250,y:260},                 // 发卡弯 1 折返
      {x:270,y:320}, {x:310,y:370}, {x:350,y:400},                 // S 弯
      {x:420,y:410}, {x:480,y:390}, {x:520,y:350},                 // S 弯
      {x:540,y:300}, {x:540,y:240},                                 // 短直道
      {x:520,y:190}, {x:470,y:160}, {x:420,y:150},                 // 发卡弯 2
      {x:370,y:160}, {x:340,y:190}, {x:330,y:240},                 // 发卡弯 2 折返
      {x:350,y:290}, {x:390,y:330}, {x:430,y:350},                 // 出弯
      {x:500,y:360}, {x:560,y:340}, {x:600,y:300},                 // 回直道
      {x:630,y:240}, {x:640,y:170}, {x:630,y:120}                  // 收尾
    ],
    width: 55,
    carStart: { x: 700, y: 130, angle: Math.PI * 0.4 }
  };
}
```

- [ ] **Step 2: 实现 `resetCar()` 函数**

在 `resetCar()` 位置插入：

```javascript
function resetCar() {
  const t = tracks[currentTrack];
  car.x = t.carStart.x;
  car.y = t.carStart.y;
  car.angle = t.carStart.angle;
  car.speed = 0;
  car.driftOffset = 0;
}
```

---

### Task 3: 实现键盘输入处理

**Files:**
- Modify: `index.html` — 替换 `bindInput()` 占位

- [ ] **Step 1: 实现 `bindInput()` 函数**

找到 `function bindInput()` 行，替换为：

```javascript
function bindInput() {
  const keyMap = {
    'KeyW': 'forward',     'ArrowUp': 'forward',
    'KeyS': 'backward',    'ArrowDown': 'backward',
    'KeyA': 'left',        'ArrowLeft': 'left',
    'KeyD': 'right',       'ArrowRight': 'right'
  };

  window.addEventListener('keydown', (e) => {
    const action = keyMap[e.code];
    if (action) {
      e.preventDefault();
      input[action] = true;
    }
  });

  window.addEventListener('keyup', (e) => {
    const action = keyMap[e.code];
    if (action) {
      e.preventDefault();
      input[action] = false;
    }
  });

  // 窗口失焦时重置所有按键，防止卡输入
  window.addEventListener('blur', () => {
    input.forward = false;
    input.backward = false;
    input.left = false;
    input.right = false;
  });
}
```

---

### Task 4: 实现物理引擎（加速、惯性、转向、漂移）

**Files:**
- Modify: `index.html` — 在 `<script>` 中 `constants` 之后、`init()` 之前插入物理函数

- [ ] **Step 1: 在全局常量之后插入物理更新函数**

在 `// ==== 初始化 ====` 行之前插入：

```javascript
// ==== 物理 ====
function updatePhysics(dt) {
  // 限制 dt 防止切后台后帧跳跃
  const t = Math.min(dt, 0.05);

  // --- 加减速 ---
  if (input.forward) {
    car.speed += ACCEL * t;
  } else if (input.backward) {
    car.speed -= REV_ACCEL * t;
  } else {
    // 惯性摩擦：每帧按比例衰减
    // 使用 pow 保证 dt 独立性
    car.speed *= Math.pow(FRICTION, t * 60);
    // 低速时直接归零，避免无限趋近
    if (Math.abs(car.speed) < 1) car.speed = 0;
  }
  car.speed = Math.max(-MAX_REV_SPEED, Math.min(MAX_SPEED, car.speed));

  // --- 转向 + 漂移 ---
  const speedRatio = Math.abs(car.speed) / MAX_SPEED;
  const turning = input.left || input.right;

  if (Math.abs(car.speed) > 1 && turning) {
    // 转向角度
    const dir = input.left ? -1 : 1;
    const turnAmount = TURN_RATE * speedRatio * t;
    car.angle += turnAmount * dir;

    // 漂移侧向偏移：柔和施加侧向力
    const targetDrift = dir * DRIFT_FORCE * speedRatio;
    car.driftOffset += (targetDrift - car.driftOffset) * 4 * t;

    // 漂移时轻微减速
    car.speed *= Math.pow(DRIFT_SPEED_LOSS, t * 60);
  } else {
    // 松开方向键：漂移量衰减回正
    car.driftOffset *= Math.pow(DRIFT_RECOVERY, t * 60);
    if (Math.abs(car.driftOffset) < 0.5) car.driftOffset = 0;
  }

  // --- 位移 ---
  const headingX = Math.cos(car.angle);
  const headingY = Math.sin(car.angle);
  const lateralX = Math.cos(car.angle + Math.PI / 2);
  const lateralY = Math.sin(car.angle + Math.PI / 2);

  car.x += headingX * car.speed * t;
  car.y += headingY * car.speed * t;
  // 漂移侧向位移
  car.x += lateralX * car.driftOffset * t;
  car.y += lateralY * car.driftOffset * t;
}
```

---

### Task 5: 实现赛道边界碰撞检测

**Files:**
- Modify: `index.html` — 在物理函数之后插入碰撞函数

- [ ] **Step 1: 插入碰撞检测函数**

在 `updatePhysics()` 函数之后插入：

```javascript
// ==== 线段最近点 ====
function closestPointOnSegment(px, py, a, b) {
  const dx = b.x - a.x;
  const dy = b.y - a.y;
  const lenSq = dx * dx + dy * dy;
  if (lenSq === 0) return { x: a.x, y: a.y };
  let t = ((px - a.x) * dx + (py - a.y) * dy) / lenSq;
  t = Math.max(0, Math.min(1, t));
  return { x: a.x + t * dx, y: a.y + t * dy };
}

// ==== 边界约束 ====
function constrainToTrack() {
  const t = tracks[currentTrack];
  const wps = t.waypoints;
  const halfW = t.width / 2;

  let minDist = Infinity;
  let nearX = car.x, nearY = car.y;

  for (let i = 0; i < wps.length; i++) {
    const j = (i + 1) % wps.length;
    const cp = closestPointOnSegment(car.x, car.y, wps[i], wps[j]);
    const dx = car.x - cp.x;
    const dy = car.y - cp.y;
    const dist = Math.sqrt(dx * dx + dy * dy);
    if (dist < minDist) {
      minDist = dist;
      nearX = cp.x;
      nearY = cp.y;
    }
  }

  if (minDist > halfW) {
    // 投影回边界内
    const dx = car.x - nearX;
    const dy = car.y - nearY;
    const dist = Math.sqrt(dx * dx + dy * dy) || 1;
    car.x = nearX + (dx / dist) * halfW;
    car.y = nearY + (dy / dist) * halfW;
    // 速度反向衰减（轻微回弹）
    car.speed *= -BOUNDARY_BOUNCE;
    car.driftOffset *= -0.5;
  }
}
```

---

### Task 6: 实现渲染函数（草地、赛道、赛车）

**Files:**
- Modify: `index.html` — 在碰撞函数之后插入渲染函数

- [ ] **Step 1: 插入渲染函数**

在 `constrainToTrack()` 之后插入：

```javascript
// ==== 预生成草地噪点（只生成一次）====
let grassDots = [];
function generateGrassDots() {
  grassDots = [];
  for (let ti = 0; ti < 3; ti++) {
    const t = tracks[ti];
    const dots = [];
    for (let i = 0; i < 150; i++) {
      dots.push({
        x: Math.random() * t.canvasW,
        y: Math.random() * t.canvasH
      });
    }
    grassDots.push(dots);
  }
}

// ==== 渲染 ====
function drawGrass() {
  const t = tracks[currentTrack];
  ctx.fillStyle = '#3a5a3a';
  ctx.fillRect(0, 0, t.canvasW, t.canvasH);
  // 预生成的噪点纹理（不随帧变化）
  ctx.fillStyle = 'rgba(50,90,50,0.3)';
  const dots = grassDots[currentTrack];
  for (const d of dots) {
    ctx.fillRect(d.x, d.y, 3, 3);
  }
}

function drawTrack() {
  const t = tracks[currentTrack];
  const wps = t.waypoints;

  // 构建路径（复用以减少重复代码）
  function tracePath() {
    ctx.beginPath();
    ctx.moveTo(wps[0].x, wps[0].y);
    for (let i = 1; i < wps.length; i++) {
      ctx.lineTo(wps[i].x, wps[i].y);
    }
    ctx.closePath();
  }
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';

  // 1. 路肩（最底层，最宽）
  tracePath();
  ctx.strokeStyle = '#cccccc';
  ctx.lineWidth = t.width + 8;
  ctx.stroke();

  // 2. 红白路肩条纹
  tracePath();
  ctx.strokeStyle = '#e63946';
  ctx.lineWidth = t.width + 4;
  ctx.setLineDash([20, 20]);
  ctx.stroke();
  ctx.setLineDash([]);

  // 3. 沥青路面（中间层）
  tracePath();
  ctx.strokeStyle = '#4a4a4a';
  ctx.lineWidth = t.width;
  ctx.stroke();

  // 4. 路面中线虚线（最顶层）
  tracePath();
  ctx.strokeStyle = '#f0f0f0';
  ctx.lineWidth = 2;
  ctx.setLineDash([14, 18]);
  ctx.stroke();
  ctx.setLineDash([]);
}

function drawCar() {
  ctx.save();
  ctx.translate(car.x, car.y);
  ctx.rotate(car.angle);

  // 车身阴影
  ctx.fillStyle = 'rgba(0,0,0,0.3)';
  ctx.fillRect(-12, -6, 28, 16);

  // 车身（矩形）
  ctx.fillStyle = '#e63946';
  ctx.fillRect(-14, -8, 28, 16);
  // 车窗
  ctx.fillStyle = '#1d3557';
  ctx.fillRect(-6, -5, 10, 10);
  // 车头三角指示
  ctx.fillStyle = '#f1fa8c';
  ctx.beginPath();
  ctx.moveTo(14, 0);
  ctx.lineTo(8, -5);
  ctx.lineTo(8, 5);
  ctx.closePath();
  ctx.fill();

  ctx.restore();
}

function render() {
  drawGrass();
  drawTrack();
  drawCar();
}
```

---

### Task 7: 实现游戏主循环 + 地图切换 UI

**Files:**
- Modify: `index.html` — 插入游戏循环和 UI 绑定代码

- [ ] **Step 1: 插入游戏循环**

在 `render()` 之后插入：

```javascript
// ==== 游戏循环 ====
let lastTime = 0;
function gameLoop(timestamp) {
  if (lastTime === 0) lastTime = timestamp;
  const dt = (timestamp - lastTime) / 1000;
  lastTime = timestamp;

  updatePhysics(dt);
  constrainToTrack();
  render();

  requestAnimationFrame(gameLoop);
}
```

- [ ] **Step 2: 插入 UI 绑定函数**

在 `init()` 函数之后（注意 `init()` 中已调用 `bindUI()`），插入 `bindUI()` 实现：

```javascript
function bindUI() {
  const buttons = [
    document.getElementById('btn-map1'),
    document.getElementById('btn-map2'),
    document.getElementById('btn-map3')
  ];

  buttons.forEach((btn, i) => {
    btn.addEventListener('click', () => {
      currentTrack = i;
      buttons.forEach(b => b.classList.remove('active'));
      btn.classList.add('active');
      resizeCanvas();
      resetCar();
    });
  });
}
```

- [ ] **Step 3: 插入 `resizeCanvas` 和 `resetCar` 实现（如果尚未完整）**

确认 `init()` 函数完整：

```javascript
function init() {
  canvas = document.getElementById('game');
  ctx = canvas.getContext('2d');
  defineTracks();
  generateGrassDots();
  currentTrack = 0;
  resizeCanvas();
  resetCar();
  bindInput();
  bindUI();
  window.addEventListener('resize', () => resizeCanvas());
  lastTime = 0;
  requestAnimationFrame(gameLoop);
}

// 页面加载后启动
window.addEventListener('DOMContentLoaded', init);
```

替换文末的 `window.addEventListener(...)` 行。

---

### Task 8: 最终集成 — 验完整文件结构 + 调整参数

**Files:**
- Modify: `index.html` — 审查完整文件，微调参数

- [ ] **Step 1: 验证文件完整性**

Run: `wc -l index.html`
Expected: 约 450-550 行

- [ ] **Step 2: 检查关键函数是否存在**

Run: `grep -c "function " index.html`
Expected: at least 12 functions defined

Key functions to verify exist:
- `init`, `defineTracks`, `resetCar`, `bindInput`, `bindUI`, `resizeCanvas`
- `updatePhysics`, `constrainToTrack`, `closestPointOnSegment`
- `drawGrass`, `drawTrack`, `drawCar`, `render`, `gameLoop`

- [ ] **Step 3: 启动本地 HTTP 服务器做手动测试**

Run: `cd /home/jly/game/racing && python3 -m http.server 8080 &`
Open: `http://<server-ip>:8080/`

手动测试清单：
1. 页面加载，显示地图1（椭圆环形），赛车在起点
2. W/↑ 前进加速，S/↓ 后退，A/← 和 D/→ 转向
3. 前进最大速度有限，松手有惯性滑行
4. 过弯按住方向键触发漂移（侧向偏移），松键回正
5. 驶出赛道边界感受回弹
6. 点击顶部按钮切换 3 张地图正常
7. 地图 1 宽赛道好操控，地图 3 窄赛道有挑战

---

### Task 9: 参数手感调优

**Files:**
- Modify: `index.html` — 调整常量区的物理参数

- [ ] **Step 1: 根据实际手感微调常量**

当前默认值，可根据实机测试调整：

```javascript
const ACCEL = 400;            // 可调范围 300-500，决定加速快慢
const REV_ACCEL = 300;        // 倒车加速
const MAX_SPEED = 350;        // 可调范围 250-400
const MAX_REV_SPEED = 120;
const FRICTION = 0.97;        // 可调范围 0.95-0.985，越高惯性越长
const TURN_RATE = 3.5;        // 可调范围 2.5-4.5，越高转向越灵敏
const DRIFT_FORCE = 60;       // 可调范围 40-80，漂移力度
const DRIFT_SPEED_LOSS = 0.995; // 可调范围 0.99-0.998
const DRIFT_RECOVERY = 0.90;    // 可调范围 0.85-0.95
const BOUNDARY_BOUNCE = 0.4;    // 可调范围 0.2-0.6
```

如果感觉漂移太猛：降 `DRIFT_FORCE` 到 40-50
如果感觉转向太迟钝：升 `TURN_RATE` 到 4.0-4.5
如果感觉边界弹太狠：降 `BOUNDARY_BOUNCE` 到 0.2-0.3
