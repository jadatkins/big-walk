**Note: This description is bilingual. The Chinese section is provided below the English section.**  
**说明：本描述为中英双语版本，中文内容位于英文内容下方。**

---

# Mod Settings Menu (English)
Adds a Mod Settings button below Settings in the main menu and to the in-game pause menu, providing a unified configuration interface for mods.

## Main Features
- Shows configurable mod names on the left, the selected mod's settings in the center, and its information on the right.
- Supports localized config sections, names and values, custom ordering, per-mod restore defaults, and optional Nexus Mods or Thunderstore links.

## For Players
This mod does not change gameplay by itself. It provides a common settings screen for compatible mods.  
Only loaded mods with at least one configuration entry appear in the list. The available settings depend on the mods you have installed.

## Mod Author API
- Registration is optional. A loaded BepInEx IL2CPP plugin with normal `Config.Bind` entries is detected automatically.
- Use `ModSettingsRegistry.Register` only when you want to provide custom metadata or Nexus Mods and Thunderstore links.
- Use `ModSettingsTags.Section` and `ModSettingsTags.Entry` in `ConfigDescription` when categories or entries need a custom order or numeric sliders need a custom step. Lower order values appear first; unspecified order values use `1000`.

Complete example mods: [Big Walk example mods](https://github.com/ibox233/Ice_Box_Studio_ExampleMod/tree/83923eb57d15f5727c94e80c813710e2c087b7c0/Big%20Walk)

Sorting example:

```csharp
Config.Bind("General", "Enabled", true, new ConfigDescription("Enable this mod.", null, ModSettingsTags.Section("General", order: 10), ModSettingsTags.Entry(order: 10)));
Config.Bind("General", "SpeedMultiplier", 1f, new ConfigDescription("Adjust the speed multiplier.", new AcceptableValueRange<float>(0.1f, 5f), ModSettingsTags.Entry(order: 20, sliderStep: 0.25d)));
```

Put the section tag on any one entry in that section. Put an entry tag on each setting that needs a custom order or slider step. Without `sliderStep`, integer sliders use `1` and decimal sliders use `0.1`.

Apply configuration changes:

```csharp
private ConfigEntry<bool> _enabled;

public override void Load()
{
    _enabled = Config.Bind("General", "Enabled", true, "Enable this mod.");
    _enabled.SettingChanged += OnEnabledChanged;
}

private void OnEnabledChanged(object sender, EventArgs args)
{
    ApplyEnabled(_enabled.Value);
}
```

Use BepInEx's built-in `ConfigEntry.SettingChanged` or `ConfigFile.SettingChanged` to apply changed settings. The menu does not send network messages or make settings take effect by itself. Implement `ApplyEnabled` and any multiplayer synchronization in your own mod.

Optional hard dependency example:

```csharp
using BepInEx;
using BepInEx.Configuration;
using BepInEx.Unity.IL2CPP;
using ModSettingsMenu.Api;
using UnityEngine;

[BepInPlugin(PluginInfo.PLUGIN_GUID, PluginInfo.PLUGIN_NAME, PluginInfo.PLUGIN_VERSION)]
[BepInDependency(ModSettingsMenu.PluginInfo.PLUGIN_GUID)]
public sealed class MyPlugin : BasePlugin
{
    public override void Load()
    {
        Config.Bind("General", "Enabled", true, "Enable this mod.");
        Config.Bind("General", "SpeedMultiplier", 1f, new ConfigDescription("Adjust the speed multiplier.", new AcceptableValueRange<float>(0.1f, 5f)));
        Config.Bind("Controls", "QuickAction", KeyCode.F7, "Choose the quick action key.");

        ModSettingsRegistry.Register(
            PluginInfo.PLUGIN_GUID,
            new ModSettingsModOptions
            {
                Name = "My Mod",
                Description = "A short description shown in this mod's information panel.",
                Author = "Author Name",
                Version = PluginInfo.PLUGIN_VERSION,
                NexusModsId = 6,
                ThunderstoreTeam = "MyTeam",
                ThunderstoreModName = "MyMod"
            });
    }
}
```

## Localization API Example
For localized config entries and mod metadata, register the Big Walk Localization API JSON file from the same directory as your plugin DLL. `ModSettingsModOptions.Description` is saved when the mod is registered, so register again after `LocalizationApi.LanguageChanged` to refresh the current-language description.

```csharp
using System.IO;
using System.Reflection;
using BepInEx;
using BepInEx.Configuration;
using BepInEx.Unity.IL2CPP;
using ModSettingsMenu.Api;
using BigWalkLocalizationAPI.Api;
using UnityEngine;

[BepInPlugin(PluginInfo.PLUGIN_GUID, PluginInfo.PLUGIN_NAME, PluginInfo.PLUGIN_VERSION)]
[BepInDependency(ModSettingsMenu.PluginInfo.PLUGIN_GUID)]
[BepInDependency(BigWalkLocalizationAPI.PluginInfo.PLUGIN_GUID)]
public sealed class MyPlugin : BasePlugin
{
    private ModLocalizer _localizer;

    public override void Load()
    {
        _localizer = LocalizationApi.For(PluginInfo.PLUGIN_GUID);
        string directory = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);
        _localizer.RegisterJson(Path.Combine(directory, "MyPlugin.Localization.json"));
        LocalizationApi.LanguageChanged += OnLanguageChanged;

        Config.Bind("General", "Enabled", true, _localizer.Config("config.enabled", 10, "General", "config.section.general", 10));
        Config.Bind("General", "SpeedMultiplier", 1f, _localizer.Config("config.speed_multiplier", 20, "General", "config.section.general", 10, new AcceptableValueRange<float>(0.1f, 5f), sliderStep: 0.25d));
        Config.Bind("Controls", "QuickAction", KeyCode.F7, _localizer.Config("config.quick_action", 10, "Controls", "config.section.controls", 20));

        RegisterSettings();
    }

    private void RegisterSettings()
    {
        ModSettingsRegistry.Register(
            PluginInfo.PLUGIN_GUID,
            new ModSettingsModOptions
            {
                Name = "My Mod",
                LocalizedName = () => _localizer.GetLocalizedText("mod.name"),
                Description = _localizer.GetLocalizedText("mod.description"),
                Author = "Author Name",
                Version = PluginInfo.PLUGIN_VERSION
            });
    }

    private void OnLanguageChanged(string language)
    {
        RegisterSettings();
    }
}
```

`MyPlugin.Localization.json` must contain an `English` object. Add any other supported game language such as `SimplifiedChinese`; missing languages and keys fall back to English.

```json
{
  "English": {
    "mod.name": "My Mod",
    "mod.description": "A short description of my mod.",
    "config.section.general": "General",
    "config.section.controls": "Controls",
    "config.enabled.name": "Enabled",
    "config.enabled.description": "Enable this mod.",
    "config.speed_multiplier.name": "Speed Multiplier",
    "config.speed_multiplier.description": "Adjust the speed multiplier.",
    "config.quick_action.name": "Quick Action",
    "config.quick_action.description": "Choose the quick action key."
  },
  "SimplifiedChinese": {
    "mod.name": "我的模组",
    "mod.description": "我的模组简介。",
    "config.section.general": "常规",
    "config.section.controls": "控制",
    "config.enabled.name": "启用",
    "config.enabled.description": "启用这个模组。",
    "config.speed_multiplier.name": "速度倍率",
    "config.speed_multiplier.description": "调整速度倍率。",
    "config.quick_action.name": "快速操作",
    "config.quick_action.description": "选择快速操作按键。"
  }
}
```

`ThunderstoreTeam` and `ThunderstoreModName` must be provided together and may only contain ASCII letters, numbers, and underscores. The example opens `https://thunderstore.io/c/big-walk/p/MyTeam/MyMod/`. Do not pass a full URL.

Version and author are shown on separate lines. Registered Nexus Mods and Thunderstore packages appear as separate `NEXUS` and `THUNDERSTORE` buttons.

The menu chooses controls from the config entry type and acceptable values:

- `bool`: off/on buttons
- `UnityEngine.KeyCode`: native-style key binding row
- Numeric value with `AcceptableValueRange`: slider with an optional custom step
- Enum or `AcceptableValueList`: left/right selector
- Other supported serialized values: text field

## Compatibility
- Game version: `1.4.8+`

## Bug Reports & Feature Suggestions
If you have any questions or feature suggestions, please submit them through [GitHub Issues](https://github.com/ibox233/IceBox_Mods_Issues), contact me on Discord at iceboxcool, or email me at 764884112@qq.com or ibox2333@gmail.com.

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

# Mod Settings Menu (中文)
在主菜单的设置下方和游戏内暂停菜单中添加模组设置按钮，为模组提供统一的配置界面。

## 主要功能
- 左侧显示可配置的模组名称，中间显示当前模组的设置项，右侧显示模组信息。
- 支持配置分类、名称和值的本地化、自定义排序、单模组恢复默认以及可选的 Nexus Mods 或 Thunderstore 链接。

## 给玩家
本模组不会自行修改游戏玩法，只为兼容的模组提供统一设置界面。  
列表中只会显示已经加载并且至少包含一个配置项的模组。实际可用设置由你安装的其他模组决定。

## 模组作者 API
- 注册不是必需的。已加载的 BepInEx IL2CPP 模组只要使用普通 `Config.Bind` 配置项，就会被自动识别。
- 只有需要自定义元数据或提供 Nexus Mods、Thunderstore 链接时，才需要调用 `ModSettingsRegistry.Register`。
- 需要自定义分类、配置项顺序或数值滑条步进时，在 `ConfigDescription` 中使用 `ModSettingsTags.Section` 和 `ModSettingsTags.Entry`。排序数值越小越靠前，未指定排序时使用 `1000`。

完整例子模组：[Big Walk 例子模组](https://github.com/ibox233/Ice_Box_Studio_ExampleMod/tree/83923eb57d15f5727c94e80c813710e2c087b7c0/Big%20Walk)

排序示例：

```csharp
Config.Bind("General", "Enabled", true, new ConfigDescription("启用这个模组。", null, ModSettingsTags.Section("General", order: 10), ModSettingsTags.Entry(order: 10)));
Config.Bind("General", "SpeedMultiplier", 1f, new ConfigDescription("调整速度倍率。", new AcceptableValueRange<float>(0.1f, 5f), ModSettingsTags.Entry(order: 20, sliderStep: 0.25d)));
```

每个分类只需在其中任意一个配置项上放置分类标签。需要自定义顺序或滑条步进的配置项分别添加配置项标签。未指定 `sliderStep` 时，整数滑条步进为 `1`，小数滑条步进为 `0.1`。

应用配置修改：

```csharp
private ConfigEntry<bool> _enabled;

public override void Load()
{
    _enabled = Config.Bind("General", "Enabled", true, "启用这个模组。");
    _enabled.SettingChanged += OnEnabledChanged;
}

private void OnEnabledChanged(object sender, EventArgs args)
{
    ApplyEnabled(_enabled.Value);
}
```

使用 BepInEx 内置的 `ConfigEntry.SettingChanged` 或 `ConfigFile.SettingChanged` 应用修改后的配置。菜单不发送网络消息，也不会自动让配置生效。`ApplyEnabled` 和任何多人同步逻辑均由你的模组自行实现。

可选硬依赖示例：

```csharp
using BepInEx;
using BepInEx.Configuration;
using BepInEx.Unity.IL2CPP;
using ModSettingsMenu.Api;
using UnityEngine;

[BepInPlugin(PluginInfo.PLUGIN_GUID, PluginInfo.PLUGIN_NAME, PluginInfo.PLUGIN_VERSION)]
[BepInDependency(ModSettingsMenu.PluginInfo.PLUGIN_GUID)]
public sealed class MyPlugin : BasePlugin
{
    public override void Load()
    {
        Config.Bind("General", "Enabled", true, "启用这个模组。");
        Config.Bind("General", "SpeedMultiplier", 1f, new ConfigDescription("调整速度倍率。", new AcceptableValueRange<float>(0.1f, 5f)));
        Config.Bind("Controls", "QuickAction", KeyCode.F7, "选择快速操作按键。");

        ModSettingsRegistry.Register(
            PluginInfo.PLUGIN_GUID,
            new ModSettingsModOptions
            {
                Name = "我的模组",
                Description = "显示在这个模组信息面板中的简短介绍。",
                Author = "作者名称",
                Version = PluginInfo.PLUGIN_VERSION,
                NexusModsId = 6,
                ThunderstoreTeam = "MyTeam",
                ThunderstoreModName = "MyMod"
            });
    }
}
```

## 本地化 API 示例
需要本地化配置项和模组信息时，应从模组 DLL 所在目录加载 Big Walk Localization API 的 JSON 文件。`ModSettingsModOptions.Description` 会在注册时保存，因此必须在 `LocalizationApi.LanguageChanged` 后重新注册，才能刷新为当前语言的简介。

```csharp
using System.IO;
using System.Reflection;
using BepInEx;
using BepInEx.Configuration;
using BepInEx.Unity.IL2CPP;
using ModSettingsMenu.Api;
using BigWalkLocalizationAPI.Api;
using UnityEngine;

[BepInPlugin(PluginInfo.PLUGIN_GUID, PluginInfo.PLUGIN_NAME, PluginInfo.PLUGIN_VERSION)]
[BepInDependency(ModSettingsMenu.PluginInfo.PLUGIN_GUID)]
[BepInDependency(BigWalkLocalizationAPI.PluginInfo.PLUGIN_GUID)]
public sealed class MyPlugin : BasePlugin
{
    private ModLocalizer _localizer;

    public override void Load()
    {
        _localizer = LocalizationApi.For(PluginInfo.PLUGIN_GUID);
        string directory = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location);
        _localizer.RegisterJson(Path.Combine(directory, "MyPlugin.Localization.json"));
        LocalizationApi.LanguageChanged += OnLanguageChanged;

        Config.Bind("General", "Enabled", true, _localizer.Config("config.enabled", 10, "General", "config.section.general", 10));
        Config.Bind("General", "SpeedMultiplier", 1f, _localizer.Config("config.speed_multiplier", 20, "General", "config.section.general", 10, new AcceptableValueRange<float>(0.1f, 5f), sliderStep: 0.25d));
        Config.Bind("Controls", "QuickAction", KeyCode.F7, _localizer.Config("config.quick_action", 10, "Controls", "config.section.controls", 20));

        RegisterSettings();
    }

    private void RegisterSettings()
    {
        ModSettingsRegistry.Register(
            PluginInfo.PLUGIN_GUID,
            new ModSettingsModOptions
            {
                Name = "我的模组",
                LocalizedName = () => _localizer.GetLocalizedText("mod.name"),
                Description = _localizer.GetLocalizedText("mod.description"),
                Author = "作者名称",
                Version = PluginInfo.PLUGIN_VERSION
            });
    }

    private void OnLanguageChanged(string language)
    {
        RegisterSettings();
    }
}
```

`MyPlugin.Localization.json` 必须包含 `English` 对象，也可以添加 `SimplifiedChinese` 等游戏支持的语言；缺少当前语言或目标键时会回退到英语。

```json
{
  "English": {
    "mod.name": "My Mod",
    "mod.description": "A short description of my mod.",
    "config.section.general": "General",
    "config.section.controls": "Controls",
    "config.enabled.name": "Enabled",
    "config.enabled.description": "Enable this mod.",
    "config.speed_multiplier.name": "Speed Multiplier",
    "config.speed_multiplier.description": "Adjust the speed multiplier.",
    "config.quick_action.name": "Quick Action",
    "config.quick_action.description": "Choose the quick action key."
  },
  "SimplifiedChinese": {
    "mod.name": "我的模组",
    "mod.description": "我的模组简介。",
    "config.section.general": "常规",
    "config.section.controls": "控制",
    "config.enabled.name": "启用",
    "config.enabled.description": "启用这个模组。",
    "config.speed_multiplier.name": "速度倍率",
    "config.speed_multiplier.description": "调整速度倍率。",
    "config.quick_action.name": "快速操作",
    "config.quick_action.description": "选择快速操作按键。"
  }
}
```

`ThunderstoreTeam` 和 `ThunderstoreModName` 必须成对填写，并且只能包含 ASCII 字母、数字和下划线。上面的例子会打开 `https://thunderstore.io/c/big-walk/p/MyTeam/MyMod/`，不需要传入完整 URL。

版本和作者分别显示在两行。注册 Nexus Mods 和 Thunderstore 信息后，会显示独立的 `NEXUS` 与 `THUNDERSTORE` 按钮。

菜单根据配置项类型和可接受值选择控件：

- `bool`：开/关按钮
- `UnityEngine.KeyCode`：原生风格按键绑定行
- 带 `AcceptableValueRange` 的数值：支持可选自定义步进的滑条
- 枚举或 `AcceptableValueList`：左右选择器
- 其他支持序列化的值：文本输入框

## 兼容性
- 游戏版本：`1.4.8+`

## Bug 提交 & 新功能建议
如果你有任何问题或新功能建议，请通过 [GitHub Issues](https://github.com/ibox233/IceBox_Mods_Issues) 提交，也可以通过 Discord：iceboxcool，或邮箱 764884112@qq.com、ibox2333@gmail.com 联系我。
