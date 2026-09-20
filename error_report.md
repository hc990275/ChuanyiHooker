# 错误与适配排查报告 (Error & Adaptation Report)

## 1. Hills v1.9.0 购买校验重定向失效与掉解锁

### 问题现象
- Hills v1.9.0 启动时 Pro 会亮起，但几秒钟后立即掉回到未购买状态。
- Xposed 日志中显示 `verification endpoint never showed up in memory`，或者 `no verification endpoint among candidates`。
- 静态补丁脚本 `patch_hills.py` 报错 `find_verify_url` 返回 `None`。

### 原因分析
1. **Dart Snapshot 端点字符串结构变更**：
   - 在 Hills 旧版（v1.7.2）中，`libapp.so` 的 Isolate Snapshot 将 Supabase 请求端点固化为完整字符串 `https://api.hills.im/functions/v1/google-verify-purchase` (56 字节)。
   - 在 Hills v1.9.0 中，Flutter 端重构了 Supabase 函数调用，将基础 URL (`https://api.hills.im/functions/v1` 33 字节 / `https://api.hills.im/functions/v1/` 34 字节) 与方法名 (`google-verify-purchase` 22 字节) 拆开存储，调用时动态通过 URI 解析拼接。
2. **打分规则未命中**：
   - 原 `HillsHeap.kt` 在堆内存扫描 `https://` 时，对路径中包含 `verify` (+3)、`purchase` (+3) 打分，门槛为 4 分。拆分后，基础 URL 不含这些词，得分均为 0，被直接丢弃。
   - `VerifyBreaker.kt` 拿不到 URL，未能将网络请求重定向到本机 `127.0.0.1:45871` 的 `VerifyServer`，应用直接向真实后端发包验证，因凭据非法被远程拒绝 (`isValid: false`)，导致撤回权益。

### 解决方案
- 修改 `HillsHeap.kt`：在传统打分规则未命中时，自动回退识别 `/functions/v1` 端点（返回所有匹配的基础端点候选）。
- 修改 `VerifyBreaker.kt`：支持遍历端点列表，对 33 字节与 34 字节的基础 URL 进行等长本地重定向 (`http://127.0.0.1:45871/p...`)。
- 修改 `VerifyServer.kt`：优化转发与请求路由逻辑，准确拦截动态拼接后的 `/pppp.../google-verify-purchase` 并签发有效授权；非验证请求转发时自动剥离填充并还原真实路径。
- 修改 `patcher/tools/snapshot.py`：同步加入对 `/functions/v1` 的候选解析。

---

## 2. Yamby v2.1.0.10 Play Billing 混淆导致 Hooker 未加载

### 问题现象
- Yamby v2.1.0.10 安装后，Hooker 完全不工作，日志提示 `找不到 com.android.billingclient.api.Purchase，Play Billing 库可能被换掉了`。

### 原因分析
1. **R8 混淆深化**：
   - 在 Yamby v2.1.0.10 中，R8 规则不再保留 Google Play Billing Client 库的包名与类名，`com.android.billingclient.api.Purchase` 被混淆为阿拉伯字符类名（如 `o7.ۥۦۧ...`）。
2. **兼容性断言过窄**：
   - `YambyHooker.isCompatible` 使用硬编码字符串检测 `com.android.billingclient.api.Purchase` 及其构造器。由于类名被混淆，方法直接返回 `false`，导致 hook 未安装。
   - 实际上，Play Billing 的底层数据源依然是 AIDL 的 `Bundle`（`INAPP_PURCHASE_DATA_LIST`、`INAPP_PURCHASE_ITEM_LIST` 等），这些系统级 Key 无法被混淆，真正的核心解锁点（`installLifetime`）只要挂载到 `Bundle.getStringArrayList` 即可正常工作。

### 解决方案
- 修改 `Billing.kt`：增加 `findPurchaseClass` 动态查找机制，兼容原版与混淆后的 Purchase 类。
- 修改 `YambyHooker.kt`：放宽 `isCompatible` 判定逻辑，以 AndroidManifest 中无法混淆的 `ProxyBillingActivity` 或动态解析类为准。
- 调整 `installBillingLog` 容错，避免在调试日志开启时因类名查找异常抛错。

---

## 3. Windows 中文路径导致 CMake 4.x/Clang 溢出崩溃与 Gradle 镜像加速

### 问题现象
1. 执行 `./gradlew.bat :app:assembleRelease` 时，构建在 `:native:configureCMakeRelWithDebInfo[arm64-v8a]` 步骤报错：
   `[CXX1429] error when building with cmake ... com.android.ide.common.process.ProcessException: C++ build system [configure] failed`
2. 使用 Python/Subprocess 捕获 CMake.exe 返回码为 `3221226505` (`0xC0000409` 即 `STATUS_STACK_BUFFER_OVERRUN`)，无任何错误输出，程序在初始化阶段直接被 Windows 安全机制终止。
3. 网络下载出现 `Could not get resource 'https://dl.google.com/...' > Read timed out`。

