---
name: android-liquid-glass
license: MIT
compatibility: Requires a Gradle-based Android project using Jetpack Compose. No scripts, no network access, no binary tools. Works on any OS (verified on Windows); visual effect verified on a physical Android 16 device. Glass rendering needs RuntimeShader (API 33+) support on the device; the skill includes a fallback strategy.
description: Build Apple-style liquid glass surfaces in Android Jetpack Compose with Backdrop (io.github.kyant0:backdrop). Use for glass buttons, toggles, sliders, tab bars, nav bars, floating layers, or when glass looks like a flat box, a grey coin, or crashes in the draw pass. Covers layerBackdrop/drawBackdrop wiring, the effect stack (blur/lens/colorFilter/opacity), HighlightStyle, perf budgets, RuntimeShader fallback. 液态玻璃 / 玻璃拟态 / 折射玻璃 / 毛玻璃 / 玻璃质感 UI 也适用。
---

# Android Liquid Glass (Backdrop)

Apple 的 Liquid Glass 不是"加个模糊"。它的核心是**折射**:玻璃边缘让背后的内容弯折,高光沿边缘走,深度靠光而不是阴影表达。Backdrop 把这件事拆成三个可独立控制的部分——**source(录什么)、surface(画在哪)、effect stack(怎么变)**。绝大多数翻车都来自这三者的接线错误,而不是参数调得不好看。

## 先说清楚:什么决定观感,什么只是构建前提

这是最容易误读的地方,两件事必须分开:

| | 决定什么 | 具体是什么 |
|---|---|---|
| **观感** | 玻璃长什么样 | ① `blur` / `lens` / `Highlight` 的参数与形状 ② **source 背后内容的对比度** ③ 设备是否支持 RuntimeShader ④ 库版本的视觉 API |
| **构建** | 能不能编译、装上、跑起来 | AGP、Gradle、JDK、compileSdk、依赖坐标、Maven 镜像 |

**SDK / AGP / Gradle / JDK 的版本不会改变玻璃的样子。**改它们只会让工程从"编译不过"变成"编译通过"(或反过来),像素一个都不动。

唯一一处间接耦合:构建环境可能限制你能用哪个库版本(compileSdk 只到 36 就用不了要求 37 的版本),而库版本*可能*带来新的视觉参数。但这仍然是"能不能用某个参数"的问题,不是"环境改变了效果"。实测 `backdrop:1.0.6` 与 `2.0.x` 的 `lens()` 签名完全一致(`float, float, boolean, boolean`),观感能力等同。

所以要复现一个玻璃效果,**照抄下面两节(「effect stack」「三条铁律 2」)就够了**;后面「工程配置」那一节只保证你能把它跑起来。

## 何时用 / 何时不用

用:
- 顶部导航栏、底部标签栏、悬浮操作按钮、弹层、头部折叠时的覆盖条
- 需要"浮在内容之上、但能透出内容"的 chrome
- iOS 风格界面的关键控件

不用:
- 列表行、内容卡片、整屏背景——**玻璃只做浮动层**,铺满屏幕既贵又难看
- 玻璃上再叠玻璃(见「铁律」一节)
- 低端机占比高的场景,除非接受降级成半透明纯色

## 工程配置(只影响能否编译,不影响观感)

**这一节全部是构建问题。**这里任何一行写错只会让编译失败,不会让玻璃变好看或变难看——不要把它当成"复现效果的配方"。

```kotlin
dependencies {
    implementation("io.github.kyant0:backdrop:<latest>")
    // lens() 内部用到 io.github.kyant0:shapes;通常由 backdrop 传递引入,
    // 若解析不到 RoundedRectangularShape 再显式声明
    implementation("io.github.kyant0:shapes:<latest>")
}
```

版本号去 Maven Central 查 `io.github.kyant0:backdrop`,**不要凭记忆填**。选哪个版本只受一个约束——**`minCompileSdk`**——它决定"能不能用",不决定"好不好看":

| backdrop | minCompileSdk | 说明 |
|---|---|---|
| `1.0.6` + `shapes:1.2.0` | **36** | compileSdk 36 项目用这个(已实测跑通) |
| `2.0.0` / `2.0.1` + `shapes:1.2.1` | **37** | 需要 AGP 9.1+ 与 platform 37 |

