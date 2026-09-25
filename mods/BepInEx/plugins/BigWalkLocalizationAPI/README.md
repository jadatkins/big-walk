**Note: This description is bilingual. The Chinese section is provided below the English section.**  
**说明：本描述为中英双语版本，中文内容位于英文内容下方。**

---

# BigWalkLocalizationAPI (English)
BigWalkLocalizationAPI is a shared localization API for Big Walk mods.

## What This Mod Does
- Loads external JSON localization files for individual mods.
- Uses Big Walk's active language and raises a language-change event for mod UI.
- Provides one-call localization for config sections, names, descriptions, and enum or list values.

## For Players
This mod is an API/dependency. It does not add gameplay features by itself.  
Install it only when another Big Walk mod lists BigWalkLocalizationAPI as a requirement.

## For Mod Authors
Reference `BigWalkLocalizationAPI.dll` and add a hard dependency:

```csharp
using System;
using System.IO;
using System.Reflection;
using BepInEx;
using BepInEx.Configuration;
using BepInEx.Unity.IL2CPP;
using BigWalkLocalizationAPI.Api;

[BepInPlugin(PluginInfo.PLUGIN_GUID, "My Mod", "1.0.0")]
[BepInDependency(BigWalkLocalizationAPI.PluginInfo.PLUGIN_GUID)]
public sealed class MyPlugin : BasePlugin
{
    public override void Load()
    {
        string directory = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);
        ModLocalizer localizer = LocalizationApi.For(PluginInfo.PLUGIN_GUID);
        localizer.RegisterJson(Path.Combine(directory, "MyMod.Localization.json"));
        Config.Bind("General", "Enabled", true, I18n.Localizer.Config("config.enabled", 20, "General", "config.section.general", 10));
        LocalizationApi.LanguageChanged += OnLanguageChanged;
    }

    private static void OnLanguageChanged(string language)
    {
        // Refresh this mod's existing UI text here.
    }
}
```

Load the JSON file from the directory containing your mod DLL. This works with generated mod-manager folder names:

```csharp
using System.IO;
using System.Reflection;
using BigWalkLocalizationAPI.Api;

internal static class I18n
{
    private const string FileName = "MyMod.Localization.json";
    private static readonly ModLocalizer _localizer = Load();

    internal static ModLocalizer Localizer
    {
        get { return _localizer; }
    }

    internal static string Text(string key, params object[] args)
    {
        return _localizer.GetLocalizedText(key, args);
    }

    private static ModLocalizer Load()
    {
        string directory = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);
        ModLocalizer localizer = LocalizationApi.For(PluginInfo.PLUGIN_GUID);
        localizer.RegisterJson(Path.Combine(directory, FileName));
        return localizer;
    }
}
```

The `Config` call uses the config section and key to find localized names, descriptions, and enum or acceptable-list values. Values are normalized to lowercase snake_case suffixes. Numeric ranges can also pass an optional `sliderStep` for Mod Settings Menu:

```csharp
Config.Bind("General", "Mode", MyMode.SafeMode, I18n.Localizer.Config("config.mode", 20, "General", "config.section.general", 10));
Config.Bind("General", "Speed", 1f, I18n.Localizer.Config("config.speed", 30, "General", "config.section.general", 10, new AcceptableValueRange<float>(0.5f, 3f), sliderStep: 0.25d));
```

Single JSON file example:

```json
{
  "English": {
    "config.section.general": "General",
    "config.mode.name": "Mode",
    "config.mode.description": "Select the operating mode.",
    "config.mode.safe_mode": "Safe"
  },
  "SimplifiedChinese": {
    "config.section.general": "常规",
    "config.mode.name": "模式",
    "config.mode.description": "选择运行模式。",
    "config.mode.safe_mode": "安全"
  }
}
```

Every localization file must contain an `English` object. If the current language or requested key is missing, the API tries `English`; if the key is still missing, it returns the key itself.

## Supported Language Names
- English: `English`
- French: `French`
- Italian: `Italian`
- German: `German`
- European Spanish: `EuropeanSpanish`
- Simplified Chinese: `SimplifiedChinese`
- Japanese: `Japanese`
- Russian: `Russian`
- Korean: `Korean`
- Brazilian Portuguese: `BrazilianPortuguese`
- Polish: `Polish`
- Turkish: `Turkish`
- Czech: `Czech`
- Latin American Spanish: `LatinAmericanSpanish`
- Traditional Chinese: `TraditionalChinese`

## Compatibility
- Game version: 1.4.8+

