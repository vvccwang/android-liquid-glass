<!-- License: MIT © 2026 Faison — part of the android-liquid-glass skill.
     Example code CALLS io.github.kyant0:backdrop (Apache-2.0), which is NOT
     included or redistributed here. See ../THIRD_PARTY_NOTICES.md -->

# 完整可运行示例(已验证:真机 Android 16 跑通)

> **复现性验证记录(2026-11)**:另建了一个空工程,只照本 skill 的文本重写了一个**布局完全不同**的页面(顶部导航栏 + 底部播放条 + 圆形 FAB),**首次构建即通过(39s,零编译错误)**,真机装包后效果正常。说明下面的配置和规则是可迁移的,不是只对某一个工程有效。

这是"一个页面 + 一个玻璃按钮 + 一个玻璃标签栏"的完整实现,2026-11 在小米 23127PN0CC(Android 16)上实测通过。

> **阅读顺序提示**:下面的 Gradle/AGP/SDK 配置**只决定能不能编译**,不影响玻璃长什么样。真正决定观感的是代码里的参数和 source 背后的内容——也就是文件后半段「这版影响观感的决策」那张表。**换一套构建配置,同一份代码画出来一模一样。**

## 工程配置(示例所用组合——只影响能否编译)

`gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx2560m -Dfile.encoding=UTF-8
org.gradle.java.home=<你的 JDK 21 路径>
android.useAndroidX=true
android.nonTransitiveRClass=true
android.suppressUnsupportedCompileSdk=36
```

`build.gradle.kts`(根):

```kotlin
plugins {
    id("com.android.application") version "8.13.2" apply false
    id("org.jetbrains.kotlin.android") version "2.2.10" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.2.10" apply false
}
```

`app/build.gradle.kts`:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}

android {
    namespace = "com.example.glassbackdrop"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.example.glassbackdrop"
        minSdk = 26
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

kotlin {
    compilerOptions {
        jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17)
    }
}

dependencies {
    implementation(platform("androidx.compose:compose-bom:2025.11.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.foundation:foundation")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.activity:activity-compose:1.13.0")

    implementation("io.github.kyant0:backdrop:1.0.6")
    implementation("io.github.kyant0:shapes:1.2.0")
}
```

要点:
- **Compose BOM 2025.11.00 = Compose 1.9.4**,这是 compileSdk 36 的上限;BOM 2025.12.00 起是 1.10+ 会要求更高 SDK。
- `backdrop:1.0.6` 的 `minCompileSdk = 36`;`2.0.x` 是 37,在本组合下用不了。
- Gradle 用 8.13,JDK 用 21(JDK 17 在这台机器上 TLS 握手会失败)。

`AndroidManifest.xml` 用系统主题即可,不需要 AppCompat:

```xml
<application android:label="Liquid Glass"
             android:theme="@android:style/Theme.Material.NoActionBar">
    <activity android:name=".MainActivity" android:exported="true"
              android:configChanges="orientation|screenSize|uiMode">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
</application>
```

## Activity 代码

```kotlin
package com.example.glassbackdrop

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.kyant.backdrop.backdrops.layerBackdrop
import com.kyant.backdrop.backdrops.rememberLayerBackdrop
import com.kyant.backdrop.drawBackdrop
import com.kyant.backdrop.effects.blur
import com.kyant.backdrop.effects.lens
import com.kyant.backdrop.highlight.Highlight

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent { GlassScreen() }
    }
}

// 8 组高对比渐变:玻璃背后必须有明暗变化,否则玻璃再强也看不出来
private val cardGradients = listOf(
    listOf(Color(0xFF3B82F6), Color(0xFF8B5CF6)),
    listOf(Color(0xFFEC4899), Color(0xFFF97316)),
    listOf(Color(0xFF10B981), Color(0xFF06B6D4)),
    listOf(Color(0xFFF59E0B), Color(0xFFEF4444)),
    listOf(Color(0xFF6366F1), Color(0xFF22D3EE)),
    listOf(Color(0xFFE11D48), Color(0xFFA855F7)),
    listOf(Color(0xFF14B8A6), Color(0xFF84CC16)),
    listOf(Color(0xFFF43F5E), Color(0xFF6366F1)),
)