选错版本会在 `checkDebugAarMetadata` 阶段被挡下,报 "requires ... compile against version 37 or later"。**完整可编译示例见 `references/full-example.md`**(工程配置 + 整页代码,真机验证过)。

**构建环境坑(2026-11 实测)**:AGP 8.7.3 会在这两个库上于 `checkDebugAarMetadata` 抛 `Cannot invoke "java.util.List.get(int)" because "path" is null`,换 **AGP 8.13.x + Gradle 8.13** 即通过。AGP 9.x 会与 `org.jetbrains.kotlin.android` 抢 `kotlin` 扩展(`Cannot add extension with name 'kotlin'`),要么放弃 AGP 9,要么不用 Kotlin 插件。

**设备能力——这一条是真的会影响观感的。** 库内部有守卫,不支持时效果**静默消失**:

```kotlin
if (!isRenderEffectSupported()) return   // blur / colorFilter / opacity 走这里
if (!isRuntimeShaderSupported()) return  // lens / HighlightStyle.Default 走这里
```

好处是不崩,坏处是**在不支持的设备上效果静默消失**——你会得到一个"能跑但没效果"的界面。所以:低端机必须实测,准备好降级方案(见末尾)。

## 心智模型:source 与 surface

两个 Modifier,职责严格分开:

```kotlin
// SOURCE:把某个 composable 的渲染录进一层 graphicsLayer
Modifier.layerBackdrop(backdrop: LayerBackdrop)

// SURFACE:在自己绘制时采样那层录制,过 effect stack,再画到自己背后
Modifier.drawBackdrop(
    backdrop = backdrop,
    shape = { shape },
    effects = { ... },
    highlight = { Highlight.Default },   // 可传 null 关掉
    shadow = { Shadow.Default },         // 可传 null
    innerShadow = null,
    onDrawSurface = { ... },             // 画在玻璃上的自绘内容
)
```

最小闭环:

```kotlin
// LayerBackdrop 的构造函数是 internal,必须用工厂函数
val backdrop = rememberLayerBackdrop()
// import com.kyant.backdrop.backdrops.rememberLayerBackdrop

Box(Modifier.fillMaxSize()) {
    // ① source:内容区,必须是玻璃的兄弟节点
    LazyColumn(Modifier.fillMaxSize().layerBackdrop(backdrop)) { /* ... */ }

    // ② surface:玻璃条,浮在上面
    Row(
        Modifier
            .align(Alignment.BottomCenter)
            .padding(16.dp)
            .clip(RoundedCornerShape(28.dp))
            .drawBackdrop(
                backdrop = backdrop,
                shape = { RoundedCornerShape(28.dp) },
                effects = {
                    blur(12f)
                    lens(refractionHeight = 24f, refractionAmount = 24f)
                }
            )
    ) { /* tabs */ }
}
```

`drawPlainBackdrop` 是同一套 API 去掉 highlight/shadow 的版本,想完全自己控制描边时用。

## effect stack(已核对签名)

```kotlin
fun BackdropEffectScope.blur(radius: Float, edgeTreatment: TileMode = TileMode.Clamp)
fun BackdropEffectScope.lens(refractionHeight: Float, refractionAmount: Float,
                             depthEffect: Boolean = false, chromaticAberration: Boolean = false)
fun BackdropEffectScope.colorFilter(colorFilter: ColorFilter)
fun BackdropEffectScope.opacity(alpha: Float)
```

`BackdropEffectScope` 本身暴露 `size`、`shape`、`layoutDirection`、`padding`、`renderEffect`,所以你能按尺寸写条件逻辑(小控件关折射)。

**顺序有意义**:先 `blur` 再 `lens`,因为 lens 要弯折的是已经模糊过的画面;先 lens 后 blur 会把折射细节糊掉。参考量级(自行按设备调):小按钮 `blur 8–12f`、`lens(12f, 12f)`;大条 `blur 16–24f`、`lens(24f, 24f)`。`depthEffect` 与 `chromaticAberration` 都更贵,先关掉,观感不够再逐个打开。

## 高光:三个内置 style,选错就等于没做

