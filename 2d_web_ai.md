# 2D 俯视角坦克对战游戏 (Tank Battle 2D) - AI 开发规范文档 (PRD & Tech Spec)

> **AI 开发者指令 (System Prompt Context)**：
> 你现在是一名资深的 HTML5 游戏全栈工程师。请严格按照本文档的架构设计、数据结构、物理规则和分步指南来编写代码。
> **核心铁律**：
> 1. **零外部静态资源依赖**：严禁使用外部图片 (`.png`/`.jpg`) 或音频文件。所有坦克、地形、特效必须通过 `Canvas 2D Context API`（几何图形、渐变、路径）程序化绘制；音效使用 `Web Audio API` 合成或留空接口。
> 2. **双轴自由操控**：采用**键盘 WASD 移动车体 + 鼠标 360° 瞄准射击**的现代俯视角射击机制（非红白机四方向锁死），提升可玩性。
> 3. **代码高内聚低耦合**：严格遵守文档定义的面向对象状态结构，确保帧循环 (`requestAnimationFrame`) 中包含精准的 `deltaTime` 时间步长计算。

---

## 1. 游戏核心概述 (Game Overview)

* **游戏类型**：2D 俯视角战术射击闯关游戏 (Top-Down Tactical Tank Shooter)
* **对战模式**：单人玩家 (Player) vs 多种战术 AI 敌军 (AI Enemies)
* **核心玩法**：玩家操控主战坦克在网格化竞技场中游走，利用掩体躲避弹幕、利用反弹墙进行战术射击、拾取掉落道具强化自身，消灭当前波次所有 AI 坦克以进入下一关。
* **画风与视觉**：现代极简霓虹/几何矢量风格（深色网格背景 + 高对比度发光几何车体 + 粒子爆炸与履带残影特效）。

---

## 2. 技术栈与工程结构 (Tech Stack & Architecture)

为了适配 AI 快速生成与浏览器即开即玩，支持 **单文件模式 (`index.html`)** 或 **模块化原生 JS/TS 模式**。

### 2.1 技术选型
* **渲染引擎**：原生 HTML5 `<canvas>` (2D Context)
* **核心逻辑**：Vanilla JavaScript (ES6+ Class) 或 TypeScript
* **UI 界面**：HTML5 + CSS3 Overlay（血条、波次提示、升级菜单、结算弹窗覆盖在 Canvas 之上）
* **数学计算**：2D 向量运算 (Vector2D)、AABB 矩形碰撞检测、圆与矩形碰撞检测、射线视线检测 (Line of Sight)

### 2.2 逻辑模块划分 (Class Design)
无论写成单文件还是多文件，代码中必须包含以下核心类：
1. `Vector2`：二维向量工具类（加、减、归一化、距离、点乘、角度转换）。
2. `Game`：游戏主控制器（管理游戏状态机、主循环 `loop`、关卡加载、实体池管理）。
3. `InputHandler`：输入监听器（维护 `keys` 按键状态字典与 `mouse` 坐标/点击状态）。
4. `MapManager`：地图与网格系统（解析二维数组地图、绘制地形、提供障碍物碰撞查询）。
5. `Tank` (基类) -> 派生出 `PlayerTank` 与 `EnemyTank`：处理移动、转向、炮塔旋转、射击冷却、受击扣血。
6. `Bullet`：子弹实体（处理飞行轨迹、墙壁反弹计数、拖尾绘制、生命周期）。
7. `Particle`：粒子特效实体（受击火花、坦克爆炸碎片、炮口火焰、移动履带印）。
8. `Prop`：战场掉落物（回血包、护盾、散弹强化、移速加成）。

---

## 3. 核心数据结构与实体规范 (Data Structures)

AI 在编写代码时，必须严格对齐以下属性字段：

