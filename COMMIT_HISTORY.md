# Claude Code Best — 项目演进全记录与工作进度分析

> 本文档梳理项目从反编译到可用版本的完整演进过程（125 条提交，2026-03-31 ~ 2026-04-03），
> 并结合当前代码状态，分析「已完成」与「待完成」的工作，为后续规划提供参考。
>
> 数据来源：上游仓库 https://github.com/claude-code-best/claude-code (同步至 `e944633`)

---

## 第一部分：提交历史追踪

### 阶段一：反编译落地与首次启动（3/31 19:00 ~ 21:00）

**核心问题：如何让反编译出来的 Claude Code 源码跑起来？**

反编译产物面临三大障碍：
1. 大量 `@ant/*` 内部包和 `*-napi` 原生模块无法解析
2. `feature()` 函数来自 `bun:bundle` 构建时 API，运行时不存在
3. 依赖版本不匹配，`bun install` 直接失败

| Commit | 做了什么 |
|--------|---------|
| `f90eee8` | 初始提交 — 导入全部反编译源码（src/、bun.lock、package.json），包含 QueryEngine、Tool 系统、Bridge、REPL 等完整模块 |
| `8fc3ddb` | 补全缺失的 npm 依赖，让 `bun install` 能通过 |
| `c26d614` | 进一步调整依赖版本和配置 |
| `bd756cc` | 为 Anthropic 内部包创建 stub（空壳实现），解决 `@ant/*` 和 `*-napi` 包的 import 报错 |
| `751a684` | **里程碑：第一个可启动版本** — 修复 `cli.tsx` 入口，注入 `feature()` polyfill（始终返回 false），绕过所有内部 feature flag |
| `3d4cb09` | **Monorepo 构建完成** — 在 `packages/` 下创建所有内部包的 stub（@ant/computer-use-*、*-napi 等），Bun workspace 解析通过 |
| `c4d9217` | 完成大部分操作模块的基础修复 |

**解决方案**：通过 stub 包 + `feature()` polyfill 两板斧，把 Anthropic 内部依赖全部架空，让反编译代码能在 Bun 上跑起来。

---

### 阶段二：类型系统修复（3/31 21:40 ~ 4/1 01:00）

**核心问题：反编译产生了大量 `unknown`/`never`/`any` 类型，tsc 报错 1341 个，如何清理？**

反编译器无法还原原始类型信息，产出的代码充斥着 `any`、`unknown`、`never`、`` 等占位类型。虽然 Bun 运行时不检查类型，但这严重影响代码可维护性和 IDE 体验。

| Commit | 做了什么 |
|--------|---------|
| `2c759fe` | 第一轮类型修复 — 修复最明显的类型不匹配 |
| `4c0a655` | 大规模清理 — 同时处理类型问题和依赖问题 |
| `d7a729c` | 第二版类型清理 — 深入各模块逐个修复 |
| `dd9cd78` | 封包处理 — 解决模块打包相关的类型问题 |
| `91f77ea` | 又一大波类型修复 — 此时仍有不少 `any` 作为过渡 |
| `fac9341` | **里程碑：tsc 零错误** — 修复 33 个原始编译错误 + 清理 176 处 `any` 标注 + 修复清理过程中引入的 41 个回归错误。最终 0 tsc 错误、0 个 `any`，构建产物 25.75MB |

**解决方案**：分四轮迭代，先粗后细。先用 `any` 占位让编译通过，再逐步替换为真实类型。最后一轮集中攻坚，一次性消灭所有 `any`。

---

### 阶段三：NAPI 原生包实现（4/1 01:07 ~ 08:48）

**核心问题：原版依赖 Node.js NAPI 原生模块和 Swift 原生代码，Bun 环境下怎么替代？**

Claude Code 官方版使用了多个 C++/Swift 编写的原生模块（通过 Node NAPI 绑定），这些在反编译后只有 JS 接口定义，没有原生二进制。需要用纯 TS/脚本方案替代。

| Commit | 做了什么 |
|--------|---------|
| `7e15974` | **实现 4 个 NAPI 包** — modifiers-napi（Bun FFI 调 macOS `CGEventSourceFlagsState` 检测修饰键）、image-processor-napi（集成 sharp + osascript 读剪贴板图片）、audio-capture-napi（基于 SoX/arecord 的跨平台音频录制）、url-handler-napi（函数签名补全，保持 null fallback） |
| `975b487` | **实现 @ant/computer-use-input** — 用 AppleScript + JXA 实现完整键鼠模拟 API（鼠标移动/点击/滚轮、键盘输入、获取前台应用信息），兼容 require() 调用方式 |
| `b51b2d7` | **升级 @ant/computer-use-mcp** — 从空 stub 升级为类型安全实现，`targetImageSize()` 实现真实缩放逻辑，添加 10 个 macOS 敏感应用的 sentinel 列表 |
| `722d59b` | **实现 @ant/computer-use-swift** — 用 JXA + screencapture 命令替代原始 Swift 原生模块，实现显示器信息获取、应用列表、截图等功能。实测验证通过 |

**解决方案**：用 macOS 原生脚本技术栈（AppleScript/JXA/screencapture/Bun FFI）替代 Node NAPI 和 Swift 原生模块，实现等价功能。代价是仅支持 macOS。

---

### 阶段四：工程化基础设施（4/1 01:40 ~ 07:17）

**核心问题：反编译项目没有任何工程化配置（无 lint、无测试、无 CI），如何建立代码质量保障？**

| Commit | 做了什么 |
|--------|---------|
| `074ea84` | 配置 Biome 代码格式化与 lint 工具 |
| `4319afc` | 配置 git pre-commit hook — 提交前自动运行 Biome 检查，使用 `.githooks/` + `core.hooksPath` 方案，零依赖 |
| `30e863c` | **调优 Biome 规则** — 关闭 formatter 和 organizeImports（避免对反编译代码产生大规模 diff），关闭 12 条不适用的 lint 规则，零源码改动 |
| `e443a8f` | **搭建测试基础设施** — 配置 Bun test runner + bunfig.toml，编写 3 组示例测试（array/set/color-diff），41 tests 通过 |
| `17ec716` | 添加 GitHub Actions CI — push/PR 自动运行 lint → test → build 三步检查 |
| `c587a64` | 添加 knip 冗余代码检查，扫描未使用的文件/exports/依赖 |
| `173d18b` | 添加代码健康度检查脚本（health-check.ts），一键汇总代码规模、lint、测试、构建等指标 |