```kotlin
data class Highlight(
    val width: Dp = 0.5f.dp,
    val blurRadius: Dp = width / 2f,
    val alpha: Float = 1f,
    val style: HighlightStyle = HighlightStyle.Default
)
// 便捷值:Highlight.Default / Highlight.Ambient / Highlight.Plain
```

- `HighlightStyle.Default` — **跑 shader**(`DefaultHighlightShaderString`),高光强度取决于边缘法线与光照方向的夹角
- `HighlightStyle.Plain` — `createShader()` 直接 `return null`,**没有任何 shader**,只有一个纯色 + `BlendMode.Plus`(默认白 38% alpha)
- `HighlightStyle.Ambient` — 第三套 shader,更柔的环境光

**这就是"圆按钮没有玻璃感"的根因**:`Default` 的亮度是夹角函数。胶囊按钮两条长边法线共轴,整片同时亮,看起来像描边;圆形按钮的法线扫遍 360°,只有两个方向亮,另外两个方向完全归零,落到 48dp 控件上基本看不见。

修法二选一:
1. **换 `Plain`** —— 没有方向性,任何形状均匀发亮
2. **留 `Default` 但把 edge 加宽** —— `Highlight(width = 1.dp)`(默认 0.5dp 太细,细到只剩几个像素)

同一屏里的玻璃元素要**统一用同一种策略**,否则胶囊和圆钮的质感会打架。

## 三条铁律

**1. source 不能是 surface 的祖先。**
`layerBackdrop` 录制子树,`drawBackdrop` 读那层录制。如果 surface 长在 source 的子树里,录制就依赖自己的输出 —— 渲染反馈循环,shader 在 draw pass 里挂掉或者画面全黑。

正确做法:source 和 surface 是**兄弟**,用 `Box` + `align()` 摆;如果玻璃条压在列表上方,在 source 的 `Column` 里用 `Spacer` 预留那条 strip。

**2. 空 source 会给出一块灰饼;而"没有对比度的 source"会给你一块**看起来像塑料的**平板。**

这是两条不同的失败,第二条更常见也更容易被误判成"参数没调好":

- **完全空**:背后什么都没有 → 只剩 surface 底色,一枚灰硬币。修法:铺一层 `matchParentSize()` 的地色 Box 当 source。
- **有内容但很平**(纯色页、只有渐变的背景、空白列表区):玻璃照样没有任何东西可弯,`blur`/`lens` 全都"生效了但看不出来"。**症状是"玻璃看起来是平的、像半透明塑料"。**

判据很简单:**玻璃正下方那一块区域,必须存在明暗或色相的突变**。没有就制造出来:

- 让内容铺满整屏(卡片数量要够),而不是让玻璃浮在一片空白上
- 把内容区的底部 padding 收小,让卡片能滚到玻璃下方
- 实在没有内容时,给页面铺一个带对比的底(渐变 + 一两块高饱和色斑),别给纯色

实测对照:同一套参数,4 张卡片时标签栏几乎看不出效果,8 张卡片时折射立刻明显。**这不是参数问题,是 source 问题。**

**3. 玻璃只做浮动层,且不能叠。**
玻璃叠玻璃 = 双重模糊 + 视觉糊成一片。同一层级只允许一层玻璃;模态可以用玻璃,但它下面的内容层必须是普通表面。

## 玻璃内部的元素:别叠玻璃,用"材质片"

标签栏里被选中的那一项、玻璃条上的徽标、按钮上的角标——这些都在玻璃**内部**。直觉是再给它包一层 `drawBackdrop`,但那违反铁律 3(玻璃叠玻璃),而且技术上会让它采样到的是源内容而不是身下的玻璃,画面会出现一块"锐利的洞"。

正确做法是把它做成**材质片(材质感靠光,不靠采样)**:

```kotlin
// 选中态:上亮下暗的渐变 + 顶部亮边,模拟"浮在玻璃上的一小块材质"
Modifier
    .background(
        Brush.verticalGradient(listOf(White.copy(alpha = 0.34f), White.copy(alpha = 0.10f))),
        RoundedCornerShape(29.dp),
    )
    .border(
        width = 0.8.dp,
        brush = Brush.verticalGradient(listOf(White.copy(alpha = 0.65f), White.copy(alpha = 0.05f))),
        shape = RoundedCornerShape(29.dp),
    )
```

