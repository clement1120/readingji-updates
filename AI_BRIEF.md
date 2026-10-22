# 阅读纪（ReadingJi）· 协作说明

> 给接手这个项目的 AI 看。目的是省掉重新摸底的十几轮对话。
> 本文件不含任何密码 / 令牌 / 私密信息。

## 1. 这是个什么项目

基于开源阅读器 **Legado** 做的「纯血 Material 3」风格阅读器，包名 `com.yueduji.app.debug`，
中文名「阅读纪」。用户嫌原版不好看，目标是彻底 MD3 化 + 流畅。

- 源码工作副本：`legado-md3/`
- 上游基线：`HapeLee/legado-with-MD3`（官方 `gedoor/legado` 已删库停更）
- 工具链（已装好，别重装）：`sdk` / `jdk-21` / `gradle-8.9`
- 独立版本线：`app/version.properties` 里的 `VERSION_PATCH`（3.27.x）

构建命令（**每次都要带全环境变量**）：

```bash
export JAVA_HOME="/c/Program Files/Microsoft/jdk-21.0.12.101-hotspot"
export ANDROID_HOME="<工具链>/sdk"
cd legado-md3
./gradlew :app:assembleRelease --console=plain > /path/to/build.log 2>&1
```

- Kotlin 编译任务名是 `:app:compileAppDebugKotlin`（不是 `compileDebugKotlin`）
- **构建日志必须落盘**，后台任务的输出经常丢失
- 失败构建留下的产物会被下次复用，22 秒"秒过"要先确认 APK 时间戳和 versionCode 是新的
- 本机没有 `gh` 命令，要打 GitHub API 就用 `curl`

## 2. 用户的审美要求（做 UI 时默认照这个来）

- 整体走 **HarmonyOS NEXT 液态玻璃风**：多层柔光背景、毛玻璃卡片、大圆角 18–24dp、
  渐变 + 内高光按钮、cubic-bezier 流体动效
- **不许照搬微信或苹果的设计语言**。可以参照 Material 3 / Android 12L 的规范，
  但成品不能"像微信""像 iOS"
- **深色模式不是把颜色反过来**：底层 → 普通卡片 → 强调卡片要逐级提亮，
  配内高光 + 外阴影，不能一片死黑
- 必须做横屏 / 平板适配：宽屏把底部标签栏换成左侧导航栏 + 双栏，不能拉满全宽单行
- **图标用自定义几何 SVG**，禁止拿 Unicode 字符（◍ ◔ ⚙）当图标——不同手机字体渲染不一致
- 字号要有统一层级：正文 16sp 起步，最小不低于 12.5sp，图表里的数字同样适用
- 交互要有"确认感"：点击涟漪 + 轻震动 + spring 回弹，弹窗支持下滑关闭
- 深浅色切换入口放「设置」页，不要在页面右上角放悬浮主题按钮；
  要有"跟随系统 / 浅色 / 深色"三档并持久化
- 主色倾向鸿蒙蓝紫（#3B82F6 → #4F46E5）

## 3. 协作方式（怎么跟用户配合）

- **中文沟通**
- **别频繁提问打断**。先自己查代码、自己推进；真需要确认时一次问清，
  并说明为什么必须问
- **过程中持续汇报进度**，每完成一个小步骤就同步，不要憋到最后一次性输出
- 每一处改动都要讲清楚「改了哪 + 会连锁影响哪」，不能只说"已优化"
- 代码评审按 **P0 / P1 / P2** 输出，每条附行号、根因、修复建议
- 设计上被否定后，要换一个明显不同的方向重做，不要原地微调
- 本地缺的依赖环境可以自行下载安装，不要因为缺工具就交付半成品
- 风格直接、诚实，接受负面结论（比如"现成的免费方案不存在"），但要给替代路径
- 出产物**只发最新的那一个**；产物目录只保留最新两个，历史版本靠 git

## 4. 发版规则（踩过坑，别再犯）