**解决方案**：Biome（lint only，不格式化）+ pre-commit hook + CI + knip + health-check，在不破坏反编译代码风格的前提下建立质量门禁。

---

### 阶段五：构建修复与可发布版本（4/1 09:00 ~ 10:42）

**核心问题：合并分支后构建失败，如何修复并产出可用的构建产物？**

| Commit | 做了什么 |
|--------|---------|
| `b32dd45` | 修复构建问题 — cli.tsx 缺少导入导致 bundle 失败 |
| `9a57642` | **里程碑：完成最新可构建版本** — 重写 build.ts 构建脚本，修复 modifiers-napi 兼容性，产出可用的 dist/cli.js（~25MB） |

**解决方案**：修复入口文件缺失导入 + 重写构建脚本适配 Bun 打包。

---

### 阶段六：功能解锁 — 移除 Feature Flag 限制（4/1 11:53 ~ 17:11）

**核心问题：反编译版本中 `feature()` 始终返回 false，导致部分本应公开的功能也被错误禁用，如何逐个解锁？**

这是最有价值的一个阶段 — 不是所有 feature flag 都是内部功能，有些是已经对外发布但仍通过 flag 控制的功能。需要逐个甄别并解锁。

| Commit | 问题 → 解决 |
|--------|------------|
| `33fe494` | **/loop 命令不可用** — `isKairosCronEnabled()` 依赖 `feature('AGENT_TRIGGERS')` 始终为 false，导致 /loop skill 被禁用。简化为仅检查 `CLAUDE_CODE_DISABLE_CRON` 环境变量 |
| `2934f30` | **/loop 仍然不可用** — 上次只修了一个入口点，但整条链路有 5 处被 feature flag 拦截：skills 注册、tools 加载、工具名列表、REPL hook、pipe 模式 cron 调度器。一次性全部移除 |
| `a889ed8` | **/config 命令报错** — Settings 组件引用了 Anthropic 内部的 `Gates` 组件（反编译版本中不存在），运行时抛出 ReferenceError。移除该引用 |
| `221fb6e` | **@ 文件搜索无结果** — execa 新版将 `signal` 选项重命名为 `cancelSignal`，导致 `execFileNoThrowWithCwd` 调用 `git ls-files` 时抛出 TypeError，文件索引始终为空。修复参数名，同时改进 FileIndex 的模糊匹配算法 |

**解决方案**：逐个排查被 feature flag 或内部依赖阻断的功能链路，移除 gate 或替换内部组件引用。关键经验：一个功能的 gate 可能分布在 5+ 个文件中，只修一处不够。

---

### 阶段七：重构 — 全局变量注入改为 Bun 原生 define（4/2 09:51）

**核心问题：cli.tsx 顶部用 `globalThis` 注入 MACRO/BUILD_*/feature 等全局变量，hack 味太重且不利于测试，如何优雅化？**

| Commit | 做了什么 |
|--------|---------|
| `28e40dd` | 删除 cli.tsx 顶部所有 `globalThis` 注入，新增 `scripts/defines.ts` 作为 MACRO 定义的单一来源。dev 模式通过 `bun run -d` 转译时注入，build 模式通过 `getMacroDefines()` 构建时内联。55 个 MACRO 消费文件零改动 |

**解决方案**：利用 Bun 原生的 `define` 能力（类似 webpack DefinePlugin），在转译/构建阶段替换常量，消除运行时 globalThis hack。

---

### 阶段八：大规模测试覆盖（4/1 21:00 ~ 4/2 10:11）

**核心问题：反编译代码没有任何测试，如何在不触发重依赖链（Anthropic SDK、文件系统、网络）的情况下建立测试覆盖？**

策略：纯函数优先 + `mock.module()` 切断重依赖。

#### 第一轮：核心模块测试（4/1 21:32 ~ 22:50，共 517 tests）

| Commit | 覆盖模块 | 测试数 |
|--------|---------|--------|
| `67baea3` | Tool 系统（buildTool、findToolByName、toolMatchesName 等） | 46 |
| `cad6409` | Utils 纯函数（xml、hash、stringUtils、semver、uuid 等 10 个模块） | 190 |
| `c4344c4` | Context 构建（stripHtmlComments、buildEffectiveSystemPrompt 等） | 25 |
| `583d043` | 权限规则解析器（escapeRuleContent、permissionRuleValueFromString 等） | 25 |
| `25839ab` | 模型路由（isModelAlias、getAPIProvider 等） | 40 |
| `c57950e` | 消息处理（createAssistantMessage、normalizeMessages 等） | 56 |
| `f81a767` | Cron 调度（parseCronExpression、computeNextCronRun 等） | 38 |
| `3df4b95` | Git 工具函数（normalizeGitRemoteUrl） | 18 |
| `1834213` | 配置与设置系统（Schema 验证、MCP 类型守卫等） | 62 |

#### 第二轮：补充模块测试（4/1 23:56 ~ 4/2 08:08，累计 647 tests）

| Commit | 覆盖模块 | 测试数 |
|--------|---------|--------|
| `43af260` | json/truncate/path/tokens — 用 `mock.module()` 切断 log.ts/tokenEstimation.ts 重依赖链 | 88 |
| `a28a44f` | FileEditTool utils / permissions / filterToolsByDenyRules | 42 |

#### 第三轮：Phase 1-5 扩展（4/2 08:50 ~ 10:11，累计 1177 tests）

