# 场景配方书:Catalog 十三个演示页的实现索引

Catalog(`app/src/commonMain/.../catalog/destinations/`)的 14 个演示页 = 14 个"已解决场景"。本文每个场景一张卡:**用到什么 API → 核心思路 → 参数去哪个文件抄 → 真机注意事项**。

> 所有路径相对仓库根:`https://github.com/Kyant0/AndroidLiquidGlass`。演示页文件在 `app/src/commonMain/kotlin/com/kyant/backdrop/catalog/destinations/`。

## 场景索引(按实现难度排序)

| # | 场景 | 文件 | 难度 | 一句话 |
|---|---|---|---|---|
| 1 | 静态玻璃面 | (主 SKILL 已覆盖) | ★ | blur+lens+highlight 三件套 |
| 2 | 按压形变按钮 | ButtonsContent.kt | ★★ | layerBlock 四件套 |
| 3 | 玻璃弹窗 | DialogContent.kt | ★★ | 弹窗 + scrim + 玻璃面板 |
| 4 | 滚动列表上的玻璃 | LazyScrollContainerContent.kt / ScrollContainerContent.kt | ★★ | 滚动时玻璃下内容变化 |
| 5 | 开关/滑块/标签栏 | ToggleContent.kt / SliderContent.kt / BottomTabsContent.kt | ★★★ | 见 component-recipes.md |
| 6 | 渐进模糊 | ProgressiveBlurContent.kt | ★★★ | blur 分段递减 |
| 7 | 自适应明暗玻璃 | AdaptiveLuminanceGlassContent.kt | ★★★ | 采样背景亮度自动切换 |
| 8 | 陀螺仪高光 | ControlCenterContent.kt | ★★★★ | 重力方向驱动高光 |
| 9 | 放大镜 | MagnifierContent.kt | ★★★★ | lens 放大 + 三层 backdrop 合成 |
| 10 | SDF 纹理锁屏 | LockScreenContent.kt | ★★★★★ | colorControls 重调壁纸 + SDF 纹理 |

## 卡片 3:玻璃弹窗(DialogContent)

**思路**:模态 = scrim(压暗)+ 玻璃面板(浮起)。要点:
- scrim 独立一层,不参与玻璃的 backdrop 录制(否则面板折射的是"被压暗后的世界",双重变暗)
- 弹窗玻璃的 `layerBlock` 做入场 scale(0.9→1)+ 按压行为,接 `interactive-glass.md` 配方
- 边缘 rim 高光不要省——弹窗玻璃最容易被误做成"纯色圆角框"

**参数来源**:DialogContent.kt 的 scrim alpha 与面板 effects 签名。

## 卡片 6:渐进模糊(ProgressiveBlurContent)

**思路**:整个"模糊带"不是一次 blur,而是**沿高度切成 N 条,每条 blur 半径线性递减**——模拟 iOS 状态栏下面那种"越往下越清晰"的过渡。
- 实现上是一个 For 循环画 N 个 `drawBackdrop` 条,或单条玻璃配 `graphicsLayer` 分段
- 性能:N 控制在 6~10 条;每条都开 lens 太贵,只有最顶部 1~2 条开

**1.0.6 注意**:库内置没有现成 progressive API,这里纯手写分段。参数抄 ProgressiveBlurContent.kt 的分段数与半径曲线。

## 卡片 7:自适应明暗玻璃(AdaptiveLuminanceGlassContent)

**思路**:玻璃该变黑还是变白,取决于它**底下内容**的亮度:
1. 对 source 层采样平均亮度(官方做法:把内容画进小尺寸 Bitmap,`Color.luminance()` 求均值——不需要读像素,用 `ImageBitmap` 降采样)
2. `luminance > 0.5f` → 玻璃用深色调(白底上玻璃要压暗才可见);反之用浅调
3. 切换用 `animateFloatAsState` 过渡,**不瞬切**(否则背景闪一下就翻面)

**适用**:壁纸/大图上浮玻璃的所有场景(相册、音乐、地图卡片)。参数抄 AdaptiveLuminanceGlassContent.kt 的阈值 0.5f 与过渡时长。

## 卡片 8:陀螺仪高光(ControlCenterContent)—— 复刻 iOS 控制中心

**思路**:玻璃高光方向 = 手机倾斜方向。
```kotlin
val uiSensor = rememberUISensor()
// 2.x 一行接入:HighlightStyle.Default(angle = uiSensor.gravityAngle, falloff = 2f)
// 1.0.6:HighlightStyle 无 angle 参数 → 用 gravity(Offset) 驱动方向性渐变替代(interactive-glass.md 有方案)
```
- `UISensor` 是 expect/actual:commonMain 声明接口,androidMain 用 `SensorManager.TYPE_ACCELEROMETER` 算重力角,**每帧 mutableFloatStateOf 更新**(不是 Flow,避免重组风暴)
- ControlCenter 页把一整屏小玻璃块(tile)全部接同一个 uiSensor —— 转手机时**整屏高光同步转**,这是"统一光源"的物理直觉:所有玻璃共享一个太阳

**真机**:模拟器没加速度计,必须真机;倾斜要慢,光斑弹簧(0.5f/300f)会自然拖尾。

## 卡片 9:放大镜(MagnifierContent)—— lens 的正确打开方式

**思路**:lens 的 `refractionHeight/Amount` 不只是"边缘弯折装饰",组合到位就是一个**真放大镜**:
- 放大 = 大 `refractionAmount` + 圆形形状 + 小尺寸(放大率来自曲率,尺寸越小曲率越大)
- **三层 backdrop 合成**(官方签名):
```kotlin
val combined = rememberCombinedBackdrop(pageBackdrop, contentBackdrop, cursorBackdrop)
Modifier.drawBackdrop(backdrop = combined, shape = { CircleShape }, effects = { lens(...) })
```
- 三个源:页面层(背景壁纸)+ 内容层(被放大的文字)+ 光标层(自己的高亮圈也要被玻璃折射)
- 色散:`lens(depthEffect=false, chromaticAberration=true)` —— 边缘出现 RGB 分离的彩虹边

**这是理解 lens 的分水岭**:之前只把它当"边缘装饰",这里它成了**光学器件**。MagnifierContent.kt 三层合成的顺序和每层的录制范围是唯一必读源码。

## 卡片 10:SDF 纹理锁屏(LockScreenContent)—— 最高级的一页

**思路**:锁屏时间文字"刻"在玻璃里:
- `colorControls(brightness=-0.1f, contrast=0.75f, saturation=1.5f)` 把壁纸压暗、压对比、提饱和 —— 这是玻璃后面那层"环境"的调色配方
- 时间数字用 **SDF(signed distance field)shader** 渲染成玻璃表面的凹凸纹理,配合 highlight 出现"文字嵌进玻璃"的立体感
- SDF 部分依赖 2.x 的 surface 绘制钩子与自定义 shader(`utils/SdfShader.kt`),**1.0.6 无法完整复刻**;能复刻的是调色那半(colorFilter 替代)

**判定**:此页当作"观感上限"参考,1.0.6 环境下复刻到调色层即可,别在 SDF 上耗时间。

## 通用提醒

- **演示页源码 = 参数金矿**:每个文件顶部的常量(圆角、blur、lens、颜色)都是作者调好的成品值,直接抄比自己试快
- **重参数 vs 轻参数**:Catalog 用 `blur(2dp)` 这种小值是建立在 `vibrancy()` 之上的;1.0.6 复刻统一按主 SKILL 的量级表换算
- 每复刻一个场景,回主 SKILL 的验收清单过一遍(尤其真机帧率)