三条经验:
- **渐变方向就是光照方向**:上亮下暗 → 读作"受光面";反过来会读作凹槽
- **描边也用渐变**:顶部亮、底部几乎透明,才有玻璃边缘的"光边"感;纯色描边会显得是塑料贴纸
- **alpha 不要太低**:在已经压暗的玻璃上,低于 0.25 的白色几乎看不出来

## 进阶能力:交互玻璃与高级场景(按需深入)

主文档(本文件)只覆盖**静态玻璃**。官方 Catalog 还有一整套"让玻璃活起来"的实现,已蒸馏成三份配方文档——**用户提出下列需求时,先读对应文档再动手**:

| 用户想要 | 先读 | 内容 |
|---|---|---|
| 按压形变、拖拽跟随、松手回弹、光斑跟手指、速度交接(惯性) | `references/interactive-glass.md` | 三件套(DampedDragAnimation/InteractiveHighlight/observe-only 手势)机制 + layerBlock 形变公式 + 弹簧参数选型表 |
| 复刻玻璃按钮/开关/滑块/标签栏 | `references/component-recipes.md` | 五个官方组件的配方卡 + **1.0.6 无 vibrancy 的兼容签名** + 双状态源同步等易错点 |
| 控制中心(陀螺仪高光)、放大镜、自适应明暗玻璃、玻璃弹窗、渐进模糊、SDF 锁屏 | `references/scene-playbook.md` | 13 个演示场景的逐卡拆解:API 组合 + 核心思路 + 参数源文件索引 |

**能力边界(诚实声明)**:
- 交互玻璃与场景配方的**参数签名来自官方 Catalog(2.x)**;1.0.6 的兼容替代方案已在文档中标注,但 `interactive-glass.md` 的陀螺仪替代实现尚未真机验证(标为 open)。
- SDF 纹理锁屏在 1.0.6 上**不可复刻**(依赖 2.x 的 surface 绘制钩子),文档里明确说了复刻到哪一层为止。

## 组件:库不提供,自己写

README 明确写了 **"The library does not include any high-level components"**。官方示例 App 里有可直接参考/复制的实现(Apache-2.0):

| 组件 | 路径 |
|---|---|
| LiquidButton | `app/src/commonMain/kotlin/com/kyant/backdrop/catalog/components/LiquidButton.kt` |
| LiquidToggle | `.../LiquidToggle.kt` |
| LiquidSlider | `.../LiquidSlider.kt` |
| LiquidBottomTabs | `.../LiquidBottomTabs.kt` |
| LiquidBottomTab | `.../LiquidBottomTab.kt` |

**复制这些组件的合规要求(Apache-2.0)**:保留版权声明、附 LICENSE 全文、注明你改了什么。注意库源码**文件头没有版权注释**,只有仓库根一个 LICENSE —— 你 vendor 的时候要自己补 attribution,别以为没写就不管。

另外:单独抄某个 `effects/*.kt` 是**行不通**的,它们依赖 `BackdropEffectScope`、`internal/Paint` 这套连接的类型体系;要抄就抄整个 `backdrop/` 模块,否则就老实加依赖。

## 按压与交互

玻璃控件通常自己不做点击,只是视觉外壳。如果给它加了按压缩放/跟随手势的高光:

- 手势识别必须 **observe-only**:不 `consume()`,`awaitFirstDown(requireUnconsumed = false)`,遇到已被消费的 change 立刻退出
- 否则外壳在动、里面的按钮不响应,用户会觉得"点不动"

## 性能

玻璃是这个库里最贵的东西,每帧 采样 + 模糊 + 折射。

- 一个屏幕**最多一到两处**玻璃
- `lens` 的 `refractionHeight/Amount` 直接决定 shader 开销,先小后大
- 列表里**绝对不要**放玻璃行
- 滚动时如果 source 是列表,`layerBackdrop` 每帧都在重录 —— 大列表上这是主要成本
- 掉帧了先怀疑 `depthEffect` / `chromaticAberration`,这两个默认关不是没有原因

## 降级策略

`isRenderEffectSupported()` / `isRuntimeShaderSupported()` 为 false 时,效果会静默消失。所以显式做降级:

