# gsi-codec2-video-freeze-guide

**简体中文** | [English](doc/en/README.md) | [Español](doc/es/README.md)

高通 HIDL Codec2 vendor (如 Android 14 原厂 vendor) 上刷视频时声音继续, 画面定格, 拖进度条才恢复; 本项目说明两个独立原因的排查和修复: Android 17 GSI 的 libgui 切换 Surface 时清零 generation, 以及高通解码器申请输出缓冲超时后放弃重试, 与占满的输入槽死锁 (与 GSI 无关).

[![License](https://img.shields.io/badge/License-GPL--3.0--or--later-orange)](LICENSE)

## 文档

- 项目概览: [README](README.md)

- 设计思路: [DESIGN](doc/DESIGN.md)

- 项目状态: [LOG](doc/LOG.md)
- 历史记录: [HISTORY](doc/HISTORY.md)
- 变更日志: [CHANGELOG](doc/CHANGELOG.md)

- 第三方声明: [THIRD_PARTY_NOTICES](doc/THIRD_PARTY_NOTICES.md)

## 简介

- [调试教程](doc/debug-guide.md): 按"确认问题, 抓日志, 定位原因, 热补丁验证, 做成持久补丁, 刷机复查"的顺序, 说明两个独立原因的排查和修复方法:
  - 原因 A: Android 17 `libgui` 的 `Surface::disconnect` 清零 generation, HIDL Codec2 切换 Surface 时缓冲迁移全部失败. 只出现在 Android 17 system 上, 不限于 Pixel 提取的 GSI; 播放中切换全屏或复用播放器时触发.
  - 原因 B: 高通解码器申请输出缓冲超时后放弃重试, 与占满的输入槽形成死锁. 是 vendor 自身的行为, 与 GSI 和框架版本无关 (Android 15 GSI 上同样出现); 预加载下一个视频的播放器 (如 X 用的 ExoPlayer) 刷视频时触发, 是最常见的情况.
- [TB321FU 排查记录](doc/research.md): 教程所用案例的原始记录, 包括 logcat 证据链, 反汇编与 AOSP 源码对照, uprobe 跟踪和验证数据.

案例是拯救者 Y700 三代 (TB321FU, 高通 SM8650, Android 14 vendor) 上的 Pixel Android 17 GSI, 修复后已在真机验证. 补丁实现见 [tb321fu-video-freeze-patch](https://github.com/CharlesGool/tb321fu-video-freeze-patch).

## 要求

- 有 root 的设备 (热补丁验证和 uprobe 跟踪需要)
- 电脑: `adb`, `python3` 和 `pyelftools`, 反汇编工具 (`llvm-objdump` 或 capstone)
- 能自己构建或修改 system 镜像

## 安装

无需安装, 直接阅读 [调试教程](doc/debug-guide.md).

### 快速安装

无需安装.

### 常规安装

无需安装.

## 指南

1. 按教程第 1 节确认设备和现象符合本教程.
2. 按第 2 节抓日志, 用日志特征判断是原因 A, 原因 B 还是两者都有.
3. 按对应章节定位和修复, 先做第 5 节的热补丁验证, 再按第 6 节做成持久补丁.

## 升级

暂无.

## 卸载

无需安装, 直接删除即可.

## 致谢

- [AOSP](https://source.android.com/) `frameworks/native` (`Surface.cpp`), `frameworks/av` (`MediaCodec`, Codec2 客户端): 用于对照分析, 排查记录中引用了少量片段

## 许可证

本项目以 GNU General Public License v3.0 或更新版本授权, SPDX 标识符 `GPL-3.0-or-later`, 全文见 [LICENSE](LICENSE). 引用的 AOSP 代码片段的授权见 [THIRD_PARTY_NOTICES](doc/THIRD_PARTY_NOTICES.md).