| Commit | 覆盖模块 | 新增测试数 |
|--------|---------|-----------|
| `acfaac5` Phase 1 | errors、shellRuleMatching、argumentSubstitution、CircularBuffer、sanitization、slashCommandParsing 等 8 个文件 | +134 |
| `21ac9e4` Phase 2-4 | envUtils、sleep、memoize、groupToolUses、dangerousPatterns、zodToJsonSchema、PermissionMode、mcpStringUtils、commandSemantics 等 12 个文件 | +321 |
| `4f323ef` Phase 5 | effort、tokenBudget、displayTags、taggedId、MCP normalization、gitConfigParser、hyperlink、windowsPaths、notebook 等 12 个文件 | +209 |

**最终结果**：从 0 tests 提升到 **1177 tests / 52 files**，全部通过。

---

### 阶段九：文档建设（贯穿全程）

**核心问题：反编译项目需要文档来解释架构、隐藏功能和使用方式。**

| 关键节点 | 做了什么 |
|---------|---------|
| `2de3d30` ~ `4692c3e` | 添加基础 README 和 Bun 使用说明 |
| `f6fe944` | 搭建 Mintlify 文档站 |
| `2fa9148` | 撰写「揭秘：隐藏功能与内部机制」系列 — 88+ feature flags 分类、GrowthBook A/B 测试体系、KAIROS/PROACTIVE/BRIDGE 等未公开功能深度分析 |
| `c5b55c1` | 完成大量文档内容（架构、工具、命令等） |
| `a426a50` | 完善测试规范文档 |
| `64f79dc` | SEO 优化 |
| 多次 docs 提交 | README、SECURITY.md、赞助说明、配图等 |

---

### 阶段十：上游社区贡献 — Feature Flag 体系重构与遥测脱钩（4/2 ~ 4/3）

**核心问题：feature() polyfill 方案太粗暴（全部返回 false），如何让开发者按需启用功能？遥测系统仍在向 Anthropic 发送数据，如何脱钩？**

这是上游仓库在阶段一~九基础上的重大推进，由多位社区贡献者完成。

#### 10.1 Feature Flag 体系重构

| Commit | 做了什么 |
|--------|---------|
| `be82b71` | **核心改动：用 Bun 原生 `feature()` 替代 polyfill** — `scripts/dev.ts` 和 `build.ts` 扫描 `FEATURE_*` 环境变量，转换为 Bun 的 `--feature` 参数。开发者通过 `FEATURE_BUDDY=1 bun run dev` 即可按需启用任意 flag，不再需要改代码 |
| `919cf55` | 添加开发者默认开启的 feature 列表 |
| `47d8847` | 文档：修正 feature 的正确用法（不要自己定义 feature 函数，用 `bun:bundle` 内置） |

**影响**：这彻底改变了 feature flag 的处理方式。之前的文档中说「需要逐个解锁 flag」，现在只需设置环境变量即可。CLAUDE.md 也已更新。

#### 10.2 遥测系统脱钩

| Commit | 做了什么 |
|--------|---------|
| `78144b4` | **关闭 Datadog 日志发送** — 不再向 Anthropic 的 Datadog 端点上报数据 |
| `1195185` | **更新 Sentry 错误上报** — 添加 `src/utils/sentry.ts`，可配置化 |
| `e74c009` | **GrowthBook 自定义服务器适配器** — 通过 `CLAUDE_GB_ADAPTER_URL/KEY` 环境变量连接自定义 GrowthBook 实例，无配置时所有 feature 读取返回代码默认值。彻底解除对 Anthropic GrowthBook 的依赖 |
| `e32c159` | **关闭自动更新** — 移除自动更新检查，避免连接 Anthropic 服务器 |

#### 10.3 功能修复与增强

| Commit | 做了什么 |
|--------|---------|
| `c57ad65` + `f71530a` + `7dfbcd0` | **Buddy 命令实现** — 支持 `/buddy` 命令，修复 rehatch 问题，添加完整的 buddy.ts 命令文件 |
| `4ab4506` | **修复 USER_TYPE=ant 时 TUI 无法启动** — 反编译版本中 `global.d.ts` 声明的全局函数运行时未定义，通过显式 import、stub 组件和全局 polyfill 修复 |
| `68ccf28` + `be82b71` | **Auto Mode 修复** — 补全 yolo-classifier-prompts/ 三个缺失的 prompt 模板文件，auto mode 分类器可用 |
| `c252294` | **移除反蒸馏代码** — 删除 `ANTI_DISTILLATION_CC` 相关的假工具注入逻辑 |
| `e48da39` | **修正 Web Search 工具** — 添加 Bing adapter，支持通过 API 适配器模式扩展搜索后端 |
| `d04e00f` | **调整预先检查代码** — 优化启动时的 preflight checks |
| `ac1f029` | **批量修正 external 字面量** — 统一 `USER_TYPE` 相关的字符串处理 |
| `e944633` | **修复 getAntModels is not defined** — 修复 #69 issue，解决运行时 ReferenceError |

#### 10.4 代码清理

| Commit | 做了什么 |
|--------|---------|
| `991ccc6` | 删除 `src/src/` 下的重复目录（反编译产物残留） |
| `88b45e0` | 删除垃圾脚本（`create-type-stubs.mjs`、`fix-default-stubs.mjs`、`fix-missing-exports.mjs`、`remove-sourcemaps.mjs`） |
| `87fdd45` | 删除调试代码 |
| `1f0a2e4` | 完成 debug 配置（`.vscode/launch.json`） |

#### 10.5 测试大幅扩展

| Commit | 做了什么 |
|--------|---------|
| `006ad97` ~ `ce29527` | 新增大量测试文件 — Phase 16-19，覆盖 AgentTool、LSPTool、MCPTool、PowerShellTool、WebFetchTool、WebSearchTool、compact service、MCP service、store、集成测试等 |
| 集成测试 | 新增 `tests/integration/` 目录：cli-arguments、context-build、message-pipeline、tool-chain 四个集成测试文件 |

**最终测试状态**：114 个测试文件（从 52 个翻倍），含单元测试 + 集成测试。

#### 10.6 学习文档与 Feature 文档

| Commit | 做了什么 |
|--------|---------|
| `b6f3708` | 添加 Claude Code 源码学习笔记（`learn/` 目录，启动流程 + 对话循环两个阶段） |
| `5ee49fd` | 添加 20+ 个 feature 的详细描述文档（`docs/features/`），每个 flag 一个文件 |
| 多次 docs 提交 | auto-updater、telemetry audit、GrowthBook adapter、Sentry setup、auto-mode 安全文档等 |