### 3.1 地图网格配置 (Grid & Tile System)
* **画布尺寸**：固定逻辑分辨率 `960 x 640` 像素（支持 CSS 等比缩放适配屏幕）。
* **网格大小 (`TILE_SIZE`)**：`40 x 40` 像素，整张地图为 `24 列 x 16 行` 的二维数组 `grid[16][24]`。
* **地形枚举值 (`TileType`)**：
  * `0 - EMPTY` (空地)：坦克与子弹均可正常通行。
  * `1 - BRICK` (砖墙 - 橙褐色)：阻挡坦克与子弹；可被子弹击中，拥有 `hp: 2`，受击产生裂纹，归零后变为 `0`。
  * `2 - STEEL` (钢墙 - 银灰色)：阻挡坦克；不可破坏，但**允许子弹发生镜面反射 (Bounce)**。
  * `3 - WATER` (水域 - 深蓝色)：阻挡坦克移动，但**子弹可以无视水域直接飞跃**。
  * `4 - BUSH` (草丛 - 半透明绿色)：不阻挡移动和子弹；渲染层级在坦克上方，提供视觉隐蔽。

### 3.2 坦克基类属性 (`Tank` Schema)
```javascript
class Tank {
  constructor(x, y, config) {
    this.x = x;                 // 中心点 X 坐标 (px)
    this.y = y;                 // 中心点 Y 坐标 (px)
    this.radius = 16;           // 碰撞圆半径 (略小于半个网格 20px，保证顺畅过弯)
    this.bodyAngle = 0;         // 车体朝向弧度 (rad)
    this.turretAngle = 0;       // 炮塔朝向弧度 (rad)
    
    this.hp = config.hp;        // 当前生命值
    this.maxHp = config.hp;     // 最大生命值
    this.speed = config.speed;  // 移动速度 (px/s)
    
    this.fireRate = config.fireRate;     // 射击冷却间隔 (秒)
    this.fireCooldown = 0;               // 当前剩余冷却时间 (秒)
    this.bulletSpeed = config.bulletSpeed; // 子弹初速度 (px/s)
    this.bulletDamage = config.damage;   // 单发伤害
    this.bulletBounces = config.bounces; // 子弹最大反弹次数 (默认 1)
    
    this.color = config.color;           // 主色调 (Hex/RGB)
    this.isDead = false;                 // 销毁标记
  }
}
```

### 3.3 子弹属性 (`Bullet` Schema)
```javascript
class Bullet {
  constructor(x, y, angle, speed, damage, maxBounces, isPlayer) {
    this.x = x;
    this.y = y;
    this.vx = Math.cos(angle) * speed;
    this.vy = Math.sin(angle) * speed;
    this.radius = 4;
    this.damage = damage;
    this.bouncesLeft = maxBounces; // 剩余可反弹次数
    this.isPlayer = isPlayer;      // true: 玩家发射; false: AI发射 (防止同类误伤)
    this.history = [];             // 记录最近 5 帧坐标，用于绘制流光拖尾
    this.isDead = false;
  }
}
```

---

## 4. 敌方 AI 兵种与状态机设计 (AI Behavior & Archetypes)

为了保证单人闯关的趣味性，设计 **4 种差异化 AI 坦克**。AI 逻辑无需复杂的 A* 寻路，采用 **射线避障 + 状态机 (FSM)** 即可在轻量代码中实现极佳效果。

### 4.1 四大 AI 兵种数值表
| 兵种代号 | 视觉特征 (颜色/造型) | HP | 移速 | 射速/弹道特征 | 行为逻辑特征 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Scout (侦察型)** | 青色 / 小巧单管 | 40 | 160 (快) | 射速快，无反弹 | 激进逼近玩家，高频绕圈游走 |
| **Guard (守卫型)** | 红色 / 标准主战 | 100 | 90 (中) | 标准射速，1次反弹 | 保持中距离交火，寻找掩体边缘 |
| **Sniper (狙击型)**| 紫色 / 细长炮管+红外线 | 60 | 50 (慢) | 极高弹速，2次反弹 | 远离玩家，射击前有 0.8s 红色瞄准线预警 |
| **Heavy (重装Boss)**| 暗金 / 宽大车体+双联装 | 300 | 45 (慢) | 每次齐射3发扇形弹幕 | 霸体推进，血量低于50%进入狂暴射速模式 |

### 4.2 AI 决策状态机 (每帧更新逻辑)
1. **巡逻与逼近 (Move)**：
   * 计算指向玩家的向量 $V_{target}$。
   * 若距离玩家过远 ($> 250\text{px}$)，朝玩家方向移动；若为 `Sniper` 且距离过近 ($< 200\text{px}$)，则朝反方向后退。
   * **轻量避障算法**：在当前移动方向前方 $30\text{px}$ 处做碰撞探测，若遇到墙体/水域，将移动方向顺时针或逆时针旋转 $90^\circ$ 持续 $1.0$ 秒。
