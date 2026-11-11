# 交互式玻璃:让玻璃"活起来"

静态玻璃(导航栏/按钮)只是及格线。这个库真正的精华是**可交互玻璃**——按压会形变、高光跟着手指走、松手带惯性飞过去。全部配方来自官方 Catalog 的 `app/` 示例工程(Apache-2.0),本文把它们的**内部机制**拆开讲清,让 agent 能直接照抄或改造。

## 三件套总览

Catalog 的 `app/src/commonMain/.../catalog/utils/` 下有三个工具类,是所有"活玻璃"的地基:

| 类 | 干什么 | 关键机制 |
|---|---|---|
| `DampedDragAnimation` | 按压 + 拖拽 + 速度交接的统一状态机 | 5 个独立弹簧;`VelocityTracker` 把手势速度喂给松手动画 |
| `InteractiveHighlight` | 跟随手指的高光光斑 | AGSL shader 光斑 + 弹簧拖尾;不支持 shader 时降级为整体提亮 |
| `inspectDragGestures` | 手势观察器 | `awaitFirstDown(false, PointerEventPass.Initial)` **observe-only**,绝不 consume |

**复制合规(Apache-2.0)**:vendor 这三个文件进你的工程时,保留版权声明、附 LICENSE 全文、注明修改。它们只用标准 Compose API(`Animatable`/`VelocityTracker`/`pointerInput`)+ 库的公开钩子,**backdrop 1.0.6 兼容**。

## 手势铁律:observe-only(官方实现是范本)

`inspectDragGestures` 的两个关键点,缺一玻璃就会"吃掉"点击:

```kotlin
// ① Initial pass + requireUnconsumed=false:先于子组件观察,但不拦截
val initialDown = awaitFirstDown(false, PointerEventPass.Initial)
// ② 拖拽循环里,change 被 consume 立即退出 —— 子组件(如按钮)优先
if (change.isConsumed) return null
```

这与 skill 正文「按压与交互」一节的规则完全一致。**玻璃外壳永远只观察,不消费**;点击由内部的 `clickable` 处理。

## DampedDragAnimation:五个弹簧的分工

不要把"一个动画对象"理解成一条弹簧。它内部是**五条并行弹簧**,各管一件事:

```kotlin
valueAnimationSpec          = spring(1f, 1000f, visibilityThreshold)      // ① 值:临界阻尼,不回弹
velocityAnimationSpec       = spring(0.5f, 300f, visibilityThreshold*10f) // ② 速度:带惯性,自然衰减
pressProgressAnimationSpec  = spring(1f, 1000f, 0.001f)                   // ③ 按压进度:快进快出
scaleXAnimationSpec         = spring(0.6f, 250f, 0.001f)                  // ④ X 缩放:轻微回弹
scaleYAnimationSpec         = spring(0.7f, 250f, 0.001f)                  // ⑤ Y 缩放:轻微回弹(与④不同参数!)
```

**选型规则**(直接抄这个表):
- 玻璃的**位置/状态值**(tab index、滑块 fraction)→ 临界阻尼 `spring(1f, 1000f)`,到位就是到位,不许抖
- 松手后的**惯性衰减** → `spring(0.5f, 300f)`,带一点过冲,读作"惯性"
- **按压反馈**(scale/光斑) → 0.6~0.7 阻尼 / 250ms,X 和 Y 用**不同参数**——完全一致的 XY 缩放看起来像机器,轻微不对称才有"材质被捏"的感觉

### 速度交接的核心三行

```kotlin
// 每次值变化时,把"值"当作一个虚拟点的位置喂给 VelocityTracker
velocityTracker.addPosition(now, Offset(value, 0f))
// 算出速度,按值域归一化
val targetVelocity = velocityTracker.calculateVelocity().x / (valueRange.endInclusive - valueRange.start)
// 喂给速度弹簧
velocityAnimation.animateTo(targetVelocity, velocityAnimationSpec)
```

要点:**喂给 tracker 的是动画值,不是手指位置**——这样松手瞬间动画值带着手势的真实速度继续飞,再被 ① 号弹簧拉回目标,这就是 iOS 那种"甩一下滑块自己滑过去"的全部秘密。

### release() 里最容易被忽略的一步

```kotlin
if (value != targetValue) {
    snapshotFlow { valueAnimation.value }
        .filter { abs(it - valueAnimation.targetValue) < threshold }  // 接近目标才收按压态
        .first()
}
// 然后才 animateTo(0f) 收缩 scale / 光斑
```

**按压态不能在松手瞬间立刻消失**——要等玻璃"飞到位"附近再收回,否则玻璃会在飞行途中突然塌回去,看起来像断电。

## 按压形变配方(LiquidButton 的 layerBlock,逐行拆解)

这是"液态"二字的核心。玻璃在 `layerBlock`(GraphicsLayerScope)里做四件事:

```kotlin
layerBlock = {
    val progress = interactiveHighlight.pressProgress

    // ① 基础放大:按下时整体放大 4dp 折算的比例
    val scale = lerp(1f, 1f + 4f.dp.toPx() / size.height, progress)

    // ② 位移:tanh 阻尼限幅 —— 越拖越费劲,永远不会被拖出按钮太远
    val maxOffset = size.minDimension
    val initialDerivative = 0.05f          // 初始斜率:拖 1px 位移多少。越小越"黏"
    translationX = maxOffset * tanh(initialDerivative * offset.x / maxOffset)
    translationY = maxOffset * tanh(initialDerivative * offset.y / maxOffset)

    // ③ 方向性变形:按手指相对按钮中心的角度,X/Y 分别算伸长量
    val maxDragScale = 4f.dp.toPx() / size.height
    val offsetAngle = atan2(offset.y, offset.x)
    scaleX = scale + maxDragScale * abs(cos(offsetAngle) * offset.x / size.maxDimension) *
             (width / height).fastCoerceAtMost(1f)
    scaleY = scale + maxDragScale * abs(sin(offsetAngle) * offset.y / size.maxDimension) *
             (height / width).fastCoerceAtMost(1f)
}
```