---

### 时间线总览（更新版）

```
3/31 19:00  ┃ 反编译源码导入
     ↓      ┃ 依赖补全 → stub 创建 → 首次启动
3/31 21:00  ┃ ─── 阶段一完成：能跑了 ───
     ↓      ┃ 四轮类型修复
4/1  01:00  ┃ ─── 阶段二完成：tsc 零错误 ───
     ↓      ┃ NAPI 包实现 + 工程化配置
4/1  09:00  ┃ ─── 阶段三四完成：原生能力 + CI ───
     ↓      ┃ 构建修复 + 文档站 + 功能解锁
4/1  17:00  ┃ ─── 阶段五六完成：可用版本 ───
     ↓      ┃ 大规模测试编写
4/2  10:00  ┃ ─── 阶段七八完成：1177 tests ───
     ↓      ┃ Feature Flag 体系重构 + 遥测脱钩 + 社区贡献
4/3  12:00  ┃ ─── 阶段十完成：feature 按需启用 + 遥测独立 ───
```

---

## 第二部分：当前进度分析 — 已完成 vs 待完成

> 目标：产出一个可用、可发行的 Claude Code 反编译版本。
> 以下基于代码实际状态分析，不只看 commit message。

---

### 一、内部包（packages/）实现状态

| 包名 | 当前状态 | 说明 |
|------|---------|------|
| `color-diff-napi` | ✅ 完整实现 | 纯 TS 重写，含 highlight.js 语法高亮，有测试覆盖 |
| `modifiers-napi` | ✅ 完整实现 | Bun FFI 调 macOS `CGEventSourceFlagsState`，有 graceful fallback |
| `@ant/computer-use-input` | ✅ 完整实现 | AppleScript + JXA 实现键鼠模拟全套 API，仅 macOS |
| `audio-capture-napi` | ⚠️ 部分实现 | 基于 SoX/arecord 的子进程方案，macOS/Linux 可用，Windows 不支持 |
| `image-processor-napi` | ⚠️ 部分实现 | 集成 sharp，剪贴板图片读取仅 macOS（osascript 临时文件中转） |
| `@ant/computer-use-swift` | ⚠️ 部分实现 | JXA + screencapture 替代 Swift 原生模块，部分函数降级（`prepareDisplay` 返回空、`iconDataUrl` 返回 null） |
| `@ant/computer-use-mcp` | ❌ 功能性 stub | `buildComputerUseTools()` 返回 `[]`，`createComputerUseMcpServer()` 返回 `null`，`ComputerExecutor` 为空。类型和 sentinel 数据是真实的，但核心功能不工作 |
| `@ant/claude-for-chrome-mcp` | ❌ 空 stub | `BROWSER_TOOLS = []`，`createClaudeForChromeMcpServer()` 返回 `null` |
| `url-handler-napi` | ❌ 空 stub | `waitForUrlEvent()` 始终返回 `null`，深度链接/URL 事件监听不工作 |

---

### 二、核心子系统工作状态

#### ✅ 已完成且可用

| 子系统 | 状态说明 |
|--------|---------|
| **API 客户端** | `src/services/api/claude.ts` — 完整实现，支持 Anthropic 直连、AWS Bedrock、Google Vertex、Azure Foundry 四种 provider |
| **核心对话循环** | `src/query.ts` + `src/QueryEngine.ts` — 消息发送、流式响应、工具调用、对话轮次管理均可用 |
| **REPL 交互界面** | `src/screens/REPL.tsx` — Ink 终端 UI，用户输入、消息展示、工具权限提示、快捷键均工作 |
| **Tool 系统** | 50+ 工具目录，核心工具（Bash、FileEdit、FileRead、FileWrite、Glob、Grep、Agent、WebFetch、WebSearch 等）均可用 |
| **MCP 客户端** | `src/services/mcp/` — 完整实现，支持 stdio/SSE/streamable HTTP/WebSocket/in-process 五种传输方式 |
| **权限系统** | 权限规则解析、工具权限检查、deny/allow/ask 三级控制均工作 |
| **配置系统** | settings.json 读写、CLAUDE.md 发现与加载、多级配置合并均工作 |
| **OAuth 认证** | `src/services/oauth/` — 完整实现，OAuth 授权码流程、token 刷新、API key 管理 |
| **模型路由** | `src/utils/model/providers.ts` — 根据环境变量自动选择 provider |
| **Cron 调度** | `/loop` 命令和 CronCreate/Delete/List 工具已解锁可用 |
| **构建系统** | `build.ts` — Bun code-splitting 打包，产出 `dist/cli.js` + ~450 chunk files，自动后处理 `import.meta.require` 使产物兼容 Node.js |
| **测试基础设施** | Bun test runner，114 个测试文件（含单元测试 + 集成测试），全部通过 |
| **CI/CD** | GitHub Actions 自动运行 lint → test → build |
| **Skill 框架** | 内置 skill 加载、MCP skill、plugin skill 基础设施均工作 |

#### ⚠️ 部分工作 / 降级运行

| 子系统 | 问题 |
|--------|------|
| **Computer Use** | 底层包（input/swift）有 macOS 实现，但上层 MCP server 是 stub，且被 `feature('CHICAGO_MCP')` 禁用。整条链路不通 |
| **Claude in Chrome** | `@ant/claude-for-chrome-mcp` 是空 stub，`BROWSER_TOOLS = []`，浏览器集成不工作 |
| **Voice Mode** | hook 和 service 代码存在，但被 `feature('VOICE_MODE')` 禁用，且依赖 Anthropic 的 STT 端点 |
| **深度链接** | `url-handler-napi` 是空 stub，`waitForUrlEvent()` 返回 null |
| **剪贴板图片** | 仅 macOS 可用（osascript 方案），且被 `feature('NATIVE_CLIPBOARD_IMAGE')` 部分限制 |

#### ❌ 完全不工作 / 被禁用

