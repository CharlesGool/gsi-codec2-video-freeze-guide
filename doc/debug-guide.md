---
name: debug-guide
description: 高通 HIDL Codec2 vendor 上视频声音继续, 画面定格的两个原因的排查和修复方法
metadata:
  version: "1.1.0"
  lang: "zh-CN"
---

# 调试教程

本文说明在高通 HIDL Codec2 vendor 上播放视频出现"声音继续, 画面定格, 拖进度条恢复"的排查和修复方法. 案例是拯救者 Y700 三代 (TB321FU, 高通 SM8650, Android 14 vendor) 刷 Pixel 提取的 Android 17 GSI, 原始记录见 [TB321FU 排查记录](research.md), 补丁实现见 [tb321fu-video-freeze-patch](https://github.com/CharlesGool/tb321fu-video-freeze-patch).

同一现象在案例中有两个独立原因, 修好其中一个不会让现象消失. 本文按"确认问题, 抓证据, 定位原因, 热补丁验证, 做成持久补丁, 刷机复查"的顺序写.

## 1. 确认问题类型

共同现象: 视频画面停在某一帧, 声音继续播放; 拖动进度条或重新打开后恢复. 拖进度条会 flush 并重建解码器的缓冲池, 能恢复说明卡在缓冲区而不是数据源.

两个原因的适用条件不同, 一台设备可能同时有两个:

| | 原因 A: Surface generation 被清零 | 原因 B: vendor 解码器放弃申请输出缓冲 |
| --- | --- | --- |
| system | Android 17 (`libgui` 的 `Surface::disconnect` 会清零 generation; Android 15 和 16 没有) | 与框架版本无关 (Android 14 和 17 框架里相关的 `CCodecBufferChannel` 代码一致); 案例中 Android 15 和 17 GSI 都出现 |
| vendor | Codec2 HAL 是 HIDL (`android.hardware.media.c2@1.x`), 且客户端走 Surface 包装的缓冲迁移路径 (案例中由 read-only flag `wb_media_migration` 打开, Pixel 镜像中为开启) | 高通 `libqcodec2_v4l2codec.so` 解码器 (案例为 Android 14 vendor 的 `vendor.qti.media.c2@1.0-service`) |
| 范围 | 不限于 Pixel 提取的 GSI; 其他 Android 17 GSI 是否开启该路径未验证. 使用 AIDL Codec2 的原机 (如 Pixel) 不受影响 | 与 GSI 无关, 是 vendor 自身的行为; 同一 vendor 的原厂系统是否出现未验证 |
| 触发 | 播放中切换输出 Surface: 内嵌和全屏切换, 复用播放器 | 播放器预加载下一个视频 (如 ExoPlayer), 停留 1 秒以上再滑过去 |

查看 HAL 类型 (HIDL 服务出现在 `lshal`, AIDL 服务出现在 `service list`):

```bash
adb shell lshal | grep -i media.c2
adb shell service list | grep -i media.c2
```

## 2. 抓日志

先加大 logcat 缓冲区并清空, 再复现. 两种触发方式都要试, 它们对应两个不同的原因:

- 播放中切换输出 Surface: 内嵌和全屏来回切换, 或在复用播放器的列表里来回滑.
- 预加载后停留: 在一个视频上停留 10 秒以上, 再滑到下一个已经预加载的视频.

```bash
adb logcat -G 32M
adb logcat -c
# 复现
adb logcat -d > freeze.txt
grep -E "generation number mismatch|remote graphic buffer migration|Allocating output buffer timedout|queueBuffer failed|rendring output error" freeze.txt
```

按日志特征分类:

| 日志特征 | 对应 | 见 |
| --- | --- | --- |
| `attachBuffer: generation number mismatch [buffer 0] [queue N]`, `remote graphic buffer migration 0/N` | 原因 A: 切换 Surface 时缓冲迁移失败 | [第 3 节](#3-原因-a-surface-generation-被清零) |
| `Allocating output buffer timedout, Nms elapsed since last input buffer queued (threshold 1000ms)` (高通) | 原因 B: vendor 解码器放弃申请输出缓冲 | [第 4 节](#4-原因-b-vendor-解码器放弃申请输出缓冲) |
| `queueBuffer failed: -32`, `rendring output error -32` | 两者都可能出现, 只说明帧送不上屏 | |

原因 A 只在切换 Surface 时出现, 原因 B 只在解码器长时间收不到输入时出现. 案例中两种同时存在, 修好原因 A 后 X 刷视频仍然会卡, 抓到的就是原因 B.

## 3. 原因 A: Surface generation 被清零

### 3.1 机制

`MediaCodec::connectToSurface()` 切换输出 Surface 时依次执行: 设置 generation N, 断开 Surface, 重新连接. HIDL Codec2 客户端 (`libcodec2_client` 的 `OutputBufferQueue`) 随后把旧缓冲 attach 到新 Surface.

Android 17 的 `libgui` 给 `Surface` 新增了 auto generation update: `attachBuffer` 用 Surface 缓存的 generation 覆盖缓冲自己的值, 而 `disconnect` 会把缓存清零. 结果缓存为 0, BufferQueue 为 N, 每个缓冲都被拒绝. vendor 却认为迁移成功, 缓冲池被占满后停止出帧.

AIDL Codec2 客户端 (`GraphicsTracker`) 会显式调用 `setAutoGenerationUpdate(false)`, 所以使用 AIDL HAL 的原机 (如 Pixel) 没有这个问题. Android 15 和 16 的 `Surface.cpp` 没有这段逻辑.

### 3.2 确认

1. 拉出 `libgui.so`, 找 `Surface::disconnect` 是否写 0 到 generation 字段:

   ```bash
   adb pull /system/lib64/libgui.so
   llvm-objdump -d --disassemble-symbols=_ZN7android7Surface19setGenerationNumberEj libgui.so
   llvm-objdump -d --disassemble-symbols=_ZN7android7Surface10disconnectEiNS_22IGraphicBufferProducer14DisconnectModeE libgui.so
   ```

   `setGenerationNumber` 里的 32 位 `str` 的偏移就是 `mGenerationNumber` 字段. `disconnect` 里出现 `str wzr, [xN, #<同一偏移>]` 就是清零.
2. 对照 AOSP 对应分支的 `frameworks/native/libs/gui/Surface.cpp`, 确认 `disconnect` 里有 `mGenerationNumber = 0`.
3. 确认 HIDL 客户端路径没有调用 `setAutoGenerationUpdate(false)`: 反汇编 `libcodec2_client.so` 的 `OutputBufferQueue::configure`, 和 `GraphicsTracker::configureGraphics` 对比.

也可以直接用补丁工具检查, 它按上面的方法定位:

```bash
python3 patch_libgui_surface_gen.py --check libgui.so
```

工具见 [patch_libgui_surface_gen.py](https://github.com/CharlesGool/tb321fu-video-freeze-patch/blob/main/patch/parts/002.libgui-surface-gen/patch_libgui_surface_gen.py). 输出"未打补丁"表示存在清零指令; 输出"没有清零 generation 的指令"表示这个 libgui 没有原因 A.

### 3.3 修复

把 `disconnect` 里的那条 `str wzr, [xN, #off]` 换成 `nop` (`d503201f`), 恢复 Android 16 及以前的行为. 案例中其他方案的取舍:

| 方案 | 结论 |
| --- | --- |
| `attachBuffer` 永远不覆盖 generation | 不采用: ImageWriter, 相机等跨 BufferQueue attach 的场景依赖覆盖 |
| 在 `OutputBufferQueue::configure` 中补调 `setAutoGenerationUpdate(false)` | 方向最正确, 但要在二进制里插调用, 改动大 |
| 在 MediaCodec 重连之后再设 generation | 同上, 且 `libstagefright` 被多个进程使用 |
| 改 aconfig flag `wb_media_migration` | 不可行: read-only flag 在 C++ 中编译期折叠 |

## 4. 原因 B: vendor 解码器放弃申请输出缓冲

### 4.1 机制

播放器给下一个视频提前建好解码器并预加载: 短时间内送入大量输入, 解码器把全部输出缓冲解满, 播放器拿着帧不显示. 解码器手里还有输入, 继续申请输出缓冲, 申请不到就定时重试. 案例中的高通解码器在"距上次收到输入超过 1 秒"后停止重试, 之后只有收到新输入或解码出新帧才会重新申请.

用户滑过去后, 播放器归还了输出缓冲, 但解码器已经不再申请; 播放器想送新输入, 可输入槽全被解码器占着, 拿不到空槽. 双方互相等待, 画面永久定格. 这一问题在 vendor 内部, 与框架版本无关, 案例中 Android 15 GSI 也会出现.

### 4.2 定位

1. 在 vendor 的 Codec2 库里找日志字符串, 确定所在函数:

   ```bash
   adb pull /vendor/lib64/libqcodec2_v4l2codec.so
   strings -t x libqcodec2_v4l2codec.so | grep "elapsed since last input"
   ```

   案例中在 `QC2V4l2Decoder::DecoderOutputAllocator::handleOutputBufferAllocation()`. 反汇编该函数, 在超时 (`C2_TIMED_OUT`, 值 110) 分支里找"读阈值, 为 0 则继续重试"和"读上次输入时间"两个判断.
2. 用 uprobe 确认运行时确实停止了重试. 需要 root 和内核的 `CONFIG_UPROBE_EVENTS`. 下面是语法示例, 偏移要换成自己库里的函数入口或分支地址:

   ```bash
   cd /sys/kernel/tracing
   echo 'p:hoba /vendor/lib64/libqcodec2_v4l2codec.so:0x<函数偏移>' >> uprobe_events
   echo 1 > events/uprobes/enable
   cat trace_pipe
   ```

   案例中卡住时, 预加载的解码器重试约 36 轮 (约 5.4 秒) 后再也没有进入该函数.
3. 在 system 侧确认输入槽已满: 案例中在 `CCodecBufferChannel::feedInputBufferIfAvailableInternal` 上挂 uprobe, 看到播放后每次归还缓冲都会进入, 但输入槽 10/10 全在 vendor 手里, 一次 `onInputBufferAvailable` 都没有.

### 4.3 修复

把"阈值不为 0 就放弃"的条件分支改成无条件跳到重试块, 等效于阈值 0. 案例中的改动是 `cbz x9, <重试>` 改成 `b <重试>`. 案例中其他方案的取舍:

| 方案 | 结论 |
| --- | --- |
| 加大输入槽或输出缓冲数量 | 只推后上限, 预加载会把槽位再次占满 |
| 在 CCodec 侧给 vendor 送一个空 work 唤醒 | 原理可行, 但要在二进制里构造 C2Work 并调用 HIDL 接口 |
| 改 vendor preset 里的阈值 | 阈值编译在 `libqcodec2_platform.so` 的字符串里, 改长度不便, 不如改一条分支 |

代价: 预加载后一直没播放的解码器会周期性醒来 (案例中每 150 ms 一次), 直到被释放.

## 5. 热补丁验证

改镜像前, 先在运行中的进程里改内存验证, 进程重启后自动还原. 需要 root.

1. 找到目标进程和库的映射. 原因 A 改播放器进程 (如 X) 里的 `libgui.so`, 原因 B 改 Codec2 服务进程里的 vendor 库:

   ```bash
   adb shell pidof vendor.qti.media.c2@1.0-service
   adb shell su -c "grep libqcodec2_v4l2codec.so /proc/<pid>/maps"
   ```
2. 计算地址: 找到 `r-xp` 那一行, 起始地址为 `start`, 文件偏移列为 `pgoff`, 指令的文件偏移为 `off`, 则内存地址 = `start + (off - pgoff)`.
3. 以 root 向 `/proc/<pid>/mem` 的这个地址写 4 字节新指令 (小端). 可用 `dd` (`bs=1 seek=<地址> conv=notrunc`) 或自己写的小工具; 本文作者在案例中没有记录具体命令, 写入前先读出 4 字节核对原指令.
4. 复现场景, 对照第 2 节的日志特征.

案例中用 bind mount 替换文件做不到 (用的 root shell 没有 `CAP_SYS_ADMIN`), 所以改用写进程内存.

## 6. 做成持久补丁

### 6.1 定位工具

- 按符号 (`.dynsym`) 和指令模式定位, **禁止**写死偏移. 命中数不是 1 时停止, 换底包或换固件时不会改错位置.
- 打完后检查: 与原文件只差 1 条指令 (1 到 4 字节), 再用 `--check` 确认状态已翻转.
- 两个工具都依赖 `pyelftools`, 见 [tb321fu-video-freeze-patch](https://github.com/CharlesGool/tb321fu-video-freeze-patch).

### 6.2 system 侧文件

直接替换镜像里的文件, 保留原来的属主, 权限和 SELinux 标签 (`libgui.so` 为 `u:object_r:system_lib_file:s0`).

### 6.3 vendor 侧文件

GSI 通常不改 vendor 分区. 有两种做法:

| 做法 | 要求 |
| --- | --- |
| 直接改 vendor 镜像 | 重新生成 vendor 的 hashtree 和 vbmeta, 或沿用已有的关校验方式 |
| 补丁版放进 system, 开机 bind 到 `/vendor` | 在 Codec2 服务启动前 (如 `post-fs`) 执行; bind 前核对原厂文件 sha256, 对不上就跳过; 补丁文件的 SELinux 标签与原文件相同 (`vendor_file`); 执行 bind 的域需要对 `vendor_file` 的 `mounton`, `read` 权限 |

案例用第二种, 实现见 TB321FU-GSI-Mods 的 [vendor-bind](https://github.com/CharlesGool/TB321FU-GSI-Mods/blob/main/toolkit/vendor-bind/README.md). 好处是 vendor 不匹配时自动回到原厂行为.

## 7. 刷机后复查

```bash
adb shell sha256sum /system/lib64/libgui.so
adb shell dmesg | grep vendor-bind        # 只适用于 bind 做法
adb logcat -c
# 刷视频几分钟: 在一个视频停留 10 秒以上再滑走, 来回切全屏
adb logcat -d | grep -E "generation number mismatch|remote graphic buffer migration|Allocating output buffer timedout"
```

通过标准:

- 没有 `generation number mismatch`; `remote graphic buffer migration N/N` 全部成功.
- 不再出现 `Allocating output buffer timedout ... elapsed since last input` (修复后走重试分支, 不打印这条).
- 刷视频不再卡画面.

## 8. 用到其他设备

- 原因 A 只取决于 libgui: 任何 Android 17 GSI 配 HIDL Codec2 vendor 都可能出现. `patch_libgui_surface_gen.py` 只依赖 `Surface` 的两个导出符号, 可以直接用于别的 arm64 libgui; 先 `--check`.
- 原因 B 取决于 vendor 实现. `patch_qcodec2_alloc_retry.py` 按高通 `handleOutputBufferAllocation` 的指令模式查找, 其他高通 vendor 版本需要先 `--check`; 非高通 vendor 要按第 4.2 节重新定位.
- 每次换 GSI 底包或 vendor 固件都要重新 `--check`; 上游修复后工具会报告未命中, 此时不需要补丁.