2. **瞄准 (Aim)**：
   * AI 的 `turretAngle` 使用线性插值 (`lerpAngle`) 平滑转向玩家当前坐标（加入微小的随机角速度限制，避免 AI 瞬间锁头）。
3. **开火判定 (Fire Check)**：
   * 从 AI 坐标向玩家坐标发射一条虚拟采样射线（步进 $10\text{px}$ 检测一次网格）：
     * 如果射线先碰到 `STEEL` 或 `BRICK`，则抑制开火（避免无脑打墙）。
     * 如果射线畅通直达玩家，且 `fireCooldown <= 0`，立即开火！

---

## 5. 物理碰撞与核心算法细节 (Physics & Algorithms)

> **注意**：这是 AI 写代码最容易出 Bug 的地方，请严格使用以下数学逻辑实现。

### 5.1 坦克与网格地形的碰撞（滑动碰撞响应）
* **禁止卡死**：当玩家同时按住 `W` 和 `D` 斜向撞击垂直墙壁时，坦克不能完全停下，应当沿着墙壁竖直滑动。
* **实现方法（X/Y 轴分离检测）**：
  1. 先计算 `nextX = this.x + vx * dt`，保持 `this.y` 不变。
  2. 检测 `(nextX, this.y, this.radius)` 是否与周围 $3 \times 3$ 网格内的 `BRICK / STEEL / WATER` 发生圆-矩形重叠。若无重叠，更新 `this.x = nextX`。
  3. 再计算 `nextY = this.y + vy * dt`，保持更新后的 `this.x` 不变，做同样检测。若无重叠，更新 `this.y = nextY`。

### 5.2 圆与矩形 (Circle-AABB) 碰撞检测公式
```javascript
function circleRectCollide(cx, cy, cr, rx, ry, rw, rh) {
  const closestX = Math.max(rx, Math.min(cx, rx + rw));
  const closestY = Math.max(ry, Math.min(cy, ry + rh));
  const dx = cx - closestX;
  const dy = cy - closestY;
  return (dx * dx + dy * dy) < (cr * cr);
}
```

### 5.3 子弹墙壁镜面反射算法 (Bullet Bounce)
* 当子弹从上一帧 `(prevX, prevY)` 移动到 `(x, y)` 命中墙体网格 `(tileX, tileY)` 时：
  1. 若墙体为 `BRICK`：墙体扣血，子弹直接销毁 (`isDead = true`)，生成砖块碎屑粒子。
  2. 若墙体为 `STEEL`（或画布边界）：
     * 若 `bouncesLeft > 0`：判断子弹是从墙体的**左右侧面**撞入还是**上下底面**撞入（对比 `prevX` 和 `prevY` 所在的网格行列号）。
     * 左右撞击：`this.vx = -this.vx`；上下撞击：`this.vy = -this.vy`。
     * `this.bouncesLeft--`，触发清脆的跳弹火花粒子。
     * 若 `bouncesLeft <= 0`：子弹销毁。

---

## 6. Canvas 视觉渲染规范 (Procedural Graphics)

所有图形必须封装在各实体的 `draw(ctx)` 方法中，利用 `ctx.save()` 和 `ctx.restore()` 进行矩阵变换：

1. **坦克绘制分层**：
   * **底层（阴影）**：偏移 `(3, 3)` 绘制半透明黑底。
   * **第一层（车体与履带）**：`ctx.translate(x, y)` -> `ctx.rotate(bodyAngle)`。绘制两侧深灰色矩形履带（带横向刻度线），中间绘制圆角矩形车体（填充主色调 + 深色描边）。
   * **第二层（独立炮塔）**：在 `(x, y)` 基础上 `ctx.rotate(turretAngle)`。先画炮管（矩形向X轴正方向延伸，开火时加入向后的后坐力偏移 `recoilOffset`），再画圆形炮塔底座。
   * **顶层（生命条）**：当 `hp < maxHp` 时，在坦克正上方 `(x, y - 28)` 处绘制非旋转的迷你血条（绿/黄/红渐变）。