| 子系统 | 原因 |
|--------|------|
| **Bridge / Remote Control** | 被 `feature('BRIDGE_MODE')` 禁用，且强依赖 Anthropic WebSocket 端点 (`wss://bridge.claudeusercontent.com`) |
| **Assistant Mode (KAIROS)** | 被 `feature('KAIROS')` 禁用，整套 assistant/channel/brief 模式不可用 |
| **Proactive Mode** | 被 `feature('PROACTIVE')` 禁用 |
| **Auto Permission Mode** | 被 `feature('TRANSCRIPT_CLASSIFIER')` 禁用，但 prompt 模板已补全（`yolo-classifier-prompts/`），可通过 `FEATURE_TRANSCRIPT_CLASSIFIER=1` 启用 |
| **Bash Classifier** | 被 `feature('BASH_CLASSIFIER')` 禁用，可通过环境变量启用 |
| **Team Memory Sync** | 被 `feature('TEAMMEM')` 禁用 |
| **Workflow Scripts** | 被 `feature('WORKFLOW_SCRIPTS')` 禁用 |
| **Background Sessions** | 被 `feature('BG_SESSIONS')` 禁用 |
| **Settings Sync** | 被 `feature('DOWNLOAD/UPLOAD_USER_SETTINGS')` 禁用 |
| **Buddy** | 被 `feature('BUDDY')` 禁用，但命令已实现（`src/commands/buddy/buddy.ts`），可通过 `FEATURE_BUDDY=1` 启用 |
| **Torch / Ultraplan / Fork** | 被各自 feature flag 禁用，内部高级功能 |

---

### 三、Feature Flag 全景 — 现在如何启用？

> **重大变化**：上游已将 feature flag 体系从「polyfill 全部返回 false」重构为「Bun 原生 `feature()` + 环境变量注入」。
> 现在任何 flag 都可以通过 `FEATURE_<FLAG_NAME>=1 bun run dev` 按需启用，无需改代码。

共发现 **70+ 个 flag**。按对公开发行版的价值分为三档：

#### 🔴 高价值 — 建议优先解锁

这些是已经对外发布或对用户体验有显著影响的功能：

| Flag | 控制什么 | 解锁难度 | 说明 |
|------|---------|---------|------|
| `CONTEXT_COLLAPSE` | 长对话上下文压缩/折叠 | 中 | 影响长会话稳定性，涉及 query.ts、setup.ts、analyzeContext.ts |
| `REACTIVE_COMPACT` | 响应式上下文压缩 | 中 | 与 CONTEXT_COLLAPSE 配合，自动在上下文过大时触发压缩 |
| `HISTORY_SNIP` | 历史消息裁剪 | 中 | 长会话内存管理，涉及 query.ts、QueryEngine.ts、messages.ts、commands.ts、print.ts |
| `TOKEN_BUDGET` | Token 预算追踪 | 低 | 显示 token 使用量，涉及 query.ts、prompts.ts、Spinner.tsx、PromptInput.tsx |
| `STREAMLINED_OUTPUT` | 精简输出模式 | 低 | 改善输出渲染，仅涉及 cli/print.ts |
| `NATIVE_CLIPBOARD_IMAGE` | 原生剪贴板图片 | 低 | 改善图片粘贴体验，仅涉及 imagePaste.ts |
| `MCP_SKILLS` | MCP 提供的 Skills | 低 | 让 MCP server 可以注册 skill，仅涉及 commands.ts |
| `EXPERIMENTAL_SKILL_SEARCH` | 本地 Skill 搜索 | 中 | 涉及 query.ts、commands.ts、prompts.ts、messages.ts |
| `COMMIT_ATTRIBUTION` | Git 提交归因 | 低 | 标记 Claude 参与的 commit，涉及 setup.ts、bashProvider.ts、worktree.ts |

#### 🟡 中等价值 — 可选解锁

| Flag | 控制什么 | 说明 |
|------|---------|------|
| `TRANSCRIPT_CLASSIFIER` | Auto 权限模式 + Bash 命令分类 | 有价值但依赖 Anthropic 后端分类器服务，可能无法独立运行 |
| `BASH_CLASSIFIER` | Bash 命令安全分类 | 同上，可能依赖远程服务 |
| `VOICE_MODE` | 语音输入模式 | 依赖 Anthropic STT 端点，需要替换后端 |
| `DOWNLOAD_USER_SETTINGS` / `UPLOAD_USER_SETTINGS` | 设置云同步 | 依赖 Anthropic 账户系统 |
| `BG_SESSIONS` | 后台会话 | 涉及 query.ts、main.tsx、exit 命令、会话恢复 |
| `QUICK_SEARCH` | 快速搜索快捷键 | 仅 UI 功能，涉及 keybindings 和 PromptInput |
| `HISTORY_PICKER` | 历史会话选择器 | 仅 UI 功能 |
| `AUTO_THEME` | 自动主题切换 | 仅涉及 ThemeProvider.tsx |
| `MESSAGE_ACTIONS` | 消息操作快捷键 | 仅 UI 功能 |
| `TERMINAL_PANEL` | 终端面板切换 | 仅 UI 功能 |

#### 🟢 低价值 / 内部专用 — 不建议解锁

