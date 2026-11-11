# 组件配方卡:官方五个玻璃组件的复刻指南

官方 Catalog 给出 5 个组件实现(README 点名"库不含高级组件,请参考这些")。本文把每个组件拆成**结构 + 参数 + 1.0.6 兼容注记**,让 agent 不读源码也能复刻出同款。

> 通用前提:交互基建(`DampedDragAnimation` / `InteractiveHighlight` / `inspectDragGestures`)见 `interactive-glass.md`。官方源文件都在 `app/src/commonMain/kotlin/com/kyant/backdrop/catalog/components/`,Apache-2.0,可 vendor。

## 0. 官方效果栈的统一签名(先背这个)

五个组件的 `effects = {}` 高度一致,模式是:

```kotlin
effects = {
    vibrancy()                    // ← 2.x 专属,1.0.6 没有
    blur(2f.dp.toPx())            // 官方用极小的 blur!不是 10f/20f
    lens(12f.dp.toPx(), 24f.dp.toPx())
}
```

**与主 SKILL.md 的参数量级差异,原因在这里**:官方先叠了 `vibrancy()`(提饱和度、让透过的颜色更"活"),所以 blur 可以开很小、折射就可以"薄而锐"。**1.0.6 没有 vibrancy**,直接抄官方参数会得到"又薄又灰"的结果——这就是主 SKILL 用 `blur 10-26 / lens 18-34` 补偿的原因。

### 1.0.6 的 vibrancy 替代(实测可用的两条路)

```kotlin
// 路 A:colorControls 没有就用 colorFilter 调色矩阵 —— 提饱和近似 vibrancy
effects = {
    colorFilter(ColorMatrixColorFilter(ColorMatrix(
        floatArrayOf(                      // saturation × 1.2 的矩阵(标准 Android 色彩矩阵公式)
            1.2f, -0.1f, -0.1f, 0f, 0f,
            -0.1f, 1.2f, -0.1f, 0f, 0f,
            -0.1f, -0.1f, 1.2f, 0f, 0f,
            0f, 0f, 0f, 1f, 0f,
        )
    )))
    blur(10f); lens(18f, 18f)
}

// 路 B:放弃调色,把 blur/lens 按主 SKILL 的量级开大,靠折射本身撑质感(已真机验证)
```

## 1. LiquidButton(胶囊按钮)

**源文件**:`components/LiquidButton.kt`(122 行,最短的一个,新手先抄它)

| 项 | 值 |
|---|---|
| 形状 | `Capsule()`(来自 `io.github.kyant0:shapes`) |
| effects | `vibrancy + blur(2dp) + lens(12, 24)` |
| 高光 | 默认(`Highlight.Default`,胶囊长边方向性可见) |
| 按压 | ①整体放大 4dp ②tanh 阻尼位移 ③按拖拽方向的椭圆变形——完整公式见 `interactive-glass.md` |
| 点击 | `clickable(interactionSource=null, indication=null)`,反馈全交给形变+光斑 |
| 参数 | `isInteractive`(关掉形变保留点击)、`tint`(Hue 混合染色)、`surfaceColor`(纯色覆盖) |

**tint 的染色写法**(官方做法,注意是两层):
```kotlin
onDrawSurface = {
    drawRect(tint, blendMode = BlendMode.Hue)          // 用色相重染玻璃下透出的内容
    drawRect(tint.copy(alpha = 0.75f))                  // 再压一层半透明本体色
}
```

## 2. LiquidToggle(开关)

**源文件**:`components/LiquidToggle.kt`(~150 行)

结构:胶囊轨道 + `DampedDragAnimation` 驱动的滑块。

| 项 | 值 |
|---|---|
| 状态 | `fraction: 0f..1f`(连续!),`selected: () -> Boolean` |
| 交互 | 可点(直接翻)可拖(`didDrag` 区分两种意图,松手按 `fraction >= 0.5` 归位) |
| 滑块 | `pressedScale = 1.5f`(按下滑块显著变大) |
| 外观 | iOS 配色:accent `#34C759/#30D158`,track `#787880@36%`,深浅主题各一套 |

