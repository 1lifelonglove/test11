# 旋转太极罗盘控件（taichi-ring-control）

> 一个自包含、零依赖的 Canvas 交互控件：中心旋转太极 + 可任意扩展的同心圆环，
> 每个圆环可独立控制转向、转速、偏移角、显隐与文字内容。
> 参考传统罗盘 / 太极八卦盘的视觉效果实现。

---

## 1. 快速开始

- **运行**：直接用浏览器打开 `taichi-ring-control.html` 即可（单文件、无外部依赖、无需构建）。
- **嵌入其他项目**：整个控件是一个 `<canvas>` + 一段 IIFE 原生 JS，可直接拷入任意页面；
  或作为静态资源放进 Vue / React /  Electron 项目的 `public/` 目录用 `<iframe>` / 直接拆分引入。

## 2. 功能总览

| 模块 | 能力 |
|---|---|
| 中心太极 | 旋转的 Yin-Yang 图，独立控制：转速、顺/逆时针、偏移角、显隐 |
| 同心圆环 | 任意添加 / 删除，自动从内向外重排半径 |
| 每环控制 | 转向（⟲逆/⟳顺）、转速（滑杆 + 数值输入）、偏移角、显隐、锁定、删除、编辑文字 |
| 内容主题 | 八卦、天干、地支、二十四山、二十四节气、二十八宿、数字刻度、空白刻度、自定义 |
| 自定义文字 | 添加时填文字（逗号/空格分隔，数量即段数）；已有环可点 ✎ 修改 |
| 全局 | 播放/暂停（空格键）、全部复位、转速上限自定义 |
| 回档 | 撤销 / 重做（Ctrl+Z / Ctrl+Y），快照栈 50 步，覆盖增删改与所有参数调节 |
| 交互 | 点击画布选环（金色虚线高亮 + 面板滚动定位 + 左上角信息条） |

默认预置 6 层（主流罗盘次序，从内到外）：
太极 → 八卦(8) → 天干(10) → 地支(12) → 二十四节气(24) → 二十八宿(28) → 刻度(36)

## 3. 架构说明（给接手的 AI / 开发者）

### 3.1 文件结构
单文件三段式：`taichi-ring-control.html`
- `<style>`：深色主题 UI（左侧画布 + 右侧控制面板），CSS 变量定义配色。
- `<body>`：header 工具栏 / `#cv` 画布 / 右侧面板（`#addForm` 添加表单、`#ringList` 环卡片列表、`#selinfo` 选中信息条）。
- `<script>`：一个 IIFE，包含全部逻辑。

### 3.2 核心数据模型
```js
const center = { angle, speed, dir, visible, lock };   // 中心太极
let rings = [ {                                          // 每个圆环
  id,          // 唯一 id
  theme,       // 主题 key（THEMES 的键）
  segments,    // 段数 N
  labels,      // 长度 N 的文字数组
  name,        // 显示名
  color0, color1,   // 文字双色（隔段交替）
  speed,       // °/s，恒为正值
  dir,         // 1=顺时针 -1=逆时针
  angle,       // 当前角度（度），动画循环里累加
  visible, lock,
  _inner, _outer, _mid, _thick, _rot   // 布局/运行期缓存，勿手改
} ];
```

### 3.3 关键函数
| 函数 | 职责 |
|---|---|
| `addRing(opts, silent)` | 新建环。`silent=true` 时不立即 renderPanel（初始化批量添加用，避免 TDZ） |
| `layoutRings()` | 根据环数均分半径，从内向外写 `_inner/_outer/_mid/_thick` |
| `drawRing(ring,cx,cy,geo)` | 画一个环：donut 底、辐条、文字、选中高亮 |
| `drawYinYang(cx,cy,R)` | 画中心太极（随 `center._rot` 旋转） |
| `frame(t)` | requestAnimationFrame 主循环：累加角度 → 布局 → 重绘 |
| `renderPanel()` | 重建右侧面板（中心卡 + 所有环卡），同步选中态 |
| `speedToSlider/sliderToSpeed` | 转速 ⇄ 滑杆的非线性映射（幂 2.2，低速细腻高速可达） |
| `snapshot()/restore()` | 历史快照：序列化 `{center, rings, selectedId, speedMax}` ⇄ 恢复 |
| `pushHistory()/undo()/redo()` | 撤销栈/重做栈（上限 50），**在每个修改动作前调用 `pushHistory()`** |