| Flag 类别 | 包含的 Flags |
|-----------|-------------|
| KAIROS 系列（内部 Assistant 模式） | `KAIROS`, `KAIROS_BRIEF`, `KAIROS_CHANNELS`, `KAIROS_DREAM`, `KAIROS_GITHUB_WEBHOOKS`, `KAIROS_PUSH_NOTIFICATION` |
| Bridge / Remote 系列 | `BRIDGE_MODE`, `DAEMON`, `CCR_AUTO_CONNECT`, `CCR_MIRROR`, `CCR_REMOTE_SETUP`, `DIRECT_CONNECT`, `SSH_REMOTE`, `UDS_INBOX` |
| 内部工具 | `BUDDY`, `TORCH`, `ULTRAPLAN`, `FORK_SUBAGENT`, `LODESTONE`, `MONITOR_TOOL`, `RUN_SKILL_GENERATOR`, `REVIEW_ARTIFACT` |
| 遥测/实验 | `ANTI_DISTILLATION_CC`, `COWORKER_TYPE_TELEMETRY`, `MEMORY_SHAPE_TELEMETRY`, `SHOT_STATS`, `SLOW_OPERATION_LOGGING`, `NATIVE_CLIENT_ATTESTATION` |
| 构建/平台 | `IS_LIBC_MUSL`, `IS_LIBC_GLIBC`, `HARD_FAIL`, `BREAK_CACHE_COMMAND` |
| 其他内部 | `TEAMMEM`, `WORKFLOW_SCRIPTS`, `TEMPLATES`, `COORDINATOR_MODE`, `PROACTIVE`, `AWAY_SUMMARY`, `BUILDING_CLAUDE_APPS`, `AGENT_TRIGGERS_REMOTE`, `AGENT_MEMORY_SNAPSHOT`, `VERIFICATION_AGENT`, `WEB_BROWSER_TOOL`, `CHICAGO_MCP`, `CONNECTOR_TEXT`, `CACHED_MICROCOMPACT`, `FILE_PERSISTENCE`, `TREE_SITTER_BASH_SHADOW` |

---

### 四、Anthropic 硬编码依赖 — 发行前必须处理

代码中仍存在大量 Anthropic 专属的 URL、token、品牌标识。如果要作为独立项目发行，需要逐一审查。

#### 4.1 API 与服务端点

| 类别 | 硬编码内容 | 涉及文件 |
|------|-----------|---------|
| API 基础 URL | `https://api.anthropic.com`、`api-staging.anthropic.com` | `src/services/api/client.ts`、`src/services/analytics/growthbook.ts` |
| Bridge WebSocket | `wss://bridge.claudeusercontent.com`、`wss://bridge-staging.claudeusercontent.com` | `src/bridge/remoteBridgeCore.ts`、`src/remote/SessionsWebSocket.ts` |
| Chrome 集成 | `https://claude.ai/chrome`、`https://clau.de/chrome/reconnect` | `src/utils/claudeInChrome/setup.ts`、`mcpServer.ts` |
| OAuth 端点 | Anthropic OAuth 授权/token 端点 | `src/services/oauth/client.ts` |

#### 4.2 遥测与监控

| 类别 | 硬编码内容 | 当前状态 |
|------|-----------|---------|
| Datadog | 客户端 token `pubbbf48e6d78dae54bceaa4acf463299bf` | ✅ **已关闭**（`78144b4`） |
| GrowthBook | 连接 Anthropic 托管的 GrowthBook 数据源 | ✅ **已脱钩**（`e74c009`）— 支持 `CLAUDE_GB_ADAPTER_URL/KEY` 连接自定义实例，无配置时返回默认值 |
| Sentry | 错误上报 | ✅ **已更新**（`1195185`）— 添加 `src/utils/sentry.ts`，可配置化 |
| 自动更新 | 连接 Anthropic 检查更新 | ✅ **已关闭**（`e32c159`） |
| OTEL 遥测 | 服务名 `com.anthropic.claude_code.events` | ⚠️ 仍指向 Anthropic，待处理 |

#### 4.3 品牌与标识

| 类别 | 硬编码内容 | 涉及文件 |
|------|-----------|---------|
| 浏览器扩展 Host ID | `com.anthropic.claude_code_browser_extension` | `src/utils/claudeInChrome/setup.ts` |
| macOS Bundle ID | `com.anthropic.claude-code-url-handler` | 深度链接相关 |
| GitHub Issues | `https://github.com/anthropics/claude-code/issues/...` | 多处错误提示 |
| API 版本头 | `anthropic-version: 2023-06-01`、多个 `anthropic-beta` 头 | `src/services/api/claude.ts`、`src/constants/betas.ts` |

#### 4.4 `USER_TYPE` 员工门控

代码中大量 `process.env.USER_TYPE === 'ant'` 检查，控制：
- 内部专属命令（tag、agents-platform、undercover 等）
- 调试工具（VCR、prompt dump、fault injection、mock rate limits）
- GrowthBook 覆盖机制（`CLAUDE_INTERNAL_FC_OVERRIDES`、`/config Gates`）
- 额外的 shell 环境安全白名单
- 内部 skill（debug、verify、stuck 等的部分行为）

这些对公开发行版无害（`USER_TYPE` 不会是 `ant`），但增加了代码复杂度。

---

### 五、GrowthBook 运行时门控

除了构建时 `feature()` flag，还有一套运行时 GrowthBook 远程配置系统（`tengu_*` 命名）：

| Gate 名称 | 控制什么 |
|-----------|---------|
| `tengu_streaming_tool_execution2` | 流式工具执行（边执行边输出） |
| `tengu_ccr_bridge` | Remote Control 资格检查 |
| `tengu_bridge_repl_v2` | Bridge REPL v2 路径 |
| `tengu_cobalt_harbor` | CCR 自动连接默认值 |
| `tengu_ccr_mirror` | CCR 镜像模式 |
| `tengu_ccr_bridge_multi_session` | Bridge 多会话 |
| `tengu_anti_distill_fake_tool_injection` | 反蒸馏假工具注入 |

这些 gate 通过 GrowthBook SDK 从 Anthropic 服务器拉取配置。

> **重大变化**：上游 `e74c009` 已添加自定义 GrowthBook 适配器，通过 `CLAUDE_GB_ADAPTER_URL/KEY` 环境变量可连接自己的 GrowthBook 实例。无配置时所有 gate 返回代码默认值，不再连接 Anthropic 服务器。
>
> 反蒸馏代码（`tengu_anti_distill_fake_tool_injection`）已在 `c252294` 中被移除。

---

### 六、工具（Tools）完整性盘点

当前 `src/tools/` 下共 **54 个工具目录**，按可用性分类：

#### ✅ 核心工具 — 已验证可用