三个物理直觉,调参时对着想:

1. **tanh 限幅 = 橡皮筋**。线性位移拖多远走多远,像把玻璃从按钮上撕下来;tanh 在大位移处趋近 `±maxOffset`,玻璃"被拽住"但永远在。
2. **方向性变形 = 表面张力**。往右下拖,玻璃沿拖拽方向拉长(`cos/sin` 投影),同时用 `width/height` 的比值约束防止细长控件变形过度。
3. **全部量都是 `progress` 的插值**,progress 由弹簧驱动 —— 所以松手后整个形变(位移+放大+拉伸)以弹簧曲线回弹,而不是瞬间归零。

## InteractiveHighlight:光斑的降级设计

```kotlin
// shader 可用时:AGSL 光斑,半径 = minDimension * 1.5f,中心亮 0.15 alpha,Plus 混合
drawRect(Color.White.copy(0.08f * progress), blendMode = Plus)     // 先整体提亮一层
drawRect(ShaderBrush(shader), blendMode = Plus)                    // 再叠跟随光斑

// shader 不可用时:只有整体提亮,强度提到 0.25 保证可感知
drawRect(Color.White.copy(0.25f * progress), blendMode = Plus)
```

光斑位置用 `spring(0.5f, 300f)` 拖尾跟随手指(不是 snapTo 直跟),松手后弹回起点。**两层叠加**(整体提亮 + 局部光斑)是关键——只有光斑的话,按压感太局部;只有整体提亮,又没有"光聚在指尖"的方向感。

## 完整接线顺序(从零接一块"活玻璃")

```kotlin
val scope = rememberCoroutineScope()
val interactiveHighlight = remember(scope) { InteractiveHighlight(scope) }

Box(
    Modifier
        .drawBackdrop(
            backdrop = backdrop,
            shape = { Capsule() },
            effects = { blur(10f); lens(18f, 18f) },        // 静态玻璃层(见主 SKILL)
            highlight = { Highlight(width = 1.dp) },         // 静态 rim(不要省,光斑叠加在它之上)
            layerBlock = { /* ↑按压形变配方,读 interactiveHighlight.pressProgress */ },
        )
        .clickable(interactionSource = null, indication = null) { onClick() }  // indication=null!高光自己做
        .then(interactiveHighlight.modifier)      // 光斑绘制层
        .then(interactiveHighlight.gestureModifier) // 光斑手势层(observe-only)
)
```

两个易错点:
- **`indication = null`**:保留默认 ripple 会和光斑叠加出脏脏的灰色;按压反馈完全交给光斑+形变。
- **`interactionSource = null`**:让 `clickable` 内部自建,避免与光斑手势的状态机互相干扰。

## 组件级配方:BottomTabs 的弹簧指示器

标签栏的"指示器追手指"是 DampedDragAnimation 的第二个应用:

- 指示器位置 = `dampedDragAnimation.value * tabWidth`,value 是 tab index(连续值!)
- 拖拽时 `updateValue(targetValue + dragAmount.x / tabWidth)` —— **index 可以是 0.5**,指示器就停在两个 tab 中间
- 松手时 `fastRoundToInt()` 定 tab,再 `animateToValue(index)` 让它自己滑到位
- 整条玻璃还叠加 `layerBlock` 按压缩放(`pressedScale = 78f/56f`,按宽算)

**"把离散的 tab index 当连续值驱动动画"**是这个组件最值得偷的思路——slider、分段控件、步进器全部适用。

## 陀螺仪高光(UISensor)

`rememberUISensor()` 返回 `gravityAngle`(加速度计算出的重力方向角)。接进玻璃:

```kotlin
val uiSensor = rememberUISensor()
// 2.x:HighlightStyle.Default(angle = uiSensor.gravityAngle, falloff = 2f)
// 1.0.6:HighlightStyle 没有 angle 参数,替代方案见下
```

**1.0.6 兼容方案**:高光角度在 1.0.6 是硬编码 45°,无法运行时改。替代做法是把 `UISensor.gravity` 映射到一个**方向性渐变描边/高光 layerBlock**上(用 `Offset` 画一条随重力转的亮边),观感近似、成本更低。真机验证过的替代实现待补,先标记为 open。

## 性能红线(交互玻璃专属)

- `layerBlock` 每帧都在写 graphicsLayer 属性 → **绝不在里面做对象分配**(所有 `dp.toPx()` 先在块外算好)
- 光斑 shader 每帧 `setFloatUniform` × 4 → 光斑尺寸(radius)不随帧变,考虑只在 position 变时更新
- 长列表里**禁止**放交互玻璃(按下/松手会打断列表 fling 的速度采样)

## 参考

- 官方实现(必读,本文所有配方的出处):
  - `app/src/commonMain/kotlin/com/kyant/backdrop/catalog/utils/DampedDragAnimation.kt`
  - `app/src/commonMain/kotlin/com/kyant/backdrop/catalog/utils/InteractiveHighlight.kt`
  - `app/src/commonMain/kotlin/com/kyant/backdrop/catalog/utils/DragGestureInspector.kt`
  - `app/src/commonMain/kotlin/com/kyant/backdrop/catalog/utils/UISensor.kt`(commonMain 是 expect,androidMain 是 actual)
- 场景组装示例:`components/LiquidButton.kt` / `LiquidToggle.kt` / `LiquidSlider.kt` / `LiquidBottomTabs.kt`