2. **玩家瞄准辅助**：
   * 从玩家炮口位置向鼠标位置绘制一条半透明虚线 (`ctx.setLineDash([4, 6])`)，并在鼠标位置绘制十字准星 (Crosshair)。
3. **屏幕震动 (Screen Shake)**：
   * 当坦克爆炸或玩家受击时，设置全局 `shakeTimer = 0.2` 秒，渲染主画布前执行随机偏移 `ctx.translate((Math.random()-0.5)*intensity, (Math.random()-0.5)*intensity)`。

---

## 7. 关卡与道具系统 (Level & Power-ups)

### 7.1 关卡波次演进 (5 关循序渐进)
* **Level 1 (新兵训练营)**：对称砖墙掩体，`3 x Scout` + `1 x Guard`。
* **Level 2 (钢铁回廊)**：大量 `STEEL` 反弹墙，`2 x Guard` + `2 x Sniper`（引导玩家利用跳弹击杀掩体后的狙击手）。
* **Level 3 (护城河之战)**：中央横跨 `WATER` 河流与两座桥梁，`3 x Scout` + `2 x Guard` + `2 x Sniper`。
* **Level 4 (丛林伏击)**：密集 `BUSH` 草丛与迷宫砖墙，`4 x Scout` + `3 x Guard` + `2 x Sniper`。
* **Level 5 (终焉决战)**：开阔竞技场配四角钢堡，`1 x Heavy Boss` + `2 x Guard` 护卫。

### 7.2 战场随机道具 (`Prop` 掉落)
每击毁一辆敌方坦克，有 `35%` 概率在原地生成以下一种发光旋转图标（持续 12 秒后消失）：
* `[+] 维修包 (绿色)`：立即恢复玩家 $40\text{ HP}$。
* `[S] 能量护盾 (蓝色)`：获得持续 8 秒的无敌圆形光罩，抵消一切子弹。
* `[W] 散弹模块 (橙色)`：持续 10 秒，玩家每次开火同时发射 3 枚扇形子弹（夹角 $15^\circ$）。
* `[⚡] 涡轮增压 (黄色)`：持续 10 秒，移动速度与射速提升 $40\%$。

---

## 8. HUD 与交互状态流 (UI & Game States)

游戏状态机 `gameState` 包含以下 5 个状态：
1. `MENU` (主菜单)：标题展示、操作说明（WASD移动 / 鼠标瞄准左键射击 / P键暂停）、“开始战斗”按钮。
2. `PLAYING` (战斗中)：顶部 HUD 实时显示：`当前关卡 (Level X/5)`、`剩余敌军数量`、`玩家生命值条`、`当前生效的 Buff 倒计时`、`得分 (Score)`。
3. `LEVEL_CLEAR` (过关休整)：全灭当前波次敌军后触发，弹出 **3 选 1 永久强化卡牌**（① 最大生命 +30 并回满；② 基础射速 +20%；③ 子弹反弹次数 +1 且弹速 +15%），选择后进入下一关。
4. `GAME_OVER` (战败结算)：玩家 HP 归零，显示击败坦克总数与最终得分，提供“重新挑战”按钮。
5. `VICTORY` (通关致敬)：击败第 5 关 Boss，触发全屏礼花粒子，显示通关评价。

---

## 9. 给 AI 编程模型 (GLM-5.3-Flash) 的分步生成指南

> 💡 **使用方法**：你可以直接将本文档发给 AI，并附带以下指令让它一次性或分步输出代码。

### 推荐复制给 AI 的启动 Prompt：
```text
请严格阅读上述《2D 俯视角坦克对战游戏 - AI 开发规范文档》，帮我编写完整可运行的 Web 游戏代码。
要求：
1. 将所有 HTML、CSS 和 JavaScript 整合在单个 `index.html` 文件中，方便我直接在浏览器双击运行。
2. 严禁省略任何函数实现（不要写 `// TODO` 或省略号），完整实现 5 个关卡地图数组、4 种 AI 坦克、X/Y 分离碰撞、子弹反弹、粒子爆炸、3选1过关升级面板和 Web Audio API 合成音效（开火、爆炸、跳弹、拾取道具）。
3. 确保代码结构清晰，包含完整的中文注释。现在请直接输出完整的 `index.html` 代码。
```