---
name: project-design
description: 项目架构与设计约束
metadata:
  version: "1.0.0"
  lang: "zh-CN"
---

# gsi-codec2-video-freeze-guide — 设计

## 多语言

**简体中文** | [English](en/DESIGN.md) | [Español](es/DESIGN.md)

## 文档

- 项目概览: [README](../README.md)

- 设计思路: [DESIGN](DESIGN.md)

- 项目状态: [LOG](LOG.md)
- 历史记录: [HISTORY](HISTORY.md)
- 变更日志: [CHANGELOG](CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## 设计目标

- [x] 说明在 HIDL Codec2 vendor 上运行新 GSI 时视频声音继续, 画面定格的排查, 定位和修复方法, 不依赖具体设备.
- [x] 保留一个完整案例 (TB321FU) 的原始记录作为证据.
- 非目标: 提供可直接使用的补丁. 补丁在 [tb321fu-video-freeze-patch](https://github.com/CharlesGool/tb321fu-video-freeze-patch).

## 架构

本项目只有文档和第三方许可证原文.

| 顶层目录 | 职责 |
| --- | --- |
| `doc/` | 项目文档, 教程 `debug-guide.md` 和排查记录 `research.md` |
| `third_party/` | 第三方许可证原文, 每个来源一个子目录 |

本项目没有界面文本, 也没有构建输出.

## 设计约束

- 教程中的命令只写已在案例中使用过的, 或标明为示例.
- 排查记录**禁止**包含本地路径, 私有链接和设备上的个人数据.
