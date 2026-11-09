---
name: research
description: TB321FU 上 X 视频声音继续, 画面卡住的完整排查记录
metadata:
  version: "1.0.0"
  lang: "zh-CN"
---

# TB321FU 排查记录

本文是 [调试教程](debug-guide.md) 所用案例的原始记录: TB321FU 上 X 视频"声音继续, 画面卡住"的两个独立原因, 证据, 分析和修复. 补丁实现在 [tb321fu-video-freeze-patch](https://github.com/CharlesGool/tb321fu-video-freeze-patch).

日期: 2026-11-09. 设备 TB321FU(SM8650 / pineapple, vendor = 原厂 Android 14),
系统 = Pixel 提取的 A17 GSI(`Lenovo/TB321FU/TB321FU:17/DP11.261016.003/16708287`).

## 0. 总览(先看这里)

同一个现象背后有 **两个独立原因**, 两个补丁都要打:

| | 原因 1(第 2~6 节) | 原因 2(第 8 节) |
|---|---|---|
| 触发 | 播放中切换输出 Surface(全屏/inline 切换, 播放器复用) | 刷视频: 下一个视频已预加载, 停留 >1 秒后再滑过去 |
| 根因 | A17 libgui `Surface::disconnect` 清零 generation → HIDL Codec2 缓冲迁移全部失败 | 高通 vendor 解码器"距上次输入 >1s 就放弃申请输出缓冲" + 输入槽被自己占满 → 永久死锁 |
| 影响版本 | 只有 A17 框架 | **与框架版本无关**(A15/A17 GSI 都会, 解释了 A15 也卡) |
| 补丁 | `/system/lib64/libgui.so` 4 字节 | `/vendor/lib64/libqcodec2_v4l2codec.so` 4 字节 |
| 上机验证 | 在 X 进程内存热补丁验证: migration N/N | 在 codec 服务进程内存热补丁验证: 必现场景 0 卡顿 |

## 1. 现象

X(com.twitter.android)刷视频时大概率出现: 声音继续播, 画面定格; 手动拖一下进度条恢复.

## 2. 原因 1 结论(一句话)

Pixel A17 的 `libgui.so` 里 **`Surface::disconnect()` 会把 Surface 缓存的 `mGenerationNumber` 清零**,
而 MediaCodec 设置输出 Surface 时恰好是"先设 generation → 再 disconnect → 再 connect".
结果 Surface 缓存的 generation = 0, BufferQueue 的 generation = N. HIDL Codec2 客户端在切换 Surface 时
用这个 Surface 去 attachBuffer, 每个缓冲区都被盖上 generation 0, **全部被 BufferQueue 拒绝**.
vendor 解码器以为缓冲区已经迁移过去, 实际一个都没到, 缓冲池被"幽灵槽位"占满, 拿不到输出缓冲区,
**视频停止出帧, 音频不受影响**. 拖进度条会 flush/重建解码器, 缓冲池重新分配, 所以能恢复.

Pixel 原机不出问题, 是因为 Pixel(Tensor)的 Codec2 是 AIDL HAL, 走 `GraphicsTracker`,
它会显式 `setAutoGenerationUpdate(false)` 绕开这个问题; 我们的高通 A14 vendor 是 HIDL `c2@1.2`,
走 `OutputBufferQueue` 那条路径, 那条路径没有调用它.

## 3. 证据链(logcat)

X 在播放中调用 `setOutputSurface` 切换 Surface(inline/全屏切换, 播放器复用都会触发):

```
MediaCodec: [c2.qti.avc.decoder] setting surface generation to 3365953
SurfaceUtils: disconnecting from surface ..., reason connectToSurface(reconnect)        <- 这一步把 Surface 缓存的 generation 清成 0
SurfaceUtils: connecting to surface ..., reason connectToSurface(reconnect-with-listener)
BufferQueueProducer: [SurfaceTexture-0-3287-32] attachBuffer: generation number mismatch [buffer 0] [queue 3365953]
Surface2HGraphicBufferProducer@2.0: Surface2HGraphicBufferProducer::attachGraphicBuffer failed: -22
Codec2-OutputBufferQueue: remote graphic buffer migration 0/6                           <- 一个都没迁过去
C2BqBuffer (vendor, pid 2103): local generation change 3365953 ... migrated buffers # 12  <- vendor 以为迁了 12 个
Codec2-OutputBufferQueue: outputBuffer -- buffers from old generation to 3365953 , diff: 2
CCodecBufferChannel: [c2.qti.avc.decoder#525] queueBuffer failed: -32
MediaCodec: rendring output error -32                                                    <- 每一帧都送不上屏
QC2V4l2Decoder: [v4lavcD_716] Allocating output buffer timedout, 1034ms elapsed ...      <- vendor 拿不到新缓冲区
```

统计(约 1 分半钟的使用):
- 带缓冲区的 Surface 切换共 17 次(0/14 ×10, 0/15 ×3, 0/16 ×3, 0/6 ×1), **迁移成功率 0%**.
- vendor `Allocating output buffer timedout` 共出现 242 次, 分布在 10 个解码器实例上. 其中一部分可能是 X 在屏幕外
  预加载, 还没被消费的解码器, 不一定每次都是这个 bug, 补丁后对比次数即可区分.
- 实例 #525: 23.604 开始每帧都送不上屏, 24.749 之后完全不出帧, 直到 39.587 解码器被重置才结束.

`[buffer 0]` 是关键: 被 attach 的缓冲区 generation 是 0, 不是 Codec2 客户端设的 N.

## 4. 根因分析(反汇编 + AOSP 源码对照)

工具: pyelftools + capstone, 用一个按符号反汇编并解析 PLT 的小脚本(未发布).
二进制取自设备 `/system/lib64/libgui.so`(sha256 `222ca5e3…3007`)和 `/system/lib64/libcodec2_client.so`.

### 4.1 libgui: Surface 新增的 "auto generation update"(A17 新增, A15/A16 没有)

AOSP `frameworks/native` android17-release `libs/gui/Surface.cpp`:

```cpp
int Surface::attachBuffer(...) {
    uint32_t priorGeneration = graphicBuffer->mGenerationNumber;
    if (mAutoGenerationUpdate) {                       // 默认 true
        graphicBuffer->mGenerationNumber = mGenerationNumber;   // 用 Surface 缓存的值覆盖缓冲区自己的 generation
    }
    ... mGraphicBufferProducer->attachBuffer(...)
}

int Surface::disconnect(int api, DisconnectMode mode) {
    ...
    mGenerationNumber = 0;          // <-- A17 新增: disconnect 清零缓存的 generation
    mAutoGenerationUpdate = true;
    ...
}
```

设备二进制里对应: 构造函数 `+0x3b4 = 1`(`movi v0.2s,#1; str s0,[x19,#0x3b4]`), `attachBuffer` 里
`ldrb w9,[x19,#0x3b4]; cbz w9,…; ldr w9,[x19,#0x3b0]; str w9,[x8,#0xe4]`,
`disconnect` 里 `0x11c28c: str wzr,[x19,#0x3b0]` / `0x11c290: strb w8(=1),[x19,#0x3b4]`.
`+0x3b0 = mGenerationNumber`, `+0x3b4 = mAutoGenerationUpdate`.

对照: android15-qpr2-release, android16-release, android16-qpr2-release 的 Surface.cpp 里
**都没有** `mAutoGenerationUpdate`, disconnect 也不清 generation.

### 4.2 libstagefright: MediaCodec 的 "先设 generation 再断开重连"

AOSP `MediaCodec::connectToSurface()`(多个版本一直如此):

```cpp
surface->setGenerationNumber(*generation);       // BufferQueue gen = N, Surface 缓存 = N
// HACK: clear any free buffers ...
nativeWindowDisconnect(surface.get(), "connectToSurface(reconnect)");   // A17: Surface 缓存 -> 0, BQ 仍为 N
surfaceConnectWithListener(surface, listener, "connectToSurface(reconnect-with-listener)");
```

A16 及以前 disconnect 不碰缓存值, 这段 HACK 没问题; A17 libgui 改了 disconnect 语义, MediaCodec 这里没跟着改.

### 4.3 libcodec2_client: HIDL 与 AIDL 两条路径不对称

- AIDL 路径 `GraphicsTracker::configureGraphics(sp<Surface>, gen)`: 反汇编可见
  `mov w1, wzr; bl Surface::setAutoGenerationUpdate` → 关掉自动覆盖, 用缓冲区自身的 generation(客户端已设成 N), 所以没问题. **Pixel 原机走这条. **
- HIDL 路径 `OutputBufferQueue::configure(sp<Surface>, …)`(Pixel 的 `wb_media_migration` 实现,
  `com.android.graphics.libgui.flags.wb_media_migration` = enabled, read-only): 用
  `Surface2HGraphicBufferProducer` 包一层 Surface, **没有调用 `setAutoGenerationUpdate(false)`**.
  `Surface2HGraphicBufferProducer::attachGraphicBuffer` 直接调 `Surface::attachBuffer`, 缓冲区被盖成 0.
  **TB321FU 的高通 vendor(`vendor.qti.media.c2@1.0-service`, HIDL)走这条. **

### 4.4 为什么最后是"卡住"而不只是丢几帧

客户端迁移失败, vendor `C2BqBuffer` 却认为迁移成功(`migrated buffers # 12`), 它的 `C2SurfaceSyncMemory` 把这些
槽位算作已 dequeue. 这些旧 generation 的输出被客户端当作 stale 丢掉, 再也回不到新 BufferQueue. 可用槽位很快耗尽,
vendor `Allocating output buffer timedout`, 解码器停止出帧. 音频是另一条路径, 不受影响.

## 5. 影响范围

- **受影响:** Pixel A17 框架 + HIDL Codec2 vendor(A14 及更早的高通 vendor 基本都是). 会在播放中切换输出 Surface
  的 app 都会中招, X 最明显; 不切 Surface 的播放器基本碰不到.
- **Android 15 GSI:** AOSP A15/A16 的 libgui 没有这段逻辑, 所以 **A15 上的卡顿不是原因 1, 而是原因 2**(第 8 节,
  vendor 侧, 与框架版本无关).
- **其他(非 Pixel 提取)A17 GSI:** 只要 libgui 用的是 A17 AOSP 代码, Codec2 客户端走 Surface 包装路径, 同样会中招.

## 6. 修复(系统制作侧)

### 方案 A(采用): libgui.so 去掉 `Surface::disconnect` 里的 `mGenerationNumber = 0`

- **改动:** `/system/lib64/libgui.so` 一条指令 → NOP, 4 字节. 当前 build: 文件偏移 `0x11c28c`,
  `7f b2 03 b9`(`str wzr,[x19,#0x3b0]`) → `1f 20 03 d5`(`nop`).
- **效果:** disconnect 后 Surface 保留缓存的 generation(= A16 及以前的行为). MediaCodec 设 N →
  断开重连后缓存仍是 N → attachBuffer 盖上 N, 与 BufferQueue 一致, 迁移成功.
- **副作用评估:**
  - `mAutoGenerationUpdate = true` 那条照常执行.
  - AIDL GraphicsTracker 路径本来就关掉了自动覆盖, 不受影响.
  - 其他会设非零 generation 的只有 MediaCodec, 它在 `disconnectFromSurface()` 里会主动 `setGenerationNumber(0)` 复位, 所以不依赖 disconnect 清零.
  - 整个 libgui 里写 `+0x3b0 = 0` 的只有构造函数和 disconnect 两处, 没有其他内联副本.
- **工具:** [`patch_libgui_surface_gen.py`](https://github.com/CharlesGool/tb321fu-video-freeze-patch/blob/main/patch/parts/002.libgui-surface-gen/patch_libgui_surface_gen.py)
  - 依赖 pyelftools.
  - 定位方式: 从 `Surface::setGenerationNumber` 里唯一的 STR 推出字段偏移, 再在 `Surface::disconnect` 里找
    `str wzr,[xN,#偏移]`, 必须恰好 1 处才改, 否则中止. 不硬编码偏移, 换 build 也能用.
  - 打补丁: `python3 patch_libgui_surface_gen.py <in> <out>`; 只检查: `--check <so>`.

镜像制作里的集成(保留原权限, 属主和 SELinux 标签):

```bash
LIB="$MNT/system/lib64/libgui.so"
python3 -I patch_libgui_surface_gen.py --check "$LIB"            # 必须报"未打补丁"
MODE=$(sudo stat -c %a "$LIB"); UID_=$(sudo stat -c %u "$LIB"); GID=$(sudo stat -c %g "$LIB")
LABEL=$(sudo getfattr -n security.selinux --only-values "$LIB" | tr -d '\0')
python3 -I patch_libgui_surface_gen.py "$LIB" /tmp/libgui.so      # 打印改动偏移 + 前后 sha256
sudo install -m "$MODE" -o "$UID_" -g "$GID" /tmp/libgui.so "$LIB"
sudo python3 -I -c 'import os,sys; os.setxattr(sys.argv[1],"security.selinux",sys.argv[2].encode()+b"\0")' "$LIB" "$LABEL"
python3 -I patch_libgui_surface_gen.py --check "$LIB"            # 必须报"没有清零 generation 的指令"
cmp -l <原 libgui.so> "$LIB"                                     # 只允许 0x11c28c 那 4 字节内有差异
```

注意:
- **只改 lib64. ** 32 位 `/system/lib/libgui.so` 只有 32 位进程会用, 几乎没有 32 位 app 播视频, 不用改.
- **每次换 Pixel base 都要重跑. ** 先 `--check`; 如果 Google 修了(disconnect 不再清零, 或 HIDL 路径补上了
  `setAutoGenerationUpdate(false)`), 脚本会报未命中, 跳过即可.

### 不采用的方案(及原因)

| 方案 | 不采用的原因 |
|---|---|
| `Surface::attachBuffer` 永远不覆盖 generation | 影响面太大: ImageWriter, 相机等跨 BufferQueue attach 的场景都依赖覆盖成目标 Surface 的 generation |
| `libcodec2_client` 的 `OutputBufferQueue::configure` 里补调 `setAutoGenerationUpdate(false)`(与 AIDL 路径对齐) | 方向最"正统", 但需要在原生代码里插调用, 没有现成空间, 二进制改动大 |
| 改 MediaCodec, 在 reconnect 之后再设 generation | 同上, 要插代码; 而且 `libstagefright` 被多个进程使用 |
| 改 aconfig flag `wb_media_migration` | read-only flag, C++ 里编译期常量折叠, 改 flag 存储不生效 |

## 7. 原因 1 验证

### 离线
- 对设备上拉下来的 libgui.so 打补丁: `--check` 前后状态正确, `cmp -l` 只有 0x11c28c 那 4 字节内有差异(3 个字节不同).
- 整个 libgui 的反汇编里, 写 `[*, #0x3b0] = 0` 的只有构造函数和 disconnect 两处.

### 上机(进程内存热补丁, 已做)
bind mount 替换做不了(`ush` 没有 `CAP_SYS_ADMIN`), 改为 root 写 `/proc/<X pid>/mem`, 把 X 进程里 libgui
`0x11c28c` 那条指令改成 NOP(只影响这一个进程, 重启 X 即还原). 结果: 切 Surface 后
`remote graphic buffer migration N/N` 全部成功, 不再有 `generation number mismatch`.
**但 X 刷视频仍然会卡** —— 这就是原因 2(第 8 节), 两个补丁要一起打.

刷进镜像后的复查:

```bash
adb shell ush "sha256sum /system/lib64/libgui.so"      # 应为补丁版的 sha256
adb logcat -G 32M; adb logcat -c                       # 然后刷 X 视频 1~2 分钟, 多切几次全屏/来回滑
adb logcat -d | grep -E "generation number mismatch|remote graphic buffer migration|Allocating output buffer timedout"
```

通过标准:
- 没有 `generation number mismatch`.
- `remote graphic buffer migration N/N`, 且 N > 0 时全部成功.
- 没有成串的 `Allocating output buffer timedout`.
- X 刷视频不再卡画面(需要原因 2 的补丁也打上).

---

## 8. 原因 2: 高通 vendor 解码器"放弃分配" + 输入槽占满 → 死锁

### 8.1 结论

X(ExoPlayer)会给 **下一个视频提前建好解码器并预加载**: 在约 0.07 秒内送进约 30 个输入, vendor 把
**全部 18 个输出缓冲区**(OutputBufferQueue `max dequeue count 18`)解码满, X 拿着这些帧不显示(还没轮到它播).
vendor 还剩约 10 个输入在手里, 想继续解码, 但申请不到输出缓冲:

1. vendor `DecoderOutputAllocator::handleOutputBufferAllocation` 每 150ms 重试一次
   (`fetchGraphicBlock` → `C2_BLOCKING` → 等 sync-obj fence 150ms → `C2_TIMED_OUT`).
2. 超时后的判断(反汇编 `0x16e050` 起): **如果距离"上一次收到输入" > 阈值(1000ms), 打一条
   `Allocating output buffer timedout, Nms elapsed since last input buffer queued (threshold 1000ms)` 然后
   不再重投分配事件 —— 彻底停止申请**. 阈值来自 `libqcodec2_platform.so` 里编译进去的 preset
   `private_max_retry_alloc_output_timeout = 1000`, 没有属性可以覆盖.
3. vendor 只在两种情况下重新开始分配(`fetchAndQueueOutputs` ← `MSG_ID_CODEC_QUEUE_OUTPUTS`):
   **收到新输入**(`handleInputs`)或 **解码出新帧**(`onOutputsDone`).

用户滑到这个视频后:
- X 开始播放, 把手里的帧逐个渲染, 归还(BufferQueue 释放, sync-obj `cur 18 → 4`, futex 唤醒都正常).
- 但 vendor 已经停止申请, 没人去取空出来的缓冲 → 不出新帧.
- X 想送新输入: 这个解码器的输入槽只有 **10 个**(`inputDelay + pipelineDelay + 4`), **10 个全在 vendor 手里**
  (vendor 等输出缓冲才能消化它们), CCodec 给不出新的输入 buffer → vendor 收不到新输入 → 永远不恢复.
- 拖进度条 = flush, vendor 清空并重新配置分配器, 所以恢复.

停留 <1 秒就滑过去的视频不卡(还在重试窗口内), 停留越久越必现 —— 与"大概率卡"的体感一致.

### 8.2 证据(uprobe)

- vendor 侧(up7, 预加载的 avcD_82): config 后约 5.4 秒内 `q_hoba → q_alloc → fence 110 → q_allocr 110` 循环 36 轮,
  最后一轮 `q_allocr 110` 之后再无 `q_msg/q_hoba` —— 放弃.
- CCodec 侧(up8, 卡住的通道 `0xb40000771fb54350`): 预加载阶段 `qib=30 ibd=20`(10 个输入没归还);
  播放后每次渲染(`cc_rob`)/释放(`cc_bro`)都会走 `feedInputBufferIfAvailableInternal`, `pipelineFull=0`,
  但 `iact=10 islots=10` → 直接退出, 一次 `onInputBufferAvailable` 都没有, 也再没有 `queueInputBuffer`.
- 对比: A14 与 A17 的 `CCodecBufferChannel`(输入槽公式, `feedInputBufferIfAvailableInternal`, extra slots 逻辑)
  完全一致; sync-obj 实现 system/vendor 两侧也一致 —— **不是框架回归, A15/A17 GSI 都会中招**.

### 8.3 修复(采用): vendor 超时后永远重试

- **改动:** `/vendor/lib64/libqcodec2_v4l2codec.so` 一条指令, 4 字节. 当前 build(sha256 `e328076b…df25`):
  文件偏移 `0x16e078`, `89 16 00 b4`(`cbz x9, #0x16e348`, 阈值为 0 才重试) →
  `b4 00 00 14`(`b #0x16e348`, 无条件重试).
  补丁后 sha256 `faaeaa8c…a35c`.
- **含义:** 等同于把 vendor 自带的阈值设成 0("no time limitation for output buffer allocation"), 这是高通
  代码里本来就支持的模式, 不是新逻辑. 输出缓冲一空出来, 下一次 150ms 重试就拿到, 解码继续.
- **代价:** 被预加载, 但一直没轮到播放的解码器会每 150ms 醒一次(一次 fence 等待), 直到被释放, CPU 开销可忽略.
  非 110(真正的分配错误)仍然走原来的报错路径, 不受影响.
- **工具:** [`patch_qcodec2_alloc_retry.py`](https://github.com/CharlesGool/tb321fu-video-freeze-patch/blob/main/patch/parts/001.codec2-alloc-retry/patch_qcodec2_alloc_retry.py)
  - 定位方式: 在 `handleOutputBufferAllocation` 里找 "`ldr Xt,[Xn,#K]; cbz Xt,T`" 与
    "`ldr Xs,[Xm,#K-8]; cbz Xs,T`" 成对(阈值字段 0x280 / 上次输入时间 0x278, 跳到同一个重试块), 必须恰好 1 处.
  - 打补丁 `python3 patch_qcodec2_alloc_retry.py <in> <out>`; 检查 `--check <so>`.
- 只有 lib64(设备上没有 32 位版本).
- **这是 vendor 分区的改动**, GSI 本身改不到; 我们出的是整包(9008), 打进 vendor 镜像即可. 注意 vendor 有
  dm-verity/AVB, 改了文件要重新生成 vendor 镜像的 hashtree / vbmeta(或沿用现有的关校验方式).

镜像制作里的集成(同第 6 节的保权限/SELinux 标签流程):

```bash
LIB="$MNT_VENDOR/lib64/libqcodec2_v4l2codec.so"
python3 -I patch_qcodec2_alloc_retry.py --check "$LIB"          # 必须报"未打补丁"
MODE=$(sudo stat -c %a "$LIB"); UID_=$(sudo stat -c %u "$LIB"); GID=$(sudo stat -c %g "$LIB")
LABEL=$(sudo getfattr -n security.selinux --only-values "$LIB" | tr -d '\0')
python3 -I patch_qcodec2_alloc_retry.py "$LIB" /tmp/libqcodec2_v4l2codec.so
sudo install -m "$MODE" -o "$UID_" -g "$GID" /tmp/libqcodec2_v4l2codec.so "$LIB"
sudo python3 -I -c 'import os,sys; os.setxattr(sys.argv[1],"security.selinux",sys.argv[2].encode()+b"\0")' "$LIB" "$LABEL"
python3 -I patch_qcodec2_alloc_retry.py --check "$LIB"          # 必须报"已打补丁"
```

### 8.4 不采用的方案

| 方案 | 不采用的原因 |
|---|---|
| 系统侧加大输入槽 / 输出缓冲数量 | 只是推后上限: 预加载时 X 会一直送输入直到槽满, 槽再多也会被 vendor 占满 |
| 系统侧 CCodec 在"输入槽满 + 输出被归还"时给 vendor 送一个空 work 踢醒 | 原理可行(空 work 会走 `handleInputs`), 但要在二进制里构造 C2Work 并调用 HIDL queue, 改动大, 风险高 |
| 系统侧走非 sync-obj 分配路径 | vendor 对非 110 的错误直接报 `Buffer allocation Error` 致命错误 |
| 改 preset 阈值 | preset 是编译进 `libqcodec2_platform.so` 的 JSON 字符串, 改字符串长度会变, 不如改一条分支干净 |

### 8.5 验证(codec 服务进程内存热补丁, 已做)

root 写 `/proc/<vendor.qti.media.c2@1.0-service pid>/mem`, 把 `0x16e078` 改成 `b`(只影响当前服务进程,
服务重启即还原), 在 X 里用 adb 脚本自动滑动(未发布; 截图对比视频区域是否变化,
左下角剩余时间是否在走):

| 场景 | 补丁前 | 补丁后 |
|---|---|---|
| 预加载 >10 秒后滑过去 | 每次都卡(多轮 4/4 FROZEN) | 4/4 正常 |
| 连续滑 25 个视频 | — | 0 卡顿 |
| 长时间停留(40/60/45/90 秒)再滑 | — | 4/4 正常 |

补丁后 trace: 预加载的解码器(如 avcD_93)在 19 秒里持续每 ~150ms 重试一次, 不再出现
"最后一轮 110 后停止"; 补丁前卡住的那类通道补丁后正常出帧(如 `qib=568 rob=510`).

## 9. 刷入镜像后的验证

两个补丁按 [tb321fu-video-freeze-patch](https://github.com/CharlesGool/tb321fu-video-freeze-patch) 构建进 DP11 `BTtest-v1.7-cust_bebug`(libgui 补丁版 sha256 `9a9ec193…4b4a`, v4l2codec 补丁版 `faaeaa8c…a35c`, 开机经 vendor-bind 盖到 `/vendor`).
2026-11-09 真机刷入后, X 刷视频不再出现画面定格.
