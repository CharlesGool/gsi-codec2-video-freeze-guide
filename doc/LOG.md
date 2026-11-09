---
name: project-log
description: 项目缺陷, 限制, 决策和交接
metadata:
  version: "1.0.0"
  lang: "zh-CN"
---

# 日志

## 多语言

**简体中文** | [English](en/LOG.md) | [Español](es/LOG.md)

## 文档

- 项目概览: [README](../README.md)

- 设计思路: [DESIGN](DESIGN.md)

- 项目状态: [LOG](LOG.md)
- 历史记录: [HISTORY](HISTORY.md)
- 变更日志: [CHANGELOG](CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](THIRD_PARTY_NOTICES.md)

## 记录

- [历史记录](HISTORY.md)
- [变更日志](CHANGELOG.md)

## 缺陷

暂无.

## 限制

- 教程第 5 节写进程内存的具体命令在案例中没有记录, 只给出了地址计算方法和示例.
- 原因 B 的定位方法只在高通 vendor 上验证过.

## 决策

| 日期 | 决策 | 状态 | 原因 |
| --- | --- | --- | --- |
| 2026-11-09 | 教程和补丁分成两个公开仓库, 排查记录放在教程仓库 | 接受 | 与 tango-porting-guide 和 tb321fu-tango-patch 的组织方式一致. |

## 交接

- 当前分支: `main`.
- 已完成: 简体中文教程 `doc/debug-guide.md`; 排查记录 `doc/research.md` (由 TB321FU-GSI-Mods 中的 ANALYSIS.md 去掉本地路径和未发布脚本名后整理); 核心文档. 英文和西班牙语文档仍是模板.
- 检查: 文档链接检查; 本项目没有构建和测试.
- 剩余步骤:
  - [ ] 翻译英文和西班牙语文档.
- 阻碍: 无.
- 临时项目规则: 未发现.
