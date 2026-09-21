# CapyPlayer 逆向与热补丁技术分析报告（开源分享版）

## 1. 概述
- **目标类型**：Flutter 跨平台影音播放器 (ARM64)
- **分析版本**：v1.1.5
- **技术要点**：Dart AOT 快照热补丁、Riverpod 状态管理逆向、ARM64 寄存器约定

---

## 2. 授权判定架构简述
CapyPlayer 采用 Flutter 构建前端界面，底层解码依赖 mpv / ffmpeg。业务层授权状态完全在 Dart AOT 快照镜像（`libapp.so`）中判定。

UI 角标（`ProBadge`）与功能拦截器均基于 Riverpod 的 `isSubscribedProvider` 与 `proFeatureEntitlementProvider` 进行单向数据流监听。

---

## 3. 常见问题排查：为什么简单入口 Patch 会导致异常
在针对复杂 Dart 框架（如 Riverpod Family Provider）实施 AOT 代码段补丁时，需要严格区分两类函数：
1. **Provider 实例工厂闭包**：用于向依赖注入容器注册并返回 ProviderElement 节点，其返回类型为复合对象引用。
2. **Provider 业务求值函数**：真正执行业务判定的逻辑主体，返回布尔值。

若直接将工厂函数改写为返回常量布尔值，将破坏整个依赖图树，导致运行时类型强转异常（`type 'bool' is not a subtype of type 'Provider<bool>'`）。必须下潜至实际执行求值的业务分支函数处实施返回劫持。

---

## 4. 补丁设计建议
- 采用局部内存特征码（Pattern Matching）动态定位，提取具有函数特异性且不含 PC 相对偏移的机器码切片。
- 在函数入口处注入 ARM64 Dart 寄存器单例赋值指令：
  - 返回 true：`add x0, x22, #0x20; ret`
  - 返回 null (void 正常出口)：`mov x0, x22; ret`
- 对抛出异常的安全阻断点，采用返回 null 的策略规避异常抛出流程。

---

## 5. 多架构兼容与序言自愈反汇编
1. **多 ABI 指令适配**：
   - 针对 ARM64、ARM32 与 X86_64 设备分别合成目标平台的 Dart 常量返回指令块。
2. **轻量反汇编自愈机制**：
   - 当微小版本变动引起特征偏移微移时，自动回溯安全窗口逆向识别函数序言，实现补丁落点的自愈纠偏。
