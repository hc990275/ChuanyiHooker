# ChuanyiHooker 架构设计与逆向分析深度报告

> 项目路径：[project_11_chuanyi_hooker](file:///d:/DeskTop/GitHub/测/project_11_chuanyi_hooker)  
> 远端地址：https://github.com/MiChongs/ChuanyiHooker  
> 分析日期：2026-09-20  

---

## 1. 项目核心定位与设计哲学

ChuanyiHooker 是一个设计极度严谨、工程化程度极高的高级 **Xposed 模块宿主与逆向治理框架**。
它的核心理念是：**「一个目标应用一个 Hooker 独立子模块，框架本体对具体应用零感知」**。

- **高解耦插件体系**：目标应用的逆向逻辑全部封闭在 `hookers/<name>` 独立 Gradle 子模块中。
- **运行时动态发现**：利用 Java SPI 标准机制（`META-INF/services/com.chuanyi.hooker.core.AppHooker`）在运行时自发现并动态加载。
- **作用域编译期聚合**：各子模块只需声明自己的 `META-INF/xposed/scope.list`，宿主在构建时通过 Gradle 任务（`:checkHookerScope`）完成自动校验并合并为全局作用域清单。
- **无感静默注入**：配置 `staticScope=true` 与 `autoHotReload=true`，实现全自动作用域匹配与免杀进程的原地热重载换 Hook。

---

## 2. 核心技术栈与架构设计

```
project_11_chuanyi_hooker/
├── core/            SPI 核心契约、Hooker 运行时、跨进程 RemotePreferences 读写桥、日志中继
├── native/          Dobby Inline Hook、预编译常量返回桩、JNI 动态注册拦截、匿名执行内存解密壳
├── hookers/         23 款针对主流商业应用的独立 Hooker 逆向插件模块
├── app/             模块宿主 APK（libxposed API 102 入口、热重载控制、Miuix 界面、设置中心）
└── patcher/         脱离 Xposed 框架的纯静态 APK 离线字节流补丁链路（针对 Flutter AOT）
```

### 2.1 依赖底层与工具链
- **Xposed 规范**：采用现代化 [libxposed API 102](https://github.com/libxposed/api) 标准接口，结合 [EzHookTool](https://github.com/lingqiqi5211/EzHookTool) 的 `hook-xposed-102` 实现 Kotlin DSL 声明式 Hook。
- **反射引擎**：[KavaRef 1.1.0](https://github.com/HighCapable/KavaRef)（core + extension + android），替代脆弱的原生 Java 反射。
- **Native 挂钩**：集成 LSPosed fork 的 [Dobby](https://github.com/LSPosed/Dobby) 原生内联 Hook 引擎，支持 ARM64/ARMv7/x86 跨架构符号解析与内存修补。
- **DexKit 2.2.0**：用于在代码高度混淆、类名随机变换的场景下，通过字符串常量池锚点与方法签名结构进行动态模糊匹配。
- **UI 审美**：基于 [Miuix 0.9.3](https://github.com/compose-miuix-ui/miuix) 与 Navigation 3 构建现代简约控制台，采用纯色语义配色、渐进式毛玻璃叠加与动态 Monet 调色。
- **构建环境硬要求**：JDK 25+、compileSdk 37.1、NDK r30、CMake 4.1.2、Kotlin 2.4.10。

---

## 3. 关键机制与底层亮点

### 3.1 跨进程配置安全设计 (RemotePreferences)
- **读写分离陷阱规避**：Hook 代码运行于目标应用进程内，持有的是 libxposed `RemotePreferences` 的只读视图；宿主管理 App 持有可写句柄。写配置严格使用 `ModuleSettings.write()` 强制同步落盘，杜绝 `apply()` 后台线程在被系统划掉时丢数据的缺陷。
- **日志中继广播环 (LogRelay)**：目标进程对模块私有存储只读，日志无法通过常规文件落盘；框架采用内存批量缓冲（400ms / 64条）通过动态 Intent 广播回宿主应用，由 `LogReceiver` 落入轮转日志文件中。

### 3.2 原生层防杀与无痕替换 (Native Hook)
- **指令页与数据页权限隔离**：严格区分 `patchMemory`（写完强制恢复为 `RX`，仅限代码段）与 `writeMemory`（保留原内存属性，针对数据段），避免修改标志位或指针时引发延迟发生的 `SEGV_ACCERR` 崩溃。
- **JNI 动态注册拦截 (watchJniRegistrations)**：在目标应用调用 `RegisterNatives` 时，直接在 ART 虚拟机绑定函数指针前替换 `JNINativeMethod.fnPtr`。由于宿主 `.so` 磁盘与内存字节未发生任何更改，可完美绕过应用自身的 PLT 钩子检测与代码段哈希校验。
- **常量返回桩**：内置 32 个预编译轻量级汇编桩函数，无需实时生成机器码或 patch `mov x0, #imm; ret`，原生层支持完全可撤销的无痕常量替换。

### 3.3 激活闸门与防盗用机制 (ActivationGuard & TgGuard)
框架内置了极其巧妙的“TG 加群激活验证机制”：
1. **私有数据库直读**：利用 Telegram-Android 分支共用 `files/cache4.db` 的特性，`TgGuardHooker` 运行于 TG 进程中，通过底层 C++ 手写轻量 SQLite B 树查找算法，在不加载完整 sqlite3 库的前提下，以纳秒级速度探测 `dialogs` 与 `chats` 表中是否存在指定的群组 rowid（`4404720340`）。
2. **匿名内存执行壳 (Payload Shell)**：判定逻辑被单独编译为位置无关的裸机器码（No-relocation / Freestanding ELF），在运行时经 FNV-1a 篡改哈希校验后解密映射入匿名内存（`mmap PROT_READ|PROT_EXEC`），验证完成后立即执行内存抹零（`munmap`）。
3. **SipHash-2-4 动态令牌**：校验通过后签发包含版本号、包名 Hash 与时间戳的 `CYT1` 加密令牌，被注入的各目标应用在冷启动时验证 MAC 签名，未通过则全量 Hook 自动熔断不挂载。

### 3.4 静态改包脱机方案 (Patcher)
针对 Flutter/Dart AOT 编译应用（以 Hills 为例），由于内存反序列化和 JIT 竞争剧烈，框架开辟了独立的 `patcher/` 静态离线改包方案：
- **Snapshot 等长覆写**：直接对 `libapp.so` 内序列化的 Dart Cluster 流（`_kDartIsolateSnapshotData`）进行等长替换，重定向鉴权 API 域名与 RSA 公钥。
- **原包证书提取与替换**：直接解析原包 v2/v3 签名块（`apk_signing_block.py`），在 Dalvik 虚拟机层拦截 `Signature.toByteArray()`，将原包证书的字节输入伪装替换，天然兼容上层的 SHA-256 签名校验。

---

## 4. 内置 23 款目标应用攻防实录

| 序号 | 模块标识 | 目标应用包名 | 逆向突破点与实现手法 |
|---|---|---|---|
| 1 | `airmusic` | `app.airmusic.trial` | 拦截试用噪音注入机制，接管核心授权判定与原生自检 |
| 2 | `astraflow` | `com.astraflow.tool` | 突破注入端 HMAC 鉴权门，复用目标内部签名算法自签凭据 |
| 3 | `bridgeaudio` | `app.bridgeaudio` | 绕过本地权益布尔检查，旁路 Google PairIP 许可证离线强验 |
| 4 | `capyplayer` | `com.feifeiduck.capyplayer` | 伪造 Play Billing 订阅交易凭据，同步接管服务端复核逻辑 |
| 5 | `cellularpro` | `make.more.r2d2.cellular_pro` | 突破 nmmp 虚机与调用栈帧深度反 Hook 校验，通过 JNI 动态注册置换高级会员 |
| 6 | `chuckle` | `app.jjyy.chuckle` | 接管纯本地权益判定，阻断服务端权限撤销与风控埋点上报 |
| 7 | `esj` | `com.gx.sw.qa.fkssj004.esj` | 协议观察台与离线模式支持（绕过服务端鉴权依赖） |
| 8 | `flix` | `com.ifreedomer.flix` | 针对 Dart 侧权益逻辑，在 Java 计费层与本地状态流双端切入拦截 |
| 9 | `gameclick` | `com.pbb.gameclick` | 篡改登录回包、绕过 Native 原生复核与心跳定时联网校验 |
| 10 | `gboard` | `com.google.android.inputmethod.latin` | 彻底移除剪贴板大小与条数硬限制，放开滑动输入，终端免切密码盘 |
| 11 | `gifshop` | `com.gif.gifmaker` | 接管 Zipoapps PremiumHelper 辅助类与 Google Play Billing v7 接口 |
| 12 | `hills` | `com.mountains.hills` | 拦截 Play Billing 回包，并针对 Flutter `libapp.so` 进行离线静态等长替换 |
| 13 | `instashot` | `com.camerasideas.instashot` | 接管 Play 结算生命周期与买断凭据（无壳纯纯本地校验） |
| 14 | `osmin` | `com.osmin` | 地图与导航专业版功能放行与离线包限制解除 |
| 15 | `paisa` | `dev.hemanths.paisa` | 拦截 RevenueCat CustomerInfo 映射实体，在 Dart 反序列化前置入 Pro 状态 |
| 16 | `poweramp` | `com.maxmpz.audioplayer` | 接管原生授权引擎返回的 Bundle，篡改共享状态块，绕过设置加解密层 |
| 17 | `secretshoot`| `com.weixikeji.secretshoot.googleV2` | 会员状态常驻、广告去除、强行开放免登录拍摄功能 |
| 18 | `skypulse` | `com.skypulse.weather` | 本地 HMAC 激活码拦截与新版服务端 RSA JWT 验签规避 |
| 19 | `tgguard` | `Telegram 各分支` | 读取本地 `cache4.db` 会话数据库，无感实现社区资格核验与令牌自签发 |
| 20 | `wink` | `com.meitu.wink` | 接管 `VipInfoData` 掩码计算函数（SVIP=12），就地修补 2099 永久有效期 |
| 21 | `womic` | `com.wo.voice2` | 解除 Play 订阅购买限制，剔除 AdMob 广告横幅与 Google UMP 弹窗 |
| 22 | `yamby` | `com.hush.yamby` | 针对被 nmmp/dex2c 抽空的代码，在 AIDL 计费层注入合法购买 Bundle |
| 23 | `zenneko` | `io.github.wisyh.zenneko` | 绕过 VIP 校验逻辑与 Native 原生环境反调试自检 |

---

## 5. 新增 Hooker 开发规范（SOP）

根据项目规范，新增一个目标应用的 Hook 支持需遵循以下 7 步无侵入流程：

1. **新建子模块**：在 `hookers/` 下创建子目录 `<target_name>`，编写 `build.gradle.kts`。
2. **实现接口**：继承 `com.chuanyi.hooker.core.AppHooker`，编写主逻辑文件 `<Target>Hooker.kt`。
3. **注册 SPI**：在 `src/main/resources/META-INF/services/com.chuanyi.hooker.core.AppHooker` 写入实现类的全限定名。
4. **声明作用域**：在 `src/main/resources/META-INF/xposed/scope.list` 写入目标包名（**末尾必须保留换行符**）。
5. **配置可见性**：在 `src/main/AndroidManifest.xml` 的 `<queries>` 标签中添加该目标包名（确保 Android 11+ 包可见性生效）。
6. **引入工程配置**：在 `settings.gradle.kts` 中追加 `include(":hookers:<target_name>")`。
7. **宿主依赖接入**：在 `app/build.gradle.kts` 的 dependencies 中加入 `implementation(project(":hookers:<target_name>"))`。