| 规则 | 原因 |
|---|---|
| `bases/` 里**只放 release 包** | debug 包当基线会让差分包退化成 57MB（≈全量），13 条就 595MB，直接爆掉配额 |
| **绝不能把 debug 包留在用户手机上当日常版本** | `debuggable=true` 时 ART 不做 AOT、Compose 开重组跟踪、无 R8，用户会反馈"非常非常卡"。装完必须换回 release 并主动告知 |
| 每次发版要为**所有在用版本**各生成一条补丁 | 覆盖没升级到上一版的用户 |
| 发版前逐个检查 `update.json` 里每个 from-version 都指向当前版 | 脚本只按 key 覆盖，旧条目会残留成"升级到旧版" |
| 发版前确认 **versionName 和 versionCode 都递增** | versionCode 由 version.properties 派生：major*10000+minor*100+patch |
| 上传失败必须**回滚云端清单** | 否则 APP 提示有新版本却下载失败 |
| 装机抓完性能数据后**必须换回 release** | 同第 2 条 |

验证 debuggable：`aapt2 dump badging x.apk | grep -i debuggable`（release 应无输出）。

## 5. 更新分发（2026-10-22 起走 GitHub）

坚果云免费版每月只有 1GB 上传配额，还要和 APP 内 WebDAV 书籍备份抢，
2026-10-21 已耗尽 → 3.27.9 之后所有版本传不上去，云端一度停在 3.27.8。
现在改成 **GitHub Releases**：

- 仓库：独立的分发仓库，只放 APK / 补丁 / update.json，**不放源码**
- `update.json` 走 raw 分支（改完立刻生效）
- APK 和补丁作为 **Release 附件**（不进 Git 历史，避免仓库膨胀）
- 下载用 `releases/latest/download/<文件名>` 免版本号直链，公开仓库无需鉴权
- 上传脚本：`upload_to_github.sh <token>`
- 代码位置：`app/src/main/java/io/legado/app/help/update/AppUpdateGh.kt`
- **改这个类里的嵌套数据类后，记得同步 proguard 的 `-keep` 规则**，
  否则 R8 混淆后 GSON 解析为空，检测不到更新

## 6. 上游同步规则

- **禁止盲 merge**。本地与上游有分歧的改动要按文件逐一审阅后 cherry-pick
- 本地独有的改动：settings.gradle 阿里云镜像优先、包名/应用名、独立 3.27.x 版本线、
  ABI splits / debug keystore 签名、自建更新通道、proguard 的 GSON 保留规则、菜单预热

## 7. 已知的技术坑

**Compose 常驻隐藏层**：`graphicsLayer{alpha=0}` 和 `drawWithContent{}` 只跳过绘制，
**不跳过命中测试**。任何"隐藏但常驻"的覆盖层都会继续吃掉触摸。
必须显式在 `PointerEventPass.Initial` 里 `consume()`。

**Compose 重组隔离**：想让子树跳过重组，捕获的必须是恒定实例（State / lambda），
把 `.value` 的读取推到最小作用域（`Modifier.composed` 是官方为此准备的）。
注意 `PointerInputChange.consume()` 是成员函数，不要 import。

**删死代码的方法论**：按文件名批量删之前，先 `grep -rnw` 该文件里**每个顶层符号**，
确认无外部引用。只 grep 文件名对应的类名会漏。删完必跑一次构建；
单元测试源集平时不编译，孤儿测试要单独扫。

**真机性能取证**：一加 PKJ110 是 **120Hz**，单帧预算 8.29ms（不是 16.7ms）。
工具在 `reader-perf/`：`auto_menu_capture.py`（带逐步校验，支持 `--scenario menu/page/both`）、
`compare_traces.py`（两份 trace 对比）。
这个 ROM 上 perfetto 只能写 `/data/misc/perfetto-traces`，且不能带 `-b` 参数。
trace_processor 的 SQL：`slice` 表没有 `utid` 列，要 JOIN `thread_track` 才拿得到线程名。

**长期存在的系统级开销**：透明导航栏 + `enableEdgeToEdge()` 时，若
`isNavigationBarContrastEnforced = false`，系统会用 `RegionSamplingHelper`
持续采样屏幕像素（trace 里叫 `Fetching RGB`），实测每 300ms 一次、每次约 300ms，
无缝占用一个 system_server 线程。这是系统级开销，APP 内怎么优化都影响不到它。