### 原因分析
1. **中文路径与编码溢出**：
   - 工程位于包含中文的父目录 `d:\DeskTop\GitHub\测\...`。Windows 系统代码页默认是 936 (GBK)，AGP 通过 ProcessBuilder 传递带中文的 `-HD:\...` 绝对路径给 `cmake.exe`。
   - CMake 4.1.2 在 Windows 环境下内部尝试进行宽字符/UTF-8 路径解析与参数复制，当遇到 GBK 编码的双字节中文字符时发生栈缓冲区越界，触发 `0xC0000409` 崩溃。
2. **Junction 与 CanonicalPath 特性**：
   - 尝试创建 Windows Directory Junction (`D:\ChuanyiHookerBuild`) 规避，但在 Java/Gradle 内部，`File.getCanonicalPath()` / `getCanonicalFile()` 会自动将 Junction 逆向解析为其物理源路径，导致传递给 AGP 的路径再次变回包含中文字符的路径。
3. **海外 Maven 源超时**：
   - 国内网络直接拉取 `dl.google.com` 偶发 Read timed out。

### 解决方案
1. **虚拟根盘符映射 (subst)**：
   - 在 Windows 上使用 `subst Z: "d:\DeskTop\GitHub\测\project_11_chuanyi_hooker"` 将项目目录映射为一个顶层独立根驱动器 `Z:\`。
   - 在 Java/Win32 底层，`Z:\` 的 `canonicalPath` 为根盘符自身，不会被解析回底层物理路径，CMake 接收到的路径为纯 ASCII 字符 `Z:\...`，彻底杜绝了栈溢出崩溃。
2. **镜像源优先配置**：
   - 在 `settings.gradle.kts` 的 `pluginManagement` 与 `dependencyResolutionManagement` 前置引入阿里云 Maven 镜像源：
     `maven("https://maven.aliyun.com/repository/public")` 与 `maven("https://maven.aliyun.com/repository/google")`，实现毫秒级依赖下载并规避海外源超时。
3. **C 盘零占用闭环**：
   - 工具链（JDK 26、Android SDK、NDK 30、CMake 4.1.2）全部落地在 `D:\` 盘，并建立软链接重定向，完美遵循磁盘保护准则。

---

## 4. CapyPlayer v1.1.5 (11512) Dart AOT 指令重排与热补丁失效

### 问题现象
- 用户提供 `capyplayer-arm64-v8a-release-1.1.5(11512).apk`。
- 原 1.1.3 版热补丁在 1.1.5 上加载时提示 `Patch failed` 或由于找不到匹配的特征码而未生效，Pro 订阅未解锁。

### 原因分析
1. **Dart 编译器版本升级与指令布局重排**：
   - 在 CapyPlayer 1.1.3 中，`isSubscribed` 与 `proFeatureEntitlement` 等核心判断函数的锚点特征包含确定的 prologue 结构 (`stp x29, x30, [sp, -16]!` 即 `FD7BBEA9`)。
   - 在 1.1.5 中，Flutter/Dart 引擎升级，`libapp.so` 代码重排：
     - `isSubscribed` 旧锚点完全消失，新函数位于 `0x7849d4`，其前导不再是单纯的标准栈帧开辟，而是寄存器初始化序列。其关键 12 字节特征为：`20F040B800801C8B706F4AF9` (位于偏移 `+0x14`)。
     - `_checkIfActive` 移至 `0x8220a8`，新特征为 `EF4100D1E00302AAE20303AAA3831FF8` (偏移 `+0x8`)。
     - `proFeatureEntitlement` 依然保留在 `0x784860`，原 12 字节特征 `FF0110EB29080054403F40F9` 全库唯一且兼容。
2. **函数序言校验过窄**：
   - `DartPatch.kt` 原逻辑只接受标准 Prologue：`FD7BBEA9` (`stp x29, x30, [sp, #-16]!`) 与 `BF0300D5` (`nop`)。
   - Dart AOT 中存在大量直接跳转跳转块或使用 `sub sp, sp, #imm`、`b #imm` 的优化函数头，过于严苛的 prologue 校验导致有效 patch 点被错误判定为非法代码而放弃 hook。

### 解决方案
1. **拓展多版本锚点字典 (Multi-Version SITES)**：
   - 在 `Entitlement.kt` 中引入双版本兼容策略：优先匹配 1.1.5 新特征，未命中时回退到 1.1.3 旧特征。
2. **放宽函数 Prologue 识别 (isPrologue)**：
   - 在 `DartPatch.kt` 中增加对 `stp x29, x30` (`fd7b..a9`)、`sub sp, sp, #imm` (`..00d1`)、`b #imm` (`......14` / `......17`) 等常见 Dart AOT 代码头的支持。
3. **保持热补丁幂等与零延迟**：
   - 补丁逻辑直接注入 `mov x0, #1; ret`，瞬时放行全部 Pro 权益。
