# Android Gacha Bot Porting

米哈游游戏助手 Android 移植方案：基于对 [MAA-Meow](https://github.com/Aliothmoon/MAA-Meow)、[BetterGI](https://github.com/babalae/better-genshin-impact)、[March7thAssistant](https://github.com/moesnow/March7thAssistant) 三个项目的深入代码分析，撰写的技术调研与移植方案文档。

## 文档

- [移植方案全文](PORTING_PLAN.md) — 包含架构分析、技术路径、可行性评估

## 背景

| 项目 | 游戏 | 平台 | 语言 | Stars |
|---|---|---|---|---|
| MAA-Meow | 明日方舟 | Android | Kotlin | ~1K |
| BetterGI | 原神 | Windows | C# | ~14K |
| March7thAssistant | 星穹铁道 | Windows | Python | ~11K |

MAA-Meow 已完成将 MAA（明日方舟助手）从 PC 移植到 Android。本文档分析其移植策略，并探讨将 BetterGI 和 March7thAssistant 移植到 Android 的可行路径。