@Composable
private fun GlassScreen() {
    val backdrop = rememberLayerBackdrop()
    var taps by remember { mutableIntStateOf(0) }
    var tab by remember { mutableIntStateOf(0) }

    Box(Modifier.fillMaxSize().background(Color(0xFF06070C))) {

        // ① SOURCE —— 必须与下面两个玻璃面互为兄弟
        Column(
            Modifier
                .fillMaxSize()
                .layerBackdrop(backdrop)
                .background(
                    Brush.verticalGradient(
                        0.0f to Color(0xFF0A1030),
                        0.45f to Color(0xFF1E2C6B),
                        1.0f to Color(0xFF48205F),
                    )
                )
                .verticalScroll(rememberScrollState())
                .padding(horizontal = 20.dp)
                .padding(top = 84.dp, bottom = 150.dp)
        ) {
            Text("Liquid Glass", color = Color.White, fontSize = 34.sp, fontWeight = FontWeight.Bold)
            Text("Backdrop · 折射玻璃演示", color = Color(0xFFAFC0EE), fontSize = 14.sp)
            Spacer(Modifier.height(24.dp))

            // 卡片数量要够:内容必须铺满整屏,标签栏背后才有东西可折射
            cardGradients.forEachIndexed { i, colors ->
                Row(
                    Modifier
                        .fillMaxWidth()
                        .padding(vertical = 6.dp)
                        .background(Brush.linearGradient(colors), RoundedCornerShape(22.dp))
                        .padding(14.dp),
                    verticalAlignment = Alignment.CenterVertically,
                ) {
                    Box(
                        Modifier.size(40.dp)
                            .background(Color.White.copy(alpha = 0.22f), RoundedCornerShape(13.dp))
                    )
                    Spacer(Modifier.size(12.dp))
                    Column {
                        Text("卡片 ${i + 1}", color = Color.White, fontSize = 15.sp, fontWeight = FontWeight.SemiBold)
                        Text(
                            "玻璃下方需要有内容才能折射",
                            color = Color.White.copy(alpha = 0.78f),
                            fontSize = 11.sp,
                        )
                    }
                }
            }
        }

        // ② SURFACE —— 玻璃按钮
        Box(
            Modifier
                .align(Alignment.Center)
                .drawBackdrop(
                    backdrop = backdrop,
                    shape = { RoundedCornerShape(28.dp) },
                    effects = {
                        blur(10f)
                        lens(refractionHeight = 18f, refractionAmount = 18f)
                    },
                    highlight = { Highlight(width = 1.dp) },
                )
                .clickable { taps++ }
                .padding(horizontal = 34.dp, vertical = 18.dp)
        ) {
            Text(
                text = if (taps == 0) "点我" else "已点击 $taps 次",
                color = Color.White,
                fontSize = 18.sp,
                fontWeight = FontWeight.SemiBold,
            )
        }

        // ③ SURFACE —— 玻璃标签栏
        Row(
            Modifier
                .align(Alignment.BottomCenter)
                .fillMaxWidth()
                .navigationBarsPadding()
                .padding(horizontal = 16.dp)
                .padding(bottom = 22.dp)
                .height(70.dp)
                .drawBackdrop(
                    backdrop = backdrop,
                    shape = { RoundedCornerShape(35.dp) },
                    effects = {
                        // 大面积 chrome 要比按钮更重,否则整条看起来是"平的一块"
                        blur(26f)
                        lens(refractionHeight = 34f, refractionAmount = 34f)
                    },
                    highlight = { Highlight(width = 1.2.dp) },
                )
                .padding(7.dp),
            horizontalArrangement = Arrangement.spacedBy(4.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            listOf("首页", "发现", "我的").forEachIndexed { index, title ->
                val selected = tab == index
                Box(
                    Modifier
                        .weight(1f)
                        .fillMaxHeight()
                        .then(
                            if (selected) {
                                // 选中态:不再叠一层玻璃(违反铁律 3),而是用
                                // "上亮下暗的渐变 + 顶部亮边" 做出浮起的小材质片
                                Modifier
                                    .background(
                                        Brush.verticalGradient(
                                            listOf(
                                                Color.White.copy(alpha = 0.34f),
                                                Color.White.copy(alpha = 0.10f),
                                            )
                                        ),
                                        RoundedCornerShape(29.dp),
                                    )
                                    .border(
                                        width = 0.8.dp,
                                        brush = Brush.verticalGradient(
                                            listOf(
                                                Color.White.copy(alpha = 0.65f),
                                                Color.White.copy(alpha = 0.05f),
                                            )
                                        ),
                                        shape = RoundedCornerShape(29.dp),
                                    )
                            } else Modifier
                        )
                        .clickable { tab = index },
                    contentAlignment = Alignment.Center,
                ) {
                    Text(
                        text = title,
                        color = if (selected) Color.White else Color.White.copy(alpha = 0.60f),
                        fontSize = 15.sp,
                        fontWeight = if (selected) FontWeight.SemiBold else FontWeight.Normal,
                    )
                }
            }
        }
    }
}
```

## 这版影响观感的决策(照抄时别省)

下面每一条都会改变像素;上面那些版本号不会。

| 决策 | 为什么 |
|---|---|
| 卡片放 8 张 | 内容不满一屏时,标签栏背后是纯渐变 → 玻璃看起来是平的一块 |
| 底部 padding 150dp | 让卡片能滚到标签栏下方,滚动时折射更明显 |
| 标签栏 `blur 26 / lens 34` | 面积越大越需要更强的折射,否则"存在感"很弱 |
| 按钮 `blur 10 / lens 18` | 小控件别用大参数,会糊成一团 |
| 选中态不用玻璃 | 玻璃叠玻璃违反铁律 3;渐变+描边足够做出"浮起"感 |
| `Highlight(width = 1.dp)` | 默认 0.5dp 在 70dp 高的条上几乎看不见 |