## Bug Reports & Feature Suggestions
If you have any questions or feature suggestions, please submit them through [GitHub Issues](https://github.com/ibox233/IceBox_Mods_Issues), contact me on Discord at `iceboxcool`, or email `764884112@qq.com` or `ibox2333@gmail.com`.

---

<div align="center">

If you enjoy my mods, feel free to support me! / 如果你喜欢我的模组，请支持我一下吧！

<a href="https://ko-fi.com/I3I1WNP4">
  <img src="https://storage.ko-fi.com/cdn/kofi3.png?v=6" width="320" alt="Ko-fi">
</a>
&nbsp;
<a href="https://www.ifdian.net/a/iceboxstudio">
  <img src="https://temp-rr-icebox.cn-nb1.rains3.com/ifdian.png" width="320" alt="爱发电">
</a>

</div>

---

# BigWalkLocalizationAPI (中文)
BigWalkLocalizationAPI 是供 Big Walk 模组使用的共享本地化 API。

## 主要功能
- 让各个模组加载自己的外部 JSON 本地化文件。
- 读取 Big Walk 当前语言，并在语言切换后通知模组刷新 UI。
- 一次注册配置分类、名称、说明以及枚举或列表值的本地化。

## 给玩家
这是一个 API/依赖模组，本身不会添加玩法内容。  
只有其他 Big Walk 模组要求安装 BigWalkLocalizationAPI 时才需要安装它。

## 给模组作者
在项目中引用 `BigWalkLocalizationAPI.dll`，并添加硬依赖：

```csharp
using System;
using System.IO;
using System.Reflection;
using BepInEx;
using BepInEx.Configuration;
using BepInEx.Unity.IL2CPP;
using BigWalkLocalizationAPI.Api;

[BepInPlugin(PluginInfo.PLUGIN_GUID, "My Mod", "1.0.0")]
[BepInDependency(BigWalkLocalizationAPI.PluginInfo.PLUGIN_GUID)]
public sealed class MyPlugin : BasePlugin
{
    public override void Load()
    {
        string directory = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);
        ModLocalizer localizer = LocalizationApi.For(PluginInfo.PLUGIN_GUID);
        localizer.RegisterJson(Path.Combine(directory, "MyMod.Localization.json"));
        Config.Bind("General", "Enabled", true, I18n.Localizer.Config("config.enabled", 20, "General", "config.section.general", 10));
        LocalizationApi.LanguageChanged += OnLanguageChanged;
    }

    private static void OnLanguageChanged(string language)
    {
        // 在这里刷新模组已经创建的 UI 文本。
    }
}
```

从模组 DLL 自己所在的目录加载 JSON。这样即使模组管理器生成了不同的插件文件夹名，也能正确找到语言文件：

```csharp
using System.IO;
using System.Reflection;
using BigWalkLocalizationAPI.Api;

internal static class I18n
{
    private const string FileName = "MyMod.Localization.json";
    private static readonly ModLocalizer _localizer = Load();

    internal static ModLocalizer Localizer
    {
        get { return _localizer; }
    }

    internal static string Text(string key, params object[] args)
    {
        return _localizer.GetLocalizedText(key, args);
    }

    private static ModLocalizer Load()
    {
        string directory = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);
        ModLocalizer localizer = LocalizationApi.For(PluginInfo.PLUGIN_GUID);
        localizer.RegisterJson(Path.Combine(directory, FileName));
        return localizer;
    }
}
```

`Config` 会按配置分类和配置键查找名称、说明以及枚举或可接受值列表的本地化，并自动使用小写下划线后缀。数值范围还可以传入可选的 `sliderStep`，供 Mod Settings Menu 设置自定义滑条步进：

```csharp
Config.Bind("General", "Mode", MyMode.SafeMode, I18n.Localizer.Config("config.mode", 20, "General", "config.section.general", 10));
Config.Bind("General", "Speed", 1f, I18n.Localizer.Config("config.speed", 30, "General", "config.section.general", 10, new AcceptableValueRange<float>(0.5f, 3f), sliderStep: 0.25d));
```

本地化文件示例：

```json
{
  "English": {
    "config.section.general": "General",
    "config.mode.name": "Mode",
    "config.mode.description": "Select the operating mode.",
    "config.mode.safe_mode": "Safe"
  },
  "SimplifiedChinese": {
    "config.section.general": "常规",
    "config.mode.name": "模式",
    "config.mode.description": "选择运行模式。",
    "config.mode.safe_mode": "安全"
  }
}
```

每个语言文件都必须包含 `English` 对象。当前语言或目标 key 缺失时，API 会尝试 `English`；如果英语中仍缺少该 key，则直接返回 key 本身。

## 支持的语言名
- 英语：`English`
- 法语：`French`
- 意大利语：`Italian`
- 德语：`German`
- 欧洲西班牙语：`EuropeanSpanish`
- 简体中文：`SimplifiedChinese`
- 日语：`Japanese`
- 俄语：`Russian`
- 韩语：`Korean`
- 巴西葡萄牙语：`BrazilianPortuguese`
- 波兰语：`Polish`
- 土耳其语：`Turkish`
- 捷克语：`Czech`
- 拉丁美洲西班牙语：`LatinAmericanSpanish`
- 繁体中文：`TraditionalChinese`

## 兼容性
- 游戏版本：1.4.8+

## Bug 提交 & 新功能建议
如果你有任何问题或新功能建议，请通过 [GitHub Issues](https://github.com/ibox233/IceBox_Mods_Issues) 提交，也可以通过 Discord：`iceboxcool`，或邮箱 `764884112@qq.com`、`ibox2333@gmail.com` 联系我。