### 3.4 重要设计决策（改代码前必读）
1. **文字朝向（用户最终拍板，勿再改）**：径向排列，**字顶朝外、字底朝圆心**，全环方向一致、无翻转，位置随环转动。
   实现：`ctx.translate(lx,ly); ctx.rotate(midAng + Math.PI/2); ctx.fillText(label,0,0)`，其中 `midAng = rot + (k+0.5)*segAng`。
2. **donut 必须 `moveTo` 抬笔**：外圈 arc 与内圈 arc 之间若不 `ctx.moveTo(cx+inner,cy)`，
   Canvas 会把两圈在角度 0（正东）处连成一条**固定不旋转的横线**（每环一条，叠成横贯全盘的线）。这是已修过的真实 bug。
3. **TDZ 陷阱**：`renderPanel()` 依赖 `listEl/selinfoEl` 等 const，预设环必须 `addRing(..., true)` 静默添加、最后统一渲染一次，否则脚本在初始化即抛 ReferenceError 全灭。
4. **速度恒正 + dir 控制方向**：`speed` 不存负值，避免"负负得正"混乱。
5. **转速上限自定义**：`SPEED_MAX`（let）由顶部输入框驱动，改动后 `renderPanel()` 重映射所有滑杆量程；`SPEED_HARD_MAX=200000` 是数值框硬上限。
6. **频闪提示**：60fps 下超过约 10800°/s 会出现马车轮效应（倒转错觉），属正常现象。
7. **选择器用 class**（`.spd-range/.ang-range/.speed-num`），不要按 `input[type=range]` 的索引取值，加控件时不会错位。
8. **新增任何"修改状态"的操作都要先 `pushHistory()`**：滑杆在 `pointerdown`/`keydown` 时存（拖动只存一次），
   数字框在 `focus` 时存，按钮类在点击时存；否则该操作无法撤销。快照不含运行期字段（`_inner/_rot` 等），恢复后由 `layoutRings()` 每帧重算。错位。

### 3.5 扩展主题
在 `THEMES` 加一条即可（`segs` 固定段数或 null=用户填）：
```js
const THEMES = {
  mytheme: { name:"我的主题", segs:16, labels:["…","…", /* 16 个 */] },
};
```
并在 `#themeSel` 下拉加对应 `<option>`。自定义主题走 `customLabels` 输入框 → `parseLabels()`。

## 4. 集成到其他项目

### 4.1 作为独立页面（最简单）
整文件拷入项目，直接打开或 iframe 引入：
```html
<iframe src="taichi-ring-control.html" style="width:100%;height:800px;border:0"></iframe>
```

### 4.2 拆分为组件（Vue / React 思路）
- 把 `<canvas id="cv">` 和 IIFE 里「数据模型 + 绘制 + frame 循环」抽成 `TaichiCompass.js`；
- 暴露命令式 API：`new TaichiCompass(canvas).addRing({theme:'trigram', speed:14, dir:1})`；
- UI 面板用框架重写，`renderPanel()` 改为响应式状态（rings 数组放 store/ref）。
- 绘制层与 UI 层已天然解耦：绘制只读 `rings[]/center`，改数据即生效。

### 4.3 在另一个 AI 会话中继续优化
把本 README 与 html 一起发给 AI，开头说明：
> "这是旋转太极罗盘控件，请先读 README.md 第 3 节架构与第 3.4 节设计决策，再按我的需求修改 taichi-ring-control.html。"

## 5. 已知边界 & 后续优化方向
- [ ] 圆环层级/粗细暂不可拖拽调整（当前按添加顺序均分）。
- [ ] 无配置导出/导入（可加 `JSON.stringify({center,rings})` 一键导出）。
- [ ] 文字字号随环厚自适应（当前 clamp 9~13px），超多层时较挤。
- [ ] 移动端触摸事件未适配（点击选择基于 mouse 坐标，可直接复用为 touch）。
- [ ] 高速下可换成"绕圈亮点"等无频闪的动效表达。

## 6. 版本记录
- 2026-09-07 初级原型 → 修复 TDZ 点击失效、donut 固定横线 bug；
  新增中心太极控制、自定义转速上限、自定义文字主题；
  文字朝向定稿为"径向、字底朝圆心、随环转动"；
  新增撤销/重做（回档）功能，Ctrl+Z / Ctrl+Y，快照栈 50 步。
