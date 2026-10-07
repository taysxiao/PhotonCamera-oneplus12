# Photon Camera — OnePlus 12 适配版

[English](./README_EN.md) | [简体中文](./README_CN.md) | [日本語](./README_JA.md)

> 这是 [bjzhou/PhotonCamera](https://github.com/bjzhou/PhotonCamera) 的 **OnePlus 12 专用适配分支**。
> 上游功能说明见原项目 README；本文只记录为一加 12（Snapdragon 8 Gen 3 / PJD110）所做的适配与修复。
>
> 设备环境：Android 16（PixelOS），已 root，vendor/odm 保留一加原厂相机 HAL（QTI CAMX）。

---

## 为什么需要适配版

一加 12 刷 PixelOS 后，原版 PhotonCamera 有三个具体问题：

| 问题 | 表现 | 状态 |
|---|---|---|
| 手动色温滑杆只在暖调段有效 | 红通道增益被钉死，往暖调拖只动蓝，怎么调都偏蓝 | ✅ 已修（`7529280`） |
| RAW 成片整体偏蓝 | 即使色温拖到最暖（2900K），成片仍然全蓝 | ✅ 已修（`ba073d9`） |
| 预览防抖开关变灰 | 专业模式里开关灰色不可点 | ✅ 已修（`7529280`） |
| RAW 噪声模型不匹配 | X9 Ultra 的噪声 profile 不适用于一加 12 传感器 | ✅ 已修（`f2f3273`） |
| 噪声模型被设置页重置 | 选完 adaptive 会被打回 Pixel 5 | ✅ 已修（`1763793`） |

---

## 一、手动色温滑杆在暖调段完全失效

### 现象
专业模式拖动色温滑杆，**预览正常，成片整体偏蓝**，且往暖调拖也救不回来。

### 根因
`Camera2Controller.kelvinToRggbGains()` 用**三通道最小值**做归一化：

```kotlin
val minGain = minOf(redGain, greenGain, blueGain).coerceAtLeast(1e-3f)
return RggbChannelVector(
    (redGain / minGain).coerceIn(1f, 4f),   // ← 恒等于 1.0
    ...
)
```

增益是光源 RGB 的倒数，**红光越少 → 红增益越大**。在 2500K–6500K 区间，
红通道倒数始终最大，取 min 就永远落在红通道上：

```
redGain / minGain ≡ redGain / redGain ≡ 1.0
```

**红增益被钉死成常数 1.0，形成横跨 3500K 的死区。** 用户往暖调拖时只有蓝通道在动，
红通道纹丝不动 → R/B 持续走低 → 画面持续偏蓝。

数值复现（用实机 DNG 里量到的自动白平衡增益 `(1.96, 1.0, 1.0, 1.76)` 作冻结锚点）：

| 目标色温 | 修复前 R | 修复后 R | 修复前 R/B | 修复后 R/B |
|---|---|---|---|---|
| 3000K | **1.0000** | 0.6949 | 0.3984 | 0.4310 |
| 4000K | **1.0000** | 0.8071 | 0.4968 | 0.6513 |
| 5000K | **1.0000** | 0.8942 | 0.6305 | 0.8076 |
| 6000K | **1.0000** | 0.9653 | 0.7982 | 0.9288 |
| 6500K | **1.0000** | 0.9965 | 0.9571 | 0.9806 |
| 8000K | 1.1527 | 1.0388 | 1.1527 | 1.1527 |

### 另外两处叠加缺陷

**逐通道截断破坏比例** —— `scaleFrozenWhiteBalanceGain()` 对四个通道各自
`.coerceAtLeast(1f)`，而各通道缩放比例不同，逐个 clamp 等于在归一化后又扭曲一次 R/B。

**25K 冻结死区** —— `resolveManualMatrixGains()` 里 `abs(target - base) <= 25` 直接返回
冻结值，滑杆步进小于 25K 时**设置完全无效**，滑块在动但画面不变。

### 修复
新增 `normalizeRggbGains()`，改为**以绿色通道为基准**归一化，末尾统一 clamp 一次：

```kotlin
val safeGreen = ((greenEven + greenOdd) / 2f).coerceAtLeast(1e-3f)
// 三通道比值被完整保留，R 不再被钉死
val overallScale = when {
    minGain < MIN_GAIN_SCALE -> MIN_GAIN_SCALE / minGain   // 0.25f
    maxGain > MAX_GAIN_SCALE -> MAX_GAIN_SCALE / maxGain   // 4f
    else -> 1f
}
```

保留大于 1 的下界余量（0.25），暖调侧红通道才有向下收缩的空间。

修复后 R 增益从 0.25 单调升到 1.04，**R/B 严格递增**，符合普朗克轨迹的物理预期。

### 已排除的假设
「预览与成片走两条不同白平衡链路」—— **证伪**。预览与拍照调用的是同一个
`applyWhiteBalanceSettings()`，且 MATRIX 分支里 `isCapture` 参数**根本没被使用**，
两条链路的 gains 与 transform 逐字节相同。

**v4 实机复验**（PJD110 上拍摄，DNG 内嵌诊断）：

| 色温 | R | G | B | R/B |
|---|---|---|---|---|
| 2900K | 2.0713 | 1.0 | 1.7217 | 1.2031 |
| 8000K | 3.1562 | 1.0 | 0.9180 | 3.4383 |
| 自动 AWB | 1.9600 | 1.0 | 1.7600 | 1.1136 |

R 上升、B 下降、R/B 单调递增（跨度 2.86 倍），红通道调节范围从 0（死区）
恢复到 52%。**修复已确认生效。**

---

## 二、RAW 成片整体偏蓝（仅手动白平衡）

> 与色温滑杆**完全独立**的第二个 bug。特征非常明确：**自动白平衡正常，只有手动白平衡全蓝**。

### 现象

| 拍摄模式 | 成片 R/B | |
|---|---|---|
| 自动白平衡 | 1.17 | ✅ 正常 |
| 手动白平衡 | **0.039** | ❌ 整张泛蓝 |

### 根因

手动白平衡下 HAL 上报的白平衡增益异常（`wb=[0.875, 1, 1, 5.333]`），把白点推到 2059K。
色适应矩阵因此接近奇异——`pcsToCamera` 的蓝对角被压到 0.033，求逆后放大 22 倍，
最终用于渲染的色彩校正矩阵（CCM）蓝对角达到 **15.04**（正常只有 2 左右）。

由于该矩阵仍然把中性映射到中性，伤害表现为**对任何蓝色过量施加 15 倍增益**，
于是整幅画面泛蓝。

DNG 内嵌诊断给出的完整链路：

```
wb=[0.87521374, 1.0, 1.0, 5.3333335]
cameraWhite=[1.0, 0.8752135, 0.1641025]
whiteXY=[0.5300357, 0.4274876]  cct=2059.2893
ccm=[0.55488247, 0.20795092, 1.6040871,
     -0.23171294, 1.4500076, -0.2279171,
     -0.9661846, -0.5739047, 15.043851]      ← 蓝通道对角 15.04
```

### 判据：只能用元素幅值

| | 自动 | 手动 |
|---|---|---|
| CCM 最大元素 | 2.047 | **15.044** |

（曾试图用"白点色度必须等于 D50"作判据，但实测**自动和手动都映射到 (0.3333, 0.3333)**，
偏离 0.0252 完全相同——该判据根本无法区分健康与病态矩阵。这一点是靠拿自动模式的
样本反向验证才发现的。）

### 修复

1. **`RawDemosaicProcessor.convertDngRawDataToMetadata`** —— 渲染**实际使用**的 CCM 来源。
   DNG/HAL 提供的矩阵病态时（任一元素绝对值 > 6），退回 metadata 矩阵，最后退回单位矩阵。
2. **`DngSdkColorSpec.cameraToPcsForWhite`** —— 用幅值判据替换原先的白点色度判据，
   并同时覆盖 ForwardMatrix 与 ColorMatrix 两条分支。

### 实机验证

| | v5 | v6 |
|---|---|---|
| 自动 R/B | 1.170 | 1.183 |
| 自动 CCM 最大元素 | 2.047 | 2.034 |
| 手动 R/B | **0.039** | **1.376** |
| 手动 CCM 蓝对角 | **15.04** | **1.637** |

自动模式护栏不触发，渲染路径完全不变 —— **零回归**。

### 一并排除的假设

| 假设 | 结论 | 依据 |
|---|---|---|
| DCP 的 ForwardMatrix 是占位 sRGB | ❌ | `assets/dcp/` 目录不存在；DNG 里也无 ForwardMatrix 标签 |
| FM 分支的 `inverseWhite` 双重白平衡 | ❌ 不适用 | 本机不走 ForwardMatrix 分支 |
| 换用 DNG 规范归一化 | ❌ | 异常场景偏差反而更差 |
| 钳制白平衡增益的 span | ❌ | 会误伤 8000K 正常场景 |
| 用 D50 色度做判据 | ❌ | 健康与病态矩阵该值完全相同（见上）|

## 三、预览防抖开关永久变灰

### 现象
专业模式「预览防抖」开关灰色不可点。

### 一加 12 的 HAL 能力组合是矛盾的

| 能力 | dumpsys 实测 |
|---|---|
| `android.lens.opticalStabilizationMode` | **`[ON]`** ← 声明 OIS 开启 |
| `android.statistics.info.availableOisDataModes` | **完全不存在** |
| `statistics.oisSamples` | **完全不存在** |
| `STATISTICS_LENS_INTRINSICS_SAMPLES`（API 35） | **不存在** |

冲突探测逻辑：

```kotlin
val opticalConflict = opticalStabilizationActive && !opticalCorrectionAccepted
```

OIS 报告 ON，而 `opticalCorrectionAccepted` 要求每帧至少 2 个光学校正采样 ——
HAL 从来不提供，恒为 false。两者一拼 `opticalConflict` **恒为真**，
连续 5 帧后 `halStabilizationConflictDetected = true`，开关永久锁死。

**这是误判**：HAL 压根没实现这套遥测，「没有采样」不等于「OIS 和 EIS 真的打架」。

### 修复
只有 HAL **声明过**采样通道时，缺少采样才判为冲突：

```kotlin
val reportsOpticalCorrectionTelemetry = supportsOisSamples || supportsLensIntrinsicsSamples

val opticalConflict = opticalStabilizationActive &&
    calibration?.reportsOpticalCorrectionTelemetry == true &&
    !opticalCorrectionAccepted
```

一个从不声明遥测通道的 HAL，无法与它自己矛盾。

顺带修了一个 UI 问题：`calibration` 是普通 `@Volatile` 字段，首帧后才赋值，
Compose 不会重组，开关会永远停在初始 `false`。新增 `AtomicLong` 版本号供 UI 订阅。

> 补充：`SENSOR_INFO_TIMESTAMP_SOURCE` 实测 6 个相机**全部是 `REALTIME`**，不是障碍。

---

## 四、设备配置（`oneplus_12.json`）

```json
{
  "enable_logical_multi_camera_discovery": true,
  "hdr_plus_frame_count": 8,
  "nr_level": 0,
  "edge_level": 0,
  "raw_lens_shading_correction_enabled": true,
  "use_p010": true,
  "use_p3_color_space": true,
  "raw_noise_profile_id": "adaptive_x9_ultra",
  "raw_dcp_id": "builtin_dcp_OPPO Find X8 Ultra back camera 8.67mm f1.8 Adobe Standard"
}
```

匹配型号：`PJD110` / `CPH2573` / `CPH2581` / `CPH2583`

| Key | 作用 |
|---|---|
| `raw_noise_profile_id` | RAW 噪声模型。原版用 X9 Ultra 的 profile，与一加 12 传感器不匹配；已解除内置配置对 adaptive profile 的排除 |
| `raw_dcp_id` | RAW 色彩匹配配置。一加 12 主摄与 X8 Ultra 同为 8.67mm f1.8，光学特性接近 |
| `hdr_plus_frame_count` | HDR+ 融合帧数，上限可到 20，8 帧是画质与速度的平衡点 |
| `enable_logical_multi_camera_discovery` | 启用逻辑多摄物理绑定 |
| `use_p010` / `use_p3_color_space` | 10-bit 与 Display P3 |

### 关于 `nr_level` / `edge_level` 保持 0
这两个 key **影响不了 RAW 成片**。源码里 RAW 拍摄时整段跳过：

```kotlin
if (!isRawCapture) {
    // 6. 图像质量设置（锐化、降噪）
    applyImageQualitySettings(builder, currentState)
}
```

实测 `original.dng` 内嵌诊断 JSON 里 `edgeMode` / `noiseReduction` / `nrLevel` /
`edgeLevel` 出现次数**全部为 0**，且 `dng.layout = LINEAR_RAW_RGB`、
`pipeline = SPATIAL_RGB` —— 成片完全来自 App 自研 RAW demosaic 管线。

真正控制成片降噪的只有 App 侧的**降噪滑杆**和**画质调优开关**。

---

## 五、实测数据

### 硬件与 HAL
- 主摄 RAW：4096×3072（传感器 50MP，binned）
- 长焦 RAW：4624×3472（传感器 64MP，binned）
- Camera2 相机 ID：`2` 主摄 / `3` 超广角 / `4` 长焦，均为 `LEVEL_3`，支持 RAW
- HAL 能力含 `MANUAL_POST_PROCESSING`（手动色温矩阵路径可用）
- GPU：Adreno 750，OpenGL ES 3.2

### RAW 管线（DNG 内嵌诊断）
```
qualityTuningSensorAreaMm2 = 63.13602   ← 画质调优开启（关闭时为 null）
lumaScales                 = 0.3108     ← 与受控实验的 ON 状态值一致
dng.layout                 = LINEAR_RAW_RGB
merge.outputMode           = RGB
```

### 超分与镜头
| 镜头 | 源尺寸 | 输出尺寸 | scale | 说明 |
|---|---|---|---|---|
| 主摄 id2 | 4096×3072 | 4096×3072 | 2.0 | RAISR 放大到 6144×8192 再缩回 |
| 长焦 id4 | 4624×3472 | **3840×2884** | 2.0 | 裁切，非放大 |

RAISR 单次 667–685ms，是 RAW 管线最大固定开销。
长焦 2× 是从 binned RAW 放大，**不产生新的光学细节**。

### 画质开关的实测影响
| 设置 | 纹理 | 噪声 | 说明 |
|---|---|---|---|
| 画质调优 ON | 基准 | 基准 | 降噪强度 ×0.31，**保纹理，推荐保持开** |
| 画质调优 OFF | **−50%** | −59% | 降噪强度 ×3.2，抹平细节 |
| RAISR vs Lanczos-3 | +23% | +35% | 学习型放大器，高频重建更强 |

**结论**：画质调优保持 ON；追求锐度用 RAISR，开 Ultra HDR 时换 Lanczos-3；
降噪用滑杆连续微调（它是乘在画质调优强度之上的）。

---

## 六、使用建议

| 项目 | 推荐值 |
|---|---|
| 画质调优 | **开** |
| 超分算法 | 日常 RAISR；开 Ultra HDR / 追求少颗粒时 Lanczos-3 |
| 降噪滑杆 | 默认 1.0 起步，按 ISO 调 |
| HDR+ 帧数 | 8（可上调到 20，更慢） |
| 手动色温 | 现在可用了，5000K 起步按环境微调 |
| 预览防抖 | 现在应该可点了 |

> ⚠️ 色彩风格取决于 `raw_dcp_id`。若偏好更中性，可换其他 DCP 或在 App 内调 LUT。

---

## 构建

本机无 JDK / Android SDK，使用 GitHub Actions 云端构建：

- workflow：`android-build.yml`
- 触发方式：**手动 dispatch**（`push` 不触发）
- 渠道：`default` / `dev`
- 签名：仓库自带 `app/debug.keystore`（alias `androiddebugkey` / 密码 `android`）

APK 从 build 产出的 artifact 获取：`release-apk-dev`。

---

## 已知限制

- **无法突破 binned RAW 上限**：主摄约 4096×3072、长焦约 4624×3472。
  ColorOS 相机自己的 50MP 全尺寸也走私有 HAL 通道，第三方 App 到不了。
- **长焦 2× 不增加光学细节**，只是放大。
- **防抖遥测受限于 HAL**：一加 12 的 PixelOS HAL 不提供 OIS 采样，
  App 无法做镜头畸变校正（profile 7 需要 intrinsics）。修复后开关可用，
  但效果以陀螺仪 EIS 为主。
- release 构建无调试日志，需要诊断信息时用 `dev` 渠道。

---

## 相关文档

- `build/本轮修复说明.md` — 本轮两个修复的完整技术分析
- `build/受控实验结论-调优与放大算法.md` — 三张隔离样片的量化对比
- `build/超分改版-复验报告.md` — 超分算法复验
- `build/白平衡-专业模式色温问题.md` — 色温问题早期诊断
- `build/ColorOS相机移植可行性报告.md` — ColorOS 相机移植评估

---

## License

遵循上游 [Apache License 2.0](./LICENSE)。