```kotlin
val glassSupported = remember { isRuntimeShaderSupported() }

val surfaceModifier = if (glassSupported) {
    Modifier.drawBackdrop(backdrop, { shape }, { blur(12f); lens(20f, 20f) })
} else {
    Modifier.background(Color.White.copy(alpha = 0.72f), shape)  // 普通半透明
}
```

同屏内所有玻璃元素**用同一个判定结果**,否则会出现"有的玻璃有的不是"的割裂感。

## 验收清单

玻璃效果模拟器上看不出来,**必须真机**。

- [ ] 深色/浅色两套主题下,玻璃上的文字都可读(不达 4.5:1 就加 scrim 或调 `Highlight(alpha=)` / 背景层不透明度)
- [ ] 玻璃下方**真的有内容**可折射;纯色区域是否出现了灰饼
- [ ] 圆形与胶囊控件的高光强度是否一致(不一致就是 `HighlightStyle` 选错)
- [ ] source 与 surface 是兄弟关系(`grep` 一下所有 `layerBackdrop(` 的包围块)
- [ ] 滚动时玻璃条不会跟着滚走,且不挡内容
- [ ] 快速反复进出该页面,无 shader 崩溃、无内存增长
- [ ] 低端机/模拟器(minSdk 边界)上降级分支生效且不难看
- [ ] 打开 GPU 渲染剖析,确认没有持续超过 16ms 的帧

## 常见症状速查

| 症状 | 先查 |
|---|---|
| 玻璃是平的圆角矩形,没有折射 | `lens()` 是否被 `isRuntimeShaderSupported()` 拦掉了;参数是否 ≤ 0 |
| **玻璃看起来像半透明塑料,参数怎么加都没用** | **source 正下方没有对比度** → 让内容铺满整屏、减小底部 padding(见铁律 2) |
| 大面积条(标签栏/导航栏)存在感很弱 | 参数按面积给:70dp 高的条用 `blur 24–26f` / `lens 32–34f`,别沿用按钮的 `10f` |
| 玻璃里的选中项没质感 | 别叠玻璃;用渐变+渐变描边做材质片(见「玻璃内部的元素」) |
| 48dp 圆按钮完全没有高光 | `HighlightStyle.Default` 的方向性 → 换 `Plain` 或 `width = 1.dp` |
| 控件渲染成一枚灰饼 | source 是空的 → 给它铺地色,或换不用 glass |
| 整个玻璃区域全黑 / shader 崩 | source 嵌套在 surface 内部(反馈循环) |
| 只有部分设备有效果 | 缺降级分支,老设备 shader 不支持 |
| 文字在玻璃上发灰看不清 | 缺 scrim;或 `colorFilter` / `opacity` 把对比度拉低了 |
| 一滚动就掉帧 | source 是长列表;或 `depthEffect` 开着 |

## 参考

- **完整可运行示例**:`references/full-example.md` —— 工程配置(Gradle/AGP/Compose BOM/依赖坐标)+ 整页代码,真机 Android 16 验证过
- **进阶配方**(见上方「进阶能力」一节):`references/interactive-glass.md` / `references/component-recipes.md` / `references/scene-playbook.md`
- 仓库:https://github.com/Kyant0/AndroidLiquidGlass
- 文档:https://kyant.gitbook.io/backdrop
- 官方 Catalog 示例工程(配方出处,三模块可直接构建):仓库根 `app/` + `androidApp/`;预编译 APK 在 `androidApp/release/`
- 许可:**Apache-2.0**(可商用;复制源码需保留声明 + LICENSE + 标注修改)

## 来源与许可

- 本 skill 的文字、结构与示例代码为**原创内容**,未复制 AndroidLiquidGlass 的任何源码。
- 示例代码**调用**了 Kyant0 的 Backdrop 库(Apache-2.0)。该库**不随本 skill 分发**;使用者需自行通过 Maven Central 引入 `io.github.kyant0:backdrop`,库本身的使用受 Apache-2.0 约束。
- 完整第三方声明见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
- 本 skill 以 **MIT 协议**开源,允许免费使用与商用(见 [LICENSE](LICENSE))。MIT 仅覆盖本 skill 自身的文字与示例代码,**不能**也无法重新授权 Backdrop 库。
- Apple、Liquid Glass、Android、Jetpack Compose 为各自所有者的商标;本 skill 与 Apple、Google、Kyant0 无隶属或背书关系。