| 工具 | 功能 |
|------|------|
| `BashTool` | Shell 命令执行 |
| `FileEditTool` | 文件编辑（精确字符串替换） |
| `FileReadTool` | 文件读取（支持图片、PDF、Notebook） |
| `FileWriteTool` | 文件写入 |
| `GlobTool` | 文件模式匹配搜索 |
| `GrepTool` | 文件内容搜索（基于 ripgrep） |
| `AgentTool` | 子 Agent 启动 |
| `WebFetchTool` | URL 内容抓取 |
| `WebSearchTool` | 网页搜索 |
| `AskUserQuestionTool` | 向用户提问 |
| `EnterPlanModeTool` / `ExitPlanModeTool` | 计划模式 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree 隔离 |
| `NotebookEditTool` | Jupyter Notebook 编辑 |
| `LSPTool` | 语言服务器协议交互 |
| `MCPTool` | MCP 工具调用 |
| `SkillTool` | Skill 调用 |
| `ConfigTool` | 配置管理 |
| `ScheduleCronTool` | Cron 定时任务（已解锁） |
| `TaskCreate/Get/List/Update/Output/StopTool` | 任务管理系列 |
| `SendMessageTool` | Agent 间消息发送 |
| `SleepTool` | 等待 |
| `TodoWriteTool` | TODO 列表 |

#### ⚠️ 可能工作但未充分验证

| 工具 | 不确定因素 |
|------|-----------|
| `PowerShellTool` | Windows 专用，macOS 上未测试 |
| `REPLTool` | 代码执行环境，依赖运行时配置 |
| `ListMcpResourcesTool` / `ReadMcpResourceTool` | 依赖 MCP server 是否正确暴露 resources |
| `McpAuthTool` | MCP OAuth 认证，简化过的实现 |
| `BriefTool` | 可能依赖被禁用的 KAIROS_BRIEF |
| `DiscoverSkillsTool` | 依赖 `EXPERIMENTAL_SKILL_SEARCH` flag |

#### ❌ 确定不工作或内部专用

| 工具 | 原因 |
|------|------|
| `WebBrowserTool` | 被 `feature('WEB_BROWSER_TOOL')` 禁用 |
| `MonitorTool` | 被 `feature('MONITOR_TOOL')` 禁用 |
| `SnipTool` | 被 `feature('HISTORY_SNIP')` 禁用 |
| `ReviewArtifactTool` | 被 `feature('REVIEW_ARTIFACT')` 禁用 |
| `WorkflowTool` | 被 `feature('WORKFLOW_SCRIPTS')` 禁用 |
| `ToolSearchTool` | 被 `feature('EXPERIMENTAL_SKILL_SEARCH')` 禁用 |
| `RemoteTriggerTool` | 依赖 Bridge/Remote 系统 |
| `SuggestBackgroundPRTool` | 依赖 KAIROS 系统 |
| `TeamCreateTool` / `TeamDeleteTool` | 依赖 TEAMMEM 系统 |
| `TungstenTool` | 内部专用 |
| `TerminalCaptureTool` | 内部专用 |
| `VerifyPlanExecutionTool` | 被 `feature('VERIFICATION_AGENT')` 禁用 |
| `OverflowTestTool` | 测试用工具 |
| `SendUserFileTool` | 依赖 Bridge 系统 |
| `SyntheticOutputTool` | 内部专用 |

---

### 七、命令（Commands）完整性盘点

`src/commands.ts` 注册了所有 slash 命令，按可用性分类：

#### ✅ 可用命令

`/help`, `/status`, `/login`, `/logout`, `/config`, `/model`, `/permissions`, `/mcp`, `/memory`, `/compact`, `/diff`, `/copy`, `/branch`, `/files`, `/review`, `/doctor`, `/install`, `/theme`, `/vim`, `/output-style`, `/rate-limit-options`, `/upgrade`, `/version`, `/usage`, `/tasks`, `/skills`, `/plugin`, `/loop`

#### ❌ 被 Feature Flag 禁用的命令