**两个状态源的双向同步**(容易写错的地方):
```kotlin
LaunchedEffect(dampedDragAnimation) {           // UI 拖拽 → fraction → 动画值
    snapshotFlow { fraction }.collectLatest { dampedDragAnimation.updateValue(it) }
}
LaunchedEffect(selected) {                      // 外部状态变化 → 动画接管(带惯性翻过去)
    snapshotFlow { selected() }.collectLatest {
        dampedDragAnimation.animateToValue(if (it) 1f else 0f)
    }
}
```
**关键**:`updateValue`(跟手)与 `animateToValue`(带弹簧惯性)是两个不同方法——跟手用前者,状态翻转用后者。

## 3. LiquidSlider(滑块)

**源文件**:`components/LiquidSlider.kt`(~180 行)

| 项 | 值 |
|---|---|
| 值域 | `valueRange: ClosedRange<Float>`,进度 = `animation.progress` |
| 填充轨 | `onDrawSurface` 里按 `progress` 画 `drawRoundRect`(宽 = trackWidth × progress) |
| 滑块 | 两种形态:静止小圆 ⇄ 按下变成胶囊(scaleX/Y 分别 `lerp(2/3→1, 0→1, progress)`) |
| blur 联动 | **`blur(8dp × (1 - progress))`** —— 按住滑块时玻璃变清晰,这是"捏住玻璃"的触感 |

**值得偷的细节**:blur 与按压进度联动。静态 blur 是死的;让 blur 跟着交互状态变化,玻璃立刻有"被触碰就变实"的质感。此技巧适用于任何玻璃组件。

## 4. LiquidBottomTabs(标签栏)—— 最复杂的一个

**源文件**:`components/LiquidBottomTabs.kt`(11.7 KB)+ `LiquidBottomTab.kt`(单 tab)

| 项 | 值 |
|---|---|
| 指示器 | 不画一个框,而是**整条玻璃的 layerBlock 里按 value 偏移一个"内嵌玻璃面板"** |
| 连续 index | value 0..(n-1) 连续可插值,指示器能停在两 tab 中间;松手 `fastRoundToInt` 归位 |
| 弹簧 | 归位 `spring(1f, 300f, 0.5f)`;面板间过渡逐 tab 变形 |
| 按压缩放 | `pressedScale = 78f/56f`(按宽度算,约 1.39x) |
| 手势 | 整条玻璃可横向拖拽换页 —— 指示器跟着手指走 |
| 中心高光 | `InteractiveHighlight(position = ...)` 的 position 重写为**指示器中心**(不是手指位置!)——光斑永远聚在选中项上 |

**三层结构**(官方实现,复刻时的骨架):
1. 外层 `drawBackdrop`(玻璃条本体,vibrancy+blur(8dp)+lens(24,24))+ `layerBlock` 按压缩放 + `onDrawSurface { drawRect(containerColor) }`(容器色打底)
2. 中层 `layerBackdrop(tabsBackdrop)` + 隐藏的原始 tab 行(`alpha(0f)` 只为录制)
3. 内层玻璃面板 `drawBackdrop(tabsBackdrop, ...)` —— **玻璃采样玻璃的录制层**,这是合法的"玻璃叠玻璃"(下层的 tab 行是普通绘制,不是 drawBackdrop),注意不要混淆

## 5. LiquidBottomTab(单 tab)

选中态的材质片写法已收录在主 SKILL.md「玻璃内部的元素」。补充官方版两点:
- tab 图标随 `LocalLiquidBottomTabScale`(由 pressProgress 驱动的 1→1.2x)联动缩放
- 选中色直接 `onDrawSurface { drawRect(accentColor) }` 打底,不搞复杂渐变

## 复刻清单(按顺序抄,每步真机验证)

- [ ] 1. LiquidButton —— 学会 layerBlock 形变 + observe-only 手势
- [ ] 2. LiquidToggle —— 学会 DampedDragAnimation 的 fraction 状态机
- [ ] 3. LiquidSlider —— 学会 onDrawSurface 动态绘制 + blur 联动
- [ ] 4. LiquidBottomTabs —— 学会连续 index + 三层玻璃结构 + 光斑 position 重写
- [ ] 每步在 compileSdk 36 + backdrop 1.0.6 下编译,套用「0. 效果栈」的兼容签名

## 参考

- 组件源文件:`app/src/commonMain/kotlin/com/kyant/backdrop/catalog/components/`(GitHub: Kyant0/AndroidLiquidGlass)
- shapes 库的 `Capsule`:`io.github.kyant0:shapes:1.2.0`(1.0.6 兼容)
- 动效基建:`references/interactive-glass.md`
