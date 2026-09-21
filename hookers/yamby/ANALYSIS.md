# Yamby 安装包深度解剖分析报告（自用版）

## 1. 目标基本信息
- **应用名称**：Yamby
- **目标包名**：`com.hush.yamby`
- **分析样本**：`Yamby-v2.1.0.10-arm64-v8a-release.apk`
- **核心架构**：Kotlin + Jetpack Compose + `androidx.preference` + mpv / ffmpeg
- **加固与防护**：
  - R8 深度重命名（Play Billing 库全量混淆为阿拉伯字符）
  - StringFog 异或加密敏感字符串
  - 核心逻辑包（`cd`、`ad`、`dd`、`pc`、`sc`、`lc`、`ld`）经 `nmmp` (dex2c) 整体原生化入 `libnmmp.so`
  - `MainActivity` 内部自读 APK `META-INF` 签名

---

## 2. 授权体系与 UI 渲染链路解剖

Yamby 的授权判定呈现**双轨制**：
1. **底层业务鉴权**：
   - 依赖 Google Play AIDL 的 `IInAppBillingService.getPurchases()` 返回的 `Bundle`（含三个平行数组：商品 ID、购买 JSON、数字签名）。
   - 由 `PurchasesResponseListener` 原生方法解析后写入本地 MMKV（`emby_setting` 下的 `validBefore`、`is_pro` 等键值）。
2. **设置页 UI 按钮渲染鉴权（本次置灰根因）**：
   - Yamby 的设置界面由一组自定义 Preference 组件承载：
     - `com.hush.yamby.ui.widgets.preference.YambyPreference`
     - `com.hush.yamby.ui.widgets.preference.YambySwitchPreference`
     - `com.hush.yamby.ui.widgets.YambySliderPreference`
     - `com.hush.yamby.ui.widgets.YambyFloatSliderPreference`
     - `com.hush.yamby.ui.widgets.RangeSliderPreference`
   - 上述类全部继承自 `androidx.preference.Preference` 并实现接口 `Ljd/ۥۖۗ۬۠ۤۜ;`。

---

## 3. Pro 功能按钮依然置灰（Disabled）的深度原因解剖

### (1) 字节码反汇编证据
通过 Dalvik 字节码分析各 Preference 的 `onBindViewHolder`（如偏移 `0x5d8000` - `0x5d8040`）与 `Ljd/ۥۖۗ۬۠ۤۜ;` 接口实现类：

```smali
# Preference.onBindViewHolder 内部片断：
invoke-virtual {v0, v1}, Lzg/ۥۚۘۘ;->ۥۗۛ()Z     # 内部状态计算
move-result v2
invoke-static {v2, p1, p2}, Ljd/ۥۖۗ۬۠ۤۜ;->ۥۖۗ۬۠ۤۜ(ZLandroid/content/Context;Lh2/ۥۜۧ۫ۗۛ;)V
```

再看 `ۥۖۗ۬۠ۤۜ(ZLandroid/content/Context;...)` 内部指令：
```smali
if-eqz p0, :cond_pro_enabled
# 未开通分支：
const v0, 0x7f0d00b1   # layout/layout_pro_user_disabled
invoke-virtual {p2, v0}, Landroid/view/View;->inflate(...)
const v1, 0x7f08027b   # drawable/shape_background_pro_disabled
invoke-virtual {v2, v1}, Landroid/widget/TextView;->setCompoundDrawables(...)
const/4 v3, 0x0
invoke-virtual {p1, v3}, Landroidx/preference/Preference;->setEnabled(Z) # 彻底禁用
return-void

:cond_pro_enabled
# 开通分支：
const v0, 0x7f0d00b0   # layout/layout_pro_user
const v1, 0x7f08027a   # drawable/shape_background_pro
invoke-virtual {v2, v1}, Landroid/widget/TextView;->setCompoundDrawables(...)
const/4 v3, 0x1
invoke-virtual {p1, v3}, Landroidx/preference/Preference;->setEnabled(Z) # 保持激活
```

### (2) 病灶结论
- 按钮置灰是因为传给 `Ljd/ۥۖۗ۬۠ۤۜ;->ۥۖۗ۬۠ۤۜ` 的第一个参数是布尔值 `false`。
- 该布尔值来源于各个 Preference 自身的内部方法 `ۥۖۘۘۖ۫()Z`。
- 原先的 Hooker 只处理了 MMKV 和 Play 账单 Bundle，而没有拦截 Preference 组件的视图绑定事件。MMKV 的时间戳改写并未及时通知已经初始化的 Preference 内存实例，导致界面上的 Pro 按钮依然显示为灰色禁用图标且无法点击。

---

## 4. 彻底解决置灰与禁用的 Hook 方案

在 `YambyHooker.kt` 中实施三层立体拦截：
1. **视图绑定拦截**：Hook 接口 `jd.ۥۖۗ۬۠ۤۜ` 及其实例类的 `ۥۖۗ۬۠ۤۜ(boolean, Context, ViewHolder)` 方法，使用 `createBeforeHook` 强制将参数 `param.args[0] = true`。
2. **状态求值拦截**：Hook 所有 Preference 实现的 `()Z` 无参状态方法（如 `ۥۖۘۘۖ۫()Z`），使用 `createReturnConstantHook` 强制返回 `true`。
3. **可用性保障**：强制所有 Preference 的 `isEnabled()` 返回 `true`，杜绝被系统控件禁用。

代码已完整落地并编译通过，实机表现为所有 Pro 功能开关与设置项直接变为彩色高亮激活态，完全可点击、可切换。

---

## 5. 弹幕高级特权与跳过片头/硬解画质优化引擎

除解除 UI 设置项置灰外，新增了两项增强特性（`danmaku_pro` 与 `media_opt`）：
1. **弹幕特权与高级过滤引擎**：
   - 拦截 MMKV 存储中与 `danmu`/`danmaku` 相关的布尔值读取，强制开启高级个性化特权弹幕渲染与彩色弹幕。
   - 拦截过滤规则屏蔽词上限（`limit`/`max`/`count`），将默认额度扩展至 99999 条。
2. **自动跳过片头片尾与硬件解码画质优化**：
   - 自动预置 `skip_intro` / `skip_outro` / `auto_skip_ending` 相关的布尔开关为 `true`。
   - 优化 mpv 底层解码器配置，将 `hwdec` 预置为 `mediacodec-copy` 硬件解码加速，解除高刷与 HDR 画面渲染限制。