| 命令 | 依赖的 Flag |
|------|------------|
| `/proactive` | `PROACTIVE` |
| `/brief` | `KAIROS_BRIEF` |
| `/assistant` | `KAIROS` |
| `/bridge` | `BRIDGE_MODE` |
| `/voice` | `VOICE_MODE` |
| `/force-snip` | `HISTORY_SNIP` |
| `/workflows` | `WORKFLOW_SCRIPTS` |
| `/remote-setup` | `CCR_REMOTE_SETUP` |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` |
| `/ultraplan` | `ULTRAPLAN` |
| `/torch` | `TORCH` |
| `/peers` | `UDS_INBOX` |
| `/fork` | `FORK_SUBAGENT` |
| `/buddy` | `BUDDY` |
| `/daemon` | `DAEMON` + `BRIDGE_MODE` |

---

### 八、待完成工作清单 — 通往「可发行版本」的路线图

基于以上分析，按优先级排列待完成的工作。标注 ✅ 的为上游已完成项。

#### P0 — 必须完成（核心可用性）

| # | 工作项 | 说明 | 状态 |
|---|--------|------|------|
| 1 | **Feature Flag 按需启用机制** | 让开发者可以按需启用任意 flag，而不是全部硬编码 false | ✅ 已完成（`be82b71`）— 通过 `FEATURE_*` 环境变量 + Bun 原生 `--feature` 实现 |
| 2 | **关闭 Datadog 遥测** | 不再向 Anthropic 的 Datadog 端点上报数据 | ✅ 已完成（`78144b4`） |
| 3 | **GrowthBook 脱钩** | 不再依赖 Anthropic 托管的 GrowthBook 服务器 | ✅ 已完成（`e74c009`）— 支持自定义 GrowthBook 实例，无配置时返回默认值 |
| 4 | **关闭自动更新** | 不再连接 Anthropic 检查更新 | ✅ 已完成（`e32c159`） |
| 5 | **Sentry 可配置化** | 错误上报可控 | ✅ 已完成（`1195185`）— 添加 `src/utils/sentry.ts` |
| 6 | **移除反蒸馏代码** | 删除假工具注入等反蒸馏逻辑 | ✅ 已完成（`c252294`） |
| 7 | **Auto Mode prompt 模板** | 补全 yolo-classifier-prompts/ 缺失文件 | ✅ 已完成（`be82b71`） |
| 8 | **修复 USER_TYPE=ant TUI 崩溃** | 全局函数未定义导致 ReferenceError | ✅ 已完成（`4ab4506`） |
| 9 | **修复 getAntModels 未定义** | 运行时 ReferenceError | ✅ 已完成（`e944633`） |
| 10 | **Web Search 工具修正** | 添加 Bing adapter，支持适配器模式 | ✅ 已完成（`e48da39`） |
| 11 | **删除 src/src/ 残留** | 反编译产物中的重复目录 | ✅ 已完成（`991ccc6`） |
| 12 | **删除废弃脚本** | create-type-stubs、fix-default-stubs 等不再需要的脚本 | ✅ 已完成（`88b45e0`） |
| 13 | **端到端功能验证** | 系统性测试核心工作流：启动 → 登录 → 对话 → 工具调用 → 长会话压缩 → 退出 | ⬜ 待完成 |
| 14 | **跨平台基础验证** | 在 Linux 上验证 NAPI 替代实现（modifiers-napi FFI、image-processor osascript、audio-capture SoX） | ⬜ 待完成 |

#### P1 — 应该完成（发行质量）

| # | 工作项 | 说明 | 状态 |
|---|--------|------|------|
| 15 | **清理 Anthropic 品牌硬编码** | 替换或参数化 API URL、Bundle ID、GitHub Issues 链接、扩展 Host ID 等 | ⬜ 待完成 |
| 16 | **OTEL 遥测脱钩** | 1P 遥测仍指向 `com.anthropic.claude_code.events`，需要改为可选或移除 | ⬜ 待完成 |
| 17 | **移除死代码** | 利用 knip 清理永久禁用的代码路径，减少体积和维护负担 | ⬜ 待完成 |
| 18 | **构建产物优化** | 当前 code-splitting 产出 ~450 chunk files，可以进一步 tree-shaking | ⬜ 待完成 |
| 19 | **npm 发布配置** | 添加 `bin` 字段、发布脚本、版本管理，让用户可以 `bun install -g` 或 `npx` 使用 | ⬜ 待完成 |
| 20 | **url-handler-napi 实现** | 当前是空 stub，深度链接不工作。可用 Bun HTTP server 监听本地端口替代 | ⬜ 待完成 |
| 21 | **逐个验证高价值 Feature Flag** | 通过 `FEATURE_*=1` 启用后实际测试：CONTEXT_COLLAPSE、HISTORY_SNIP、TOKEN_BUDGET 等是否真正可用，是否有运行时依赖缺失 | ⬜ 待完成 |

#### P2 — 锦上添花（增强功能）

| # | 工作项 | 说明 | 状态 |
|---|--------|------|------|
| 22 | **Computer Use 完整链路** | 底层包已有 macOS 实现，但上层 MCP server 是 stub。需要实现 `buildComputerUseTools()` 和 `createComputerUseMcpServer()` | ⬜ 待完成 |
| 23 | **Voice Mode 替代方案** | 原版依赖 Anthropic STT 端点，可集成 Whisper API 或开源 STT | ⬜ 待完成 |
| 24 | **Linux/Windows NAPI 包适配** | modifiers-napi 需要 X11/Wayland 方案，image-processor 需要 xclip/xsel，computer-use 需要 xdotool | ⬜ 待完成 |
| 25 | **React Compiler 反编译清理** | 代码中大量 `const $ = _c(N)` 样板代码，影响可读性，可写 codemod 批量清理 | ⬜ 待完成 |

---

### 九、关键数字总结

| 指标 | 当前值 |
|------|--------|
| 总提交数 | 125 条 |
| 开发周期 | ~65 小时（3/31 19:00 ~ 4/3 12:00） |
| 贡献者 | 多位社区贡献者（含 PR 合并） |
| tsc 错误 | 1341 → 0 |
| any 标注 | 176 → 0 |
| 测试文件 | 0 → 114 个（含单元测试 + 集成测试） |
| 构建产物 | code-splitting，dist/cli.js + ~450 chunks |
| 内部包 | 9 个（3 完整实现 / 3 部分实现 / 3 空 stub） |
| Feature Flags | 70+ 个，全部可通过 `FEATURE_*` 环境变量按需启用 |
| 工具 | 54 个目录（~22 核心可用 / ~6 待验证 / ~15 可通过 flag 启用 / ~11 内部专用） |
| 命令 | ~30 可用 / ~15 可通过 flag 启用 |
| 遥测脱钩 | Datadog ✅ / GrowthBook ✅ / Sentry ✅ / 自动更新 ✅ / OTEL ⬜ |

---

### 十、完成度评估

```
反编译 & 可启动        ████████████████████ 100%
类型系统修复           ████████████████████ 100%
NAPI 原生包替代        ██████████████░░░░░░  70%  (3/9 完整, 3/9 部分, 3/9 stub)
工程化基础设施         ████████████████████ 100%
构建系统              ████████████████████ 100%
核心功能可用           ████████████████░░░░  80%  (对话/工具/MCP 可用, 长会话管理待验证)
Feature Flag 体系     ████████████████████ 100%  (环境变量按需启用，不再需要改代码)
遥测系统脱钩          ████████████████░░░░  80%  (Datadog/GrowthBook/Sentry/自动更新已脱钩, OTEL 待处理)
品牌硬编码清理         ████░░░░░░░░░░░░░░░░  20%  (反蒸馏已移除, API URL/品牌标识待处理)
跨平台支持            ████████░░░░░░░░░░░░  40%  (macOS 可用, Linux/Windows 未验证)
测试覆盖              ████████████████░░░░  80%  (114 文件, 含集成测试, 缺端到端)
发行准备              ████░░░░░░░░░░░░░░░░  20%  (无 npm 配置, 无版本管理)
文档                  ██████████████████░░  90%  (架构/功能/feature/学习文档充分)
```

**总体评估**：项目已经完成了最困难的部分（反编译落地、类型修复、核心功能打通），且上游社区在阶段十中完成了关键的遥测脱钩和 Feature Flag 体系重构。距离「可发行版本」主要差在：端到端功能验证、跨平台测试、品牌硬编码清理、npm 发布配置这几块工作。相比之前的评估，进度已大幅推进。
