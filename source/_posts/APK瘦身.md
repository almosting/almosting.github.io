---
title: APK 瘦身
date: 2019-04-30 17:06:19
categories: Android
tags: 优化
---
## APK 结构

在讨论如何减小应用的大小之前，了解应用 APK 的结构会很有帮助。APK 文件由 Zip 压缩文件（其中包含构成应用的所有文件）组成。这些文件包括 Java 类文件、资源文件和包含已编译资源的文件。

APK 包含以下目录：

- `META-INF/`：包含 `CERT.SF` 和 `CERT.RSA` 签名文件，以及 `MANIFEST.MF` 清单文件。
- `assets/`：包含应用的资源；应用可以使用 [AssetManager](https://developer.android.com/reference/android/content/res/AssetManager.html?hl=zh-CN) 对象检索这些资源。
- `res/`：包含未编译到 `resources.arsc` 中的资源。
- `lib/`：包含特定于处理器软件层的编译代码。此目录包含每种平台类型的子目录，如 `armeabi`、`armeabi-v7a`、`arm64-v8a`、`x86`、`x86_64` 和 `mips`。

APK 还包含以下文件。在这些文件中，只有 `AndroidManifest.xml` 是必需的。

- `resources.arsc`：包含已编译的资源。此文件包含 `res/values/` 文件夹的所有配置中的 XML 内容。打包工具会提取此 XML 内容，将其编译为二进制文件形式，并将相应内容进行归档。此内容包括语言字符串和样式，以及未直接包含在 `resources.arsc` 文件中的内容（例如布局文件和图片）的路径。
- `classes.dex`：包含以 Dalvik/ART 虚拟机可理解的 DEX 文件格式编译的类。
- `AndroidManifest.xml`：包含核心 Android 清单文件。此文件列出了应用的名称、版本、访问权限和引用的库文件。该文件使用 Android 的二进制 XML 格式。

## 减少资源数量和大小

APK 的大小会影响应用加载速度、使用的内存量以及消耗的电量。减小 APK 大小的一种简单方法是减少其包含的资源数量和大小。具体来说，您可以移除应用不再使用的资源，并且可以用可伸缩的 [Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable.html?hl=zh-CN) 对象取代图片文件。此部分将讨论上述这些方法，以及另外几种可减少应用中的资源以减小 APK 总大小的方法。

### 移除未使用的资源

[lint](https://developer.android.com/studio/write/lint.html?hl=zh-CN) 工具是 Android Studio 中附带的静态代码分析器，可检测到 `res/` 文件夹中未被代码引用的资源。当 `lint` 工具发现项目中有可能未使用的资源时，会显示一条消息，如下例所示。

```xml
res/layout/preferences.xml: Warning: The resource R.layout.preferences appears to be unused [UnusedResources]
```

**注意**：`lint` 工具不会扫描 `assets/` 文件夹、通过反射引用的资源或已链接到应用的库文件。此外，它也不会移除资源，只会提醒您它们的存在。

您添加到代码的库可能包含未使用的资源。如果您在应用的 `build.gradle` 文件中启用了 [shrinkResources](https://developer.android.com/studio/build/shrink-code.html?hl=zh-CN)，则 Gradle 可以代表您自动移除资源。

```groovy
    android { 
        { 
            release { 
                minifyEnabled true 
                shrinkResources true 
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro' 
            } 
        } 
    }
```

要使用 `shrinkResources`，您必须启用代码压缩功能。在编译过程中，首先，[ProGuard](https://developer.android.com/studio/build/shrink-code.html?hl=zh-CN) 会移除未使用的代码，但会保留未使用的资源。然后，Gradle 会移除未使用的资源。

要详细了解 ProGuard 以及 Android Studio 帮助您减小 APK 大小的其他方式，请参见 [压缩代码和资源](https://developer.android.com/studio/build/shrink-code.html?hl=zh-CN)。

在 Android Gradle Plugin 0.7 及更高版本中，您可以声明应用支持的配置。Gradle 会使用 `resConfig` 和 `resConfigs` 风格以及 `defaultConfig` 选项将这些信息传递给编译系统。随后，编译系统会阻止来自其他不受支持配置的资源出现在 APK 中，从而减小 APK 的大小。有关此功能的详情，请参见 [移除未使用的备用资源](https://developer.android.com/studio/build/shrink-code.html?hl=zh-CN#unused-alt-resources)。

### 减少库中的资源使用

在开发 Android 应用时，您通常需要使用外部库来提高应用的可用性和多功能性。例如，您可以引用 [Android 支持库](https://developer.android.com/topic/libraries/support-library/index.html?hl=zh-CN) 来提升旧设备上的用户体验，也可以使用 [Google Play 服务](https://developers.google.com/android/guides/overview?hl=zh-CN) 检索应用中文本的自动翻译。

如果库是为服务器或桌面设备设计的，则它可能包含应用不需要的许多对象和方法。要仅包含您的应用所需的库部分，您可以编辑库的文件（如果相应的许可允许您修改库）。您还可以使用其他适合移动设备的库来为应用添加特定功能。

**注意**：[ProGuard](https://developer.android.com/studio/build/shrink-code.html?hl=zh-CN) 可以清理随库导入的一些不必要代码，但它无法移除库的大型内部依赖项。

### 仅支持特定密度

Android 支持数量非常广泛的设备（包含各种屏幕密度）。在 Android 4.4（API 级别 19）及更高版本中，框架支持各种密度：`ldpi`、`mdpi`、`tvdpi`、`hdpi,`、`xhdpi`、`xxhdpi` 和 `xxxhdpi`。尽管 Android 支持所有这些密度，但您无需将光栅化资源导出到每个密度。

如果您知道只有一小部分用户拥有具有特定密度的设备，请考虑是否需要将这些密度捆绑到您的应用中。如果您不添加用于特定屏幕密度的资源，Android 会自动扩缩最初为其他屏幕密度设计的现有资源。

如果您的应用仅需要扩缩的图片，则可以通过在 `drawable-nodpi/` 中使用图片的单个变体来节省更多空间。我们建议每个应用至少包含一个 `xxhdpi` 图片变体。

有关屏幕密度的详情，请参见 [屏幕尺寸和密度](https://developer.android.com/about/dashboards/index.html?hl=zh-CN#Screens)。

### 使用可绘制对象

某些图片不需要静态图片资源；框架可以在运行时改为动态绘制图片。[Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable.html?hl=zh-CN) 对象（XML 中为 `<shape>`）会占用 APK 中的少量空间。此外，XML [Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable.html?hl=zh-CN) 对象会生成符合 Material Design 准则的单色图片。

### 重复使用资源

您可以为图片的变体添加单独的资源，例如同一图片经过色调调整、阴影设置或旋转的版本。不过，我们建议您重复使用同一组资源，并在运行时根据需要对其进行自定义。

Android 提供了一些实用程序来更改资源的颜色，每个实用程序在 Android 5.0（API 级别 21）及更高版本上都使用 `android:tint` 和 `tintMode` 属性。对于较低版本的平台，则使用 [ColorFilter](https://developer.android.com/reference/android/graphics/ColorFilter.html?hl=zh-CN) 类。

您还可以省略仅是另一个资源的旋转等效项的资源。以下代码段提供了一个示例，展示了通过绕图片中心位置旋转 180 度，将“拇指向上”变为“拇指向下”：

```xml
    <?xml version="1.0" encoding="utf-8"?> 
    <rotate xmlns:android="http://schemas.android.com/apk/res/android" 
    		android:drawable="@drawable/ic_thumb_up" 
    		android:pivotX="50%" 
    		android:pivotY="50%" 
    		android:fromDegrees="180" />
```

### 从代码进行渲染

您还可以通过按一定程序渲染图片来减小 APK 大小。按一定程序渲染可以释放空间，因为您不再在 APK 中存储图片文件。

### 压缩 PNG 文件

`aapt` 工具可以在编译过程中通过无损压缩来优化放置在 `res/drawable/` 中的图片资源。例如，`aapt` 工具可以通过调色板将不需要超过 256 种颜色的真彩色 PNG 转换为 8 位 PNG。这样做会生成质量相同但内存占用量更小的图片。

请记住，`aapt` 具有以下限制：

- `aapt` 工具不会压缩 `asset/` 文件夹中包含的 PNG 文件。
- 图片文件需要使用 256 种或更少的颜色才可供 `aapt` 工具进行优化。
- `aapt` 工具可能会增大已压缩 PNG 文件。为防止出现这种情况，您可以使用 Gradle 中的 `cruncherEnabled` 标记为 PNG 文件停用此过程：

  ` aaptOptions { cruncherEnabled = false }`

### 压缩 PNG 和 JPEG 文件

您可以使用 [pngcrush](http://pmt.sourceforge.net/pngcrush/)、[pngquant](https://pngquant.org/) 或 [zopflipng](https://github.com/google/zopfli) 等工具减小 PNG 文件的大小，同时不损失画质。所有这些工具都可以减小 PNG 文件的大小，同时保持肉眼感知的画质不变。

`pngcrush` 工具尤为有效：该工具会迭代 PNG 过滤器和 zlib(Deflate) 参数，使用过滤器和参数的每个组合来压缩图片。然后，它会选择可产生最小压缩输出的配置。

要压缩 JPEG 文件，您可以使用 [packJPG](http://www.elektronik.htw-aalen.de/packjpg/) 和 [guetzli](https://github.com/google/guetzli) 等工具。

### 使用 WebP 文件格式

如果以 Android 3.2（API 级别 13）及更高版本为目标，您还可以使用 [WebP](https://developers.google.com/speed/webp/?hl=zh-CN) 文件格式的图片（而不是使用 PNG 或 JPEG 文件）。WebP 格式提供有损压缩（如 JPEG）以及透明度（如 PNG），不过与 JPEG 或 PNG 相比，这种格式可以提供更好的压缩效果。

您可以使用 Android Studio 将现有 BMP、JPG、PNG 或静态 GIF 图片转换为 WebP 格式。有关详情，请参见 [使用 Android Studio 创建 WebP 图片](https://developer.android.com/studio/write/convert-webp.html?hl=zh-CN)。

**注意**：仅当 [启动器图标](https://material.io/guidelines/style/icons.html#icons-product-icons) 使用 PNG 格式时，Google Play 才会接受 APK。

### 使用矢量图形

您可以使用矢量图形创建与分辨率无关的图标和其他可伸缩媒体。使用这些图形可以极大地减少 APK 占用的空间。矢量图片在 Android 中以 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html?hl=zh-CN) 对象的形式表示。借助 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html?hl=zh-CN) 对象，100 字节的文件可以生成与屏幕大小相同的清晰图片。

不过，系统渲染每个 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html?hl=zh-CN) 对象需要花费大量时间，而较大的图片则需要更长的时间才能显示在屏幕上。因此，请考虑仅在显示小图片时使用这些矢量图形。

有关使用 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html?hl=zh-CN) 对象的详情，请参见 [使用可绘制资源](https://developer.android.com/training/material/drawables.html?hl=zh-CN)。

将矢量图形用于动画图片

请勿使用 [AnimationDrawable](https://developer.android.com/reference/android/graphics/drawable/AnimationDrawable.html?hl=zh-CN) 创建逐帧动画，因为这样做需要为动画的每个帧添加单独的位图文件，而这会大大增加 APK 的大小。

您应改为使用 [AnimatedVectorDrawableCompat](https://developer.android.com/reference/android/support/graphics/drawable/AnimatedVectorDrawableCompat.html?hl=zh-CN) 创建 [动画矢量可绘制资源](https://developer.android.com/training/material/animations.html?hl=zh-CN#AnimVector)。

## 减少原生和 Java 代码

您可以使用多种方法来减小应用中 Java 和原生代码库的大小。

移除不必要的生成代码

确保了解自动生成的任何代码所占用的空间。例如，许多协议缓冲区工具会生成过多的方法和类，这可能会使应用的大小增加一倍或两倍。

避免使用枚举

单个枚举会使应用的 `classes.dex` 文件增加大约 1.0 到 1.4 KB 的大小。这些增加的大小会快速累积，产生复杂的系统或共享库。如果可能，请考虑使用 `@IntDef` 注释和 [ProGuard](https://developer.android.com/studio/build/shrink-code.html?hl=zh-CN) 移除枚举并将它们转换为整数。此类型转换可保留枚举的各种安全优势。

减小原生二进制文件的大小

如果您的应用使用原生代码和 Android NDK，您还可以通过优化代码来减小发布版本应用的大小。移除调试符号和不提取原生库是两项很实用的技术。

### 移除调试符号

如果应用正在开发中且仍需要调试，则使用调试符号非常合适。您可以使用 Android NDK 中提供的 `arm-eabi-strip` 工具从原生库中移除不必要的调试符号。之后，您便可以编译发布版本。

### 避免解压缩原生库

在编译应用的发布版本时，您可以通过在应用清单的 [application](https://developer.android.com/guide/topics/manifest/application-element.html?hl=zh-CN) 元素中设置 `android:extractNativeLibs="false"`，打包 APK 中未压缩的 **`.so`** 文件。停用此标记可防止 [PackageManager](https://developer.android.com/reference/android/content/pm/PackageManager.html?hl=zh-CN) 在安装过程中将 `.so` 文件从 APK 复制到文件系统，并具有减小应用更新的额外好处。

## 维持多个精简 APK

APK 可能包含用户下载但从不使用的内容，例如其他语言或针对屏幕密度的资源。要确保为用户提供最小的下载文件，您应该 [使用 Android App Bundle](https://developer.android.com/topic/performance/reduce-apk-size?hl=zh-CN#app_bundle) 将应用上传到 Google Play。通过上传 App Bundle，Google Play 能够针对每位用户的设备配置生成并提供经过优化的 APK，因此用户只需下载运行您的应用所需的代码和资源。您无需再编译、签署和管理多个 APK 以支持不同的设备，而用户也可以获得更小、更优化的下载文件包。

如果您不打算将应用发布到 Google Play，则可以将应用细分为多个 APK，并按屏幕尺寸或 GPU 纹理支持等因素进行区分。

当用户下载您的应用时，他们的设备会根据设备的功能和设置接收正确的 APK。这样，设备不会接收用于设备所不具备的功能的资源。例如，如果用户具有 `hdpi` 设备，则不需要您可能会为具有更高密度显示器的设备提供的 `xxxhdpi` 资源。

有关详情，请参见 [配置 APK 拆分](https://developer.android.com/studio/build/configure-apk-splits.html?hl=zh-CN) 和 [维持多个 APK](https://developer.android.com/training/multiple-apks/index.html?hl=zh-CN)。

## 减少 Native 体积

```groovy
android {
    ...
    defaultConfig {
        ...
        //在 defaultConfig 节点下配置 externalNativeBuild 块
        externalNativeBuild {
            // For ndk-build, instead use ndkBuild {}
            cmake { //指定 cmake 的编译选项
                // Sets optional flags for the C compiler.
                cFlags "-fvisibility=hidden  -ffunction-sections -fdata-sections"
                // Sets optional flags for the C++ compiler.
                cppFlags "-fvisibility=hidden  -ffunction-sections -fdata-sections -std=c++11"

                arguments "-DANDROID_TOOLCHAIN=gcc"//设置使用 gcc 编译，cmake 默认使用 clang 编译
            }
        }
    }

    buildTypes {...}
  
}
```
