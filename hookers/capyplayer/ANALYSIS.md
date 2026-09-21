# CapyPlayer (卡皮) 安装包深度解剖分析报告（自用版）

## 1. 目标基本信息
- **应用名称**：CapyPlayer (卡皮播放器)
- **目标包名**：`com.feifeiduck.capyplayer`
- **分析样本**：`capyplayer-arm64-v8a-release-1.1.5(11512).apk`
- **核心架构**：Flutter (Dart 3.x AOT 快照编译，未启用 `--obfuscate`) + mpv / ffmpeg 原生解码内核
- **业务授权模式**：Google Play Billing 客户端发起 + 自家后端复验 (`/subscriptions/verify/google`) + Riverpod 状态流驱动 UI

---

## 2. Dart AOT 权益判定拓扑与调用链解剖

由于该 APK 未启用 `--obfuscate` 混淆，`libapp.so`（16.9 MB）中的类名、函数名与源文件路径均明文保存在快照符号池中。经 ELF 与指令流反编译，其核心依赖判定链如下：

```
SubscriptionNotifier.build()  →  SubscriptionState
  ├── _loadSubscription() → _checkIfActive()      ← 【落点 active】0x8220a0
  ├── _verifyWithServer()      POST /subscriptions/verify/google
  └── SubscriptionSyncService.mergeSubscriptions(local, server)
                    ↓ ref.watch
         isSubscribed(Ref) : bool                 ← 【落点 subscribed】0x7849b8 ★ 根状态
                    ↓
         proFeatureEntitlementProvider(Ref, feature)
              ├── 工厂闭包 (Family Provider 构造器)   ← 0x784800 (返回 Provider 对象，严禁 Patch!)
              └── 求值判定主体函数                    ← 0x784a88 【落点 entitlement】(返回 bool)
                    ↓
  PaywallGuard|ensureEntitled() / ensureEntitledAsync()
  NativePlayerSubtitleSearchBridge
    ._ensureSubtitleSearchEntitled()              ← 【落点 subtitle】0xfbb040 (未开通时抛异常)

  ProBadge.build()  ─────────────────────────────→ 直接读 isSubscribedProvider
```

---

## 3. 1.1.5 版本中 Pro 功能不可用的根因分析

### (1) Riverpod Provider 工厂闭包与求值函数混淆（致命病灶）
- **现象**：打上旧补丁后，功能点击依然无效或触发闪退/回退。
- **逆向分析**：
  - 在旧版 1.1.3 锚点中，`0x784800` 被误作为 `proFeatureEntitlement(Ref, ProFeature)`，并在函数入口强行注入指令：
    ```assembly
    add x0, x22, #0x20   ; 返回 Dart 单例 true
    ret
    ```
  - 反汇编 1.1.5 发现：`0x784800` 实际上是 Riverpod 的 Family Provider 工厂闭包。它的退出点在 `+0x164`，其返回值是一个 Provider 实例对象（类型派生自 `AutoDisposeProvider`）。
  - 将一个返回 Provider 对象的工厂强制改成返回布尔值 `true`，导致上游 Riverpod 在执行 `ref.watch(proFeatureEntitlementProvider(feature))` 时发生强转异常（`type 'bool' is not a subtype of type 'Provider<bool>'`），引起逻辑崩溃中断。
  - **真正的求值判定函数**位于 `0x784a88`。该函数内部提取 feature 参数，在 `+0x38` 执行 `add x0, x22, #0x20; ret` 返回 `true`，在 `+0x48` 执行 `add x0, x22, #0x30; ret` 返回 `false`。

### (2) 字幕搜索异常抛出未放行
- `_ensureSubtitleSearchEntitled` 在 1.1.5 中代码段重排，入口移动到 `0xfbb040`。
- 旧 1.1.3 的锚点失效，导致未命中。当用户点击在线字幕搜索时，函数直接走到 `0xfbb084` 分支，抛出 `"Pro entitlement required for subtitle search"` 异常。

---

## 4. 1.1.5 补丁站点设计与特征码清单

为了保证补丁在重启、重新编译后依然稳定可靠，所有特征码均取自函数体内部，不含任何 PC 相对寻址（如 `BL`、`B`、`ADR`、`ADRP`），且避开入口前 8 个字节改写区。

| 站点 ID | 目标函数 / 作用 | 目标入口地址 | 锚点偏移 | 全库唯一特征码 (HEX) | 补丁指令与效果 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`subscribed`** | `isSubscribed(Ref) [isSubscribedProvider]` | `0x7849b8` | `0x1c` | `20F040B800801C8B706F4AF9` | `add x0, x22, #0x20; ret` (恒返回 true) |
| **`entitlement`** | `proFeatureEntitlement(Ref, ProFeature) 求值主体` | `0x784a88` | `0x1c` | `20F040B800801C8B7027409110EE44F9` | `add x0, x22, #0x20; ret` (恒返回 true) |
| **`active`** | `SubscriptionNotifier._checkIfActive` | `0x8220a0` | `0x8` | `EF4100D1E00302AAE20303AAA3831FF8` | `add x0, x22, #0x20; ret` (恒返回 true) |
| **`subtitle`** | `NativePlayerSubtitleSearchBridge._ensureSubtitleSearchEntitled` | `0xfbb040` | `0x8` | `EF2100D1502740F9FF0110EB89020054207040B800801C8BE10300AA` | `mov x0, x22; ret` (返回 null，阻断抛出异常) |

---

## 5. 修复与验证结论
- 代码落地于 `hookers/capyplayer/src/main/kotlin/com/chuanyi/hooker/hookers/capyplayer/Entitlement.kt`。
- 经全量静态扫描与 Gradle 构建验证，4 处站点命中率 100%，序言检测无误，Pro 功能与在线字幕搜索完全解锁。

---

## 6. 多架构兼容 (ARM64 / ARM32 / X86_64) 与轻量反汇编序言自愈扫描器

为解决跨 ABI 设备（如 x86 模拟器、老旧 32 位设备）以及小版本微调引起的偏移漂移问题，重构了 `DartPatch.kt`：
1. **跨架构原生指令合成**：
   - **ARM64**：`add x0, x22, #0x20; ret` (`0xC0, 0x82, 0x00, 0x91, 0xC0, 0x03, 0x5F, 0xD6`)
   - **ARM32**：`add r0, r8, #16; bx lr` (`0x10, 0x00, 0x88, 0xE2, 0x1E, 0xFF, 0x2F, 0xE1`)
   - **X86_64**：`lea rax, [r14 + 0x20]; ret` (`0x49, 0x8D, 0x46, 0x20, 0xC3`)
2. **轻量反汇编特征回溯扫描器**：
   - 若锚点命中但预设的 `anchorOffset` 偏离函数入口，不再直接判定失效；
   - 自动在 `[anchorAt - 128, anchorAt]` 区间内按指令步长向前逆向扫描函数序言（`isPrologue`）；
   - 一旦在安全窗口内识别到唯一的合法序言，自动矫正入口地址并输出纠偏日志，实现跨 ABI 及微小版本的自愈补丁。
