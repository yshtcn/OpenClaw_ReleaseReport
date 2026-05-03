# OpenClaw v2026.5.2 更新分析报告

- **版本**: 2026.5.2
- **发布日期**: 2026-05-02
- **Commit**: 8842a5bd43a6874c86645d00dab80611a94d5850
- **发布者**: steipete
- **总更新条数**: 379 条（含 5 条 Highlights 总结 + 37 条 Changes + 337 条 Fixes）

## 目录

### Highlights 总结
- #000–#004

### Changes（新功能/改进）
- #005–#041

### Fixes（Bug 修复）
- #042–#378

---

## Highlights 总结（#000–#004）

本次 2026.5.2 版本更新要点如下：

1. **插件体系大升级**：外部插件安装、更新、doctor 修复、依赖报告、artifact 元数据全面覆盖 npm-first cutover，支持过期配置安装修复、缺失包载荷检测和 beta 通道插件回退。
2. **Gateway/Agent 性能优化**：启动、会话列表、任务维护、prompt 准备、插件加载、工具描述规划、文件系统路径检查和大型运行时配置的热路径全部精简。
3. **Control UI / WebChat 健壮性**：Sessions、Cron、长时间 Gateway WebSocket、分组消息宽度、斜杠命令反馈、iOS PWA 边界、选择对比度和 Talk 诊断均有改进。
4. **消息通道修复**：WhatsApp Channel/Newsletter 目标、Telegram topic 命令和网络、Discord 投递/启动边界、Slack 线程、Signal 群组/媒体和可见回复路由。
5. **Provider/媒体修复**：OpenAI 兼容 TTS/Realtime、OpenRouter/DeepSeek 回放、Anthropic 兼容流式、LM Studio reasoning 元数据、Brave/SearXNG/Firecrawl 网页搜索、媒体路径、音乐和语音通话路由。

---

## Changes（新功能/改进）

### #005 Gateway 启动跳过 auth overlay 以降低延迟

**更新原文**：Gateway/startup and restart: skip plugin-backed auth-profile overlays during startup secrets preflight, reducing gateway readiness latency while keeping reload and OAuth recovery paths overlay-capable; add `openclaw gateway restart --force` and `--wait <duration>`, log active task run IDs before restart deferral timers, and report timeout restarts as explicit forced restarts. ([#68327](https://github.com/openclaw/openclaw/pull/68327)) Thanks [@JIRBOY](https://github.com/JIRBOY).

**痛点**：Gateway 启动时，secrets preflight 阶段需要加载所有 auth-profile overlay（包括外部 CLI 和插件支持的认证配置），这在配置了较多 auth profile 的部署中会显著拖慢启动速度。社区报告 Windows 上的启动时间可达 137 秒，其中 auth overlay 加载是重要瓶颈之一。此外，缺乏 `--force` 和 `--wait` 等重启控制选项，在有活跃任务运行时难以安全执行计划内重启。

**如何解决**：PR #68327 在 `src/gateway/server-startup-config.ts` 中引入条件判断：当 `activationParams.reason` 为 `"startup"` 或 `"restart-check"` 时，注入 `loadAuthProfileStoreWithoutExternalProfiles`（仅从持久化存储加载），跳过外部 CLI/plugin-backed 的 overlay 加载。而 reload 和 OAuth recovery 路径仍保持 overlay-capable，不影响运行时行为。核心改动极小（+39/-0 行，3 个文件），但精准命中了启动延迟的一个关键路径。值得注意的是，更新日志原文还提到了 `--force`/`--wait` 参数和超时重启报告，这些在最终 PR diff 中未出现，推测由其他 commit 或 PR 覆盖。此 PR 由 @JIRBOY 发起，经 @steipete 缩减范围后合入，体现了"最小化改动"的代码审查原则。

### #006 ClawHub ClawPack 元数据贯穿插件安装记录

**更新原文**：Plugins/ClawHub: make diagnostics, onboarding, doctor repair, and channel setup carry ClawPack metadata through install records while keeping explicit `clawhub:` installs on ClawHub and bare package installs on npm for the launch cutover. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：ClawHub 插件安装系统原先使用 "StorePack" 命名管理 artifact 元数据（SHA-256、manifest 哈希、尺寸等），但在向 ClawPack 品牌迁移的过程中，diagnostics、onboarding、doctor repair 和 channel setup 等环节尚未携带新的 ClawPack 元数据。此外，升级到新版本后，已配置的插件可能因安装记录指向旧的 artifact 路径而无法正常使用，需要手动干预。

**如何解决**：由三个 PR 构成的功能栈协同完成。PR #75864（@vincentkoc）将全链路的 StorePack 命名统一重命名为 ClawPack，涉及 14 个文件（类型定义、Zod schema、API 类型、安装记录、测试），确保元数据字段名一致。PR #75866（Draft）新增了从 ClawHub 下载 ClawPack artifact 的能力，包括 SHA-256 双重校验（响应头 + 下载字节）。PR #76129（@steipete）实现了 2026.5.2 版本的一次性 doctor repair 流程，通过 `meta.lastTouchedVersion` 检测是否需要修复，优先从 ClawHub 安装活跃使用的可下载插件（npm 作为回退），并新增了 315 行的 `release-configured-plugin-installs.ts` 模块。整体策略保持了"显式 `clawhub:` 安装走 ClawHub，裸包名走 npm"的双轨模式，为后续 ClawHub 全面上线预留了切换空间。

### #007 插件列表输出依赖安装状态

**更新原文**：Plugins/CLI: include package dependency install state in `openclaw plugins list --json` so scripts can spot missing plugin dependencies without runtime-loading plugins.

**痛点**：运维脚本和自动化工具在使用 `openclaw plugins list --json` 检查插件状态时，无法得知插件的依赖包是否已安装。要确认依赖状态，必须实际运行时加载每个插件，这不仅慢而且可能触发副作用（如插件初始化时的网络请求）。这给诊断"插件装了但依赖缺失导致运行时失败"的问题增加了难度。

**如何解决**：Commit `4a3ad396`（@steipete）在 `plugins list --json` 输出中新增 `dependencyStatus` 字段。实现方式是读取插件 `package.json` 的 `dependencies`/`optionalDependencies`，沿 `node_modules` 查找路径检查每个包是否存在。整个过程纯文件系统操作，无需加载插件运行时，对大型插件集合也不会造成性能问题。这为 `openclaw doctor` 等诊断工具提供了快速检测缺失依赖的能力。

### #008 Beta 通道插件更新 @beta 优先回退

**更新原文**：Plugins/update: on the beta OpenClaw update channel, default-line npm and ClawHub plugin updates try `@beta` first and fall back to default/latest when no plugin beta release exists.

**痛点**：在 beta 更新通道上，当 OpenClaw 核心切换到 beta 版本时，插件更新应优先尝试对应的 `@beta` dist-tag。但之前的行为是直接使用 `latest`，导致 beta 用户可能拿到不兼容的稳定版插件，或者找不到对应的 beta 版本时报错中断。

**如何解决**：在插件更新逻辑中增加 beta 通道的 dist-tag 回退策略：当 `openclaw update` 检测到处于 beta 通道时，对 npm 和 ClawHub 的插件更新先尝试 `@beta` tag，如果该 tag 不存在（即插件没有发布 beta 版本），则回退到 `default`/`latest`。这确保了 beta 用户不会因某个插件缺少 beta 版本而更新失败，同时优先使用与 beta 核心匹配的插件版本。经搜索未找到直接关联的 PR，推测为 @steipete 直接 commit 合入。

### #009 运行时插件预加载限定至有效插件集合

**更新原文**：Plugins/runtime: scope broad runtime preloads to the effective plugin ids derived from config, startup planning, configured channels, slots, and auto-enable rules instead of importing every discoverable plugin.

**痛点**：Gateway 启动时的插件预加载阶段会导入所有可发现的插件（`all` scope），而非仅导入实际需要的插件。在安装了大量插件但只启用少部分的场景下，这导致不必要的 I/O 和模块加载开销。Issue #76187 报告 Gateway 在 `plugins.bootstrap` 阶段永久挂起，根因之一就是过宽的预加载范围触发了有问题的插件初始化代码。

**如何解决**：Commit `da2a8bd`（@steipete）将 runtime preload 的 `all` scope 替换为 `resolveEffectivePluginIds` 的返回值。该函数综合考虑 config 中的插件配置、startup planning 结果、已配置的 channels、slots 以及 auto-enable 规则，推导出当前实际需要的插件 ID 集合。只有这些插件才会在启动时被预加载，其余插件延迟到首次使用时再加载。这是一个精准的性能优化，减少了启动时间和内存占用，同时避免了无关插件初始化代码的副作用。

### #010 复用启动时加载的插件注册表

**更新原文**：Agents/runtime: reuse the startup-loaded plugin registry for request-time providers, tools, channel actions, web/capability/memory/migration helpers, and memoized provider extra-params, and memoize transcript replay-policy resolution for stable config and process-env runs while preserving model-specific transport hook patches and custom-env provider behavior. Thanks [@DmitryPogodaev](https://github.com/DmitryPogodaev).

**痛点**：Issue #48380 报告了 Gateway 启动回归：bundled provider 插件导致 O(N×M) 次完整重载。每次 agent 请求到达时，系统会重新构建整个 plugin registry（包括加载所有插件运行时），即使启动时已经构建过一次。对于配置了大量插件的部署，这意味着每次请求都有数秒的额外延迟，严重影响响应速度。

**如何解决**：PR #76004（@DmitryPogodaev + @steipete）引入 WeakMap 缓存 `resolvePreparedExtraParams`，将启动时构建的 plugin registry 复用到 provider、tool、channel action、web/capability/memory/migration helper 等所有运行时路径。同时缓存 transcript replay-policy 的解析结果，使 stable config 和 process-env 运行避免重复计算。但 model-specific 的 transport hook patches 和 custom-env 的 provider 行为仍保持独立，避免跨请求污染。实测将 `buildAgentRuntimePlan` 从约 2.8 秒降至约 0.9 秒，效果显著。

### #011 文件系统路径包含检查快速路径

**更新原文**：Infra/path-guards: add a fast path for canonical absolute POSIX containment checks, avoiding repeated `path.resolve` and `path.relative` work in hot filesystem walkers. Refs [#75895](https://github.com/openclaw/openclaw/issues/75895), [#75575](https://github.com/openclaw/openclaw/issues/75575), and [#68782](https://github.com/openclaw/openclaw/issues/68782). Thanks [@Enderfga](https://github.com/Enderfga).

**痛点**：`isPathInside()` 函数是文件系统操作（sandbox 文件读写、插件路径验证等）的热路径，每次调用都执行 `path.resolve` + `path.relative` 来判断路径包含关系。在大量文件遍历（如 workspace 扫描、插件加载）的场景下，这些重复的路径规范化操作成为 CPU 瓶颈。关联的三个 issue 反映了路径操作导致的挂起和性能问题。

**如何解决**：Commit `f7b24a6`（@steipete）在 `src/infra/path-guards.ts` 的 `isPathInside()` 中为 POSIX 绝对路径添加快速路径：当父子路径都是规范化的绝对 POSIX 路径时，直接通过字符串前缀比较判断包含关系，跳过 `path.resolve`/`path.relative` 调用。核心代码仅 +13 行，新增 +9 行测试。这是一个经典的微优化——在热路径上减少不必要的系统调用，对大规模文件操作有可观的累积效果。

### #012 缓存插件工具描述符以消除重复加载

**更新原文**：Tools/plugins: add a platform-level tool descriptor planner for descriptor-first visibility, generic availability checks, and executor references, and cache plugin tool descriptors captured from `api.registerTool(...)` so repeated prompt-time planning can skip plugin runtime loading while execution still loads the live plugin tool. ([#76079](https://github.com/openclaw/openclaw/pull/76079)) Thanks [@shakkernerd](https://github.com/shakkernerd).

**痛点**：Issue #75956 报告了严重的 Gateway 性能问题：Windows 上每条消息需要约 78 秒处理时间，根因是每次 prompt-time planning 都会重新加载所有插件的工具工厂（tool factory），触发完整的插件运行时初始化。即使工具列表没有变化，也会重复执行昂贵的模块加载和注册操作。

**如何解决**：PR #76079（@shakkernerd + @steipete）引入平台级的 tool descriptor planner 和缓存机制。核心思路是"描述符先行"：在 `api.registerTool(...)` 调用时捕获工具的静态描述符（名称、schema、可用性），缓存后供 prompt-time planning 直接使用，跳过插件运行时加载。实际执行工具时仍加载 live plugin tool，确保使用最新代码。这是一个经典的"读时加载 vs 用时加载"分离——规划阶段用缓存的静态信息，执行阶段才做完整加载。

### #013 文档澄清 Codex 订阅运行时路径

**更新原文**：Docs/Codex: clarify that ChatGPT/Codex subscription setups should use `openai/gpt-*` with `agentRuntime.id: "codex"` for native Codex runtime, while `openai-codex/*` remains the PI OAuth route. Thanks [@pashpashpash](https://github.com/pashpashpash).

**痛点**：OpenClaw 支持两种 Codex 运行时路径：`openai/gpt-*` + `agentRuntime.id: "codex"` 走原生 Codex runtime，`openai-codex/*` 走 PI OAuth 代理。但文档中两种路径的推荐顺序不清晰，导致用户经常配错，尤其是 ChatGPT/Codex 订阅用户可能误用了 PI OAuth 路径而遇到认证问题。

**如何解决**：PR #75910（@pashpashpash）纯文档更新（10 个 md 文件，+143/-101 行），重排 Codex 路由推荐顺序，将 `openai/*` + `agentRuntime.id: "codex"` 提升为 ChatGPT/Codex 订阅用户的首选推荐路径，`openai-codex/*` 明确标注为"仅 PI OAuth"的窄路径。这是一次文档层面的用户体验优化，减少用户的配置困惑。

### #014 源码检出使用 extensions 工作区加载插件

**更新原文**：Plugins/source checkout: load bundled plugins from the `extensions/*` pnpm workspace tree in source checkouts, so plugin-local dependencies and edits are used directly while packaged installs keep using the built runtime tree. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：开发者从 git clone 源码后，插件仍从预构建的 runtime tree 加载，无法使用 `extensions/` 目录下的本地依赖和实时编辑。每次修改插件代码都需要重新构建才能测试，严重降低了开发效率。

**如何解决**：Commit `25011bdb`（@vincentkoc）在 `resolveBundledPluginsDir()` 中新增 `isSourceCheckoutRoot()` 检测，自动识别 git source checkout 环境。检测到源码检出后，优先从 `extensions/*` pnpm workspace 目录加载插件源码，使插件本地依赖和编辑立即可用。而 packaged installs（npm 安装的正式版）仍使用构建后的 runtime tree。两种模式自动切换，开发者无需手动配置环境变量。

### #015 外部化 ACPX 和诊断插件，准备 beta 发布

**更新原文**：Plugins/beta: externalize ACPX behind `@openclaw/acpx` and diagnostics OpenTelemetry behind `@openclaw/diagnostics-otel`, keeping their heavier runtime stacks out of the core package until installed; prepare Google Chat, LINE, Matrix, Mattermost, BlueBubbles, diagnostics Prometheus, Google Meet, Nextcloud Talk, Nostr, Zalo, Zalo Personal, diagnostics OpenTelemetry, Discord, Diffs, Lobster, Memory LanceDB, Microsoft Teams, QQ Bot, Voice Call, WhatsApp, Brave, Codex, Feishu, Synology Chat, Tlon, and Twitch for `2026.5.1-beta.1`/`2026.5.1-beta.2` npm and ClawHub publishing, and keep publishable plugin dist trees out of the core npm package. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：ACPX 和 diagnostics OpenTelemetry 插件的运行时依赖栈较重（涉及 OpenTelemetry SDK、诊断协议等），将它们打包在核心 npm 包中会增加所有用户的安装体积和启动时间，即使大部分用户不需要这些功能。同时，20+ 个插件需要为 beta 通道发布做好准备，但核心包不应包含它们的 dist tree。

**如何解决**：Commit `010f7a5`（@steipete）将 ACPX 外部化为 `@openclaw/acpx`，diagnostics OpenTelemetry 外部化为 `@openclaw/diagnostics-otel`，它们在用户显式安装前不会出现在运行时中。同时为 26 个插件（Google Chat、LINE、Matrix、Discord、WhatsApp、Feishu、Twitch 等）准备了 `2026.5.1-beta.1`/`2026.5.1-beta.2` 的 npm 和 ClawHub 发布元数据，确保 publishable plugin dist tree 不包含在核心 npm 包中。这是 OpenClaw 插件架构向"核心精简 + 按需安装"模式演进的重要一步。

### #016 xAI 默认模型升级至 Grok 4.3

**更新原文**：Providers/xAI: add Grok 4.3 to the bundled catalog and make it the default xAI chat model.

**痛点**：xAI 的 Grok 模型已迭代到 4.3 版本，支持 reasoning（1M context，text+image 输入），但 OpenClaw 的内置模型目录和默认配置仍指向旧版 Grok 模型，用户需要手动配置才能使用最新版本。

**如何解决**：Commit `a273441`（@steipete）将 Grok 4.3 加入内置模型目录，并设为 xAI 的默认聊天模型。涉及 11 个文件（模型定义、文档、测试、live test 配置）。用户无需额外配置即可使用 Grok 4.3，同时旧模型仍可通过显式指定使用。

### #017 Google Meet 空间访问控制、结束会议和听诊探针

**更新原文**：Google Meet: let API-created rooms set `accessType` and `entryPointAccess`, add `googlemeet end-active-conference` for closing managed spaces after a call, and add `googlemeet test-listen` plus the matching `google_meet` `test_listen` action so transcribe-mode joins wait for real caption or transcript movement before reporting listen-first health. ([#74824](https://github.com/openclaw/openclaw/pull/74824); refs [#72478](https://github.com/openclaw/openclaw/issues/72478)) Thanks [@BsnizND](https://github.com/BsnizND) and [@DougButdorf](https://github.com/DougButdorf).

**痛点**：通过 API 创建的 Google Meet 房间无法设置访问策略（`accessType`/`entryPointAccess`），导致所有会议默认为开放访问，不适合需要限制参与者的场景。通话结束后没有编程方式关闭托管空间，导致会议空间无限累积。此外，transcribe 模式加入会议后无法可靠检测是否真正接收到字幕/转录数据，健康检查可能误报就绪。

**如何解决**：PR #74824（@BsnizND + @steipete）新增三大功能：(1) API 创建房间时支持 `accessType`（如 `restricted`）和 `entryPointAccess`（如 `invited`）参数；(2) `googlemeet end-active-conference` 命令用于关闭 API 管理的会议空间；(3) `googlemeet test-listen` + `google_meet test_listen` 探针以 transcribe 模式加入，等待真实的字幕或转录数据变化后才报告 listen-first 健康，避免了无数据时的误判。

### #018 ClawHub 版本化 Artifact 带摘要校验的安装

**更新原文**：Plugins/ClawHub/onboarding: prefer versioned ClawPack artifacts when ClawHub publishes digest metadata, verify ClawPack response headers and downloaded bytes, persist ClawPack digest/artifact metadata on install/update records and install-on-demand provider setup entries, and allow official bundled-plugin cutovers to record ClawHub artifact metadata while preserving npm as the launch default for bare package specs and retaining npm/local fallback paths. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：ClawHub 插件安装之前只支持通用的 package archive 路径，无法利用版本化的 ClawPack artifact（包含预构建的二进制依赖和完整性校验）。安装过程缺少下载内容与声明摘要的比对，存在中间人篡改的潜在风险。升级后已安装插件的 artifact 元数据也未持久化，导致后续更新和诊断无法追溯安装来源。

**如何解决**：基于 PR #75864（ClawPack 重命名）构建的 PR #75866 新增了完整的 ClawPack artifact 安装管线。当 ClawHub 发布 `clawpack.sha256` 摘要元数据时，优先下载版本化的 ClawPack artifact，要求响应头 `X-ClawHub-ClawPack-Sha256` 与下载字节匹配。安装后将 artifact 类型、npm integrity、shasum 和 tarball 元数据持久化到安装记录。同时保留旧版 package archive 回退路径和 npm/local 安装通道，确保迁移期间不影响现有用户。

### #019 Crestodian 插件管理 + ClawHub 搜索

**更新原文**：Plugins/Crestodian: add ClawHub plugin search plus Crestodian plugin list/search/install/uninstall operations, with approval and audit coverage for install and uninstall.

**痛点**：Crestodian（OpenClaw 的 AI 辅助管理工具）之前无法直接搜索 ClawHub 插件市场，也无法执行插件的安装/卸载操作。用户需要在 CLI 和 Crestodian 之间切换来完成插件生命周期管理，操作效率低。

**如何解决**：PR #75869（@steipete）为 Crestodian 新增了完整的插件管理能力：ClawHub 插件搜索（关键词匹配），以及 Crestodian 原生的 list/search/install/uninstall 操作。install 和 uninstall 操作需经过 approval 审批流程，所有操作都有审计日志覆盖。同时新增了 message-channel rescue 的安全边界，防止插件安装被滥用于非预期目的。

### #020 统一线程绑定会话生成配置

**更新原文**：Channels/thread bindings: replace split subagent/ACP thread-spawn toggles with `threadBindings.spawnSessions`, default thread-bound spawns on, and let `openclaw doctor --fix` migrate the legacy keys. ([#75943](https://github.com/openclaw/openclaw/pull/75943))

**痛点**：之前线程绑定会话的生成由两个独立开关控制：`spawnSubagentSessions`（子代理）和 `spawnAcpSessions`（ACP），配置分散且容易混淆。用户经常不确定应该启用哪个，或者忘记同时启用两者。此外，默认行为是关闭的，导致新用户需要手动配置才能使用线程绑定功能。

**如何解决**：PR #75943（@steipete）将两个独立开关合并为统一的 `threadBindings.spawnSessions`，默认开启线程绑定生成。同时引入 `defaultSpawnContext` 配置和自动策略（移除 `parentForkMaxTokens` 手动配置），通过 `openclaw doctor --fix` 自动迁移旧配置键。涉及 68 个文件的大规模重构，是配置简化和用户体验改进的典型案例。

### #021 OpenAI 兼容 TTS 端点 extraBody 透传

**更新原文**：Providers/OpenAI: add `extraBody`/`extra_body` passthrough for OpenAI-compatible TTS endpoints, so custom speech servers can receive fields such as `lang` in `/audio/speech` requests. Fixes [#39900](https://github.com/openclaw/openclaw/issues/39900). Thanks [@R3NK0R](https://github.com/R3NK0R).

**痛点**：Issue #39900 报告，自部署的 OpenAI 兼容 TTS 服务器（如 Kokoro TTS）通常需要额外参数（如 `lang` 指定语言、`speed` 控制语速），但 OpenClaw 的 OpenAI provider 不支持在 `/audio/speech` 请求中透传这些非标准字段，导致自建语音服务的用户无法使用这些自定义参数。

**如何解决**：为 OpenAI 兼容的 TTS provider 添加 `extraBody`（config 配置）和 `extra_body`（env 变量）透传机制，允许自定义语音服务器在标准 OpenAI `/audio/speech` 请求中接收额外字段。实现简洁，仅在 TTS 请求构建时将 extraBody 合并到请求体中，不影响标准的 OpenAI 端点行为。@R3NK0R 提出了 issue，代码可能由维护者直接 commit 合入。

### #022 WhatsApp Channel/Newsletter 出站目标支持

**更新原文**：Channels/WhatsApp: support explicit WhatsApp Channel/Newsletter `@newsletter` outbound message targets with channel session metadata instead of DM routing. Fixes [#13417](https://github.com/openclaw/openclaw/issues/13417); carries forward the narrow outbound target idea from [#13424](https://github.com/openclaw/openclaw/pull/13424). Thanks [@vincentkoc](https://github.com/vincentkoc) and [@agentz-manfred](https://github.com/agentz-manfred).

**痛点**：Issue #13417 反映，WhatsApp 的 Channel/Newsletter（类似公众号/广播频道）是重要的信息分发渠道，但 OpenClaw 之前只支持 DM（一对一私聊）路由。当用户尝试向 Channel/Newsletter 发送消息时，系统会将其当作普通 DM 处理，导致消息无法正确投递到频道。

**如何解决**：PR #73393（@vincentkoc，基于 @agentz-manfred 的原始 PR #13424 重写）为 WhatsApp 出站消息新增 `@newsletter` JID 目标支持。改动包括：目标规范化（将 `@newsletter` 格式转为 WhatsApp JID）、绕过 DM 的 `allowFrom` 检查、通过 channel session 元数据路由而非 DM 路由、抑制 composing presence 通知（避免在频道中显示"正在输入"状态）。原始 PR #13424 因不活跃被关闭，最终由维护者重写合入。

### #023 工作区依赖版本刷新

**更新原文**：Dependencies: refresh workspace, bundled runtime, and plugin dependency pins, including TypeBox 1.1.37, AWS SDK 3.1041.0, Microsoft Teams 2.0.9, Marked 18.0.3, Pi 0.71.1, OpenAI 6.35.0, Codex 0.128.0, Zod 4.4.1, and Matrix 41.4.0. Thanks [@mariozechner](https://github.com/mariozechner), [@aws](https://github.com/aws), and [@microsoft](https://github.com/microsoft).

**痛点**：依赖版本长期未更新可能导致安全漏洞、性能缺失和与新 API 不兼容的问题。TypeBox、AWS SDK、Zod 等核心依赖的更新通常包含重要的 bug 修复和性能改进。

**如何解决**：常规的依赖版本批量刷新，涉及 9 个核心包的版本升级。未找到直接关联的 PR，推测由 @steipete 通过 commit 直接合入。这类更新通常是例行维护工作，确保项目使用最新稳定版本的依赖。

### #024 Discord 可复用消息通道访问组

**更新原文**：Discord/channels: add reusable message-channel access groups plus Discord channel-audience DM authorization, so allowlists can reference `accessGroup:<name>` across channel auth paths. ([#75813](https://github.com/openclaw/openclaw/pull/75813))

**痛点**：Discord 频道的访问控制配置分散，DM 授权需要单独维护每个用户/角色的白名单。当多个频道共享相同的授权策略时（如"所有管理员频道的 DM 都授权给同一组用户"），需要在每个频道重复配置，维护成本高且容易出错。

**如何解决**：PR #75813（@steipete）引入顶层 `accessGroups` 配置概念，支持 `message.senders` 静态组（手动列举用户 ID）和 `discord.channelAudience` 动态组（基于 Discord ViewChannel 权限自动推导）。DM 白名单现在可以引用 `accessGroup:<name>` 而非逐一列举用户 ID。此机制不仅适用于 Discord，还扩展到 Google Chat、WhatsApp、Zalo 等其他渠道，实现了跨渠道的统一访问控制策略。

### #025 Crabbox 脚本打印二进制信息并拒绝过期版本

**更新原文**：Crabbox/scripts: print the selected Crabbox binary, version, and supported providers before `pnpm crabbox:*` commands, and reject stale binaries that lack `blacksmith-testbox` provider support.

**痛点**：Crabbox（OpenClaw 的测试/沙盒执行环境）的二进制文件可能过期，缺少新引入的 `blacksmith-testbox` provider 支持。开发者在运行 `pnpm crabbox:*` 命令时，过期的二进制会静默失败或产生难以诊断的错误，因为无法直观看到当前使用的是哪个版本。

**如何解决**：在 `pnpm crabbox:*` 命令执行前，打印当前选中的 Crabbox 二进制路径、版本号和支持的 provider 列表。同时增加版本检查：如果二进制不支持 `blacksmith-testbox` provider，则明确拒绝执行并提示更新。未找到直接关联的 PR，推测为内部开发工具链改进。

### #026 Codex happy-path prompt 快照

**更新原文**：Agents/Codex: add committed happy-path prompt snapshots for Codex/message-tool Telegram direct, Discord group, and heartbeat turns so prompt drift can be reviewed. Thanks [@pashpashpash](https://github.com/pashpashpash).

**痛点**：Codex 的 prompt 模板（Telegram 直接消息、Discord 群组、心跳轮次等场景）会随着代码迭代而变化，但没有机制跟踪这些变化。当 prompt 意外偏离预期的"happy path"时，可能导致 agent 行为异常，但开发者难以发现这种 drift。

**如何解决**：PR #75807（@pashpashpash）为 Codex/message-tool 运行时的关键场景新增了 committed prompt 快照文件，覆盖 Telegram 直接消息、Discord 群组和心跳轮次。这些快照存放在代码仓库中，可以通过 git diff 检测 prompt 变化。配套的生成脚本和 CI drift 检测确保 prompt 偏离能被及时发现。后续 PR #76229 增加了 Codex 模型 prompt 层快照，#76274 重命名了 fixture 目录。

### #027 跳过可选工作区引导文件

**更新原文**：Agents/workspace: add `agents.defaults.skipOptionalBootstrapFiles` for skipping selected optional workspace files during bootstrap without disabling required workspace setup. ([#62110](https://github.com/openclaw/openclaw/pull/62110)) Thanks [@mainstay22](https://github.com/mainstay22).

**痛点**：OpenClaw 的 workspace bootstrap 会创建一系列引导文件（SOUL.md、USER.md、HEARTBEAT.md、IDENTITY.md 等），其中部分是可选的。但之前只有全局开关：要么全部创建，要么全部禁用。用户如果只想禁用某些可选文件（如不需要 HEARTBEAT.md），就必须完全禁用 bootstrap，同时失去了必需文件（AGENTS.md、TOOLS.md）的自动创建。

**如何解决**：PR #62110（@mainstay22）新增 `agents.defaults.skipOptionalBootstrapFiles` 配置项，允许用户逐个指定要跳过的可选引导文件列表，同时保留必需文件的自动创建。涉及 28 个文件，+251/-40 行。配置粒度从"全有或全无"变为"按需选择"，灵活性显著提升。

### #028 一等公民 git: 插件安装

**更新原文**：Plugins/CLI: add first-class `git:` plugin installs with ref checkout, commit metadata, normal scanner/staging, and `plugins update` support for recorded git sources. Thanks [@badlogic](https://github.com/badlogic).

**痛点**：之前安装 Git 仓库中的插件需要手动 clone、构建和链接，操作繁琐且无法通过 `openclaw plugins update` 进行版本管理。对于开发者自己的插件或尚未发布到 npm 的第三方插件，没有标准化的安装和更新流程。

**如何解决**：Commit `7ddf28c`（@steipete）新增了 `git:` 前缀的插件安装协议。核心模块 `src/plugins/git-install.ts`（298 行）实现了完整的 Git 插件生命周期：spec 解析（支持 `git:<url>` 和 `git:<url>#<ref>`）、浅克隆 + ref checkout、commit 元数据记录、安全扫描集成。`plugins update` 命令会重新解析 git 源并拉取最新 ref。后续 commit `f7fd803` 修复了 git clone 失败时认证 URL 泄露的安全问题。30 个文件，+1188/-74 行，是一项重大的新功能。

### #029 Google Meet Chrome transcribe 字幕健康监控

**更新原文**：Google Meet: add live caption health for Chrome transcribe mode, including caption observer state, transcript counters, last caption text, and recent transcript lines in status and doctor output. Refs [#72478](https://github.com/openclaw/openclaw/issues/72478). Thanks [@DougButdorf](https://github.com/DougButdorf).

**痛点**：Issue #72478 反映，无头 agent 通过 Chrome transcribe 模式加入 Google Meet 后，缺乏可靠的方式检测是否真正接收到字幕数据。字幕可能因权限、DOM 变化或网络问题而停止，但系统无法感知，导致 agent 在没有实际转录数据的情况下继续运行。

**如何解决**：Commit `f221bc85`（@steipete）为 Google Meet Chrome transcribe 模式新增 caption observer，基于 MutationObserver 监听 DOM 字幕区域。在 `status` 和 `doctor` 命令输出中暴露 captioning 状态、transcript 计数器、lastCaptionText 和 recentTranscript 等健康字段。`runtime.status()` 改为 async 以支持实时刷新字幕状态。这为运维人员提供了直观的诊断入口。

### #030 Twilio Meet 加入阶段日志增强

**更新原文**：Voice Call/Google Meet: add Twilio Meet join phase logs around pre-connect DTMF, realtime stream setup, and initial greeting handoff for easier live-call debugging. Thanks [@donkeykong91](https://github.com/donkeykong91) and [@PfanP](https://github.com/PfanP).

**痛点**：Twilio 通过电话桥接加入 Google Meet 时，加入过程涉及多个阶段（phone leg 建立、DTMF PIN 输入、realtime 流设置、初始问候），但在调试日志中这些阶段缺乏明确的阶段标记。当通话建立失败时，运维人员难以定位是哪个阶段出了问题。

**如何解决**：Commit `1634f91`（@steipete）将 Twilio Meet 加入流程从单次 `voicecall.start` 拆为三阶段顺序执行（phone leg → delayed DTMF → delayed intro speech），每阶段带详细日志。同时 voice-call 插件支持 provider call ID 归一化和已结束通话的历史状态报告。感谢的 @donkeykong91 和 @PfanD 可能贡献了需求分析和测试反馈。

### #031 macOS 菜单栏 Context 子菜单

**更新原文**：macOS app: move recent session context rows into a Context submenu while keeping usage and cost details root-level, so the menu bar companion stays compact with many active sessions. Thanks [@Guti](https://github.com/Guti).

**痛点**：macOS 菜单栏伴侣在有多个活跃会话时，根菜单会被大量 session context 行挤占，导致 usage 和 cost 等关键信息被推到菜单底部，需要滚动才能看到。

**如何解决**：PR #75489（@ngutman/Guti）将 session context 行折叠到 "Context" 子菜单中，Usage 和 Cost 信息保持在根菜单。新增 `ContextRootMenuLabelView` 视图组件，重构了 `MenuSessionsInjector` 的注入逻辑。改动简洁但有效解决了菜单空间管理问题。

### #032 Gateway SDK 工具调用 RPC

**更新原文**：Gateway/SDK: add SDK-facing tools.invoke RPC with shared HTTP policy, typed approval/refusal results, and SDK helper support. Refs [#74705](https://github.com/openclaw/openclaw/issues/74705). Thanks [@BunsDev](https://github.com/BunsDev) and [@ai-hpc](https://github.com/ai-hpc).

**痛点**：Issue #74705 反映，SDK 侧调用 Gateway 工具的路径不完善：`oc.tools.invoke()` 返回 `unsupported`，SDK 开发者只能通过 HTTP 端点调用工具，无法享受 Gateway 内部的策略管道（权限检查、审批流程等）。同时 HTTP 和 RPC 两条路径的策略逻辑不一致，导致行为差异。

**如何解决**：PR #74804（@ai-hpc + @BunsDev）为 SDK 新增 `tools.invoke` Gateway RPC 方法。核心改动是将原有 HTTP 专用的工具调用逻辑抽取为共享模块 `tools-invoke-shared.ts`，RPC 和 HTTP 两条路径共用同一套策略管道。SDK 侧 `oc.tools.invoke()` 返回类型化的 approval/refusal 结果（而非抛异常）。新增 `approvalMode` 区分 "report"（仅报告阻塞）和 "request"（发起审批）两种模式。涉及 24 个文件，+932/-251 行。

### #033 Discord 组件注册表持久化

**更新原文**：Discord: keep active buttons, selects, and forms working across Gateway restarts until they expire, so multi-step Discord interactions are less likely to break during upgrades or restarts. Thanks [@amknight](https://github.com/amknight).

**痛点**：Discord 的按钮、下拉选择器和表单等交互组件的回调注册表原本存储在内存中。当 Gateway 因升级或崩溃重启时，所有活跃组件的回调处理器丢失，用户点击按钮或提交表单时不会产生任何响应。对于多步骤的 Discord 交互（如配置向导、审批流程），这意味着用户必须从头开始。

**如何解决**：PR #75584（@amknight）将 Discord 组件/模态注册表从纯内存 Map 升级为内存优先 + SDK-backed SQLite 持久化回退。活跃组件在创建时自动持久化，Gateway 重启后从 SQLite 恢复注册表（直到 TTL 过期）。持久化失败时自动降级回纯内存行为，不影响正常功能。

### #034 文档澄清 BodyForAgent 与 Signal 覆盖

**更新原文**：Messages/docs: clarify that `BodyForAgent` is the primary inbound model text while `Body` is the legacy envelope fallback, and add Signal coverage so channel hardening patches target the real prompt path. Refs [#66198](https://github.com/openclaw/openclaw/pull/66198). Thanks [@defonota3box](https://github.com/defonota3box).

**痛点**：PR #66198（@defonota3box）尝试为 Signal DM 添加安全包裹，但错误地包裹了 `Body`（legacy 兼容字段）而非 `BodyForAgent`（实际进入 agent prompt 的字段）。维护者 @steipete 指出这个问题后关闭了 PR，但这也暴露了文档中 `BodyForAgent` 与 `Body` 的契约关系不够清晰。

**如何解决**：@steipete 通过 commit `32d429e` 直接修复：(1) 澄清文档，明确 `BodyForAgent` 是 inbound model text 的主要字段，`Body` 仅作为 legacy envelope 的向后兼容回退；(2) 为 Signal 通道添加 `BodyForAgent` 的回归测试覆盖。这确保后续的 channel hardening 补丁能正确命中实际的 prompt 路径。

### #035 Slack App Home 标签页 + 线程参与持久化

**更新原文**：Slack: publish a safe default App Home tab view on `app_home_opened`, include the Home tab event in setup manifests, and keep track of bot-participated threads across restarts so ongoing threaded conversations can continue auto-replying after the Gateway restarts. Fixes [#11655](https://github.com/openclaw/openclaw/issues/11655); refs [#52020](https://github.com/openclaw/openclaw/issues/52020). Thanks [@TinyTb](https://github.com/TinyTb) and [@amknight](https://github.com/amknight).

**痛点**：Issue #11655 反映，Gateway 重启后 Slack 线程自动回复中断——bot 之前参与的线程在重启后不再自动回复，因为线程参与记录只存在于内存中。Issue #52020 则涉及 App Home 标签页未处理 `app_home_opened` 事件，用户看到空白页面。

**如何解决**：两个独立 PR 协同解决。PR #73389（@vincentkoc）添加了 `app_home_opened` 事件处理和安全的默认 Home tab 视图发布，将 Home tab 事件纳入 setup manifests。PR #33845（@zeroaltitude）将 bot 参与的线程缓存持久化到磁盘，使 Gateway 重启后能自动恢复线程参与状态，保持自动回复的连续性。

### #036 Usage Mosaic 精确 token 桶

**更新原文**：Control UI/Usage: add UTC quarter-hour token buckets for the Usage Mosaic and reuse them for hour filtering, keeping the legacy session-span fallback for older summaries. ([#74337](https://github.com/openclaw/openclaw/pull/74337)) Thanks [@konanok](https://github.com/konanok).

**痛点**：Usage Mosaic 的 token 使用统计之前按 session 时间跨度均摊，精度低且不准确。当多个 session 跨越小时边界时，无法准确知道每个小时实际消耗了多少 token。长期运行的 session 更是模糊了时间维度的精度。

**如何解决**：PR #74337（@konanok）引入 UTC quarter-hour（15 分钟）粒度的 token 桶（`utcQuarterHourTokenUsage`），替代旧的按 session 时间跨度均摊逻辑。这些 15 分钟粒度的桶同时复用于小时级过滤，避免了重复计算。对于升级前生成的旧 summary，保留 session-span 回退机制确保向后兼容。涉及 7 个文件，+518/-70 行。

### #037 BlueBubbles 回复上下文 API 回退 + 日志脱敏

**更新原文**：BlueBubbles: add opt-in `channels.bluebubbles.replyContextApiFallback` that fetches the original message from the BlueBubbles HTTP API when the in-memory reply-context cache misses...Also promotes BlueBubbles attachment download failures from verbose to runtime error so silently-dropped inbound images are visible at default log level, and extends `sanitizeForLog` to redact `?password=…`/`?token=…` query params and `Authorization:` headers before they reach the log sink (CWE-532). ([#71820](https://github.com/openclaw/openclaw/pull/71820)) Thanks [@coletebou](https://github.com/coletebou) and [@zqchris](https://github.com/zqchris).

**痛点**：多实例部署共享一个 BlueBubbles 账号时，内存中的回复上下文缓存可能未命中（因为上下文在其他实例处理时建立）。Gateway 重启后、TTL/LRU 淘汰后也会丢失上下文，导致 bot 无法知道用户回复的是哪条消息。同时，附件下载失败仅记录 verbose 级别日志，静默丢弃了入站图片。此外，日志中可能包含 URL 查询参数里的密码/token（CWE-532 安全漏洞）。

**如何解决**：PR #71820（@coletebou + @zqchris）包含三项改进：(1) 新增 opt-in `replyContextApiFallback`，缓存未命中时通过 BlueBubbles HTTP API 获取原始消息，每次获取都经过 SSRF 防护和 reply-id 验证，并发请求自动合并；(2) 附件下载失败从 verbose 提升为 runtime error，在默认日志级别可见；(3) 扩展 `sanitizeForLog` 脱敏 `?password=`/`?token=` 查询参数和 `Authorization:` 头，修复 CWE-532。

### #038 代理验证命令

**更新原文**：CLI/proxy: add `openclaw proxy validate` so operators can verify effective proxy configuration, proxy reachability, and expected allow/deny destination behavior before deploying proxy-routed OpenClaw commands. ([#73438](https://github.com/openclaw/openclaw/pull/73438)) Thanks [@jesse-merhi](https://github.com/jesse-merhi).

**痛点**：在企业代理环境中部署 OpenClaw 时，代理配置错误（URL 格式、allow/deny 规则、可达性）通常在运行时才暴露，导致请求静默失败或超时。运维人员缺乏在部署前验证代理配置的工具。

**如何解决**：PR #73438（@jesse-merhi）新增 `openclaw proxy validate` CLI 子命令，包含完整的代理验证引擎：URL 解析优先级检查、allowed/denied 目标匹配测试、loopback canary 否定检测（确保代理不会短路到本地）、URL 脱敏输出（避免在终端暴露敏感信息）。配套 9 个文件共 +1398 行，为代理环境下的运维提供了预检工具。

### #039 Codex 动态工具 native-first 默认值

**更新原文**：Agents/Codex: default Codex app-server dynamic tools to native-first, keeping OpenClaw integration tools while leaving file, patch, exec, and process ownership to the Codex harness; default Codex-harness direct source replies to the OpenClaw `message` tool when visible reply delivery is not explicitly configured, keeping channel-visible output as a deliberate tool call. ([#75308](https://github.com/openclaw/openclaw/pull/75308), [#75765](https://github.com/openclaw/openclaw/pull/75765)) Thanks [@pashpashpash](https://github.com/pashpashpash).

**痛点**：OpenClaw 的 Codex 集成之前将 OpenClaw 自身的文件读写、编辑、执行等工具注入 Codex 环境，与 Codex harness 原生的同名工具产生冲突和重复。同时 Codex harness 的 source reply 默认自动发送到频道，而非通过 `message` tool 显式发送，导致回复行为不可控。

**如何解决**：两个 PR 协同修复。PR #75308（@pashpashpash）将 Codex 动态工具设为 native-first：移除与 Codex 原生重复的 read/write/edit/exec 等工具，仅保留 OpenClaw 独有的集成工具（如 message tool、web search）。PR #75765 将 Codex harness 的 source reply 默认改为通过 message tool 发送，使频道可见的输出成为显式的工具调用而非自动行为。

### #040 结构化心跳响应工具

**更新原文**：Heartbeats/agents: add a structured `heartbeat_respond` tool for tool-capable heartbeat runs so agents can record quiet outcomes or explicit notification text without relying only on `HEARTBEAT_OK` parsing. ([#75765](https://github.com/openclaw/openclaw/pull/75765)) Thanks [@pashpashpash](https://github.com/pashpashpash).

**痛点**：之前心跳轮次的结果依赖纯文本 `HEARTBEAT_OK` 约定——agent 需要在回复文本中包含特定字符串来表示"无事发生"。这种方式脆弱且不灵活：agent 无法同时返回"安静无事"和"需要通知"两种状态，也无法提供结构化的心跳元数据（如优先级、摘要等）。

**如何解决**：PR #75765（@pashpashpash）新增 `heartbeat_respond` 结构化工具，支持 `outcome`（结果类型）、`notify`（是否通知）、`summary`（摘要文本）、`notificationText`（通知内容）和 `priority`（优先级）等字段。agent 可以在不产生任何文本输出的情况下记录心跳结果，或同时提供通知文本。该工具与 #039 在同一 PR 中合入，涉及 39 个文件改动（size: L）。

### #041 配置 include 审批目录

**更新原文**：Gateway/config: allow `$include` directives to read files from operator-approved `OPENCLAW_INCLUDE_ROOTS` directories while preserving default config-directory confinement. Thanks [@ificator](https://github.com/ificator).

**痛点**：Gateway 的 `$include` 配置指令默认只能读取配置目录内的文件，这在配置文件分布在多个目录（如共享的 secrets 目录、组织级配置仓库）的场景下不够灵活。用户不得不将所有配置文件复制到配置目录中，或使用符号链接，增加了维护复杂度。

**如何解决**：PR #75746（@ificator）引入 `OPENCLAW_INCLUDE_ROOTS` 环境变量，允许 `$include` 指令从操作员审批的额外目录解析文件。同时保留默认的 config 目录 confinement 和符号链接安全检查，确保只有显式审批的目录才能被访问。这是一个安全可控的配置灵活性改进。


## Fixes (#042-#378)

### #042 GPT-5 SSE Responses Transport 默认值修复

**更新原文**：Agents/OpenAI: default GPT-5 API-key sessions to the SSE Responses transport unless WebSocket is explicitly selected, restoring replies in fresh Control UI and WebChat beta installs where the auto WebSocket path connected but produced no model events.

**痛点**：在全新的 Control UI 和 WebChat beta 安装中，使用 API-key 的 GPT-5 会话默认走 WebSocket 路径。虽然 WebSocket 连接成功建立，但模型不会产生任何事件（`toolcall_delta` 等），导致用户看不到任何回复——静默失败，调试困难。

**如何解决**：将 GPT-5 API-key 会话的默认传输从 WebSocket 切换为 SSE Responses，除非用户显式选择 WebSocket。SSE Responses 是基于 HTTP 的单向推送，与 OpenAI Responses API 配合更稳定。关联 PR #75281 修复了 Codex/Responses 传输层的畸形工具调用参数，并将 `openai-responses` transport 纳入修复范围。PR #70743/70772 则从运行时路径加固角度做了配套改进。

### #043 会话状态保持

**更新原文**：Agents/sessions: preserve terminal lifecycle state when final run metadata persists from a stale in-memory snapshot, preventing sessions from staying stuck as running after completed or timed-out turns.

**痛点**：当 agent 运行完成或超时后，内存中的旧快照可能仍保留旧的运行状态，导致会话在 UI 中持续显示"运行中"，实际已结束。用户看到假阳性状态，交互体验混乱。

**如何解决**：在最终运行元数据持久化时，检查内存快照的 stale 状态，如果 session 已 terminal 则不覆盖为 running，保持正确的终态。该修复属于 agents/sessions 模块，涉及 Pi embedded runner 的状态管理逻辑。

### #044 Gateway 启动与配置修复

**更新原文**：Gateway/CLI/status: make `openclaw gateway start` repair stale managed service definitions that point at old OpenClaw versions, missing binaries, or temporary installer paths before starting; add concrete service, config, listener-owner, and log collection next steps when gateway probes fail and Bonjour finds no local gateway; avoid repeated plugin tool descriptor config hashing so large runtime configs do not block reply startup and trigger reconnect/timeouts. Refs [#49012](https://github.com/openclaw/openclaw/issues/49012). ([#75944](https://github.com/openclaw/openclaw/pull/75944)) Thanks [@vincentkoc](https://github.com/vincentkoc) and [@joshavant](https://github.com/joshavant).

**痛点**：Gateway 启动时如果 service 定义指向旧版本、缺失二进制或临时路径，无法自我修复，导致启动失败或进入不可用状态。同时大型运行配置反复进行 plugin tool descriptor hash 计算会阻塞回复启动，引发超时和重连。

**如何解决**：PR #75944（@vincentkoc, @joshavant）在 `gateway start` 前增加修复检查，清理陈旧的 managed service 定义；probe 失败且 Bonjour 无结果时给出具体的 service/config/listener-owner/log 排查步骤；引入缓存机制避免重复的 plugin tool descriptor hash 计算。该修复涉及 3 个主要模块：gateway start 逻辑、status 子命令、config hashing 优化。

### #045 插件更新配置重构

**更新原文**：Plugins/update/config: stop treating the non-plugin `auth` command root as a bundled plugin id, keep packaged upgrades and beta external plugin installs on stable runtime aliases and matching prerelease npm specs, detect tracked plugin install records whose package directories disappeared during `openclaw update`, reinstall them before normal plugin updates, fail the update if install records still point at missing disk payloads, and validate configured web-search providers plus statically suppressed model/provider pairs against the active plugin set at config load. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：OpenClaw 升级过程中，如果插件包目录在更新期间消失，相关插件安装记录仍被保留但指向不存在的位置，导致配置验证失败。此外 `auth` 命令根被错误当作 bundled plugin id 处理，引发意外行为。beta 外部插件安装依赖的 prerelease npm spec 不稳定，容易出现版本不匹配。

**如何解决**：停止将 `auth` 命令根当作 bundled plugin id；保持 packaged upgrades 和 beta 外部插件安装使用 stable runtime aliases；更新时检测并重新安装目录消失的插件记录；配置加载时验证 web-search providers 和 suppress 模型/提供商对_ACTIVE plugin set 的匹配。该修复由 @vincentkoc 完成，涉及插件系统的完整生命周期管理。

### #046 Codex app-server 二进制解析

**更新原文**：Codex/app-server: resolve managed binaries from bundled `dist` chunks and from the `@openai/codex` package bin when installs do not provide a nearby `.bin/codex` shim, avoiding false missing-binary startup failures.

**痛点**：当 installs 没有提供邻近的 `.bin/codex` shim 时，Codex app-server 无法正确解析 managed binaries，错误地报告"缺少二进制文件"并导致启动失败。这是一个假阳性错误报告——二进制实际存在，只是路径解析逻辑有缺陷。

**如何解决**：从 bundled `dist` chunks 和 `@openai/codex` package bin 两个路径解析 managed binaries，即使没有 `.bin/codex` shim 也能正确定位二进制文件，避免错误的 startup 失败。


### #047 CLI update 服务标记阻断

**更新原文**：CLI/update: treat inherited Gateway service markers as origin hints and only block package replacement when the managed Gateway is still live, so self-updates can stop the service and continue safely. ([#75729](https://github.com/openclaw/openclaw/pull/75729)) Thanks [@hxy91819](https://github.com/hxy91819).

**痛点**：从 gateway 服务进程内部（通过 agent exec 工具）调用 `openclaw update` 时，包更新会成功完成，但受监管模式（systemd）下的重启静默失败——磁盘二进制变成新版本，gateway PID 和内存中的代码仍是旧的。操作员没有任何信号发现问题。这是一个严重的状态不一致问题：日志显示成功，实际情况却是部分更新。

**如何解决**：PR #75729（@hxy91819）引入精细的阻断逻辑：新增 `shouldBlockPackageUpdateFromGatewayServiceEnv()` 函数，通过服务环境标记和运行时状态检查判断 gateway 是否仍在运行。只有在 gateway 实际运行时才阻断更新；服务已停止或从未运行时允许更新继续。同时在 post-core update 子进程中清除 `OPENCLAW_SERVICE_MARKER` 环境变量，避免服务标记遗传。该修复解决了 issue #75691（PPID-ancestry guard）的核心风险，同时保留了服务标记作为安全路径指示器。

### #048 Agent failover 工具执行超时豁免

**更新原文**：Agents/failover: exempt run-level timeouts that fire during tool execution from model fallback, timeout-triggered compaction, and generic timeout payload synthesis, avoiding misleading "LLM request timed out" errors after the primary model has already responded. Fixes [#52147](https://github.com/openclaw/openclaw/issues/52147). ([#75873](https://github.com/openclaw/openclaw/pull/75873)) Thanks [@simonusa](https://github.com/simonusa).

**痛点**：当 agent 运行超时恰好发生在工具执行期间时，failover 逻辑会错误地将主模型已成功产生的响应触发为"LLM request timed out"错误，进而触发模型 fallback、timeout-triggered compaction 和泛化的 timeout payload 合成。用户看到误导性的超时错误，而实际问题可能是工具执行本身慢而非模型响应慢。

**如何解决**：PR #75873（@simonusa）引入 `timedOutDuringToolExecution` flag，在 failover policy、timeout-triggered compaction 和 timeout error emission 三处均增加豁免条件：如果超时发生在工具执行期间，则不触发模型 fallback 和误报的错误信息。该修复针对 issue #52147，确保工具执行期间的超时能正确归因，不误导用户。

### #049 Docker Bun 1.3.13 digest pinning

**更新原文**：Docker: copy Bun 1.3.13 from a digest-pinned image and keep CI on the same version. Fixes [#74356](https://github.com/openclaw/openclaw/issues/74356). Thanks [@fede-kamel](https://github.com/fede-kamel) and [@sallyom](https://github.com/sallyom).

**痛点**：之前 Dockerfile 使用不安全的 `curl | bash` 安装 Bun，引入供应链攻击风险。此外使用版本号而非 digest-pinned 镜像意味着每次构建可能拉取到不同的小版本，存在潜在的构建不一致性。

**如何解决**：PR #74359（关联 issue #74356）将 Dockerfile 中的 `curl | bash` 安装替换为基于 digest-pinned `oven/bun:1.3.13` 多阶段 COPY，完全消除供应链攻击向量。CI 保持使用该固定版本，确保构建一致性。这是一个安全加固和可重现性改进。

### #050 z.ai 类 providers 的 compaction 上下文保留

**更新原文**：Agents/compaction: keep prior context on consecutive turns against z.ai-style providers ([z.ai](http://z.ai/) direct, openrouter z-ai/\*, in-house GLM gateways), avoiding accidental Pi state reset after successful turns. ([#76056](https://github.com/openclaw/openclaw/pull/76056)) Thanks [@openperf](https://github.com/openperf).

**痛点**：在 z.ai 类 providers（直接 z.ai、openrouter z-ai/*、自托管 GLM gateways）上连续 turns 时，Pi 的 auto-compaction 机制可能在追加用户消息之前误触发，导致 `state.messages` 被意外清空。这是 `_checkCompaction` 的 race condition——compaction 检查在消息追加之前运行，在某些 provider 的特定行为模式下会产生错误的 compaction 判断。

**如何解决**：PR #76056（@openperf）新增 `isSilentOverflowProneModel` 辅助函数，通过 endpointClass、normalized provider id 和 model-id prefix 三信号检测 z.ai 风格 provider。在 `applyPiAutoCompactionGuard` 处对这类 provider 禁用 Pi 的 auto-compaction，防止 `_checkCompaction` 误触发；在 `resourceLoader.reload()` 后二次应用 guard 防止重新启用。该修复涉及 7 个文件、284 行新增，属于较深层的状态管理修正。

### #051 Doctor/plugins 一次性版本修复

**更新原文**：Doctor/plugins: run a one-time 2026.5.2 configured-plugin install repair based on `meta.lastTouchedVersion`, update stale configured plugin manifests that still declare channels without `channelConfigs`, install actively used downloadable OpenClaw plugins through the configured external source, preserve unmanaged third-party plugin `node_modules`, and then mark the config touched for the release.

**痛点**：2026.5.2 版本前配置的一些插件manifest 缺失 `channelConfigs` 字段，导致配置不一致；同时某些可下载插件的安装记录指向的包目录已在更新期间消失，但记录未清理。此外第三方插件的 `node_modules` 可能被误删。

**如何解决**：基于 `meta.lastTouchedVersion` 的一次性修复路径：更新缺失 `channelConfigs` 的陈旧插件 manifest；通过配置的外部源重新安装已消失的可下载插件；保留第三方插件的 `node_modules` 不被清理；最后标记配置已触碰（touched）以防重复修复。该修复由 @vincentkoc 完成，属于版本迁移的预防性维护。


### #052 Gateway/channels 启动扇出上限 + Bonjour 竞态修复

**更新原文**：Gateway/channels: cap startup fanout at four channel/account handoffs and recover from Bonjour ciao self-probe races, reducing Windows startup stalls with many Telegram accounts. Fixes [#75687](https://github.com/openclaw/openclaw/issues/75687).

**痛点**：Windows 多 Telegram 账户环境下，gateway 启动时事件循环立即饱和至 100%——所有 Telegram provider 同时发起连接请求（无并发限制），API 调用全部失败且不恢复。同时 Bonjour 插件抛出未处理的 "Can't probe for a service which is announced already" 错误导致 gateway 崩溃。这两个 Bug 叠加导致用户在 .24-.29 版本中完全无法使用多 Telegram bot 配置，每次启动都在循环崩溃中挣扎。

**如何解决**：通过 commit `bf2711b40ef7` 两处修复：1）引入 `CHANNEL_STARTUP_CONCURRENCY = 4` 常量，在 `server-channels.ts` 中使用 `runTasksWithConcurrency` 替代无限制 `Promise.all`，对每账户/每频道启动施加 4 个并发波次限制，避免多账户同时压垮事件循环；2）在 ciao 扩展中将 "Can't probe for a service which is announced already" 分类为自探测竞态，通过进程错误处理器抑制而非崩溃，请求广告商恢复。配套 `server-channels.test.ts`（6 账户 + 6 频道插件 4 并发波次测试）和 `ciao.test.ts` 回归覆盖。

### #053 sessions.list 轮询优化

**更新原文**：Gateway/sessions: keep `sessions.list` polling responsive on large session stores by reusing list-safe session cache/indexes and returning a lightweight compaction checkpoint preview instead of heavyweight summaries. Thanks [@rolandrscheel](https://github.com/rolandrscheel).

**痛点**：当 session store 规模较大时，每次 `sessions.list` 轮询都要重建索引并返回完整的重量级会话摘要，导致控制面板和 CLI 的会话列表刷新出现明显延迟。

**如何解决**：复用 list-safe session cache/indexes，避免每次轮询重复计算；返回 lightweight compaction checkpoint preview 而非完整的 heavyweight summaries，显著降低 CPU 和内存开销。该修复由 @rolandrscheel 完成。

### #054 Control UI/Gateway 综合修复

**更新原文**：Control UI/Gateway: keep long-running dashboard WebSocket sessions alive with protocol pings, keep Stop available after reconnect or reload by recovering session-scoped active-run abort state, contain standalone iOS PWA viewports with safe-area-aware document locking, use high-contrast text selection colors, and show inline feedback when local slash-command dispatch is unavailable or fails unexpectedly. Fixes [#70991](https://github.com/openclaw/openclaw/issues/70991), [#60850](https://github.com/openclaw/openclaw/issues/60850), and [#52105](https://github.com/openclaw/openclaw/issues/52105); supersedes [#60854](https://github.com/openclaw/openclaw/pull/60854). Thanks [@alexandre-leng](https://github.com/alexandre-leng), [@kvncrw](https://github.com/kvncrw), [@Badschaff](https://github.com/Badschaff), [@efe-arv](https://github.com/efe-arv), and [@MooreQiao](https://github.com/Moqiao).

**痛点**：这是一个综合修复包，处理 5 个 Control UI 和 Gateway 的交互问题：1）长时间运行的 dashboard WebSocket 因空闲超时断开；2）WebSocket 重连或页面刷新后 Stop 按钮消失（abort 状态未持久化）；3）iOS PWA 独立模式下视口未正确处理安全区域；4）深色主题文本选择颜色对比度不足；5）本地斜杠命令失败时无用户反馈。

**如何解决**：1）使用 WebSocket 协议级 ping 保活；2）重连时恢复 session-scoped active-run abort 状态；3）iOS PWA 使用 safe-area-aware 文档锁定；4）高对比度文本选择色；5）命令失败时显示内联反馈。该修复 supersedes PR #60854，由 5 位贡献者协作完成。


### #055 CLI update 服务标记阻断（与 #047 相同内容，已在前文记录）

**更新原文**：CLI/update: treat inherited Gateway service markers as origin hints and only block package replacement when the managed Gateway is still live, so self-updates can stop the service and continue safely. ([#75729](https://github.com/openclaw/openclaw/pull/75729)) Thanks [@hxy91819](https://github.com/hxy91819).

**痛点**：从 gateway 服务进程内部调用 `openclaw update` 时，磁盘文件被替换但 gateway 内存仍是旧版本（systemd Restart=always 静默失败）。操作员无任何信号发现问题。

**如何解决**：PR #75729 引入精细阻断逻辑：只在 gateway 实际运行时阻断更新，服务已停止时允许继续。在 post-core 子进程中清除 `OPENCLAW_SERVICE_MARKER` 避免遗传。新增 `shouldBlockPackageUpdateFromGatewayServiceEnv()` 和 `stripGatewayServiceMarkerEnv()` 函数，+147 行测试覆盖。

### #056 Agents failover 工具执行超时豁免

**更新原文**：Agents/failover: exempt run-level timeouts that fire during tool execution from model fallback, timeout-triggered compaction, and generic timeout payload synthesis, avoiding misleading "LLM request timed out" errors after the primary model has already responded. Fixes [#52147](https://github.com/openclaw/openclaw/issues/52147). ([#75873](https://github.com/openclaw/openclaw/pull/75873)) Thanks [@simonusa](https://github.com/simonusa).

**痛点**：当 agent 执行长时间工具调用（如 `process(poll)` 轮询后台进程）导致总运行时间超过 600 秒超时阈值时，failover 逻辑错误地将主模型已成功产生的响应触发为 "LLM request timed out" 错误，进而触发模型 fallback。用户看到完全误导的超时消息，而主模型早已正确响应，回退模型白白消耗 ~61K input tokens 产出 0 个 completion tokens。

**如何解决**：PR #75873 引入 `timedOutDuringToolExecution` flag——通过 `toolStartData` map 检测是否有工具仍在活跃执行中，沿用 PR #46889 的 `timedOutDuringCompaction` 先例模式。failover policy 添加并行豁免：工具执行期间的超时不触发任何模型轮换或回退，而 `shouldRotateAssistant` 继续返回 `continue_normal`。该修复涉及 18 个文件，与 compaction 超时豁免逻辑平行。

### #057 Docker Bun digest-pinning（与 #049 相同内容，已在前文记录）

**更新原文**：Docker: copy Bun 1.3.13 from a digest-pinned image and keep CI on the same version. Fixes [#74356](https://github.com/openclaw/openclaw/issues/74356). Thanks [@fede-kamel](https://github.com/fede-kamel) and [@sallyom](https://github.com/sallyom).

**痛点**：Dockerfile 使用不安全的 `curl | bash` 从 `bun.sh/install` 安装 Bun，无版本锁定、无校验和。如果 bun.sh 或其 CDN 被入侵，每次构建都会静默执行恶意代码。而同一文件的 Node 基础镜像全部使用 SHA256 digest 锁定。

**如何解决**：将 `curl | bash` 替换为从 `oven/bun:1.3.13` 镜像的多阶段 COPY，与 Node 镜像采用相同的安全策略，消除供应链攻击向量。


### #058 z.ai 类 providers 的 compaction 上下文保留

**更新原文**：Agents/compaction: keep prior context on consecutive turns against z.ai-style providers ([z.ai](http://z.ai/) direct, openrouter z-ai/\*, in-house GLM gateways), avoiding accidental Pi state reset after successful turns. ([#76056](https://github.com/openclaw/openclaw/pull/76056)) Thanks [@openperf](https://github.com/openperf).

**痛点**：使用 z.ai 类提供商（直接 z.ai、openrouter z-ai/*、自托管 GLM gateways）时，Pi 的 `_checkCompaction` 在追加用户消息之前运行，对溢出提供商会返回 true 即使 `stopReason: "stop"` 已触发。这导致 `agent.state.messages` 被压缩为单个摘要或空数组——trajectory 事件已捕获完整上下文，但实际发送给提供商的请求体接近空。

**如何解决**：PR #76056 新增 `isSilentOverflowProneModel` 辅助函数，通过 endpointClass、normalized provider id 和 model-id prefix 三信号检测 z.ai 风格 provider。在 `applyPiAutoCompactionGuard` 处对这类 provider 禁用 Pi 的 auto-compaction；在 `resourceLoader.reload()` 后二次应用 guard 防止重新启用。涉及 7 个文件、284 行新增，25 个测试全部通过。

### #059 Doctor/plugins 一次性版本修复

**更新原文**：Doctor/plugins: run a one-time 2026.5.2 configured-plugin install repair based on `meta.lastTouchedVersion`, update stale configured plugin manifests that still declare channels without `channelConfigs`, install actively used downloadable OpenClaw plugins through the configured external source, preserve unmanaged third-party plugin `node_modules`, and then mark the config touched for the release.

**痛点**：从旧版本升级到 2026.5.2 时，可能存在已配置但未安装的插件、过时的 manifest（缺少 `channelConfigs`）、活跃使用但缺失的可下载插件等问题。修复过程中还可能破坏第三方插件的 `node_modules`。

**如何解决**：在 `openclaw doctor` 中添加一次性 2026.5.2 版本修复逻辑：基于 `meta.lastTouchedVersion` 检查触发修复；重新安装未安装的已配置插件；更新过时 manifest；通过外部源安装活跃官方插件；保留第三方插件 `node_modules`；完成后标记版本防止重复运行。

### #060 Session 写入锁超时提升

**更新原文**：Sessions/transcripts: use one `session.writeLock.<wbr/>acquireTimeoutMs` policy for session transcript lock acquisitions and raise the default wait to 60 seconds, avoiding user-visible lock timeouts during legitimate slow prep, cleanup, compaction, and mirror work. Fixes [#75894](https://github.com/openclaw/openclaw/issues/75894). Thanks [@shandutta](https://github.com/shandutta).

**痛点**：v2026.4.29 上合法的 session-write 锁持有时间可达 40-55 秒，但至少一个调用站点使用硬编码的 10 秒锁获取超时。当 gateway 处于 cron/agent prep 负载时，交互式 Telegram 消息会失败并显示 "Something went wrong"——尽管阻塞锁是合法的且稍后会释放。用户看到 10 秒超时错误，而实际锁持有时间达 55 秒。

**如何解决**：引入统一的 `session.writeLock.acquireTimeoutMs` 配置策略，替代各站点的硬编码值；将默认超时从 10 秒提升到 60 秒；确保所有 `acquireSessionWriteLock` 调用站点使用配置项而非硬编码。


### #061 Agent restart recovery topic 后缀匹配修复

**更新原文**：Agents/restart recovery: match cleaned transcript locks by exact transcript lock paths plus the canonical session fallback, so interrupted main sessions using topic-suffixed transcripts resume after gateway restart. Refs [#76052](https://github.com/openclaw/openclaw/pull/76052). Thanks [@anyech](https://github.com/anyech).

**痛点**：Gateway 重启时恢复中断主会话的逻辑，通过 lock 文件名派生 session id 再与 `entry.sessionId` 比较来识别需恢复的会话。对于 topic-suffixed transcript（如 Telegram forum topic），lock 文件名包含 topic 后缀，派生值不等于存储的 session id，导致这些会话无法被标记为需要恢复，重启后静默丢失。

**如何解决**：PR #76052 改为比较规范化的 transcript lock 路径：通过 `resolveSessionFilePath()` 解析每个 session 条目的预期 transcript 文件路径，精确比较预期的 `<transcript>.lock` 路径，而非派生 session id。同时保留 session-id transcript lock 的回退匹配。配套回归测试覆盖 topic 后缀场景、误匹配防护、相对路径清理和元数据过时回退。

### #062 Runtime 系统提示前缀缓存

**更新原文**：Agents/runtime: cache the stable system-prompt prefix and reuse prompt-report tool schema stats during dispatch prep, reducing repeated CPU work before streaming starts. Fixes [#75999](https://github.com/openclaw/openclaw/issues/75999); supersedes [#76061](https://github.com/openclaw/openclaw/issues/76061). Thanks [@zackchiutw](https://github.com/zackchiutw) and [@STLI69](https://github.com/STLI69).

**痛点**：v2026.4.29 上每次 agent dispatch 需要约 73 秒同步 CPU 工作（system-prompt 23s + stream-setup 21s）后才能开始流式输出。`buildEmbeddedSystemPrompt` 每次 dispatch 都进行大量同步字符串拼接 + XML 转义 + 条件渲染所有 skill 元数据，没有任何缓存。同一机器上 Python agent runtime 使用相同模型只需 <10 秒。事件循环阻塞还级联触发 fallback fetch 超时。

**如何解决**：按 (skills hash + workspace files hash) 键值缓存 `buildEmbeddedSystemPrompt` 结果，仅在文件变更时失效；复用 dispatch prep 阶段已计算的 prompt-report tool schema 统计。两项优化直接针对最大开销源，将 dispatch prep 从 ~73s 降至合理范围。该修复 supersedes issue #76061。

### #063 Telegram plugin commands topic session 修复

**更新原文**：Telegram/native commands: pass persisted session files into plugin commands for topic-bound sessions, so `/codex bind` works from Telegram forum topics. Refs [#75845](https://github.com/openclaw/openclaw/pull/75845) and [#76049](https://github.com/openclaw/openclaw/pull/76049). Thanks [@MatthewSchleder](https://github.com/MatthewSchleder).

**痛点**：在 Telegram forum 模式下，每个 topic 有独立 session。用户执行 `/codex bind` 时，命令处理器无法找到当前 topic 对应的 session 文件路径，导致 topic 绑定操作失败。PR #75845（已关闭）尝试修复但未完全解决，#76049 作为后续补充修复。

**如何解决**：在执行 Telegram 插件命令前解析并持久化 session 元数据（`sessionId` 和 `sessionFile`），传递给插件命令处理器，使 topic 邀请的命令路由能正确找到 session 文件。新增 `bot-native-commands.session-meta.test.ts` 回归测试覆盖 topic `/codex` 插件命令路由场景。


### #064 Security audit plugins 目录过滤修复

**更新原文**：Security audit/plugins: ignore plugin install backup, disabled, and dependency debris directories when enumerating installed plugin roots, avoiding false-positive findings for `.openclaw-install-backups` after plugin updates. Fixes [#75456](https://github.com/openclaw/openclaw/issues/75456).

**痛点**：`openclaw security audit --json` 在枚举已安装插件根目录时，会将隐藏的 `.openclaw-install-backups` 目录当作活跃插件扫描，产生误报。此外，审计的插件收集器通过运行时加载器获取插件元数据，会触发捆绑运行时依赖项的暂存（`runtimeDeps.stage`），即使审计本应是只读安全检查，导致审计超时、进程持续消耗 CPU。

**如何解决**：在 `listInstalledPluginDirs` 中添加过滤逻辑，排除以 `.openclaw-install-backups` 开头的目录、禁用插件目录和依赖碎片目录，消除误报并减少审计运行时间。

### #065 Telegram 绑定群组原生命令路由

**更新原文**：Telegram: honor runtime conversation bindings for native slash commands in bound top-level groups, so commands like `/status@bot` route to the active non-`main` session instead of falling back to the default route. Fixes [#75405](https://github.com/openclaw/openclaw/issues/75405); supersedes [#75558](https://github.com/openclaw/openclaw/pull/75558). Thanks [@ziptbm](https://github.com/ziptbm) and [@yfge](https://github.com/yfge).

**痛点**：v2026.4.27 引入回归 bug：在绑定到非 `main` agent 的 Telegram 群组中，原生斜杠命令（`/status@bot` 等）被静默丢弃——消息到达 session bookkeeping（`updatedAt` 更新，`systemSent: false`），但 handler 链从未运行，消息静默消失。DM 正常工作。

**如何解决**：原生命令处理器增加运行时 conversation bindings 查询——当群组绑定到非 `main` agent 时，命令路由到该绑定的活跃 session 而非回退默认路由。修复 supersedes PR #75558。

### #066 Task registry 维护性能优化

**更新原文**：Gateway/tasks: make task registry maintenance use pass-local backing-session lookups and fresh active child-session indexes, avoiding repeated full task snapshots and session-store clones on large stale registries. Fixes [#73517](https://github.com/openclaw/openclaw/issues/73517) and [#75708](https://github.com/openclaw/openclaw/issues/75708); supersedes [#74406](https://github.com/openclaw/openclaw/pull/74406) and [#75709](https://github.com/openclaw/openclaw/pull/75709). Thanks [@Lightningxxl](https://github.com/Lightningxxl), [@glfruit](https://github.com/glfruit), and [@jared-rebel](https://github.com/jared-rebel).

**痛点**：长期运行的安装中，当 `runs.sqlite` 包含过时 task/session 条目时，gateway 启动后消耗 ~90-105% 单核 CPU 长达数分钟，RSS 达 700-1000MB。CPU profile 显示热路径在 `shouldMarkLost -> hasBackingSession -> loadSessionStore` 循环，且 per-task 维护循环内部调用 `listTaskRecords()` 导致 O(n^2)。

**如何解决**：维护扫描取一次 task 快照并在 per-task 决策中复用；使用新鲜活跃 child-session 索引避免重复 `loadSessionStore` + `structuredClone`；`cleanupTerminalAcpSession` 使用已有 tasks 快照而非 per-task 重新调用 `listTaskRecords()`。从 O(n^2) 降至 O(n)，supersedes #74406 和 #75709。

### #067 structuredClone 内存泄漏修复

**更新原文**：Auth/sessions: JSON-clone auth-profile cache/runtime snapshots and remaining session cleanup previews instead of using `structuredClone`, preserving mutation isolation while avoiding native-memory growth on large stores. Fixes [#45438](https://github.com/openclaw/openclaw/issues/45438). Thanks [@markus-lassfolk](https://github.com/markus-lassfolk).

**痛点**：Gateway 以 ~1GB/min 的速率泄漏原生内存——RSS 在数分钟内增长到 4-5GB，而 V8 堆仅 ~1.2GB。`structuredClone` 分配的结构化克隆缓冲区属于 C++ 原生内存，V8 GC 无法回收。热点路径包括 `store-cache.ts`、`sessions/store.ts`、`auth-profiles/store.ts` 中的多处调用。大型 store + 频繁 compaction 的场景下，gateway 在 3-5 分钟内 OOM。

**如何解决**：将 session store 数据（纯 JSON，无 Map/Set/Date/循环引用）的 `structuredClone` 替换为 `JSON.parse(JSON.stringify())`。JSON-clone 所有内存保持在 V8 堆内，GC 可正常回收，同时保留深拷贝的 mutation isolation 语义。

### #068 models list --provider 恢复

**更新原文**：Models CLI: restore `openclaw models list --provider <id>` catalog and registry fallback rows for unconfigured providers, so provider-specific verification commands no longer report "No models found." Fixes [#75517](https://github.com/openclaw/openclaw/issues/75517); supersedes [#75615](https://github.com/openclaw/openclaw/pull/75615). Thanks [@lotsoftick](https://github.com/lotsoftick) and [@koshaji](https://github.com/koshaji).

**痛点**：v2026.4.27 后，`openclaw models list --provider google` 返回空结果 "No models found."——之前能正常返回模型列表。`--provider` 过滤逻辑仅查询已配置的 provider，未配置 provider 即使 catalog 中有数据也返回空。

**如何解决**：对未配置的 provider fallback 到 catalog 和 registry 查询，使用户不需要先配置 provider key 就能查看模型列表。supersedes PR #75615。

### #069 macOS LaunchAgent PATH 规范

**更新原文**：Gateway/macOS: write LaunchAgent services with a canonical system PATH and stop preserving old plist PATH entries, so Volta, asdf, fnm, and pnpm shell paths no longer affect gateway child-process Node resolution. Fixes [#75233](https://github.com/openclaw/openclaw/issues/75233); supersedes [#75246](https://github.com/openclaw/openclaw/pull/75246). Thanks [@nphyde2](https://github.com/nphyde2).

**痛点**：macOS 上 LaunchAgent plist 捕获了交互式 shell PATH（包含 Volta/asdf/fnm/pnpm 版本管理器路径），导致：非确定性 Node 解析、spawnSync ETIMEDOUT 超时、version mismatch 错误。`openclaw doctor --repair` 已能写出正确 PATH（最小规范 `/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`），但安装时未默认使用。

**如何解决**：LaunchAgent plist 安装器始终写入最小规范 PATH，不再依赖用户的交互式 shell PATH；更新 plist 时不保留旧 PATH 条目。修复后 Volta/asdf/fnm/pnpm 不再干扰 gateway 子进程 Node 解析。


### #070 Slack/hooks 保留 bot alert attachment 文本

**更新原文**：Slack/hooks: preserve bot alert attachment text in message-received hook content when command text is blank. Fixes [#76035](https://github.com/openclaw/openclaw/issues/76035); refs [#76036](https://github.com/openclaw/openclaw/pull/76036). Thanks [@amsminn](https://github.com/amsminn).

**痛点**：Slack bot 消息可以有空的顶层 `text` 但将用户可见内容放在 `attachments[0].text` 中（如 bot alert）。OpenClaw 的 prepare 路径能正确将 attachment 文本提取到 `RawBody`/`BodyForAgent`，但插件 hook 映射器以空 `BodyForCommands` 为准，不回退到 `RawBody`，导致 `message_received` hook 收到空白内容并静默忽略——而 agent 路径仍能正确回复同一 alert。

**如何解决**：在插件 hook 映射器中增加回退逻辑：`BodyForCommands` 非空白时优先使用；空白或仅空白时回退到 `RawBody`；再回退到 `Body`。确保非空白命令文本（如 `/status`）仍优先于原始消息文本，插件和内部 `message_received` 上下文暴露相同规范内容。

### #071 专用 session store writer 消除文件锁

**更新原文**：Sessions/agents: route Gateway session-store writes, CLI cleanup maintenance, and agent-delete session purges through a dedicated in-process writer and borrow the validated mutable cache during the writer slot, avoiding runtime file locks plus repeated `sessions.json` rereads and JSON clones on hot metadata updates. Refs [#68554](https://github.com/openclaw/openclaw/pull/68554). Thanks [@henkterharmsel](https://github.com/henkterharmsel).

**痛点**：33MB 的 `sessions.json` 每次写入都触发 `readFileSync + JSON.parse` 阻塞事件循环 ~3-9 秒，5 个排队调用累积成 100+ 秒阻塞，导致 lock watchdog 在 134 秒触发、WebSocket 握手超时、Telegram 断连且无法自恢复必须重启 gateway。根因是 `withSessionStoreLock` 内部调用 `loadSessionStore(..., { skipCache: true })`——缓存已在 `saveSessionStoreUnlocked` 后更新，磁盘重读是冗余的。

**如何解决**：PR #68554 从 `updateSessionStore`/`updateSessionStoreEntry` 移除 `{ skipCache: true }`。本次更新进一步引入专用 in-process writer：所有写入路由到专用 writer，借用已验证的可变缓存而非每次从磁盘重读，消除文件锁和 JSON clone 开销。写入操作从秒级降至毫秒级。

### #072 Memory/markdown CRLF 和重复块自愈

**更新原文**：Memory/markdown: replace CRLF managed blocks in place and collapse duplicate marker blocks without rewriting unmanaged markdown, so Dreaming and Memory Wiki files self-heal from repeated generated sections. Fixes [#75491](https://github.com/openclaw/openclaw/issues/75491); supersedes [#75495](https://github.com/openclaw/openclaw/pull/75495), [#75810](https://github.com/openclaw/openclaw/pull/75810), and [#76008](https://github.com/openclaw/openclaw/pull/76008). Thanks [@asaenokkostya-coder](https://github.com/asaenokkostya-coder), [@ottodeng](https://github.com/ottodeng), [@everettjf](https://github.com/everettjf), and [@lrg913427-dot](https://github.com/lrg913427-dot).

**痛点**：`replaceManagedMarkdownBlock` 正则缺少 `g` 标志导致两个失败模式：CRLF 行尾无法匹配 `\n` 标题分隔符而追加重复块；`.replace()` 不带 `g` 只能替换一个副本导致文件累积多个 managed block 副本。实际案例中 5 个连续 dreaming 块副本使文件从 ~50KB 增长到 243KB，触发截断警告且无限制增长。

**如何解决**：综合三个被取代 PR（#75495/#75810/#76008）的思路：正确匹配 CRLF 管理块并原地替换；折叠相邻重复 marker blocks；无需重写非托管 markdown。Dreaming 和 Memory Wiki 文件能从重复生成段落中自愈。

### #073 Agent restart recovery topic 后缀（与 #061 相同，已在前文记录）

### #074 Agents/tools 工具循环熔断器返回 blocked results

**更新原文**：Agents/tools: return critical tool-loop circuit-breaker stops as blocked tool results instead of thrown tool failures, so models see the guardrail and stop retrying the same call. Thanks [@rayraiser](https://github.com/rayraiser).

**痛点**：工具循环熔断器触发时，之前作为抛出的工具失败（thrown tool failure）返回给模型。模型将其解读为可重试的错误而非系统护栏，继续重试相同调用而非停止。

**如何解决**：熔断器停止事件改为以 blocked tool results 形式返回——模型看到明确的"被阻止"信号，理解这是护栏并停止重试，而非误以为临时错误。涉及 tool-result 类型系统新增状态和相关执行循环逻辑。

### #075 Heartbeat 模型覆盖不再泄漏到共享会话状态

**更新原文**：Agents/sessions: preserve pre-existing runtime model and context window after heartbeat turns so a per-run heartbeat model override does not bleed into shared-session status. Fixes [#75452](https://github.com/openclaw/openclaw/issues/75452). Thanks [@zhangguiping-xydt](https://github.com/zhangguiping-xydt).

**痛点**：配置了 `agents.defaults.heartbeat.model: "<model-B>"`（快速模型，~200k context）覆盖 `main` agent 的 primary 模型 `<model-A>`（大上下文，~1M context）时，heartbeat 轮次使用 `<model-B>`，但轮次完成后覆盖没有被清除——所有后续用户轮次继续运行在错误的 `<model-B>` 上，上下文窗口从 ~1M 降至 ~200k，静默行为降级。

**如何解决**：Heartbeat 轮次完成后，恢复之前的运行时模型和上下文窗口设置，确保 per-run 的 heartbeat 模型覆盖不会泄漏（bleed）到共享会话状态。


### #076 Model commands /model 确认消息澄清为 session-scoped

**更新原文**：Model commands: clarify direct and inline `/model` acknowledgements for non-default selections as session-scoped. Thanks [@addu2612](https://github.com/addu2612).

**痛点**：用户使用 `/model` 命令切换到非默认模型时，确认消息没有明确说明该选择仅在当前会话生效。用户可能误以为模型切换是全局持久化的，导致新会话开始时发现模型配置不符合预期，产生困惑。

**如何解决**：修改 direct 和 inline `/model` 确认消息文本，明确说明非默认模型选择是 session-scoped 的。不涉及逻辑变更，仅是措辞澄清。

### #077 Doctor/gateway 不再对未配置的 user-bin 目录发出警告

**更新原文**：Doctor/gateway: stop warning that non-existent, unconfigured user-bin directories are required in the Gateway service PATH. Fixes [#76017](https://github.com/openclaw/openclaw/issues/76017). Thanks [@xiphis](https://github.com/xiphis).

**痛点**：当 user-bin 目录不存在且未被用户显式配置时，`doctor` 和 Gateway 仍发出警告，提示该目录需要在 PATH 中。这是误报警——用户从未需要自定义二进制路径时，目录不存在不影响正常功能，警告只会造成困扰。

**如何解决**：在 PATH 验证逻辑中增加"目录是否被用户配置过"的检查——仅当用户实际配置了该目录（但 PATH 中缺失）时才发出警告；不存在且未配置的目录不再触发警告。

### #078 TUI/setup 冷启动优化

**更新原文**：TUI/setup: skip full provider model normalization during context-window warmup and bound Terminal hatch bootstrap provider requests, avoiding cold-start stalls with large model registries and first-run hatching stuck behind the watchdog. ([#76241](https://github.com/openclaw/openclaw/pull/76241)) Thanks [@547895019](https://github.com/547895019) and [@joshavant](https://github.com/joshavant).

**痛点**：TUI 启动时执行完整 provider 模型归一化，在配置大量模型注册表时会发出大量 API 请求导致显著延迟；首次运行 hatch 的引导请求如果耗时过长会被 watchdog 超时，导致首次运行卡住。两者都严重影响冷启动体验。

**如何解决**：在 context-window warmup 阶段跳过完整模型归一化，仅获取必要信息；对 bootstrap provider 请求设置并发限制防止无限等待。PR #76241 由 @547895019 和 @joshavant 完成。

### #079 Codex 和 Azure OpenAI Responses 工具调用参数修复

**更新原文**：Agents: enable malformed tool-call argument repair for Codex and Azure OpenAI Responses transports while keeping generic OpenAI Responses paths out of the repair gate. Fixes [#75154](https://github.com/openclaw/openclaw/issues/75154). Thanks [@Nimraakram22](https://github.com/Nimraakram22).

**痛点**：OpenClaw 已有 generic OpenAI Responses 路径的 malformed tool-call argument 修复机制，但 Codex 和 Azure OpenAI Responses 传输不在修复范围内。当这两个 provider 返回格式错误的工具调用参数时无法自动修复，导致工具调用失败。

**如何解决**：将 repair gate 扩展到 Codex 和 Azure OpenAI Responses transports，同时保持 generic OpenAI Responses 路径不变。涉及 tool-call argument 解析/修复管道在 transport 层的入口添加。

### #080 Memory Wiki 接受带 .md 后缀的相对链接

**更新原文**：Memory Wiki: accept relative Markdown links that include the `.md` suffix during broken-wikilink validation, avoiding false positives for native render-mode links. Thanks [@Kenneth8128](https://github.com/Kenneth8128).

**痛点**：Memory Wiki 的 broken-wikilink 验证将带 `.md` 后缀的原生 Markdown 相对链接（如 `[笔记](./notes.md)`）错误标记为断链，而 native render-mode 下这些链接实际是有效的。误报给 Wiki 维护带来困扰。

**如何解决**：在 broken-wikilink 验证逻辑中接受包含 `.md` 后缀的相对 Markdown 链接，验证前剥离 `.md` 后缀再进行页面存在性检查。


### #081 OpenAI Codex 设备配对码显示与日志隔离

**更新原文**：OpenAI Codex: show the device-pairing code in the interactive SSH/headless prompt while keeping the short-lived code out of persistent runtime logs. Fixes [#74212](https://github.com/openclaw/openclaw/issues/74212). Thanks [@da22le123](https://github.com/da22le123).

**痛点**：OpenClaw 支持设备配对，首次使用时需要输入短生命周期配对码。之前面临两难：在 SSH/headless 环境中不显示配对码则用户无法完成配对；写入持久化日志则短期安全凭证被永久保存带来安全风险。

**如何解决**：区分交互式提示和持久化日志：SSH/headless 提示中显示配对码供用户输入；日志系统写入时过滤或脱敏配对码，确保短生命周期安全凭证不泄露到持久存储。遵循"临时安全凭证不写入持久存储"最佳实践。

### #082 QA Lab 父进程消失时停止 Gateway 子进程

**更新原文**：QA Lab: stop gateway children when the suite parent disappears, so interrupted local QA runs cannot leave hot orphaned gateways behind.

**痛点**：QA 套件父进程意外消失（Ctrl+C、进程被杀掉或崩溃）时，Gateway 子进程不会被自动停止，变成孤儿继续占用资源（内存、CPU、端口），可能导致后续运行的端口冲突，需要手动清理。

**如何解决**：实现父子进程关联清理机制——当 QA suite parent 消失时自动停止其关联的所有 gateway children，确保中断的本地 QA 运行不会留下孤儿 Gateway。涉及进程信号处理和进程组管理。

### #083 Plugins/CLI 缓存注册条目

**更新原文**：Plugins/CLI: cache plugin CLI registration entries per command program so completion state generation does not repeat the full plugin sweep in one invocation. Thanks [@ScientificProgrammer](https://github.com/ScientificProgrammer).

**痛点**：生成 shell 补全状态时，每次调用都执行完整插件扫描（遍历所有已安装插件、读取 CLI 注册信息、构建命令树），在单次 Tab 补全中可能触发多次，造成性能浪费、延迟增加和资源消耗。

**如何解决**：按 command program 缓存插件 CLI 注册条目，同一次调用中后续补全生成直接使用缓存而非重复扫描。缓存 key 为 command program 标识，生命周期为 per-invocation。

### #084 Plugins 复用 gateway-bindable 加载器缓存

**更新原文**：Plugins: reuse gateway-bindable plugin loader cache entries for later default-mode loads without serving default-built registries to gateway-bound requests, reducing repeated plugin registration during dispatch. Refs [#61756](https://github.com/openclaw/openclaw/issues/61756). Thanks [@DmitryPogodaev](https://github.com/DmitryPogodaev).

**痛点**：Dispatch 过程中插件多次加载和注册时，default 模式没有复用 gateway-bindable 缓存的条目，导致同一插件被重复注册；同时 default-built 注册表可能被错误提供给 gateway-bound 请求，而后者需要更严格验证的注册表。

**如何解决**：在 default 模式加载时复用为 gateway-bindable 场景构建的缓存条目；确保 default-built 注册表不会被提供给 gateway-bound 请求，两者隔离。减少 dispatch 过程中的冗余注册操作。

### #085 Gateway/secrets 在日志中包含错误消息

**更新原文**：Gateway/secrets: include the caught error message in `secrets.reload` and `secrets.resolve` warning logs while keeping RPC errors generic, so operators can diagnose reload and permission failures. Thanks [@davidangularme](https://github.com/davidangularme).

**痛点**：`secrets.reload` 和 `secrets.resolve` 失败时，日志只显示通用错误信息，操作员无法诊断具体是 reload 失败还是权限问题。错误消息的缺失增加了排查难度。

**如何解决**：在 `secrets.reload` 和 `secrets.resolve` 的 warning 日志中包含捕获的具体错误消息，同时保持 RPC 错误层面的通用性（不泄露敏感细节到外部调用方）。使操作员能诊断 reload 和权限失败，同时保证安全性。


### #086 Gateway/secrets 日志包含错误消息

**更新原文**：Gateway/secrets: include the caught error message in `secrets.reload` and `secrets.resolve` warning logs while keeping RPC errors generic, so operators can diagnose reload and permission failures. Thanks [@davidangularme](https://github.com/davidangularme).

**痛点**：`secrets.reload` 和 `secrets.resolve` 失败时日志只记录"发生了错误"的事实，不包含具体错误消息。运维人员无法从日志判断是 reload 失败还是权限问题，需要复现问题才能定位根因。

**如何解决**：在警告日志中包含捕获的具体错误消息（如 `error.message`），同时保持 RPC 错误层面通用（不向远程调用方泄露敏感详情）。运维可见性提升，同时保持安全性平衡。

### #087 Providers 多项推理兼容性修复

**更新原文**：Providers/OpenRouter/LM Studio/Anthropic: fill DeepSeek V4 `reasoning_content` replay placeholders for `openrouter/deepseek/deepseek-v4-flash` and `openrouter/deepseek/deepseek-v4-pro`, normalize binary LM Studio reasoning metadata from Gemma 4 and other local models, and recover Anthropic-compatible stream text deltas that arrive before their matching content block. Fixes [#76018](https://github.com/openclaw/openclaw/issues/76018) and [#76007](https://github.com/openclaw/openclaw/issues/76007). Thanks [@cloph-dsp](https://github.com/cloph-dsp) and [@vliuyt](https://github.com/vliuyt).

**痛点**：三个 provider 兼容性问题：1）DeepSeek V4 通过 OpenRouter 返回的 `reasoning_content` 在回放时使用占位符未填充；2）LM Studio 返回的 Gemma 4 等本地模型二进制推理元数据无法正确解析；3）Anthropic 流式响应中文本增量有时在 content block 之前到达导致丢失。

**如何解决**：1）填充 DeepSeek V4 OpenRouter 路径的 `reasoning_content` replay 占位符；2）对 LM Studio 二进制推理元数据进行归一化处理；3）恢复 Anthropic 流式响应中提前到达的文本增量。

### #088 fix(infra) 阻止 workspace state-directory 环境变量覆盖

**更新原文**：fix(infra): block workspace state-directory env override [AI]. ([#75940](https://github.com/openclaw/openclaw/pull/75940)) Thanks [@pgondhi987](https://github.com/pgondhi987).

**痛点**：之前允许通过环境变量覆盖 workspace state-directory 路径，带来安全隐患：恶意/错误的环境变量可指向任意路径，导致敏感数据写入不安全位置，可能被用于路径遍历攻击或数据劫持。

**如何解决**：阻止通过环境变量对 state-directory 进行覆盖，state-directory 只能通过配置文件或内部默认值确定。[AI] 标记表明可能为 AI 辅助生成的安全修复。

### #089 MCP/OpenAI and media 多项修复

**更新原文**：MCP/OpenAI and media: normalize parameter-free MCP tool schemas before OpenAI tool submission, honor explicit short `[[tts:text]]...[[/tts:text]]` blocks while keeping untagged short auto-TTS suppressed, and accept home-relative `MEDIA:~/...` attachment paths under the existing file-read policy. Fixes [#75362](https://github.com/openclaw/openclaw/issues/75362), [#73758](https://github.com/openclaw/openclaw/issues/73758), and [#73796](https://github.com/openclaw/openclaw/issues/73796). Thanks [@tolkonepiu](https://github.com/tolkonepiu), [@SymbolStar](https://github.com/SymbolStar), [@yfge](https://github.com/yfge), and [@fabkury](https://github.com/fabkury).

**痛点**：三个问题：1）无参数 MCP tool schema 直接提交 OpenAI 会因格式不符失败；2）显式短 `[[tts:text]]` 块可能被忽略或自动 TTS 错误触发；3）`MEDIA:~/...` home 相对路径不被接受。

**如何解决**：1）提交前将无参数 MCP tool schema 归一化为 OpenAI 兼容格式；2）尊重显式短 TTS 块同时抑制未标记短文本的自动 TTS；3）在现有 file-read policy 下接受 `MEDIA:~/...` 路径。

### #090 Hooks/doctor transformsDir 路径校验警告

**更新原文**：Hooks/doctor: warn when `hooks.transformsDir` points outside the canonical hooks transform directory, so invalid workspace skill paths get a direct recovery hint before the Gateway crash-loops. Fixes [#75853](https://github.com/openclaw/openclaw/issues/75853). Thanks [@midobk](https://github.com/midobk).

**痛点**：`hooks.transformsDir` 指向 canonical hooks transform 目录之外时，Gateway 启动会失败并进入 crash-loop——启动→失败→重启→失败→...用户只看到不断崩溃的 Gateway，没有明确错误提示说明配置路径有问题。

**如何解决**：在 `doctor` 诊断命令中增加路径校验检查——如果 `transformsDir` 指向目录之外，发出警告并提供直接的恢复提示，让用户在 Gateway crash-loop 发生前就发现配置错误。


### #091 Proxy/audio FormData 转换修复

**更新原文**：Proxy/audio: convert standard `FormData` bodies before proxy-backed undici fetches, so audio transcription and multipart uploads no longer send `[object FormData]` when `HTTP_PROXY` or `HTTPS_PROXY` is configured. Fixes [#48554](https://github.com/openclaw/openclaw/issues/48554). Thanks [@dco5](https://github.com/dco5).

**痛点**：在配置了 HTTP/HTTPS 代理的环境中使用音频转写或 multipart 上传时，标准 `FormData` 对象传递给 undici 代理 fetch 时没有被正确转换，导致请求体变成字面量 `[object FormData]` 而非正确的 multipart 编码数据，音频转写和 multipart 上传完全失败。

**如何解决**：在使用代理 fetch 发送请求前，将标准 `FormData` 对象转换为 undici 兼容格式，确保音频转写和 multipart 上传在代理环境下正常工作。

### #092 Discord/delivery/media 多项修复

**更新原文**：Discord/delivery/media: use session-backed A2A announce target lookup for multi-account `sessions_send`, keep typing indicators alive during long tool runs and auto-compaction, preserve multipart Content-Type headers for uploads, preserve attachment and sticker filenames, and keep non-ASCII channel names in session labels while preserving ASCII-slug allowlists. Fixes [#42652](https://github.com/openclaw/openclaw/issues/42652) and [#59744](https://github.com/openclaw/openclaw/issues/59744); refs [#51626](https://github.com/openclaw/openclaw/issues/51626) and [#44773](https://github.com/openclaw/openclaw/pull/44773); supersedes [#73975](https://github.com/openclaw/openclaw/pull/73975). Thanks [@irchelper](https://github.com/irchelper), [@dpalfox](https://github.com/dpalfox), [@Lanfei](https://github.com/Lanfei), [@Squirbie](https://github.com/Squirbie), [@FunJim](https://github.com/FunJim), [@xela92](https://github.com/xela92), [@rockcent](https://github.com/rockcent), and [@swjeong9](https://github.com/swjeong9).

**痛点**：五个相关问题：1）多账号 `sessions_send` 的 A2A announce 错误使用 `accountId: "default"` 而非 session 的 `lastAccountId`；2）长工具运行和自动压缩期间 typing indicator 丢失；3）上传的 multipart Content-Type 头丢失；4）附件和贴纸原始文件名变成 UUID；5）非 ASCII 频道名在 session 标签中被破坏。

**如何解决**：1）使用 session 级别的 `lastAccountId` 进行 A2A announce 目标查找；2）长操作期间保持 typing indicator；3）保留 multipart Content-Type 头和原始文件名；4）session 标签中保留非 ASCII 频道名同时保持 ASCII-slug 白名单。supersedes PR #73975。

### #093 Discord/threads/PluralKit 修复

**更新原文**：Discord/threads/PluralKit: canonicalize proxied webhook turns to the original message id for dedupe, inject thread starter context only on the first effective thread turn, and resolve thread `ownerId`/`parentId` from Discord API-style snake_case payload fields so bot-owned autoThreads do not require unnecessary mentions. Fixes [#41355](https://github.com/openclaw/openclaw/issues/41355); supersedes [#44447](https://github.com/openclaw/openclaw/issues/44447) and [#44449](https://github.com/openclaw/openclaw/issues/44449). Thanks [@acgh213](https://github.com/acgh213), [@p3nchan](https://github.com/p3nchan), and [@mgh3326](https://github.com/mgh3326).

**痛点**：三个问题：1）PluralKit webhook 消息未规范化到原始消息 ID 导致重复；2）`ThreadStarterBody` 在线程每一轮都被重新注入导致 echo 污染和 token 浪费；3）线程 `ownerId`/`parentId` 解析问题导致 bot-owned autoThreads 需要不必要的 mention。

**如何解决**：1）将 webhook 消息规范化到原始消息 ID 去重；2）仅在首轮有效线程轮次注入线程启动器上下文（应用与 Slack 修复相同的 `!previousTimestamp` 守卫模式）；3）从 Discord API snake_case 字段解析 `ownerId`/`parentId`。

### #094 Gateway/diagnostics 启动错误消息包含在 stability bundles

**更新原文**：Gateway/diagnostics: include a bounded redacted startup error message in stability bundles, so crash-loop reports identify the failing plugin or contract without exposing secrets. Refs [#75797](https://github.com/openclaw/openclaw/issues/75797). Thanks [@ymebosma](https://github.com/ymebosma).

**痛点**：Gateway crash-loop 时，stability bundles 缺少诊断信息确定哪个插件或 contract 失败，运维无法定位根因。

**如何解决**：在 stability bundles 中包含经过有界截断和脱敏处理的启动错误消息，让 crash-loop 报告能识别失败的插件或 contract，同时不暴露密钥等敏感信息。

### #095 Gateway/pricing 延迟刷新到 ready 路径之后

**更新原文**：Gateway/pricing: defer optional model pricing catalog refresh until after sidecars and channels reach the ready path, so slow OpenRouter or LiteLLM pricing fetches cannot block Gateway readiness. Fixes [#74128](https://github.com/openclaw/openclaw/issues/74128); supersedes [#73486](https://github.com/openclaw/openclaw/pull/73486). Thanks [@ctbritt](https://github.com/ctbritt) and [@alprclbi](https://github.com/alprclbi).

**痛点**：Gateway 启动时获取模型定价目录，OpenRouter/LiteLLM 响应缓慢会阻塞 Gateway readiness，导致容器编排系统无法调度。定价数据不是核心功能硬依赖——Gateway 可以在无最新定价数据下用缓存数据正常运行。

**如何解决**：将可选的模型定价目录刷新延迟到 sidecars 和 channels 进入 ready 路径之后，使慢速定价接口无法阻塞核心启动流程。supersedes PR #73486。

### #096 Gateway/pricing 关闭时中止进行中的 fetch

**更新原文**：Gateway/pricing: abort in-flight model pricing catalog fetches when Gateway shutdown stops the refresh loop, and avoid post-stop cache writes or refresh timers. Fixes [#72208](https://github.com/openclaw/openclaw/issues/72208). Thanks [@rzcq](https://github.com/rzcq).

**痛点**：Gateway 关闭时，进行中的定价目录获取请求不会被取消，完成后仍尝试写入缓存或设置新刷新定时器，导致无效网络请求继续占用资源、在已关闭缓存上执行写入操作、设置永不触发的定时器。

**如何解决**：Gateway shutdown 停止刷新循环时中止进行中的定价目录获取请求，避免关闭后的缓存写入和刷新定时器。与 #095 共同构成定价模块生命周期完整方案。

### #097 Codex/app-server 启动重试清理所有权感知

**更新原文**：Codex/app-server: make startup retry cleanup ownership-aware so concurrent Codex lanes cannot close another lane's freshly restarted shared app-server client. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：多个 Codex lanes 共享同一个 app-server 客户端实例时，启动重试的清理阶段可能错误关闭另一个 lane 刚重启的客户端，导致该 lane 后续请求失败。这是共享资源的并发管理竞态问题。

**如何解决**：使 startup retry 的清理操作具备所有权感知，每个 lane 只能清理自己拥有的 app-server 客户端实例，防止并发 lanes 之间互相干扰。引入所有权概念是解决此类竞态的标准方案。

### #098 Google Meet/Twilio/Voice Call 多项改进

**更新原文**：Google Meet/Twilio/Voice Call: report missing dial-in details during setup, explain that Twilio needs a phone dial plan for Meet URLs, start the phone leg before Meet PIN DTMF, delay intro speech until after post-connect dialing, log each stage, and accept provider call IDs for gateway speak/continue while reporting ended-call state from history.

**痛点**：语音通话功能的用户体验问题：setup 时缺少拨入详情错误报告、TWILIO 配置困惑、DTMF 时序错误、intro speech 时机不对、各阶段缺乏日志、provider call ID 不兼容。

**如何解决**：五项改进：setup 阶段报告缺少的拨入详情；说明 Twilio 需要电话拨号计划处理 Meet URL；先启动电话呼叫侧再发 DTMF；intro speech 延迟到 post-connect dialing 完成后；各阶段添加日志；接受 provider call ID 用于 speak/continue 并从历史记录报告结束通话状态。

### #099 Control UI/Talk WebRTC 和 VAD 配置修复

**更新原文**：Control UI/Talk: allow the OpenAI Realtime WebRTC offer endpoint through the Control UI CSP, configure browser sessions with explicit VAD/transcription input settings, and surface OpenAI realtime error/lifecycle events instead of leaving Talk stuck as live with no diagnostic. Fixes [#73427](https://github.com/openclaw/openclaw/issues/73427).

**痛点**：三个问题：1）CSP 未将 OpenAI Realtime WebRTC offer 端点列入白名单导致 WebSocket/WebRTC 连接被阻止；2）浏览器会话缺少明确的 VAD 和转写输入设置，音频输入无法正确处理；3）OpenAI Realtime 错误/生命周期事件未展示，Talk 卡在 "live" 状态但连接已断开，用户无法诊断。

**如何解决**：1）在 CSP 白名单中添加 OpenAI Realtime WebRTC offer 端点；2）配置浏览器会话的明确 VAD/转写输入设置；3）展示 OpenAI realtime 错误/生命周期事件供用户诊断。

### #100 Plugins 配置重复覆盖诊断澄清

**更新原文**：Plugins: clarify config-selected duplicate plugin override diagnostics and document manifest schema updates for bundled-plugin forks. Fixes [#8582](https://github.com/openclaw/openclaw/issues/8582). Thanks [@sachah](https://github.com/sachah).

**痛点**：用户配置重复插件或覆盖内置插件时，诊断消息不清晰难以理解发生了什么；同时 bundled-plugin fork 的 manifest schema 文档缺失，开发者不知道如何正确声明 fork 插件的 manifest。

**如何解决**：改善重复插件覆盖的诊断消息，明确说明哪个插件覆盖了哪个及原因；补充 bundled-plugin fork 的 manifest schema 文档。改善插件配置问题的排查体验和开发者文档。


### #101 CLI backends/Claude JSONL turn caps 可配置

**更新原文**：CLI backends/Claude: make live-session JSONL turn caps bounded and configurable via `reliability.outputLimits`, raising the default guard for tool-heavy Claude CLI turns while preserving memory limits. Fixes [#75838](https://github.com/openclaw/openclaw/issues/75838). Thanks [@hcordoba840](https://github.com/hcordoba840).

**痛点**：工具密集型 Claude CLI 对话轮次可能产生大量输出，之前 JSONL turn 上限设置过小或缺乏可配置性，导致关键内容被截断；同时 Claude CLI 后端在长时间会话中的输出限制问题需要解决。

**如何解决**：将 JSONL turn caps 改为有界且可通过 `reliability.outputLimits` 配置；提高工具密集型场景的默认守卫值，同时保留内存限制防止过度消耗资源。

### #102 Telegram/DMs/network/commands 多项修复

**更新原文**：Telegram/DMs/network/commands: keep incidental `message_thread_id` reply-with-quote metadata on flat DM sessions unless topic isolation is configured, raise outbound text and typing Bot API guards to 60 seconds with safe timeout overrides and typing fallback retries, and register/clear command menus in default and group-chat scopes so `/status` and plugin commands stay available in forum topics. Fixes [#75975](https://github.com/openclaw/openclaw/issues/75975), [#76013](https://github.com/openclaw/openclaw/issues/76013), and [#74032](https://github.com/openclaw/openclaw/issues/74032); updates [#6457](https://github.com/openclaw/openclaw/pull/6457). Thanks [@ProjectEvolutionEVE](https://github.com/ProjectEvolutionEVE), [@iaki1206](https://github.com/iaki1206), [@dae-sun](https://github.com/dae-sun), and [@WouldenShyp](https://github.com/WouldenShyp).

**痛点**：三个问题：1）平坦 DM 会话中回复引用的 `message_thread_id` 元数据可能导致路由错误；2）Bot API 超时守卫值过低导致网络缓慢时消息发送失败；3）命令菜单注册作用域不完整导致论坛话题中命令不可用。

**如何解决**：1）在未配置话题隔离时保留 `message_thread_id` 元数据；2）将超时守卫提升到 60 秒并添加 safe override 和 typing 回退重试；3）在 default 和 group-chat 两个作用域注册/清除命令菜单。

### #103 Providers/OpenAI Keychain API Key 解析

**更新原文**：Providers/OpenAI: resolve `keychain:<service>:<account>` `OPENAI_API_KEY` refs before creating OpenAI Realtime browser sessions or voice bridges, with a bounded cached Keychain lookup. Fixes [#72120](https://github.com/openclaw/openclaw/issues/72120). Thanks [@ctbritt](https://github.com/ctbritt).

**痛点**：OpenAI Realtime 功能在创建浏览器会话或语音桥接时，未正确解析 `keychain:<service>:<account>` 格式的 API Key 引用，而是将字符串字面值直接作为 Key 使用，导致认证失败。

**如何解决**：在创建 OpenAI Realtime 连接之前解析 `keychain:<service>:<account>` 引用，使用有界缓存的 Keychain 查找避免频繁系统调用，同时避免每次连接都查询 Keychain。修复了使用系统 Keychain 存储 API Key 的用户在 Realtime 功能中的认证问题。

### #104 Discord/gateway IDENTIFY 并发窗口重连修复

**更新原文**：Discord/gateway: reconnect when the gateway socket closes while waiting for the shared IDENTIFY concurrency window, instead of silently skipping IDENTIFY and leaving the bot online but unresponsive. Fixes [#74617](https://github.com/openclaw/openclaw/issues/74617). Thanks [@zeeskdr-ai](https://github.com/zeeskdr-ai).

**痛点**：Discord 网关在等待 IDENTIFY 并发窗口期间，如果 socket 关闭，系统会静默跳过 IDENTIFY，导致 bot 显示在线但无法接收或处理消息——进入"僵尸"状态，用户体验损害更大。

**如何解决**：当网关 socket 在等待共享 IDENTIFY 并发窗口期间关闭时触发重连，而非静默跳过 IDENTIFY。额外的重连资源消耗比静默失败更可接受。

### #105 Voice Call per-call 会话作用域

**更新原文**：Voice Call: add `sessionScope: "per-call"` for fresh per-call agent memory while preserving the default per-phone caller history. Fixes [#45280](https://github.com/openclaw/openclaw/issues/45280). Thanks [@pondcountry](https://github.com/pondcountry).

**痛点**：此前语音通话的 agent 记忆按来电号码管理，同一号码的所有通话共享会话历史。在号码被多人共用或同一用户需要不同通话完全隔离的场景下不够灵活。

**如何解决**：添加 `sessionScope: "per-call"` 配置选项，启用后每次通话创建全新 agent 记忆不复用历史。默认的 per-phone caller history 行为保持不变确保向后兼容。提供选择而非强制改变默认行为。


### #106 Music generation 超时和错误处理修复

**更新原文**：Music generation: raise too-small tool timeouts to the provider-safe 10-second floor and collapse cascading abort fallback errors into a clearer root-cause summary. Thanks [@shakkernerd](https://github.com/shakkernerd).

**痛点**：音乐生成工具超时设置过小，低于提供商安全底线（10秒），导致请求在提供商处理完成前被取消；同时级联中止错误产生一堆令人困惑的中间错误消息，用户看不到真正的根因（超时）。

**如何解决**：将超时值提升到提供商安全底线 10 秒；将级联中止错误折叠为清晰的根因摘要，让用户能看到实际问题是超时而非一堆中间故障。

### #107 Memory-core/dreaming 工作空间隔离修复

**更新原文**：Memory-core/dreaming: include the primary runtime workspace in multi-agent dreaming sweeps without mixing main-agent session transcripts into configured subagent workspaces. Fixes [#70014](https://github.com/openclaw/openclaw/issues/70014). Thanks [@ttomiczek](https://github.com/ttomiczek).

**痛点**：多 Agent 配置中，dreaming 的扫描范围没有包含主运行时工作空间，导致主 agent 记忆不会被后台整理。更严重的是，主 agent 的会话记录被错误混入配置的子 agent 工作空间，导致子 agent 处理了不属于它的对话内容。

**如何解决**：将主运行时工作空间纳入多 agent dreaming 扫描范围，同时确保主 agent 会话记录不会混入子 agent 工作空间，维持正确的隔离边界。

### #108 Control UI tab/RPC 时间归因和解耦

**更新原文**：Control UI: add tab/RPC timing attribution and decouple slow Overview/Cron secondary refreshes so Sessions navigation gets immediate visible feedback. Refs [#64004](https://github.com/openclaw/openclaw/issues/64004). Thanks [@WaMaSeDu](https://github.com/WaMaSeDu).

**痛点**：Control UI 的 Sessions 导航需要等待 Overview 和 Cron 的二次数据刷新完成才能显示，导致 UI 反馈延迟。根因是这些不相关模块的刷新阻塞了关键路径。

**如何解决**：为每个标签页和 RPC 调用添加耗时统计帮助定位性能瓶颈；将 Overview 和 Cron 的慢速刷新从 Sessions 导航关键路径解耦，使其在后台独立加载。Sessions 导航现在获得即时可见反馈。

### #109 Memory SQLite 索引文件 Windows 重试修复

**更新原文**：Memory: retry transient SQLite index file swaps during atomic reindex on Windows, so brief `EBUSY`, `EPERM`, or `EACCES` locks do not fail memory rebuilds. Fixes [#64187](https://github.com/openclaw/openclaw/issues/64187). Thanks [@kunpeng-ai-lab](https://github.com/kunpeng-ai-lab).

**痛点**：在 Windows 上执行 SQLite 索引文件原子重建交换时，短暂的文件锁会导致 `EBUSY`、`EPERM` 或 `EACCES` 错误使整个内存重建失败，而同样的操作在 Linux/macOS 上正常工作。

**如何解决**：在 Windows 上执行索引文件交换时添加重试机制，遇到短暂锁错误时自动重试而非直接失败，确保 Memory 重建在 Windows 上不会因短暂文件锁而中断。

### #110 Telegram startup/models 多项修复

**更新原文**：Telegram/startup/models: use the existing `getMe` request guard and higher `timeoutSeconds` configs for slow Bot API paths, and make model picker confirmations say selections are session-scoped. Fixes [#75783](https://github.com/openclaw/openclaw/issues/75783) and [#75965](https://github.com/openclaw/openclaw/issues/75965). Thanks [@tankotan](https://github.com/tankotan) and [@sd1114820](https://github.com/sd1114820).

**痛点**：在高延迟环境（如 Oracle Cloud Linux，172ms 到 Telegram API）中，`getMe` 请求超时（默认仅 2500ms）导致 scope upgrade 阻塞和 pairing deadlock，Telegram 频道完全不可用。同时 model picker 确认消息未说明选择是 session-scoped。

**如何解决**：使用已有的 `getMe` 请求守卫确保 bot 身份验证可靠性；提高 `timeoutSeconds` 配置适应慢速网络；让 model picker 确认消息明确说明选择是 session-scoped。

### #111 Control UI/slash commands 浏览器安全修复

**更新原文**：Control UI/slash commands: keep fallback command metadata on a browser-safe registry path, so provider thinking runtime imports cannot blank the Web UI with `process is not defined`. Fixes [#75987](https://github.com/openclaw/openclaw/issues/75987). Thanks [@novkien](https://github.com/novkien).

**痛点**：v2026.4.30 引入回归 bug，Web UI 显示完全空白，浏览器控制台报错 `process is not defined`。Provider thinking 模块导入 Node.js 专有代码（引用 `process` 全局变量）时，这些运行时导入链在浏览器环境中触发 ReferenceError。

**如何解决**：将回退命令元数据保持在浏览器安全的注册表路径上，确保 provider thinking 运行时导入无法影响 Web UI，使 `process is not defined` 错误不再发生。

### #112 与 #110 相同（已记录）

### #113 与 #111 相同（已记录）

### #114 Heartbeat/Discord exec 完成事件处理修复

**更新原文**：Heartbeat/Discord: keep async exec completion events out of the generic `System (untrusted)` prompt block and let the dedicated exec heartbeat prompt handle them, so Discord no longer receives raw exec failure tails as separate system-style messages. Fixes [#66366](https://github.com/openclaw/openclaw/issues/66366). Thanks [@Promee-ThaBossHoss](https://github.com/Promee-ThaBossHoss).

**痛点**：异步 exec 完成事件被错误注入到通用的 `System (untrusted)` prompt 块，导致 Discord 用户看到原始 exec 失败尾部作为单独的"系统风格"消息——用户收到了不应出现的内部执行细节。

**如何解决**：将 async exec 完成事件从通用 `System (untrusted)` prompt 块中移除，由专门的 exec heartbeat prompt 处理，避免 Discord 收到原始 exec 失败尾部消息。

### #115 Heartbeat/scheduler 时区感知调度

**更新原文**：Heartbeat/scheduler: make heartbeat phase scheduling active-hours-aware so the scheduler seeks forward to the first in-window phase slot instead of arming timers for quiet-hours slots and relying solely on the runtime guard. Non-UTC `activeHours.timezone` values (e.g. `Asia/Shanghai`) now correctly influence when the next heartbeat timer fires, avoiding wasted quiet-hours ticks and long dormant gaps after gateway restarts. Fixes [#75487](https://github.com/openclaw/openclaw/issues/75487). Thanks [@amknight](https://github.com/amknight).

**痛点**：用户配置 heartbeat activeHours 时区为 `Asia/Shanghai`（UTC+8），但调度逻辑完全按 UTC phase 对齐，`activeHours.timezone` 被忽略。结果是调度器在 quiet-hours 时段也设置计时器而非跳到第一个 in-window phase slot，导致安静时段大量无意义的 tick 和 gateway 重启后长休眠间隙。

**如何解决**：使 heartbeat phase 调度具备 active-hours 感知能力：调度器现在向前寻找第一个 in-window phase slot 而非为 quiet-hours 时段设置计时器，Non-UTC `activeHours.timezone` 值（如 `Asia/Shanghai`）正确影响下次心跳计时器触发时间。


### #116 Channels 剥离 MiniMax 和 XML 工具调用脚手架

**更新原文**：Channels: strip plain-text MiniMax and XML tool-call scaffolding from shared user-facing reply sanitization, so messaging channels do not deliver raw model tool syntax when a provider emits it as text instead of structured tool calls. Fixes [#62820](https://github.com/openclaw/openclaw/issues/62820). Thanks [@canh0chua](https://github.com/canh0chua).

**痛点**：当模型以文本形式（而非结构化 `tool_calls[]`）生成工具调用时，原始 XML 语法（如 `<minimax:tool_call><invoke name="exec">...`）直接发送到 WhatsApp 等消息频道的用户界面。用户看到的是原始的模型工具语法而非正常回复。根因是 `sanitizeUserFacingText()` 只处理 `<final>` 标签，不处理工具调用 XML，而 `stripToolCallXmlTags()` 虽然存在但未在消息投递路径中被调用。

**如何解决**：在共享的用户面向回复净化路径中增加对 MiniMax 和 XML 工具调用脚手架的 stripping，将 `stripToolCallXmlTags()` 整合到消息投递的 sanitization 流程中，确保所有消息频道都受益。

### #117 Infer/media provider 配置错误报告修复

**更新原文**：Infer/media: report missing image-understanding and audio-transcription provider configuration for `image describe`, `image describe-many`, and `audio transcribe` instead of blaming the input path when no provider is available. Fixes [#73569](https://github.com/openclaw/openclaw/issues/73569) and supersedes [#73593](https://github.com/openclaw/openclaw/pull/73593), [#74288](https://github.com/openclaw/openclaw/pull/74288), and [#74495](https://github.com/openclaw/openclaw/pull/74495). Thanks [@bittoby](https://github.com/bittoby), [@tmimmanuel](https://github.com/tmimmanuel), [@Linux2010](https://github.com/Linux2010), and [@vyctorbrzezowski](https://github.com/vyctorbrzezowski).

**痛点**：当未配置 audio/image understanding provider 时，`infer audio transcribe` 和 `infer image describe` 报错 "No transcript returned for audio: <path>" 或 "No description returned for image: <path>"——错误归咎于输入路径而非真正的问题（缺失 provider 配置）。用户无法从错误信息判断该调试文件还是 provider 配置。对比 `infer image generate` 正确报告 "No image-generation model configured..." 并指向缺失配置。

**如何解决**：在调用 provider 前先检查是否有可用 provider，无则报告明确的配置缺失错误（参照 `infer image generate` 的风格）。该修复合并了 @tmimmanuel、@Linux2010、@vyctorbrzezowski 的三个被取代 PR。

### #118 CLI/infer 拒绝 local codex/* 模型探测

**更新原文**：CLI/infer: reject local `codex/*` one-shot model probes before simple-completion dispatch and point operators at the Codex app-server runtime path instead of ending with an empty-output error.

**痛点**：通过 CLI `infer` 命令使用 `codex/*` 模型进行 one-shot 推理时，请求被错误分派到 simple-completion 路径，但 Codex 模型需要通过 Codex app-server 运行时执行，不支持 simple-completion dispatch。命令以空输出错误结束，用户不清楚原因。

**如何解决**：在 simple-completion dispatch 之前拦截 `codex/*` 模型探测请求，不再分派到 simple-completion 路径，向操作者明确指引 Codex app-server 运行时路径，避免空输出错误。

### #119 Docs/health session listing 澄清

**更新原文**：Docs/health: clarify that session listing surfaces stored conversation rows rather than Discord/channel socket liveness, and point connectivity checks at channel status and health probes. Fixes [#70420](https://github.com/openclaw/openclaw/issues/70420). Thanks [@ashersoutherncities-art](https://github.com/ashersoutherncities-art) and [@martingarramon](https://github.com/martingarramon).

**痛点**：Discord health monitor 因 stale socket 重启 provider 后，`sessions_list` 不再显示 Discord session 条目，但 `openclaw status` 显示 Discord ON/OK。用户使用 `sessions_list` 作为连接健康检查产生误报——它反映的是存储的对话记录行，不是 socket 活跃度。

**如何解决**：文档层面澄清 `sessions_list` 展示存储的对话记录行而非 channel socket 活跃状态，将连接健康检查指向 channel status 和 health probes，消除了使用 `sessions_list` 判断频道连接状态的误导。

### #120 WhatsApp/Cron DM pairing-store 审批隔离

**更新原文**：WhatsApp/Cron: keep DM pairing-store approvals out of implicit cron and heartbeat recipient fallback, so scheduled automation only uses explicit targets, active configured recipients, or configured `allowFrom` entries. Fixes [#62339](https://github.com/openclaw/openclaw/issues/62339). Thanks [@kelvinisly-collab](https://github.com/kelvinisly-collab).

**痛点**：DM pairing-store 审批通过的用户被隐式加入 cron 和 heartbeat 接收者列表，导致未配置这些联系人为接收者的用户也收到了定时自动化消息。计划任务消息可能发送给不应收到的人。

**如何解决**：将 DM pairing-store 审批从隐式 cron 和 heartbeat 接收者回退列表中移除，计划自动化现在只使用显式指定目标、活跃配置接收者或配置的 `allowFrom` 条目，防止意外发送。


### #121 Google Meet 非 macOS 主机工具可见性修复

**更新原文**：Google Meet: keep the agent-facing `google_meet` tool visible on non-macOS hosts but block local Chrome realtime actions with guidance, so Linux agents can still use transcribe, Twilio, chrome-node, and artifact flows without choosing the macOS-only BlackHole path. Refs [#75950](https://github.com/openclaw/openclaw/issues/75950). Thanks [@actual-software-inc](https://github.com/actual-software-inc).

**痛点**：在 Linux 环境下，Google Meet 插件的本地 Chrome 实时加入被音频后端阻止——BlackHole 是 macOS 专用的。用户不清楚 Linux 是完全不支持还是只是缺少文档，导致无法正确配置。

**如何解决**：保持 `google_meet` 工具在非 macOS 主机上仍可见，但阻止需要 BlackHole 的本地 Chrome 实时操作并提供引导说明。Linux agent 仍可使用 transcribe、Twilio、chrome-node 和 artifact flows 等替代方案，不再强制选择 macOS-only 的 BlackHole 路径。

### #122 macOS/settings 防止 General 设置重写配置文件

**更新原文**：macOS/settings: keep opening General from rewriting `openclaw.json` during Tailscale settings hydration, preserving `gateway`, `auth`, `meta`, and `wizard` until the user changes a setting. Fixes [#59545](https://github.com/openclaw/openclaw/issues/59545). Thanks [@Tengdw](https://github.com/Tengdw).

**痛点**：打开 macOS 应用的 Settings General 页面后，即使用户未做任何更改，`openclaw.json` 也会被重写——`gateway.bind` 从 `auto` 变为 `loopback`，`gateway.auth.token` 被刷新，`meta.lastTouchedAt` 和 `wizard` 字段也被更新。根因是 SwiftUI `.onChange` 处理器在加载/水合阶段误触发保存逻辑，与用户编辑设置时的保存逻辑相同。

**如何解决**：打开 General 页面时不再在 Tailscale 水合过程中触发配置重写，保留 `gateway`、`auth`、`meta` 和 `wizard` 字段直到用户实际修改设置。修复了加载阶段误触发保存的问题。

### #123 Discord 交互回调优先级和 Payload 验证

**更新原文**：Discord: prioritize interaction callbacks ahead of stale background REST work without polling active REST buckets, validate oversized gateway payloads and member-intent requests before send, and forward explicit component payloads from message actions. ([#75363](https://github.com/openclaw/openclaw/pull/75363))

**痛点**：交互回调（如按钮点击）可能被排队的旧 REST 请求阻塞；超大 gateway payload 和 member-intent 请求没有提前验证导致 API 错误或连接断开；组件 payload 未正确转发。

**如何解决**：PR #75363（@steipete）添加 Carbon 风格的 Discord REST 请求通道优先处理交互回调、丢弃过时后台请求；通过 payload 大小和 member-intent 验证加固发送处理；正确转发消息动作中的显式组件 payload。

### #124 Active Memory recall timeout 配置修复

**更新原文**：Active Memory: use the configured recall timeout as the blocking prompt-build hook budget by default and move cold-start setup grace behind explicit `setupGraceTimeoutMs` config, so the plugin no longer silently extends 15000 ms configs to 45000 ms on the main lane. Fixes [#75843](https://github.com/openclaw/openclaw/issues/75843). Thanks [@vishutdhar](https://github.com/vishutdhar).

**痛点**：`active-memory` 插件的 `before_prompt_build` hook 使用 `config.timeoutMs + setupGraceTimeoutMs`（默认 15000 + 30000 = 45000ms）作为超时预算，破坏了配置的 timeoutMs 预算。当 embedded recall sub-agent 准备阶段超过 15s 时，hook 锁定主命令通道 45s，导致新 WebSocket 握手掉线、`eventLoopUtilization=1`。

**如何解决**：使用配置的 recall timeout 作为 blocking prompt-build hook 的默认预算；将 cold-start setup grace 移到显式 `setupGraceTimeoutMs` 配置项后面；插件不再静默将 15000ms 配置扩展到 45000ms，操作者可以明确控制超时预算。

### #125 Plugins/web-provider 运行时 provider 解析优化

**更新原文**：Plugins/web-provider: reuse the active gateway plugin registry for runtime web provider resolution after deriving the same candidate plugin ids as the loader path, avoiding a redundant `loadOpenClawPlugins` call on every request while preserving origin and scope filters. Fixes [#75513](https://github.com/openclaw/openclaw/issues/75513). Thanks [@jochen](https://github.com/jochen).

**痛点**：ARM64（Raspberry Pi 4）上每个请求调用 `loadOpenClawPlugins` 耗时 8-17s，导致 `createWebSearchTool` 每次请求 8.3s、总响应时间 120s。`resolvePluginWebProviders` 的 cacheKey 永远无法匹配活跃的 gateway registry（启动时没有 `onlyPluginIds`），导致每个请求都触发冗余的 `loadOpenClawPlugins`。

**如何解决**：推导出与 loader 路径相同的 candidate plugin ids 后，复用活跃的 gateway plugin registry，避免每个请求冗余调用 `loadOpenClawPlugins`，同时保留 origin 和 scope filters。ARM64 上 `createWebSearchTool` 从 8.3s/请求降至 3ms/请求，warm 请求总时间从 ~47s 降至 ~14s。


### #126 Crestodian/CLI: 修复非 TTY 环境下的错误退出码

**更新原文**:
Crestodian/CLI: exit non-zero when interactive Crestodian is invoked without a TTY, so scripts and CI no longer treat the setup error as success. Fixes #73646 and supersedes #73928 and #74059. Thanks @bittoby, @luyao618, and @Linux2010.

**痛点**:
`openclaw crestodian` 在检测到 stdin 不是 TTY 时，向 stdout 打印 "Crestodian needs an interactive TTY. Use --message for one command."，但以**退出码 0** 退出。这与 OpenClaw 其他子命令的行为不一致——`models auth login` 在无 TTY 时退出码为 1，`plugins uninstall` 退出码为 13。Shell 脚本和 CI 流程看到退出码 0 就认为命令执行成功，实际上是一次 Setup 失败，静默错过了错误信号。长期积累下来，自动化流程无法可靠地检测到 Crestodian 配置错误。

**如何解决**:
改动 `src/cli/crestodian.ts`（或同等路径）的 TTY 检测逻辑，当检测到非 TTY 上下文时，将退出码从 0 改为非零值（1），与 `models auth login` 等兄弟子命令对齐。本次修复合并并取代了 @luyao618 和 @Linux2010 的两个社区贡献 PR，由 @bittoby 整合完成。Shell 脚本和 CI 现在可以用 `$?` 判断 Crestodian 是否真正成功。

---

### #127 Cron: 隔离任务 announce 投递不再污染主 session

**更新原文**:
Cron: keep implicit/default isolated cron announce deliveries out of the main session awareness queue, so isolated jobs do not accumulate in the main conversation. Fixes #61426. Thanks @Lihannon.

**痛点**:
配置了 `sessionTarget: "isolated"` 的 cron 任务，本应在独立 session 中运行和投递消息，实际上所有消息都静默积累在主对话 session（`conversation_id 6`）中。用户报告约 10 天后 `lcm.db` 中积累了数千条 cron 消息，最终导致 "Context limit exceeded" 错误，agent 崩溃，进行中的任务和推理全部丢失。这是一个回归 bug，影响所有使用 isolated cron 任务的用户——隔离功能完全失效，却没有任何报错。

**如何解决**:
在 cron announce 投递路径中，加入对 `sessionTarget: isolated` 的检测逻辑：当 job 配置为 isolated 时，其 announce 投递不再路由到主 session 的 awareness queue，而是正确地进入目标独立 session。具体改动位于 cron 调度的投递路由层（`src/cron/` 或 `src/scheduler/`），在 enqueue 阶段过滤掉应进入 isolated queue 的消息，避免与主 session 的消息流混合。

---

### #128 Subagents: 修复 sessions_send 导致回复重复投递问题

**更新原文**:
Subagents: avoid duplicate parent-visible replies when a parent uses `sessions_send` on its own persistent native subagent session, while preserving announce delivery for async sends. Fixes #73550. Thanks @sylviazhang2006-design.

**痛点**:
当父 agent 对自己持有的持久化原生 subagent session 调用 `sessions_send` 发送消息时，subagent 的回复会被投递 2-3 次到父 session 的对话中。这在 WebChat UI 中造成严重的信息冗余，用户看到的不是一条回复，而是同一条消息重复出现多次。问题的根因可能是 announce 投递模式与 sessions_send 回复路径产生了重叠，或者 subagent 接收到了自己的 announce 通知并触发反馈循环。

**如何解决**:
在 `sessions_send` 的投递路由中，新增对 "parent 对自己的 persistent native subagent 使用 sessions_send" 这一特定场景的检测。当检测到这个场景时，去除导致重复投递的次级路径，同时保留 `announce` 投递用于异步发送场景（async sends）。具体实现位于 subagent session 的消息路由层，通过去重或路径隔离来确保每个回复只被投递一次。

---

### #129 Web search/Brave: 新增 brave.http 诊断标志

**更新原文**:
Web search/Brave: add opt-in `brave.http` diagnostics for Brave request URLs/query params, response status/timing, and cache hit/miss/write events without logging API keys or response bodies. Fixes #55196. Thanks @mecampbellsoup.

**痛点**:
Brave Search provider 处理 `web_search` 调用但不提供专门的诊断标志（diagnostics flag），这与 OpenClaw 其他子系统（如 telegram）的 `http` 诊断模式不一致。用户在排查 Brave 搜索问题时，要么无法在不提升全局日志级别到 debug 的前提下看到发往 `api.search.brave.com` 的 API 调用细节，要么无法验证 provider 是否真正被使用（vs 缓存返回旧结果）。这影响了需要审计外部搜索 API 调用行为的企业用户和开发者。

**如何解决**:
为 Brave Search 添加 `brave.http` opt-in 诊断标志，与已有的 `telegram.http` 模式一致。启用后记录：出站搜索请求的 URL 和 query 参数、HTTP 响应状态和时间、缓存命中/未命中/写入事件。**不记录 API 密钥或响应体**（安全考虑）。配置方式为在 `diagnostics.flags` 中加入 `"brave.http"`，然后通过 `openclaw logs --follow | grep brave` 查看日志。低风险改动——标志默认关闭，不影响未启用用户。

---

### #130 Web search/Brave: 新增 baseUrl 配置支持代理

**更新原文**:
Web search/Brave: add `plugins.entries.brave.config.webSearch.baseUrl` for Brave-compatible proxies, including endpoint-aware cache keys for both web and LLM Context modes. Fixes #19075. Thanks @jkoprax and @vishnukool.

**痛点**:
Brave Search provider 的 API 端点是硬编码的（`https://api.search.brave.com`），没有 `baseUrl` 配置选项。而 Perplexity provider 已经支持 `baseUrl`（存在先例）。在自托管和企业部署场景中，通常需要通过本地代理路由外部 API 调用——用于缓存、审计日志、速率限制，或满足限制直接出站连接的企业网络策略。Brave 是唯一不支持此功能的搜索 provider，迫使企业用户要么直接暴露 Brave API 端点，要么放弃使用 Brave。

**如何解决**:
在 Brave 插件配置中新增 `plugins.entries.brave.config.webSearch.baseUrl` 字段。当设置该字段时，搜索请求路由到指定代理地址而非硬编码的 Brave API 端点。缓存键（cache key）也感知端点（endpoint-aware），同时支持 Web 和 LLM Context 两种模式，确保通过不同代理端点的请求被视为独立缓存条目。未设置 `baseUrl` 时行为完全不变——无破坏性变更，与 `models.providers` 对 LLM 端点处理 `baseUrl` 的方式一致。

---

### #131 Web search: 恢复 provider 配置验证

**更新原文**:
Web search/config: validate explicit `tools.web.search.provider` values against bundled and installed plugin manifests, while warning for stale third-party plugin config. Fixes #53092. Thanks @TinyTb.

**痛点**:
`tools.web.search.provider` 配置字段原本是严格枚举（只接受 `brave` 或 `perplexity`），后来改为 `z.string().optional()` 以支持扩展注册的 provider（如 tavily、duckduckgo、exa）。这个改动虽然解除了扩展限制，但也移除了所有验证——任何拼写错误或无意义的 provider 字符串都被静默接受，配置 schema 形同虚设。用户配置了一个不存在的 provider 后，OpenClaw 不报错，只是不工作，排查起来非常困难。

**如何解决**:
在配置验证阶段，显式校验 `tools.web.search.provider` 的值：对比 bundled 内置 provider（brave、perplexity）和已安装插件注册的 provider 清单，验证用户配置的 provider 名称是否在有效列表中。对于过时的第三方插件配置（曾经安装过但已移除），发出警告而非直接报错，保留一定的向前兼容性。这在严格枚举和无验证之间找到了平衡——既恢复了验证能力，又不破坏扩展性。

---

### #132 Plugins/web-provider: 默认启用插件注册表缓存

**更新原文**:
Plugins/web-provider: cache repeated bundled web search and web fetch provider registry loads by default while preserving explicit cache opt-outs. Supersedes #75992. Thanks @DmitryPogodaev.

**痛点**:
`resolveWebProviderLoadOptions` 和 `loadOpenClawPlugins` 默认 `cache: false`，导致每次消息 dispatch 时都会重复加载所有 bundled web search/fetch providers（brave、duckduckgo、exa、firecrawl、google、minimax、moonshot、ollama、perplexity、searxng、tavily、xai 等约 12 个插件），耗时约 1500ms。用户在任何配置了 web search providers 的 agent 中发送任意消息，都会触发这个重复加载——性能浪费严重，却无任何明显报错。

**如何解决**:
将 `resolveWebProviderLoadOptions` 和 `loadOpenClawPlugins`（setup 模式下）的默认缓存值从 `false` 改为 `true`。缓存 key 已通过 `buildCacheKey` 函数对 `onlyPluginIds`、`activate` 等输入参数变化进行了区分，因此相同输入的后续调用会正确命中缓存。显式传入 `cache: false` 的调用方（如 auth-choice catalog warning）保留现有 override。插件现在只在 gateway 生命周期内加载一次而非每次 dispatch 都重复加载。

---

### #133 Agents/sandbox: 保留文件权限模式避免权限降级

**更新原文**:
Agents/sandbox: preserve existing workspace file modes when sandbox edits atomically replace files, so 0644 files do not collapse to 0600 after Write/Edit/apply_patch. Fixes #44077. Thanks @patosullivan.

**痛点**:
Sandbox 的文件编辑工具（Write/Edit/apply_patch）在原子替换文件时，使用 "写入临时文件 → rename" 模式。新建的临时文件受 umask 影响，默认创建为 0600 权限。rename 操作后，原文件的 0644 权限丢失，变成 0600。这导致 host 侧的文件工具（可能以不同用户身份运行）在后续访问这些文件时遇到 EACCES（权限拒绝）错误。这是一个回归 bug，影响所有使用 sandbox 编辑功能的 agent 工作流——任何在 sandbox 中编辑过的文件都会静默失去原有的共享权限。

**如何解决**:
在原子替换操作中，保留原文件权限模式的记录。具体的 write temp + rename 流程改为：在 rename 后调用 `chmod()` 恢复原文件的权限模式。涉及 POSIX 文件系统语义中 `rename()` 不保留目标文件权限这一特性，需要在应用层主动记录和恢复权限。这是有意针对 `write/Edit/apply_patch` 路径的修复，不影响 sandbox 其他功能的正常行为。

---

### #134 Control UI/WebChat: 修复 /new 命令路由到真正的全新会话

**更新原文**:
Control UI/WebChat: route typed `/new` through the New Chat dashboard-session creation flow instead of `chat.send`, while keeping `/reset` as the explicit current-session reset. Fixes #69599. Thanks @WolvenRA.

**痛点**:
在 Control UI / WebChat 中，用户输入 `/new` 命令或点击 New Session 按钮时，实际上并未创建真正的新会话，而是回到了 canonical main session（`agent:main:main`）。这与直接通过 session key 创建的新会话不同——新会话的上下文约为 18k/272k（约 7% 已使用），而 `/new` 后的会话却继承了原来会话的完整上下文。这导致用户误以为自己开始了新会话，实际上继续在旧会话中操作，上下文无法释放，造成 token 浪费和性能下降，高 UX 影响——用户无法可靠地逃离"污染"或过大的会话。

**如何解决**:
将 `/new` 命令的路由从 `chat.send` 改为 New Chat dashboard-session 创建流程，确保每次 `/new` 都产生真正独立的新会话。`/reset` 则保持为对当前会话的重置操作，与 `/new` 有了明确的功能区分。相关 issue #66533（session 选择器标签）、#47561（/new 被路由变更覆盖）、#58353（/new 或 /reset 携带旧内容）均指向同一个核心问题：UI 层的 `/new` 没有真正创建新 session，而是 fallback 到了 main session。

---

### #135 Agents/models: 修复 cron 中 claude-cli/* 模型引用被静默丢弃问题

**更新原文**:
Agents/models: keep legacy CLI runtime model refs such as `claude-cli/*` in the configured allowlist after canonical runtime migration, so cron `payload.model` overrides keep working. Fixes #75753. Thanks @RyanSandoval.

**痛点**:
从 2026.4.27 更新到 2026.4.29 后，所有使用 `payload.model: "claude-cli/<model>"` 的 cron 任务一夜之间全部失败，错误信息：`"cron payload.model 'claude-cli/claude-sonnet-4-6' rejected by agents.defaults.models allowlist"`。即使该 model ref 已在 `~/.openclaw/openclaw.json` 的 `agents.defaults.models` 中配置。更严重的是，`openclaw config validate` 仍然报告 "Config valid"，`openclaw doctor` 也不报警——配置验证和健康检查均未捕获这个静默丢弃。只有在 cron 实际运行时才暴露问题，用户毫不知情。这是一个严重的回归 bug，报告者有 8 个生产 cron 一夜之间全部中断。

**如何解决**:
在 canonical runtime migration 中，当构建 cron 验证的 allowlist 时，同时保留旧版 `claude-cli/*` 前缀和新的 canonical 形式。修复涉及 `resolveAllowedModelRefFromAliasIndex` 和 `buildAllowedModelSetWithFallbacks`（位于 `dist/model-selection-shared-*.js`）函数，确保 alias/index 系统在处理 model 引用时不会丢弃旧版前缀。canonical 迁移逻辑现在正确保留了 `claude-cli/*` 条目，使 cron 的 `payload.model` 覆盖能继续工作。

---

### #136 Codex/app-server: 启动线程恢复时重启共享客户端

**更新原文**:
Codex/app-server: restart the shared Codex app-server client once when it closes during startup thread resume, preserving the existing thread binding instead of retrying `thread/start` on a closed client. Thanks @vincentkoc.

**痛点**:
Codex app-server 使用共享客户端与 Codex 后端通信。在应用启动时恢复之前暂停的线程（thread resume）阶段，客户端连接可能已经关闭。此前的逻辑是在已关闭的客户端上重试 `thread/start` 操作，必然失败。这会导致应用启动阶段出现静默挂起或失败，用户需要手动重启才能恢复。

**如何解决**:
当共享客户端在启动线程恢复期间关闭时，执行一次客户端重启（仅一次，避免无限重试循环），并保留现有的线程绑定（thread binding），而非在已关闭的客户端上继续重试失败的操作。这是一个防御性的生命周期管理修复，针对 startup thread resume 这一个人迹罕至的路径。

---

### #137 Gateway/watch: 在 NO_COLOR 环境下保留 tmux 日志颜色

**更新原文**:
Gateway/watch: keep colored subsystem log prefixes in the managed tmux pane even when the parent shell exports `NO_COLOR`, while preserving explicit `FORCE_COLOR=0` opt-out. Thanks @vincentkoc.

**痛点**:
`NO_COLOR` 是 no-color.org 社区标准的环境变量，告诉 CLI 工具不要输出彩色文本。OpenClaw 的 `gateway watch` 命令在 managed tmux pane 中显示带颜色的子系统日志前缀，便于用户区分 plugins、agents、providers 等子系统。之前如果父 shell 设置了 `NO_COLOR`，tmux pane 中的日志前缀颜色也会被禁用，日志可读性大幅下降，影响日常开发调试体验。

**如何解决**:
在 managed tmux pane 内部忽略父 shell 传下来的 `NO_COLOR` 环境变量，但保留 `FORCE_COLOR=0` 作为显式 opt-out 选项。这样区分了"被动继承父 shell NO_COLOR"和"用户主动设置 FORCE_COLOR=0 明确禁用颜色"两种场景——前者保留颜色提可读性，后者尊重用户明确偏好。

---

### #138 Agents/compaction: 修复预压缩内存刷写被 Anthropic 拒绝问题

**更新原文**:
Agents/compaction: submit a non-empty runtime-event marker for pre-compaction memory flush turns, so strict Anthropic providers no longer reject the silent flush as an empty user message. Fixes #75305. Thanks @sableassistant3777-source.

**痛点**:
OpenClaw 的 compaction（上下文压缩）机制在压缩历史消息前，会执行一次内存刷入（memory flush），这个 flush turn 发送一个"静默"空消息给 LLM。然而，严格遵循 Anthropic API 规范的提供商会拒绝空的用户消息（`user` 角色消息内容不能为空）。这导致启用 compaction 且使用严格 Anthropic API 的用户在压缩流程触发时遇到 API 调用失败，compaction 无法正常完成，进而可能导致上下文窗口溢出。

**如何解决**:
为 pre-compaction memory flush turns 提交一个非空的 runtime-event marker，作为用户消息的内容。这样即使 flush turn 本身是"静默"的，API 请求中的 `user` 消息也包含非空内容，满足 Anthropic API 的规范要求，同时不影响 compaction 的功能和语义。

---

### #139 Plugin SDK: 重新导出 isPrivateIpAddress 恢复源码构建

**更新原文**:
Plugin SDK: re-export `isPrivateIpAddress` from `plugin-sdk/ssrf-runtime`, restoring source-checkout builds for SearXNG and Firecrawl private-network guards. Thanks @vincentkoc.

**痛点**:
`isPrivateIpAddress` 是 SSRF（Server-Side Request Forgery）防护的核心工具函数，用于检测请求目标是否为私有 IP 地址（127.0.0.1、10.x.x.x、192.168.x.x 等）。SearXNG 和 Firecrawl 插件依赖 `plugin-sdk/ssrf-runtime` 模块中的这个函数来实现私有网络防护。在某次重构中，这个函数的导出可能被意外移除，导致从 Git 源码构建这两个插件时，私有网络守卫（private-network guards）无法找到该函数，构建失败。这是一个影响源码构建开发者的回归问题。

**如何解决**:
在 `plugin-sdk/ssrf-runtime` 模块的 index 文件中重新导出 `isPrivateIpAddress`，恢复 SearXNG 和 Firecrawl 插件的源码构建能力。这可能是一个简单的 re-export 修复，添加回一条导出声明语句即可。

---

### #140 Discord: agent 现在可以发现和发送文件附件

**更新原文**:
Discord/message actions: advertise `upload-file` and route it through Discord's send runtime with agent-scoped media reads, so agents can discover and send file attachments. Fixes #60652 and supersedes #60808, #61087, and #61100. Thanks @claw-io, @efe-arv, @joelnisharth and @sjhddh.

**痛点**:
之前通过 Discord 渠道与 OpenClaw agent 交互时，agent 无法发现和发送文件附件。这是一个重要的功能缺失——文件传输是聊天交互中的常见需求，用户期望能够通过 Discord 发送截图、文档等文件，但 agent 的能力列表中没有 `upload-file` 标识，导致这个功能完全不可用。多个社区贡献者（@claw-io、@efe-arv、@joelnisharth、@sjhddh）独立提交了相关 PR（#60808、#61087、#61100），说明这是社区强烈需求的功能。

**如何解决**:
在 Discord message actions 中新增两个能力：其一，在 agent 的能力列表中声明 `upload-file` 支持，让 agent 知道自己可以发送文件；其二，将文件发送路由到 Discord 的发送运行时（send runtime），通过 Discord API 的文件上传端点完成实际传输。同时引入 agent-scoped media reads，确保 agent 读取媒体文件时限制在自身作用域内（权限控制）。本次修复合并了四个社区 PR，由核心团队整合完成。

---

### #141 Sessions: 隐藏会话间控制回复和 agent 通告

**更新原文**:
Sessions: suppress exact inter-session control replies such as `NO_REPLY` and keep agent-to-agent announce bookkeeping out of visible transcripts. Fixes #53145. Thanks @TarahAssistant.

**痛点**:
在多 agent / 多 session 的场景中，会话间存在内部控制通信——使用 `NO_REPLY` 等控制回复表示"无需回复"，以及 agent 间通过 announce 机制协调状态。之前这些内部通信和记账信息会出现在用户可见的对话记录中，导致用户看到含义不明的 `NO_REPLY` 消息和 agent 间的内部通告，污染对话流。用户需要的是干净的对话记录，而非 agent 内部协调的噪音。

**如何解决**:
两处改动：第一，抑制精确匹配的控制回复（如 `NO_REPLY`），不在可见对话记录中显示；第二，将 agent-to-agent announce 的记账信息保留在内部数据结构中，不写入用户可见的 transcript。实际通信功能不受影响，只是将用户不该看到的内部信息隐藏起来。

---

### #142 CLI/directory: 正确报告插件不支持目录操作

**更新原文**:
CLI/directory: report unsupported directory operations for installed channel plugins instead of prompting to reinstall the plugin when it lacks a directory adapter. Fixes #75770. Thanks @lawong888.

**痛点**:
当用户对已安装的 channel 插件执行目录操作（directory operations）时，如果该插件没有实现 directory adapter（目录适配器），系统之前会错误地提示用户"重新安装插件"。这个提示是误导性的——重新安装不会解决问题，因为问题在于该插件本身就不支持目录操作功能。用户被引导去做无效的操作，浪费时间和感情。

**如何解决**:
在 `src/cli/directory.ts`（或同等路径）中加入插件能力检测逻辑：当插件缺少 directory adapter 时，报告明确的"此操作不受支持"错误信息，而非提示重新安装。这样提供更准确的错误引导，帮助用户理解真正的问题所在。

---

### #143 Web search: SearXNG/Firecrawl/Kimi 大型安全与功能复合修复

**更新原文**:
Web search/SearXNG/Firecrawl/Kimi: show the SearXNG JSON API `search.formats` prerequisite, pass through `img_src` image URLs, fail explicitly when Kimi returns ungrounded answers, keep public provider requests on strict SSRF guards, reject private/loopback/metadata/non-HTTP(S) hosted Firecrawl scrape targets, and allow explicit self-hosted private Firecrawl endpoints. Fixes #52573, #74357, #63877; supersedes #65592, #61416, #74360, #48133, #59666, #63941, #74013. Thanks @evanpaul14, @sghael, @wangwllu, @fede-kamel, @kn1ghtc, @jhthompson12, @jzakirov, @Mlightsnow, and @shad0wca7.

**痛点**:
这是一个涉及多个搜索提供商的大型复合修复，涵盖 6 个不同问题：SearXNG 缺少 JSON API 前置条件提示导致配置后不工作；`img_src` 图片 URL 被忽略导致图片搜索结果不完整；Kimi 返回无根据答案时静默接受导致结果不可信；Firecrawl 对私有/回环/元数据端点的请求缺乏安全防护存在 SSRF 风险；等等。这些问题单独看都不大，但积累起来影响大量使用这些搜索提供商的用户。

**如何解决**:
6 项改动并行推进：(1) SearXNG 检测到 JSON 格式未启用时明确提示 `search.formats` 前置条件；(2) 透传 SearXNG 搜索结果中的 `img_src` 图片 URL；(3) Kimi 返回无引用答案时显式报错而非静默接受；(4) 公共提供商请求保持严格 SSRF 防护；(5) 拒绝私有 IP、回环地址（127.0.0.1）、云元数据端点（169.254.169.254）和非 HTTP(S) 协议的 Firecrawl 抓取目标；(6) 提供显式配置选项，允许用户 opt-in 自托管私有 Firecrawl 端点（不改变默认安全策略）。本修复 supersede 了 7 个社区 PR，涉及 9 位贡献者，是典型的大型安全+功能合并。

---

### #144 CLI/models: 报告模型回退尝试并修复双重前缀

**更新原文**:
CLI/models: report gateway model fallback attempts in `infer model run --json` and avoid double-prefixing provider-qualified defaults such as `openrouter/auto` in `models status`. Partially fixes #69527. Thanks @alexifra.

**痛点**:
这里涉及两个独立的 CLI 模型显示问题，都影响用户的调试和管理体验。其一，`infer model run --json` 在模型回退（fallback）发生时不报告回退尝试的详细信息，用户无法知道实际使用了哪个模型、经历了哪些回退步骤，调试困难。其二，`models status` 中已经带提供商前缀的模型名（如 `openrouter/auto`）会被再次添加前缀，变成 `openrouter/openrouter/auto` 这样的双重前缀，严重干扰用户对模型配置的理解。

**如何解决**:
两处改动并行：(1) 在 `infer model run --json` 的 JSON 输出中加入回退尝试记录，包含原始请求模型、经历的回退步骤序列和最终使用的模型；(2) 在 `models status` 的模型名格式化逻辑中增加前缀检测——如果模型名已包含提供商前缀（如含 `/` 且前半部分为已知 provider），则不再添加前缀。注意这是 "Partially fixes"，issue #69527 还有其他子问题待后续修复。

---

### #145 Providers/OpenRouter: 修复 reasoning 模式下 prefill 被拒问题

**更新原文**:
Providers/OpenRouter: strip trailing assistant prefill turns from verified OpenRouter Anthropic model requests when reasoning is enabled, so Claude 4.6 routes no longer fail with Anthropic's prefill rejection through the OpenAI-compatible adapter. Fixes #75395. Thanks @sbmilburn.

**痛点**:
通过 OpenRouter 使用 Claude 4.6 的 reasoning（推理）模式时，API 请求会被 Anthropic API 拒绝。问题出在 OpenRouter 使用 OpenAI-compatible adapter 转发请求的过程中，请求里包含的 assistant prefill turns（助手角色预填充轮次）在 reasoning 模式下不被 Anthropic API 接受。这是一个严重的兼容性问题，导致所有通过 OpenRouter 使用 Claude 4.6 reasoning 功能的用户都会遇到静默失败或报错。

**如何解决**:
在通过 OpenRouter 发送 Anthropic 模型请求的适配层中，当检测到 reasoning 模式启用时，自动剥离（strip）尾随的 assistant prefill turns。这确保发往 Anthropic API 的请求符合其在 reasoning 模式下的格式要求。修复是针对 OpenRouter OpenAI-compatible adapter 路径的，不影响直接调用 Anthropic API 的用户。

---

### #146 Voice Call: 添加按电话号码的入站路由

**更新原文**:
Voice Call: add per-number inbound routing for dialed-number greetings, response agents/models/prompts, and TTS voice overrides. Fixes #56604. Thanks @healthstatus.

**痛点**:
OpenClaw 的 Voice Call 功能之前只支持全局配置，所有入站语音通话使用相同的问候语、agent、model、prompt 和 TTS voice。对于拥有多个电话号码的用户或组织，无法为不同号码配置不同行为——例如，客服号码和技术支持号码需要不同的问候语和响应 agent。这是功能的重大限制，阻止了 Voice Call 在多业务场景中的应用。

**如何解决**:
新增按电话号码的入站路由配置能力，支持 5 个维度的 per-number 配置：自定义问候语（dialed-number greetings）、指定响应 agent、指定语言模型、指定系统提示词，以及 TTS voice 覆盖。这样同一 OpenClaw 实例可以为不同号码路由到完全不同的 AI 配置，实现真正的多业务语音服务。

---

### #147 Feishu: 保留 HTTP 错误响应体改善诊断

**更新原文**:
Feishu: preserve Feishu/Lark HTTP error bodies for message sends, media sends, and chat member lookups, so HTTP 400 failures include vendor code, message, log id, and troubleshooter details. Fixes #73860. Thanks @desksk.

**痛点**:
OpenClaw 的飞书/Lark 集成在发送消息、媒体文件或查询群成员时遇到 HTTP 400 等错误，之前的实现丢弃了响应体中的详细诊断信息。飞书 API 的错误响应实际包含丰富的故障排查信息——厂商错误码、错误描述、飞书侧的 log id（可联系技术支持）和 troubleshooter details。这些信息的丢失让开发者面对飞书集成错误时几乎无从下手，需要反复试错才能找到根因。

**如何解决**:
在飞书/Lark API 调用的错误处理路径中（三处：消息发送、媒体发送、成员查询），保留完整的 HTTP 错误响应体，并将 vendor code、message、log id 和 troubleshooter details 注入到 OpenClaw 的错误对象中。开发者现在可以在错误日志中直接看到飞书返回的完整诊断信息，大幅缩短排查时间。

---

### #148 Agents/transcripts: 避免重打开大型 transcript 文件

**更新原文**:
Agents/transcripts: avoid reopening large Pi transcript files through the synchronous session manager for maintenance rewrites, persisted tool-result truncation, manual compaction boundary hardening, and queued compaction rotation. Thanks @mariozechner.

**痛点**:
OpenClaw 的 transcript 系统使用 Pi 格式文件存储会话历史，长期运行的会话文件可能非常庞大。此前的实现中，多个维护操作（维护重写、工具结果截断、手动压缩边界加固、压缩轮转）会通过同步会话管理器重新打开这些大型文件。同步 I/O 操作会阻塞事件循环，增加内存压力，严重时导致超时和性能崩溃。这是一个影响长期运行会话的潜在性能杀手。

**如何解决**:
在这 4 类维护场景中，避免通过同步会话管理器重新打开大型 transcript 文件。改用异步方式或复用已缓存的索引/句柄，大幅减少同步 I/O 操作。涉及 `src/agents/transcripts/` 路径下的多个文件，由核心贡献者 @mariozechner 提交。

---

### #149 Web search/Exa/MiniMax: baseUrl 支持和 API key 自动检测

**更新原文**:
Web search/Exa/MiniMax: accept Exa `webSearch.baseUrl` overrides with endpoint-partitioned caches, include MiniMax Search in setup, and let `MINIMAX_API_KEY` participate in MiniMax Search auto-detection. Fixes #54928; supersedes #54939 and #65828. Thanks @mrpl327, @lyfuci, and @Jah-yee.

**痛点**:
这里涉及两个独立的改进：(1) Exa 搜索之前不支持 `webSearch.baseUrl` 配置覆盖，自定义端点的缓存没有按端点分区，导致不同端点（如自托管地址）的搜索结果可能互相混淆；(2) MiniMax Search 未包含在 OpenClaw 的设置流程中，`MINIMAX_API_KEY` 环境变量也没有参与自动检测，用户需要手动配置，流程繁琐。这两项都是可用性问题，影响相应提供商的用户体验。

**如何解决**:
针对 Exa：新增 `webSearch.baseUrl` 配置支持，并使用端点分区的缓存 key，确保不同 baseUrl 的请求被视为独立缓存条目。针对 MiniMax：将 MiniMax Search 纳入 setup 引导流程，并让 `MINIMAX_API_KEY` 参与 MiniMax Search 的自动检测（类似其他提供商的 API key 自动发现机制）。本次修复合并了两组社区工作（分别 supersede 了 #54939 和 #65828），涉及 3 位贡献者。

---

### #150 Plugins/ClawHub: archive 安装时保留官方源信任

**更新原文**:
Plugins/ClawHub: preserve official source-linked trust through archive installs, so OpenClaw can install trusted ClawHub plugin packages that trigger the built-in dangerous-pattern scanner. Thanks @vincentkoc.

**痛点**:
OpenClaw 内置了"危险模式扫描器"（dangerous-pattern scanner）来检测插件代码中的不安全模式，这是重要的安全防护机制。当插件通过 ClawHub（OpenClaw 官方插件市场）以源码链接（source-linked）方式安装时，应该享有信任状态。但通过 archive（压缩包）方式安装时，这个信任状态丢失了——导致一个实际上来自 ClawHub 官方可信源的插件，仅仅因为安装方式的不同，就被危险模式扫描器阻止安装。正确的安装方式反而不被信任，这损害了用户体验和插件生态。

**如何解决**:
在 archive 安装流程中，检查插件包的来源信息——如果确认为来自 ClawHub 官方的源码链接包，则保留其信任状态，使其能够通过内置的危险模式扫描器。这不降低安全性——只有明确来自官方 ClawHub 源（通过源码链接标识）的插件才能享受此信任，其他来源的插件仍需通过完整扫描。

---

### #151 Plugins/ClawHub: archive 安装时安装运行时依赖

**更新原文**:
Plugins/ClawHub: install package runtime dependencies for archive-backed plugin installs, so ClawHub packages such as WhatsApp load declared dependencies after download. Thanks @vincentkoc.

**痛点**:
通过 archive（压缩包）方式安装 ClawHub 插件时，插件声明的运行时依赖没有被自动安装。以 WhatsApp 插件为例，它在 `package.json` 中声明了运行时依赖库，但 archive 安装流程只下载了插件本身的代码，没有执行依赖安装步骤。用户下载插件后满怀期待，却发现插件无法运行——错误可能是 "module not found" 或类似依赖缺失提示，排查起来费时费力。

**如何解决**:
在 archive 安装流程中新增运行时依赖安装步骤：下载插件 archive 后，解析 `package.json` 中的 `dependencies` 字段，执行 `npm install` 或等效机制安装声明的依赖包。这是与 #150（Plugins/ClawHub: archive 安装保留信任链）同属 ClawHub archive 安装流程改进，由同一贡献者 @vincentkoc 提交。

---

### #152 Plugins/tools: 缓存插件工具工厂结果避免性能雪崩

**更新原文**:
Plugins/tools: cache repeated plugin tool factory results only for matching request context, reducing per-turn tool prep without leaking sandbox, session, browser, delivery, or runtime config state. Fixes #75956. Thanks @Linux2010.

**痛点**:
OpenClaw 在 Windows 原生环境下出现严重延迟，发送一条简单消息如"你好"需要约 90 秒，其中 ~78 秒消耗在 `core-plugin-tools` 阶段。性能剖析显示根因：每次用户发消息都会遍历所有插件条目并执行 `entry.factory()`，而工厂调用是完全无缓存的。以 57 个插件加载为例，每条消息都要重复执行所有工厂调用，同步 I/O 阻塞事件循环造成 78 秒的等待。这是典型的性能雪崩——无缓存的重复工作随着插件数量线性增长。

**如何解决**:
实现插件工具工厂结果的缓存机制，核心设计决策是"仅对匹配的请求上下文进行缓存"——缓存键考虑了请求上下文的匹配性，避免跨沙盒、会话、浏览器、投递或运行时配置的状态泄漏。这样相同上下文参数的重复调用可以复用缓存结果，大幅减少 per-turn 工具准备时间，从 78 秒降到可接受范围。涉及 `tools-CCfW25J2.js` 中 `resolvePluginTools()` 函数的改动。

---

### #153 Providers/LM Studio: preload=false 让 LM Studio 管理模型生命周期

**更新原文**:
Providers/LM Studio: allow `models.providers.lmstudio.params.preload: false` to skip OpenClaw's native model-load call so LM Studio JIT loading, idle TTL, and auto-evict can own model lifecycle. Fixes #75921. Thanks @garyd9.

**痛点**:
从 2026.4.22 开始，OpenClaw 调用 LM Studio 的 `/api/v1/models/load` API 会阻止 LM Studio 的 JIT（Just In Time）模型加载系统正常工作。LM Studio 的 JIT 加载、空闲 TTL 和自动卸载功能是管理 GPU 显存的核心机制——模型应在空闲 TTL 后自动卸载。但 OpenClaw 调用了 `/api/v1/models/load` 后，LM Studio 将模型视为手动加载，不再对其应用 JIT 策略，导致所有模型永远驻留 GPU 显存，无法释放。用户的配置完全正确，但显存持续占用直到 OOM。这是一个"越俎代庖"式 的 bug：OpenClaw 以为自己比 LM Studio 更懂如何管理后者的模型生命周期。

**如何解决**:
新增配置参数 `models.providers.lmstudio.params.preload: false`。当设置为 `false` 时，OpenClaw 跳过自己的原生模型加载调用，不再调用 `/api/v1/models/load`，让 LM Studio 的 JIT 系统完全接管模型生命周期管理。这是一种典型的"控制权让渡"设计——让专业工具管理自己擅长的领域，而非让 OpenClaw 强行干预。

---

### #154 Agents/transcripts: 使用有界异步读取避免重新解析大型会话文件

**更新原文**:
Agents/transcripts: keep chat history, restart recovery, fork token checks, and stale-token compaction checks on bounded async transcript reads or cached async indexes instead of reparsing large session files. Thanks @mariozechner.

**痛点**:
OpenClaw 的 transcript 系统在需要获取聊天历史、执行重启恢复、进行 fork token 检查或过期 token 压缩检查时，每次都重新解析整个大型会话文件。这类同步操作会阻塞事件循环，随着会话历史增长，解析时间线性增加，长期运行的会话文件可达数十 MB，每次检查都要全量解析是严重的性能负担。这是 #148（避免重打开大型 transcript 文件）的延续——两处都针对 transcript 性能问题，由同一核心贡献者 @mariozechner 提交。

**如何解决**:
将这些检查操作从"每次重新解析"改为使用"有界（bounded）异步读取"或"缓存的异步索引"。有界异步读取确保内存可控，缓存的异步索引则避免重复解析同一会话文件。涉及 `src/agents/transcripts/` 路径的多个文件改动，共享同一优化思路：减少同步 I/O，提升并发能力。

---

### #155 Telegram: 继承 DNS 结果顺序并降级 IPv4 回退日志

**更新原文**:
Telegram: inherit the process DNS result order for Bot API transport and downgrade recovered sticky IPv4 fallback promotions to debug logs, while keeping pinned-IP escalation warnings visible. Fixes #75904. Thanks @highfly-hi and @neeravmakwana.

**痛点**:
用户报告 Telegram 插件在 24 小时内产生约 128 次 IPv4 回退警告日志，即使系统已设置 `NODE_OPTIONS=--dns-result-order=ipv4first`。该警告在 Gateway 中出现的频率高于其他任何日志，高噪信比严重干扰运维排查。根因是 Telegram 插件维护自己的双栈解析器/代理，完全不遵循进程级别的 DNS 结果顺序设置。这导致用户即使在系统层面配置了 DNS 偏好，Telegram 渠道的行为仍然与全局设置不一致。

**如何解决**:
两处改动并行：(1) Telegram Bot API 传输层现在继承进程级别的 DNS 结果顺序设置（读取 `process.options` 或 `dns.getDefaultResultOrder()`），尊重 `--dns-result-order` 配置；(2) 将已自动恢复的粘性 IPv4 回退提升（sticky IPv4 fallback promotion）降级为 debug 级别日志（因为已恢复，不需要运维关注），但保留 pinned-IP 升级警告为可见级别（因为那表示更严重的连接问题）。

---

### #156 Sessions: 保护持久外部会话指针不被维护任务驱逐

**更新原文**:
Sessions: keep durable external conversation pointers, including group and thread-scoped chat sessions, out of age, count, and disk-budget maintenance eviction while still allowing synthetic runtime entries to age out. Fixes #58088. Thanks @drinkflav.

**痛点**:
OpenClaw 的三个会话维护路径（`enforceSessionDiskBudget`、`pruneStaleEntries`、`capEntryCount`）会不加区分地删除会话条目。Telegram 论坛（topics）和线程会话条目被当作普通会话处理，在磁盘预算超限、条目数过多或超时后被当作"最旧条目"驱逐删除。结果是用户长期经营的讨论线程从 `sessions.json` 中消失，新消息创建的是空白会话，所有历史上下文、压缩记忆和会话状态全部丢失。用户感受到的是"AI 突然忘记了一切"——而实际上会话文件还在磁盘上（孤立状态），只是不再被引用。

**如何解决**:
将会话条目分为两类：第一类是持久外部会话指针（group 和 thread-scoped 聊天会话），从 age、count 和 disk-budget 三个维护驱逐路径中排除，确保用户期望长期保留的会话不会被意外清除；第二类是合成运行时条目（如临时子代理、cron 任务等产生的条目），仍按正常规则老化淘汰。这在"保护用户长期会话"和"清理无意义的临时会话"之间找到了平衡。

---

### #157 Web search/MiniMax: 支持 OAuth 认证并正确路由端点

**更新原文**:
Web search/Providers MiniMax: allow `MINIMAX_OAUTH_TOKEN` to satisfy MiniMax Search credentials and derive Coding Plan usage polling from the configured MiniMax base URL, so OAuth-authorized and global setups use the right endpoint. Fixes #65768 and #65054. Thanks @kikibrian, @zhouhe-xydt, @sixone74, and @Yanhu007.

**痛点**:
MiniMax Search 之前可能只支持 API Key 认证，不支持 OAuth Token。拥有 OAuth 授权的用户无法使用 MiniMax Search，因为凭证验证机制不认 `MINIMAX_OAUTH_TOKEN`。同时，Coding Plan 用量轮询（usage polling）的端点可能硬编码了某个 MiniMax 地址，导致 OAuth 授权用户和全球（global）用户使用了错误的端点——这可能表现为用量统计不准确或功能完全失效。这是影响多个 issue 的配置兼容性问题，涉及 4 位贡献者协作修复。

**如何解决**:
两处改动并行：(1) 新增 `MINIMAX_OAUTH_TOKEN` 环境变量作为 MiniMax Search 的认证凭证，满足 OAuth 授权用户的凭证需求；(2) Coding Plan 用量轮询端点从配置的 MiniMax base URL 动态派生，确保 OAuth 授权用户和全球部署用户使用各自正确的端点地址。

---

### #158 Control UI/WebChat: 跳过无法解析的过时媒体引用

**更新原文**:
Control UI/WebChat: skip assistant-media transcript supplements when stale media refs resolve to no playable media, so text-only final replies are not stored a second time as gateway-injected assistant messages. Fixes #73956. Thanks @HemantSudarshan.

**痛点**:
当助手生成包含媒体的回复时，系统会为 transcript 添加媒体补充信息（media supplement）。但如果媒体文件已被删除、过期或路径变更，transcript 中残留的媒体引用会变成过时（stale）状态，无法解析到可播放的媒体。之前系统的处理是：即使媒体引用无效，仍然以 gateway 注入的助手消息形式存储一份纯文本回复作为补充。这导致同一条助手回复在聊天记录中出现两次——一次是原始文本回复，一次是无效媒体补充导致的重复文本消息。用户看到的是凌乱的重复消息，严重影响对话历史的可读性。

**如何解决**:
在处理 assistant-media transcript 补充信息时，先检查媒体引用是否能解析到可播放的媒体。如果解析结果为空（无可用媒体），则跳过该补充信息的注入，避免将纯文本回复再次存储为 gateway 注入的助手消息。这确保 transcript 的完整性和可读性，不会因为媒体失效而产生消息重复。

---

### #159 Sessions: 拒绝向线程范围聊天会话发送代理间消息

**更新原文**:
Sessions: reject `sessions_send` targets that resolve to thread-scoped chat sessions, so inter-agent coordination cannot be injected into active human-facing Slack or Discord threads. Fixes #52496. Thanks @barry-p5cc.

**痛点**:
`sessions_send` API 允许代理之间发送消息进行协调，这是一个强大的功能——但也带来了安全风险。如果目标解析到一个线程范围的聊天会话（如 Slack 或 Discord 中的特定线程），代理间消息会被注入到人类用户正在进行的对话中。恶意或配置错误的子代理可以通过 `sessions_send` 向 Slack/Discord 的活跃线程注入消息，干扰人机交互或泄露信息给线程中的其他参与者。这是一个典型的跨权限边界渗透风险——代理间协调消息本应只在独立的会话通道中流转，不应该"泄漏"到人工交互的线程里。

**如何解决**:
在 `sessions_send` 的目标解析阶段，增加线程范围聊天会话的检测。当目标解析为线程范围的聊天会话时，拒绝该请求并返回错误，确保代理间协调只能发生在独立的会话通道中，不会渗透到 Slack/Discord 等平台的人类用户对话线程。这是一种权限边界加固，将"允许跨代理通信"和"禁止注入人工对话"做了明确区分。

---

### #160 Subagents: 正确处理 expectsCompletionMessage: false

**更新原文**:
Subagents: honor `sessions_spawn` with `expectsCompletionMessage: false` by skipping parent completion handoff delivery while still running child cleanup. Fixes #75848. Thanks @alfredjbclaw.

**痛点**:
`sessions_spawn` API 支持 `expectsCompletionMessage` 参数，允许父代理声明不期望接收子代理的完成消息。这在 fire-and-forget 场景中非常有用——父代理触发子代理执行后台任务后，不需要知道子代理何时完成，继续处理其他工作。但之前的实现存在 bug：即使设置了 `expectsCompletionMessage: false`，系统仍然会向父代理发送完成交接（completion handoff）消息。这些不需要的完成消息会干扰父代理的工作流，造成消息冗余或在某些情况下触发非预期的处理逻辑。

**如何解决**:
当 `expectsCompletionMessage` 为 `false` 时，跳过向父代理发送完成交接消息，但仍然执行子代理的资源清理（cleanup）流程。这将"通知父代理"和"清理子代理资源"两个关注点分离——前者按 `expectsCompletionMessage` 配置决定是否发送，后者无论配置如何都要执行。这是一个逻辑分离修复，让 API 的声明式行为与实际执行一致。

---

### #161 Media/completions: 避免媒体发送后重复 MEDIA: 回退帖子

**更新原文**:
Media/completions: treat media-only message-tool sends as delivered async completion output, avoiding duplicate raw `MEDIA:` fallback posts after video or music generation finishes.

**痛点**:
当代理通过工具生成视频或音乐等媒体内容后，媒体文件通过消息工具发送给用户。但系统同时按照"普通工具输出 → 生成完成回退帖子"的逻辑，将原始 `MEDIA:` 指令再次作为文本帖子发送。这导致用户看到重复内容——一份是正常的媒体文件消息，一份是 `MEDIA:` 标记的文本回退帖子。这不是 agent 重复回复，而是同一个媒体被以两种形式投递了出去，影响用户体验和对话可读性。

**如何解决**:
将纯媒体的消息工具发送标记为"已投递的异步完成输出"（delivered async completion output），系统在识别到这个标记后，跳过后续的 `MEDIA:` fallback 发送，避免重复投递。这解决了视频生成、音乐生成等场景下的重复消息问题。

---

### #162 Gateway/logging: 保留延迟启动日志的时间戳前缀

**更新原文**:
Gateway/logging: keep deferred channel startup logs on the subsystem logger, so Slack, Discord, Telegram, and voice-call startup messages keep timestamped prefixes. Thanks @vincentkoc.

**痛点**:
Slack、Discord、Telegram 和语音通话等频道在延迟启动（deferred startup）时，启动日志没有保留时间戳前缀。这导致运维人员在排查启动顺序和时序问题时缺少关键的时间参照——"Channel X 在 Channel Y 之后启动"这类信息完全丢失，只能靠日志内容推测。延迟启动是常见的——频道服务可能依赖外部服务可用性、重试机制或用户配置延迟加载。现在日志里只剩下裸消息，没有时间线可言。

**如何解决**:
确保延迟频道启动日志路由到子系统 logger（subsystem logger），而非顶层 logger。子系统 logger 保留了带时间戳的前缀格式（`[HH:MM:SS] [subsystem]`），让运维人员能够重建启动时序。

---

### #163 Codex/app-server: 恢复被换行符分割的 JSON-RPC 帧

**更新原文**:
Codex/app-server: recover JSON-RPC frames split by raw command-output newlines and include a redacted preview when malformed app-server messages still reach the console. Thanks @vincentkoc.

**痛点**:
app-server 通过 JSON-RPC 协议与客户端通信。当命令输出（如编译、构建输出）包含未转义的原始换行符时，这些换行符被插入到 JSON-RPC 帧的数据流中，将完整的 JSON-RPC 消息分割成多个片段。接收端无法正确解析完整的 JSON-RPC 消息，导致通信失败或行为异常。同时，当格式错误的消息到达控制台时，开发者无法快速判断消息内容——既不知道完整内容，也不知道是哪种格式错误。

**如何解决**:
两处改动：(1) 恢复被原始命令输出换行符分割的 JSON-RPC 帧——实现帧重组逻辑，能够在收到不完整的 JSON 时进行缓存和拼接，直到收到完整的帧；(2) 当格式仍然错误的消息到达控制台时，显示脱敏预览（redacted preview）——只显示可安全公开的部分，不泄露敏感信息（如文件路径、内部变量值），方便调试。

---

### #164 Replies/typing: 为排队等待消息保持 typing 状态

**更新原文**:
Replies/typing: keep typing alive for queued follow-up messages that are genuinely waiting behind an active run, instead of making chat surfaces look idle while work is queued. Fixes #65685. Thanks @papag00se.

**痛点**:
当用户发送的消息被排在活跃运行之后等待处理时，聊天界面上的"正在输入"（typing indicator）状态消失，界面显示为空闲状态。但实际上工作仍在队列中等待处理，agent 完全清醒。这导致用户体验混乱——用户以为代理没有在处理他们的消息，可能会重复发送消息或认为系统卡住了，增加系统负载和用户挫败感。问题根源是 typing 状态的生命周期管理只在消息实际处理期间保持活跃，一旦进入排队状态就取消了 typing indicator。

**如何解决**:
修改 typing 状态的生命周期管理逻辑：当后续消息真正在活跃运行之后排队等待时，保持 typing indicator 活跃，而非在进入排队状态时立即取消。这让用户知道系统正在处理他们的请求，只是暂时忙碌。修复后，用户界面准确反映了"正在排队处理中"的状态。

---

### #165 ACP/Discord: 抑制内联线程绑定 ACP 会话的重复投递

**更新原文**:
ACP/Discord: suppress completion announce delivery for inline thread-bound ACP session runs, so Discord thread-bound ACP replies are not delivered twice. Fixes #60780. Thanks @solavrc.

**痛点**:
ACP（Agent Communication Protocol）会话在 Discord 线程中运行时，回复被投递两次给用户。根因是存在两条投递路径：ACP 回复已经在线程中直接投递一次，但 completion announce（完成公告）机制又触发了一次额外的投递。两个投递路径产生了重复——用户在自己的 Discord 线程中看到同一条消息出现两次，严重影响使用体验。这是 ACP 与 Discord 线程交互时的投递路径重叠问题。

**如何解决**:
对于内联线程绑定的 ACP 会话运行（inline thread-bound ACP session runs），在 completion announce 投递阶段进行抑制。因为 ACP 回复已经直接投递到 Discord 线程中，不需要额外的完成公告来触发第二次投递。这将"ACP 直接投递"和"completion announce 投递"两个路径做了区分——前者在线程绑定场景下是唯一合法的投递方式，后者应当被抑制。

---

### #166 Discord/threads: 忽略已绑定线程中的 webhook 代理副本

**更新原文**:
Discord/threads: ignore webhook-authored copies in already-bound Discord session threads even when the webhook id differs, preventing PluralKit proxy copies from creating duplicate turn pressure. Fixes #52005. Thanks @acgh213.

**痛点**:
PluralKit 是一个 Discord 代理机器人，允许用户通过 webhook 以不同身份发送消息——它会创建原始消息的 webhook 副本。在已绑定到 OpenClaw 会话的 Discord 线程中，PluralKit 创建的这些消息副本被 OpenClaw 视为新的用户输入，导致每个副本都触发一次代理响应。这是重复轮次压力的来源——用户发一条消息，PluralKit 代理副本被 OpenClaw 当成新消息，agent 就同一个用户输入回复了两次。更糟的是，即使 webhook id 不同也会触发，导致 PluralKit 用户几乎无法正常使用。

**如何解决**:
在已绑定的 Discord 会话线程中，忽略 webhook 授权的消息副本——通过识别消息是 PluralKit 代理副本的特征（即使 webhook ID 不同）来跳过处理。这防止 webhook 副本创建重复的轮次压力，让 PluralKit 用户在已绑定线程中正常使用。

---

### #167 Discord/threads: 线程创建成功但初始消息失败时返回部分成功

**更新原文**:
Discord/threads: return the created thread as partial success when the follow-up initial message fails, so agents do not retry thread creation and create empty duplicate threads. Fixes #48450. Thanks @dahifi.

**痛点**:
代理在 Discord 中创建新线程时，流程分为两步——先创建线程，再在线程中发送初始消息。如果第二步（发送初始消息）失败，系统将整个操作视为失败并触发重试。但线程已经在第一步中成功创建了，重试会创建一个新的空线程。结果是 Discord 频道中出现多个空的重复线程，用户和代理都感到困惑——明明只让 agent 创建了一个线程，为什么出现了两三个？这本质上是分布式系统中"部分成功"的操作被当作"完全失败"处理的典型案例，导致盲目重试。

**如何解决**:
当初始消息发送失败但线程已成功创建时，返回"部分成功"（partial success）状态，包含已创建线程的 ID 等信息。这样代理知道线程已经存在，不会重试创建操作。这是幂等性思维的体现——避免部分成功的操作被盲目重试而产生副作用。

---

### #168 Discord/components: 非可复用组件首次点击后消费

**更新原文**:
Discord/components: consume every button or select in a non-reusable component message after the first authorized click, so single-use panels cannot fire sibling callbacks. Fixes #54227. Thanks @fujiwarakasei.

**痛点**:
Discord 的按钮和下拉选择等交互组件中，有一种"非可复用"（non-reusable）类型，意图是单次使用——点击一次后失效，防止重复操作。但之前的实现存在缺陷：非可复用组件在第一次点击后仍然可以被多次点击，每次点击都会触发回调处理。这导致单次操作面板（如"确认删除"、"执行操作"）可以被重复触发，可能造成数据不一致或重复操作。更糟的是，同一消息中的其他按钮/选择器（sibling callbacks）也会被触发，形成一连串非预期的副作用。这同时也是一个安全问题——用户或恶意方可以通过重复点击绕过单次使用限制。

**如何解决**:
在非可复用的组件消息中，第一次授权点击后，消费（consume）消息中的所有按钮和选择器。消费后的组件被标记为"已使用"，后续点击不再触发任何回调。这是典型的"消费即失效"模式，与 RESTful API 中的 DELETE 语义类似——资源一旦被使用就不能再用。

---

### #169 macOS/config: 回退写入时保留 gateway.auth 认证信息

**更新原文**:
macOS/config: preserve existing `gateway.auth` and unrelated config keys during app fallback writes, so dashboard or Talk settings changes cannot strand Control UI clients by dropping persisted auth. Fixes #75631. Thanks @Fuma2013.

**痛点**:
在 macOS 上，当应用进行"回退写入"（fallback writes）配置时（例如 dashboard 或 Talk 设置变更触发），现有的 `gateway.auth` 认证信息和不相关的配置键会被丢弃。后果是 Control UI 客户端的认证凭据被清除，已认证的客户端被"搁浅"（stranded）无法重新连接，用户需要手动重新配置认证才能恢复访问。这是一个破坏性大于修复性的 bug——一个看似无害的设置变更，却会导致整个系统的远程访问能力瘫痪。根因可能是 fallback writes 执行了完整的配置替换而非增量合并。

**如何解决**:
在 app fallback writes 中，保留现有的 `gateway.auth` 和不相关的配置键，只更新目标配置项。这是典型的增量合并（merge）策略——写入时只更新目标键，不影响其他键。涉及 macOS 配置写入路径的改动。

---

### #170 Control UI/TUI: 重新连接后恢复相同会话

**更新原文**:
Control UI/TUI: keep reconnecting chat sends bound to the same backing session id and let TUI relaunches resume the last selected session, avoiding silent fresh sessions after refresh, reconnect, or terminal restart. Fixes #63195, #68162, and #73546. Thanks @bond260312-cmyk, @zhong18804784882, and @mtuwei.

**痛点**:
用户在 Control UI 或 TUI 中刷新页面、重新连接或重启终端后，系统静默创建全新的会话，而非恢复之前的会话。这导致对话上下文丢失，用户需要手动重新选择之前的会话；更糟糕的是，用户可能没有意识到这一点，继续在新会话中操作而丢失了之前的上下文。三个独立的 issue（#63195、#68162、#73546）都指向同一个核心问题——会话持久性在连接中断后没有被正确恢复，说明这是一个影响面广且长期存在的痛点。

**如何解决**:
两处改动并行：(1) 重新连接时的聊天发送绑定到相同的后端会话 ID，而非创建新会话；(2) TUI 重新启动时恢复上次选择的会话（last selected session）。这确保了刷新、重新连接或终端重启后，用户仍然处于之前的会话上下文中——会话持久性和恢复是核心设计原则。

---

### #171 Plugins/tools: 允许清单声明静态工具可用性

**更新原文**:
Plugins/tools: let plugin manifests declare static tool availability so reply startup skips unavailable plugin tool runtimes instead of importing factories that only return `null`. Thanks @shakkernerd.

**痛点**:
在回复启动阶段，系统会导入所有注册插件的工具工厂（factories），即使某些工具在当前环境下不可用。这些不可用的工厂被导入后只返回 `null`——白白浪费了导入和执行时间，却没有任何实际效果。这与 #152（插件工具工厂缓存）是同一条优化路上的不同环节，都是插件工具准备阶段的性能瓶颈。每个不可用的工厂导入都增加了 per-turn 延迟，累积起来可能造成数十毫秒的浪费。

**如何解决**:
允许插件清单（manifests）在声明时就静态声明工具的可用性。回复启动时，先根据清单中的可用性声明判断是否需要导入，而非先导入再检查是否可用。这样不可用的工具直接被跳过，不再浪费资源导入注定返回 `null` 的工厂。这是一种"声明式可用性"设计——将可用性判断从运行时前置到声明时。

---

### #172 Discord/reactions: 不需要时跳过反应监听器注册

**更新原文**:
Discord/reactions: skip reaction listener registration when DMs and group DMs are disabled and every configured guild has `reactionNotifications: "off"`, avoiding needless reaction-event queue work. Fixes #47516. Thanks @x4v13r1120.

**痛点**:
在 Discord 集成中，即使所有服务器的 `reactionNotifications` 都设置为 `"off"`，并且 DM 和群组 DM 也被禁用，系统仍然注册了反应事件监听器。每个用户反应事件都会进入事件队列并被处理，但最终因为功能已关闭而被丢弃。在活跃的 Discord 服务器中，这会产生大量无意义的队列处理开销，增加事件循环压力，消耗 CPU 和内存资源。这是一个"不管有没有人在听，都照常演奏"的资源浪费问题。

**如何解决**:
在注册反应监听器之前增加条件检查：如果 DM 和群组 DM 被禁用，且所有配置的服务器都关闭了反应通知，则跳过监听器注册，完全不参与事件队列。这是典型的条件性资源分配原则——只有在功能确实被需要时才分配相关资源，否则提前跳过。

---

### #173 CLI sessions: 保留 manual-attach 复用绑定不过期

**更新原文**:
CLI sessions: preserve explicit manual-attach reuse bindings so trusted CLI sessions are not invalidated on the first turn when auth, prompt, or MCP fingerprints drift. Fixes #75849. Thanks @alfredjbclaw.

**痛点**:
用户通过 `manual-attach`（手动附加）建立的受信任 CLI 会话，在第一次交互时可能因为认证（auth）、提示（prompt）或 MCP 指纹（fingerprints）不匹配而被判定为无效进而失效。这些指纹是系统用来识别会话一致性的哈希值，基于认证凭据、提示内容、MCP 配置等——在不同环境下可能自然发生变化（如 MCP 服务器配置微调、环境变量变更等）。用户明确指定了手动附加，受信任的会话理应有更高的持久性，而不是被指纹漂移静默丢弃。这导致用户需要频繁重新认证或创建新会话，manual-attach 的意义被削弱。

**如何解决**:
保留显式的 manual-attach 复用绑定——即使认证、提示或 MCP 指纹发生变化也不失效。因为 manual-attach 是用户的明确授权行为，具有更高的信任级别，不应被指纹的自然漂移所影响。这提升了 manual-attach 的实用性和可靠性。

---

### #174 Telegram/streaming: 普通回复保持预览流式输出

**更新原文**:
Telegram/streaming: keep partial preview streaming enabled for plain reply-to replies, disabling drafts only for real native quote excerpts that require Telegram quote parameters. Fixes #73505. Thanks @choury.

**痛点**:
Telegram 的回复有两种形式：普通回复（plain reply-to）和原生引用摘录（native quote excerpt）。之前的实现对所有回复场景统一禁用了草稿/流式预览，但实际上只有原生引用摘录才需要 Telegram quote 参数并因此禁用流式预览。普通 reply-to 被误伤——流式预览功能在普通回复场景下被关闭，用户无法实时看到正在生成的消息片段，体验下降。这是一个过度保守的错误判断。

**如何解决**:
区分两种回复类型，对普通 reply-to 保持部分预览流式输出，仅在真正使用 Telegram quote 参数的原生引用摘录时才禁用草稿。修复后，普通回复场景下的用户仍然能够享受流式预览的体验，原生引用场景则按预期工作。

---

### #175 Config: 版本警告从每次快照读取改为每进程一次

**更新原文**:
Config: log the "newer OpenClaw" version warning once per process instead of once per config snapshot read. Fixes #75927. Thanks @RomneyDa.

**痛点**:
当配置文件由比当前运行的 OpenClaw 更新的版本创建时，系统会输出"newer OpenClaw"版本警告，提示用户配置文件版本高于当前程序。这是一个善意的提示——但问题在于，这个警告在每次读取配置快照（config snapshot read）时都会触发。配置快照在 Gateway 运行中被频繁读取（每次消息处理、每次配置变更、每次内部查询），导致同一个警告被反复打印到日志中，高频出现的大量重复警告淹没其他重要日志信息，让运维人员苦不堪言。这属于典型的"正确信息但错误频率"问题——该提示一次就够了，不需要每次快照都提醒。

**如何解决**:
将警告日志的触发频率从"每次配置快照读取"降低为"每个进程一次"。实现上使用进程级别的标志位（flag）记录是否已经输出过该警告，后续读取配置时检查标志位，如果已输出过则跳过。这是一种标准的"去重"模式——用额外的状态换区干净的日志输出。

---

### #176 Telegram: 良性删除消息 400 错误降级为警告

**更新原文**:
Telegram/message actions: treat benign delete-message 400s as no-op warnings instead of runtime errors, so stale or already-removed messages do not create noisy delete failures. Fixes #73726. Thanks @Avicennasis.

**痛点**:
当 OpenClaw 尝试删除 Telegram 中的消息时，如果消息已经被删除（已过期、被管理员删除、或超过 Telegram 的 48 小时删除时限），Telegram API 返回 400 错误。系统将此视为运行时错误，产生大量错误日志——但这些是完全可以预期的、良性的失败。消息已经不存在了，删除操作的目标已经达成，只是不是因为这次操作导致的。用户和运维看到的是大量噪音错误日志，干扰真正的问题排查。

**如何解决**:
将这类良性的 400 错误重新分类为"无操作警告"（no-op warning），而非运行时错误。系统记录一条 INFO 级别日志表示"消息已不存在，无需操作"即可，不打印 ERROR，不计入错误计数，不触发告警。

---

### #177 Telegram: 长消息分割为安全 HTML 块

**更新原文**:
Telegram: split long default markdown sends and media follow-up text into safe HTML chunks, so outbound messages over Telegram's limit no longer fail as one oversized Bot API request. Fixes #75868. Thanks @zhengsx.

**痛点**:
Telegram Bot API 对消息长度有限制（约 4096 个字符）。当 OpenClaw 生成的回复超过此限制时，整个 Bot API 请求失败，用户收不到任何内容——长回复完全无法投递。用户可能输入了一个复杂的问题，agent 生成了一个完整的回答，但超过 Telegram 限制后整个消息就丢了，用户只看到一个错误，没有任何部分内容可用。这对用户体验是灾难性的。

**如何解决**:
将长消息按 Telegram 的限制分割成多个安全的 HTML 块。首先将 markdown 转换为 Telegram 安全 HTML 格式（因为 markdown 格式在大段消息中更容易出问题），然后按字符限制分割——确保分割点不会破坏 HTML 标签或格式。分割后的多个消息依次发送，确保所有内容都能最终到达用户。这是一个"化整为零"的消息分段策略。

---

### #178 Gateway/chat history: 合并 Claude CLI transcript 导入

**更新原文**:
Gateway/chat history: merge Claude CLI transcript imports for Anthropic-routed sessions that still have a Claude CLI binding, so local chat history does not hide CLI JSONL turns. Fixes #75850. Thanks @alfredjbclaw.

**痛点**:
OpenClaw 支持 Anthropic 路由的会话，这些会话可能同时与 Claude CLI 共享。但 Gateway 的本地聊天历史功能只显示 OpenClaw 自己记录的轮次，不包含 Claude CLI 通过 JSONL 格式记录的对话轮次。用户在 OpenClaw 的聊天历史中看不到通过 Claude CLI 进行的对话，以为对话丢失了——实际上数据存在于 CLI 的 JSONL 文件中，只是两边没有打通。当用户交替使用 OpenClaw Gateway 和 Claude CLI 与同一个 Anthropic 会话交互时，聊天历史支离破碎，体验割裂。

**如何解决**:
对于仍有 Claude CLI 绑定的 Anthropic 路由会话，合并 Claude CLI 的 transcript 导入。Gateway 读取聊天历史时，会同时加载 Claude CLI 的 JSONL 文件中的对话轮次，合并展示给用户。这样来自两个来源的完整对话都能在本地聊天历史中看到。

---

### #179 Media: 裁剪 MEDIA: 指令路径后的 JSON 后缀

**更新原文**:
Media: trim serialized JSON suffixes after local `MEDIA:` directive file extensions, so generated-image metadata cannot pollute the parsed media path and cause false `ENOENT` delivery failures. Fixes #75182. Thanks @TnzGit and @hclsys.

**痛点**:
当 AI 模型生成图片时，可能在 `MEDIA:` 指令的文件路径后面附加了序列化的 JSON 元数据（例如 `MEDIA:/path/to/image.png{"width":1024,"height":768,...}`）。路径解析器将整个字符串（包括 JSON 后缀）作为文件路径，尝试打开 `image.png{"width":1024,...}` 这个不存在的文件，产生 `ENOENT`（文件不存在）投递失败错误。但实际上图片文件是存在的，只是路径被 JSON 元数据污染了。模型生成的图片无法投递，用户看到的是文件不存在的错误提示，但文件其实就在那里。

**如何解决**:
在解析 `MEDIA:` 指令路径时，增加检测逻辑：找到文件扩展名（如 `.png`）之后的部分，如果是 JSON 序列（以 `{` 开头），则将其裁剪掉，只保留真实的文件路径。这是一个路径解析的健壮性改进，让解析器能够容忍 AI 模型在生成 `MEDIA:` 指令时的格式小差异。

---

### #180 Plugins/runtime: 启用/禁用插件支持热重载

**更新原文**:
Plugins/runtime: hot-reload Gateway plugin runtime surfaces after plugin enable/disable changes while keeping source-changing plugin install, update, and uninstall operations restart-backed so loaded module code is not reused. Fixes #72097.

**痛点**:
之前启用或禁用插件后需要重启 Gateway 才能生效。每次简单的开关操作都需要完整重启 Gateway，影响用户体验和可用性——用户想要快速启用/禁用某个插件测试效果，却被迫等待 Gateway 重启。这是一个效率问题，尤其对需要频繁调整插件配置的用户影响显著。

**如何解决**:
采用双轨策略：(1) 插件启用/禁用操作后，热重载（hot-reload）Gateway 插件运行时界面（runtime surfaces），无需重启；(2) 插件安装、更新和卸载操作（涉及源代码变更）仍需要重启后端，确保已加载的模块代码不会被复用。设计取舍的理由：启用/禁用只是配置变更，现有代码逻辑不变，可以安全热重载；但安装/更新/卸载涉及源代码变更，Node.js 的模块缓存特性决定如果不重启就会复用旧模块代码，可能导致不一致状态、内存泄漏或崩溃。

---

### #181 Cron: 容错处理格式错误的持久化任务

**更新原文**:
Cron: make scheduler reload schedule comparison tolerate malformed persisted jobs, so one bad cron entry no longer aborts the whole tick. Fixes #75886. Thanks @samfox-ai.

**痛点**:
Cron 调度器在重新加载计划并比较新旧计划时，如果持久化的任务列表中包含格式错误（malformed）的条目，整个 tick（调度周期）会被中止。这意味着一个坏条目会阻止所有其他正常的 cron 任务执行——用户精心配置的所有定时任务，因为其中一条格式错误，就全部不执行了。这类错误可能来自手动编辑配置文件、版本升级导致的格式变更、或写入中断导致的文件损坏。问题的严重性在于：一个局部错误具有全局影响。

**如何解决**:
在调度器重新加载并比较计划时，增加对格式错误条目的容错处理逻辑：跳过无法解析的条目而非中止整个 tick。这实现了"优雅降级"（graceful degradation）原则——个别条目的错误不应影响整体服务的可用性。正常条目继续执行，只在日志中报告格式错误条目的问题。

---

### #182 Doctor/channels: 迁移后缺少 token 时发出警告

**更新原文**:
Doctor/channels: warn after migrations when default Telegram or Discord accounts have no configured token and their env fallback (`TELEGRAM_BOT_TOKEN` or `DISCORD_BOT_TOKEN`) is unavailable, with secret-safe migration docs for checking state-dir `.env`. Fixes #74298. Thanks @lolaopenclaw.

**痛点**:
在版本升级或配置迁移后，默认的 Telegram 或 Discord 账号可能丢失 token 配置。系统之前静默忽略这种情况——`openclaw doctor` 不会报警，用户在尝试使用时才发现频道无法工作。迁移可能改变配置格式或存储位置，导致之前配置的 token 不再被识别。更糟的是，Telegram 和 Discord 都支持环境变量回退（`TELEGRAM_BOT_TOKEN` 和 `DISCORD_BOT_TOKEN`），但 doctor 命令在迁移后也不检查这些回退是否可用。用户不知道频道为什么不工作，排查起来费时费力。

**如何解决**:
`openclaw doctor` 命令在迁移后增加对默认账号 token 配置的检查。如果配置中无 token 且环境变量回退也不可用，输出明确警告。同时提供安全的迁移文档（不会暴露实际 token 值），指导用户如何检查 state-dir 中的 `.env` 文件。

---

### #183 Gateway/diagnostics: 空闲活跃度采样仅保留在遥测中

**更新原文**:
Gateway/diagnostics: keep idle liveness samples in telemetry instead of visible warning logs unless diagnostic work is active, waiting, or queued. Thanks @vincentkoc.

**痛点**:
Gateway 的诊断系统在空闲时进行活跃度采样（liveness samples）来确认系统正常运行，但这些采样结果以警告日志（warning logs）形式输出。在 Gateway 空闲期间，大量正常的健康检查采样作为"警告"出现在日志中，淹没真正需要关注的日志信息，高噪信比让运维排查困难。这是"正确数据但错误表达媒介"的问题——遥测数据适合存入指标系统，不应进入日志流。

**如何解决**:
将空闲时的活跃度采样数据仅保留在遥测（telemetry）系统中，不输出可见的警告日志。只有在诊断工作实际处于活跃、等待或排队状态时（有实际问题需要关注），才输出可见的警告日志。这将"正常空闲时的健康采样"和"异常需要关注的状态"做了明确区分，让日志只反映真正值得关注的信号。

---

### #184 Channels/cron: 拒绝跨频道前缀不匹配的目标

**更新原文**:
Channels/cron: reject provider-prefixed targets for the wrong channel and let prefixed announce targets such as `telegram:123` select their channel when delivery falls back to `last`, so Telegram IDs cannot be coerced into WhatsApp phone numbers. Fixes #56839. Thanks @bencoremans.

**痛点**:
cron 任务和频道投递支持使用前缀来指定目标频道，如 `telegram:123`。但当投递回退到 `last`（最后活跃的频道）时，如果最后活跃的频道与前缀不匹配（如回退到 WhatsApp），Telegram 的数字 ID `123` 可能被错误地解释为 WhatsApp 的电话号码，导致消息被发送到完全错误的接收者。这是一个跨频道 ID 串联（ID coercion）安全问题——Telegram ID 在 WhatsApp 渠道中被误解释，可能导致私密消息被发送到错误的电话号码。这是严重的安全和隐私问题。

**如何解决**:
两处改动并行：(1) 当频道前缀与目标频道不匹配时，拒绝该投递而非尝试回退；(2) 允许带前缀的公告目标（如 `telegram:123`）在回退时直接选择正确的频道，而非被强制转换。这确保 Telegram ID 只在 Telegram 频道中使用，不会被错误解释为其他频道的标识符。

---

### #185 Control UI/chat: 会话别名发送时保持实时回复可见

**更新原文**:
Control UI/chat: keep live replies visible when a raw session alias such as `main` sends the chat turn but Gateway emits events under the canonical session key for the same run. Fixes #73716. Thanks @teebes.

**痛点**:
用户可以通过会话别名（如 `main`）发送消息，但 Gateway 在处理时会使用规范会话键（canonical session key）来发出事件。Control UI 在匹配实时回复时，使用的是别名而非规范键——导致事件无法匹配，用户看不到正在生成中的实时回复。问题在于：用户说"发送消息给 main"，Control UI 认为这是给 `main` 的消息，但 Gateway 内部处理时换成了规范键 `agent:main:main`，Control UI 收到事件后不知道该事件属于哪个别名会话，实时回复就被吃掉了。用户看到的不是"正在输入"的实时流，而是突然出现一条完整回复。

**如何解决**:
Control UI 在接收实时回复事件时，增加别名到规范键的映射解析。当通过别名发送的轮次收到实时事件时，先将事件的规范键转换为用户可识别的别名，确保通过别名发送的轮次也能正确显示实时回复。这是 UI 层的路由匹配问题，需要维护别名→规范键的映射表。

---

### #186 CLI/models: 拒绝 set 和 set-image 命令的 --agent 参数

**更新原文**:
CLI/models: reject `--agent` on `openclaw models set` and `set-image` instead of silently writing agent-scoped requests to global model defaults. Fixes #68391. Thanks @derrickabellard.

**痛点**:
用户在 `openclaw models set` 或 `set-image` 命令中使用 `--agent` 参数，意图为特定代理设置模型，但系统静默忽略了 `--agent` 参数，将请求写入全局模型默认配置。用户以为在配置特定代理的模型，实际上修改了全局默认值，所有代理和会话都受到影响。这是一个静默数据破坏——配置变了，但没人告诉你。更糟的是，用户需要排查为什么某个代理行为异常时，很难联想到是全局模型配置被意外修改了。

**如何解决**:
在 `openclaw models set` 和 `set-image` 命令中增加参数验证：当用户传入 `--agent` 参数时，明确拒绝该操作并返回错误提示，而非静默忽略。用户会收到明确的错误消息，知道需要使用其他方式来配置代理范围的模型。这将静默破坏改为明确报错，让错误对用户可见。

---

### #187 CLI: 不再将旧版 tool 命令当作插件 ID

**更新原文**:
CLI: stop treating the legacy singular `openclaw tool ...` token as a plugin id under restrictive `plugins.allow`, so it falls through as a normal unknown/reserved command instead of suggesting a stale allowlist entry. Fixes #64732. Thanks @efe-arv, @SweetSophia, and @hashtag1974.

**痛点**:
OpenClaw 之前有单数形式的 `openclaw tool ...` 命令（旧版），后来改为了 `openclaw tools ...`（复数）。旧版命令 token `tool` 在 `plugins.allow`（受限插件白名单）模式下被错误地当作插件 ID 处理。系统认为用户在尝试访问名为 `tool` 的插件，错误消息建议用户将 `tool` 添加到白名单——但 `tool` 是一个旧版保留命令，根本不是插件。3 位用户报告了这个问题，说明影响面较广，误导性的错误消息让用户做无效的配置尝试。

**如何解决**:
不再将旧版单数 `openclaw tool ...` 的 `tool` token 作为插件 ID 处理。让它作为普通的未知/保留命令穿透（fall through）到标准命令处理流程，而非路由到插件白名单查询逻辑中。这清理了遗留命令迁移中的兼容性问题，让旧命令给出正确的"未知命令"提示，而非误导性的"添加到白名单"建议。

---

### #188 Media: 使用同目录临时文件写入避免零字节残留

**更新原文**:
Media: write inbound media buffers through same-directory temp files before rename, so failed disk writes do not leave zero-byte artifacts for later voice transcription. Fixes #55966. Thanks @OpenCodeEngineer.

**痛点**:
入站媒体缓冲区（如语音消息）在写入磁盘时，如果写入过程中出现错误（磁盘满、权限问题、IO 中断等），目标文件位置会留下一个零字节的空文件。后续的语音转录（voice transcription）尝试处理这个空文件，转录失败——但用户不知道是文件损坏，只看到转录失败。这是一个"部分写入→文件损坏→后续功能全部失败"的级联故障。更糟的是，零字节文件不会报错，只在转录输出为空时才会被发现。

**如何解决**:
采用经典的"写临时文件再重命名"模式（write-to-temp-then-rename）：先在目标文件的同一目录写入临时文件，写入成功后才将临时文件原子重命名（atomic rename）为目标文件。同一目录确保 rename 操作在同一文件系统上，可以保证原子性。写入失败时，临时文件被自动清理，不会残留在目标路径上。这样既避免了部分写入产生零字节文件，又确保了磁盘故障不会导致空文件残留影响后续处理流程。

---

### #189 TTS/Telegram: 受信任本地音频始终加入语音消息投递队列

**更新原文**:
TTS/Telegram: keep trusted local audio generated by the TTS tool queued for voice-note delivery even when the run-level built-in tool list omits the raw `tts` name. Fixes #74752. Thanks @Loveworld3033 and @andyliu.

**痛点**:
TTS 工具生成的本地音频文件应该以语音消息（voice-note）形式投递到 Telegram。但当运行级别的内置工具列表中不包含原始的 `tts` 名称时（可能因为工具名被别名或映射覆盖），音频文件不会进入语音消息投递队列——系统认为 TTS 功能未启用。用户期待听到的语音消息变成了文本或其他形式，或者完全无法送达。这是一个功能可用性问题，影响用户对 TTS 语音功能的正常使用。

**如何解决**:
不再仅依赖运行级别内置工具列表中是否存在 `tts` 名称来判断是否进行语音消息投递。对于受信任的本地 TTS 音频文件（由 OpenClaw 内置 TTS 工具在本地生成，非外部来源，可以安全投递），始终加入语音消息投递队列。这将"工具名存在性检查"改为"音频文件来源信任性检查"，确保受信任的本地 TTS 音频能正确投递为语音消息。

---

### #190 TTS: 语音工具需要明确用户意图或配置激活

**更新原文**:
TTS: require explicit user or config audio intent for the agent speech tool so dashboard chats stay text unless audio is requested. Fixes #69777. Thanks @alexandre-leng.

**痛点**:
代理的语音工具（speech tool）在 Dashboard 聊天中默认生成音频回复，即使用户没有请求音频。这导致 Dashboard 聊天默认输出语音而非更常见的文本回复，用户不期望地听到语音播放。这违反了"最小惊讶原则"——大多数文字聊天用户不需要语音，且 TTS API 调用有成本。用户的默认体验应该是文字，只有明确需要语音时才生成语音。

**如何解决**:
引入明确的音频意图（audio intent）要求：只有当用户明确请求音频或配置中启用了音频功能时，代理才会调用 speech tool 生成语音。Dashboard 聊天默认保持纯文本交互，语音功能需要用户主动激活。这是一个典型的"opt-in vs opt-out"设计决策——把语音当作特殊功能而非默认行为。

---

### #191 Plugins/config: 捆绑源码检出插件不受运行时门控限制

**更新原文**:
Plugins/config: keep bundled source-checkout plugins from being runtime-gated by install-only `minHostVersion` metadata, accept prerelease host floors, trim plugin-service startup failures to one log line, and avoid broad channel-runtime loading during base config parsing. Thanks @vincentkoc.

**痛点**:
这里涉及四个独立的插件配置问题：(1) `minHostVersion` 元数据本应用于限制第三方插件的最低运行环境，但内置捆绑插件也被错误地门控，导致源码检出的内置插件被意外禁用；(2) 预发布版本（beta、rc）的 OpenClaw 可能不满足 `minHostVersion` 的版本比较逻辑，导致预发布版本用户无法使用某些插件；(3) 插件服务启动失败时产生大量重复日志，干扰排查；(4) 基础配置解析阶段不应该加载频道运行时，但实际发生了，导致不必要的依赖初始化。

**如何解决**:
四管齐下：(1) `minHostVersion` 检查只应用于安装的第三方插件，内置/捆绑插件跳过此检查；(2) 版本比较接受预发布版本标识符；(3) 插件服务启动失败日志精简为一行；(4) 延迟频道运行时加载到真正需要时，避免基础配置解析时加载。

---

### #192 Heartbeat mode: 防止工具调用输出泄露到用户消息

**更新原文**:
(关联 Issue #54138) Heartbeat mode 的心跳处理程序运行时，嵌入 agent 输出的 `[TOOL_CALL]` 格式工具调用结果（如 `exec`、`cron list` 等）会泄露到面向用户的 Telegram/Discord 消息中，而非被静默丢弃。

**痛点**:
当心跳处理程序（heartbeat handler）执行工具调用（如 `exec`、`cron list`）时，嵌入 agent 的输出中包含 `[TOOL_CALL]{tool: "exec", args: {...}}[/TOOL_CALL]` 格式的原始工具调用结果。这些结果本应只作为内部观察注入回模型，不应出现在用户可见的对话中。但由于心跳模式下的 `buildReplyPayloads` 函数跳过了所有文本过滤和清理，这些 bracket-format 工具调用块直接流入 `replyPayload.text`，被投递到 Telegram/Discord。用户在聊天中看到的是原始的工具调用语法，而非 agent 应该给出的正常回复。

**如何解决**:
在 `src/agent-runner-payloads.ts` 的 `buildReplyPayloads` 函数中，为 `isHeartbeat=true` 的分支添加与普通消息分支相同的文本过滤逻辑：对 `[TOOL_CALL]...[/TOOL_CALL]` 格式的块进行剥离（strip），确保心跳模式下的工具执行结果不会泄露到用户可见的回复中。之前的 PR #42260 修复了 XML 格式（`<tool_call>`），但没有覆盖 bracket-format（`[TOOL_CALL]`），本次修复填补了这个空白。

---

### #193 Voice wake mode: 修复 macOS app 中语音回复路由目标

**更新原文**:
(关联 Issue #51040) macOS app 的语音唤醒模式下，回复始终路由到主 webchat 会话，而非用户在下拉菜单中选择的会话目标（如 Telegram group）。

**痛点**:
在 macOS app 中使用语音唤醒（voice wake）模式时，用户已经通过下拉菜单选择了特定的会话目标（如某个 Telegram group），但语音唤醒的回复仍然路由到主 webchat 会话，完全忽略了用户的选择。这意味着用户对着手机说"帮我查一下..."，期望得到的结果发到 Telegram group，但实际出现在 webchat 中。用户需要手动切换会话并移动消息，语音唤醒的"即说即说"体验被破坏。这是一个 UX 路由问题——系统忽略了用户明确的会话选择。

**如何解决**:
在 macOS app 的 voice wake 路由逻辑中，读取用户在 session picker 下拉菜单中的选择，将语音唤醒响应路由到用户指定的会话目标，而非硬编码到主 webchat 会话。这需要在 voice wake 触发时读取当前选中的 session key，并将其传递到响应路由层。

---

### #194 Web search/Grok: 修复 xAI Responses API 超时问题

**更新原文**:
(关联 Issue #58063, #58733) Grok web_search 工具在 2026.3.28 更新后立即中止，错误信息为 "This operation was aborted"；Grok web_search 的 xAI Responses 路径使用了过短的 30 秒默认超时。

**痛点**:
2026.3.28 更新后，Grok web_search 工具在配置了 `tools.web.search.provider: grok` 时立即中止，没有任何超时也没有错误日志。根因分析揭示了两个相关问题：(1) xAI Responses API 路径的超时过短——只有 30 秒，而实际 xAI web search 在处理复杂查询时可能需要 40-60 秒；(2) 在 2026.3.25 的 xAI Responses API 迁移中，web_search 工具的代码路径可能发生了变更，30 秒超时对于某些 Grok web searches 来说不够长。用户尝试使用 Grok 搜索时，请求在 30 秒左右被中止，但同样的查询在 60 秒超时下可以正常完成。

**如何解决**:
将 Grok web_search 的 xAI Responses API 路径的默认超时从 30 秒调整为 60 秒（或根据实际 xAI web search 延迟数据设置更合理的默认值）。这是一个简单的超时配置调整，但需要准确理解 xAI Responses API 的实际延迟特性。同时需要确保 web_search 工具在 API 路径变更时不会静默失败，能够正确报告超时而非立即中止。

---

### #195 Config/onboard: 不再无条件覆盖主模型提供商

**更新原文**:
(关联 Issue #50268) 通过 onboard/configure 添加第二个提供商时，无条件覆盖 `agents.defaults.model.primary`，导致现有提供商（如 Bedrock）被破坏。

**痛点**:
当用户已经配置了 Bedrock 作为主模型提供商（使用 `auth: "aws-sdk"` 通过实例角色或环境变量配置，没有显式的 API key 在配置中），然后通过 `openclaw onboard` 或 `openclaw configure` 添加第二个提供商（如 xAI），onboard 流程会无条件调用 `applyAgentDefaultModelPrimary`，将主模型覆盖为新提供商的默认模型。结果是 Bedrock 的主提供商配置被覆盖为 xAI/grok，而 xAI 的 API key 可能还没有配置——导致 gateway 启动时就处于损坏状态。这创造了"自锁失败"（self-locking failure）：用户无法使用 AI 辅助恢复，因为 gateway 无法启动来修复配置，只能手动编辑 `openclaw.json`。

**如何解决**:
在 `applyXxxConfig` 函数中增加守卫逻辑：只有当初始主模型为空（或这是初始设置）时才覆盖 `agents.defaults.model.primary`。如果用户已经有一个工作的主模型配置，则保留现有配置，不做覆盖。同时在 onboard 流程中增加对"是否已有工作主模型"的检测，如果已有主模型则提示用户明确确认是否要更换主模型。这将盲目覆盖改为有条件的保留策略。

---

### #196 Slack: 修复 DM 历史上下文和多账户路由问题

**更新原文**:
(关联 Issue #64427, #58832, #62042, #41608, #41264 等) 修复 Slack DM 会话历史缺失、线程会话键错误、多账户路由混乱等多个问题。

**痛点**:
Slack 集成在本次更新中修复了多个长期存在的问题，涵盖多个独立 issue：(1) `dmHistoryLimit` 配置项虽然有文档但未实现，导致 DM 会话每次从零历史开始，跨 session 边界的对话完全丢失；(2) DM 一级消息错误地获得 `:thread:` 会话键后缀，导致每个 DM 消息都创建了隔离会话；(3) 多账户设置下，CLI 发送的消息被路由到错误的账户；(4) Slack 频道绑定在会话键构建时被忽略，所有消息都路由到 `agent:main`；(5) 公共频道的 `@ceo` 等人号 mention 无法启动对应 persona 的会话。这些问题单独看都会严重影响 Slack 的使用体验。

**如何解决**:
(1) 实现 `dmHistoryLimit`：在 DM 会话初始化时，通过 `conversations.history` API 获取最近 N 条消息；(2) 修复 DM 的会话键构建逻辑，不为顶级 DM 添加 `:thread:` 后缀；(3) 在 `message-action-runner.ts` 中实现端到端感知的多账户绑定查找；(4) 在会话键构建时正确咨询绑定配置；(5) 在 Slack mention preflight 中添加对 `<!subteam^...>` token 的处理和相关配置项。涉及 Slack 插件的多个文件和模块。

---

### #197 Slack/Delivery: 保留 Slack API 错误结构化字段

**更新原文**:
(关联 Issue #62391, #44625, #68789 等) 修复 Slack 投递队列恢复时丢失 Slack API 错误详情的问题。

**痛点**:
`recoverPendingDeliveries` 在 `delivery-queue.ts` 中使用 `err.message` 将 Slack API 错误字符串化，丢弃了 Slack API 错误中的结构化字段（`err.data.needed`、`err.data.response_metadata.scopes`）。这些字段标识了具体缺少哪个 OAuth scope。当用户遇到 `missing_scope` 投递失败时，日志和队列条目只显示 `"An API error occurred: missing_scope"`，没有任何关于需要添加哪个 scope 的可操作信息。59 个队列条目全部显示同样的错误，用户无法诊断具体需要什么 scope，只能手动测试 API。这是一个典型的基础设施 bug——功能正常但诊断能力缺失。

**如何解决**:
在 `delivery-queue.ts` 和 `deliver.ts` 的错误处理中，不仅记录 `err.message`，还要记录 Slack 的结构化错误字段（`err.data.error`、`err.data.needed`、`err.data.response_metadata.scopes`），将完整信息保留在队列条目和恢复日志中。日志格式变为：`"An API error occurred: missing_scope (needed: im:write, granted: chat:write, users:read)"`，提供可操作的诊断信息。

---

### #198 Slack: 修复文件附件、user-group mention 和线程消息获取问题

**更新原文**:
(关联 Issue #51458, #64625, #73827, #53943 等) 修复 Slack 交互式消息中文件附件被拒绝、user-group mention 无法唤醒 agent、无法获取线程中特定消息等问题。

**痛点**:
这里涉及 Slack 集成的多个独立问题：(1) Slack `send` 消息操作在存在交互元素（buttons、selects 等）时拒绝文件附件——因为 `blocks + media` 在 Slack API 中是非法组合；(2) 代理文件上传（`upload-file`）在 agent workspace 目录内的文件时失败，因为自定义 invoke 回调丢弃了 `mediaLocalRoots`；(3) `<!subteam^SXXX>` 格式的 Slack user-group mention 无法唤醒任何属于该 group 的 agent，因为 mention preflight 只识别 `<@UXXX>` 格式；(4) `message` 工具的 `read` action 无法获取线程中的特定消息，`messageId` 参数被忽略。这些问题影响了 Slack 集成的核心交互功能。

**如何解决**:
(1) 文件附件问题：采用 `replies.ts` 中的双消息模式——先发送媒体文件，再发送交互元素；(2) `upload-file` 问题：在 invoke 回调前将 `mediaLocalRoots` 和 `mediaReadFile` 注入到 toolContext 中；(3) user-group mention：引入 `groupChat.subteamMentions: string[]` 配置项，agent 自行声明哪些 subteam 应该唤醒它；(4) 线程消息获取：添加 `messageId` 参数支持到 `read` action。

---

### #199 Tools/Gemini: 将 API Key 从 URL 参数移到 Header

**更新原文**:
(关联 PR #60600) 将 Gemini PDF 原生 provider 的 API Key 从 URL query 参数 `?key=` 移到 `x-goog-api-key` header。

**痛点**:
`pdf-native-providers.ts` 将 Gemini API key 作为 URL query 参数 `?key=` 传递，这与代码库中其他所有 Google API 调用使用的 `x-goog-api-key` header 方式不一致。API key 在 URL 中会泄露到 HTTP 代理日志、CDN 访问日志和任何记录请求 URL 的中间设备。代码库自己的 `redact-sensitive-url.ts` 将 `key` 标记为敏感 query 参数，说明团队已经意识到这个风险。这是一个安全隐患——生产流量中的 API key 不应该出现在可被日志收集的 URL 字符串中。

**如何解决**:
将 API key 从 URL query 参数移到 `x-goog-api-key` header，与代码库其他地方保持一致。Google 的 `generateContent` 端点接受两种认证方式，行为完全相同，但 header 方式更安全——代理和 CDN 日志只记录路径，不记录敏感的认证凭证。这是一个纯安全修复，无功能性变更。

---

### #200 Web search/Gemini: 修复静默中止、SecretRef 解析、freshness 参数等问题

**更新原文**:
(关联 Issue #72995, #75420, #66498, #65862, #65870, #74915 等) 修复 Gemini web_search 的多个问题：静默中止、SecretRef 解析失败、freshness 参数被忽略、DuckDuckGo 未出现在 onboarding 向导、Brave Search docs URL 指向旧路径、外部插件 web fetch provider 无法解析等。

**痛点**:
这里涉及 Gemini web_search 和相关 web search 提供商的多个独立问题：(1) Gemini provider 的 `web_search` 在 2026.4.23 后间歇性即时中止（不到 1 秒），无错误日志，工具层吞掉了 abort 错误；(2) 2026.4.29 后 web_search 可收到未解析的 SecretRef，即使 `openclaw secrets audit` 报告 `unresolved=0`；(3) `freshness` 参数对 Gemini provider 静默忽略——Brave 和 Perplexity 都支持，但 Gemini 原生支持却被 wrapper 丢弃；(4) DuckDuckGo 因缺少 `onboardingScopes` 字段被 onboarding 向导过滤；(5) Brave Search 代码中的 `docsUrl` 指向 legacy 路径；(6) 外部/非捆绑插件声明的 `webFetchProviders` 在运行时无法被 `web_fetch` 解析，因为 provider lookup 限制为捆绑插件。

**如何解决**:
(1) 修复 session-level `runAbortController` 的竞态条件，确保 abort controller 不在搜索完成前触发；(2) 修复 web_search 执行路径读取原始/源配置而非已解析运行时快照的问题，确保使用已解析的 active secrets；(3) 将 `freshness` 参数传递给 Gemini 的 Google Search Grounding API；(4) 在 DuckDuckGo provider 定义中添加 `onboardingScopes: ["text-inference"]`；(5) 更新 Brave Search 的 `docsUrl` 到 canonical `/tools/brave-search` 路径；(6) 修复 `resolvePluginWebFetchProviders` 的 `bundledAllowlistCompat: true` 限制，让外部插件的 web fetch provider 能被正确解析。

---

### #201 Slack: 修复目录读取返回空结果问题

**更新原文**:
(关联 Issue #50776) Slack 目录读取在 token 验证成功的情况下返回空结果 `groups: []`、`self: null`、`peers: []`。

**痛点**:
OpenClaw 连接 Slack workspace 成功，bot/app/user token 都验证通过，直接 Slack API 调用（`auth.test`、`users.info`、`users.profile.get`）都返回 ok，但 Slack 目录命令仍然返回空结果：
- `openclaw directory groups list --channel slack` → `[]`
- `openclaw directory self --channel slack` → `null`
- `openclaw directory peers list --channel slack` → `[]`

这说明问题不在 token 认证，而在目录读取的实现本身。用户配置一切正常，却无法获取 Slack 目录信息，影响依赖目录功能的自动化工作流。160 字符的 SMS PDU 限制提示可能与 Slack 的 DM 处理有关联。

**如何解决**:
修复 Slack 目录读取的 bug——问题可能在于 DM 事件使用 `text` 字段（Slack 用作通知预览，可能有截断）而非 `blocks` 载荷（包含完整内容）。需要在 Slack 消息处理中读取完整内容而非截断的预览文本。Channel 消息使用正确路径，DM 事件没有。

---

### #202 Slack: 修复 typing indicator 和状态反应在 message_tool 模式下被静默禁用

**更新原文**:
(关联 Issue #75877) 在 `messages.groupChat.visibleReplies: "message_tool"`（v2026.4.27+ 的默认配置）下，Slack 频道处理器结构性禁用了 typing indicator、typingReaction emoji、live preview 和 status reactions。

**痛点**:
在 `messages.groupChat.visibleReplies: "message_tool"` 配置下（这是 v2026.4.27+ 的默认推荐配置），Slack 渠道的处理逻辑结构性禁用了以下功能：
- Slack assistant thread status（`setSlackThreadStatus({status: "is typing..."})`）
- `channels.slack.typingReaction` emoji（如 `:hourglass_flowing_sand:`）
- Live preview / native streaming
- Status reactions（`messages.statusReactions`）

文档明确说明："Typing indicators in groups follow `agents.defaults.typingMode`. When visible replies use the default message-tool-only mode, typing starts immediately by default so group members can see the agent is working..."

但实际行为是：在推荐/默认的工具模式下，群组成员完全得不到任何运行中的反馈——没有 typing indicator，没有 reaction，没有 preview。回复在数秒后才出现，用户完全不知道 agent 正在工作。这是一个有据可查的 UX 回归，影响依赖 typing indicator 的团队。

**如何解决**:
在 `extensions/slack/src/monitor/message-handler/dispatch.ts` 中，移除对 `sourceRepliesAreToolOnly` 的结构性依赖条件。typing indicator、status reactions、preview streaming 应该在 `message_tool` 模式下也能工作，只是最终回复的投递方式不同。修改 `statusReactionsEnabled`、`typing` controller 和 `previewStreamingEnabled` 的条件判断，允许 tool-only 模式下也启用这些反馈机制。

---

### #203 Slack: 修复 DM 消息在 160 字符处被静默截断

**更新原文**:
(关联 Issue #55358) Slack DM 消息超过 160 字符后被静默截断，每次截断都精确发生在第 160 个字符处。

**痛点**:
用户发送超过 160 字符的 Slack DM 时，agent 收到的消息被精确截断在第 160 个字符处。没有错误日志，agent 处理的是截断后的不完整消息。160 是标准 SMS PDU 字符限制——暗示代码某处有一个硬编码的 160 字符上限。Channel 消息不受影响（同一个 Slack app、同一天、206+ 字符的 channel 消息完好无损），Web UI 也不受影响。只有 DM 的 inbound 消息被截断。

推测的根因：Slack DM 事件从 `text` 字段读取（Slack 用作通知预览，可能有截断）而非 `blocks` 载荷（包含完整内容）。Channel 事件使用了正确的路径，DM 事件没有。

**如何解决**:
在 Slack DM 消息处理中，改为从 `blocks` 载荷读取完整内容，而非 `text` 字段的预览文本。确保 DM 事件使用与 channel 事件相同的完整内容读取路径，避免 160 字符的 SMS PDU 限制被误应用到 Slack DM 消息处理。

---

### #204 WhatsApp: 修复工具调用结果显示原始语法而非实际结果

**更新原文**:
(关联 Issue #63610) WhatsApp 渠道显示原始工具调用语法 `[TOOL_CALL]{tool => "web_search", args => {...}}[/TOOL_CALL]` 而非实际工具结果。

**痛点**:
当模型在 WhatsApp 渠道中调用工具（如 `web_search`）时，工具执行成功并返回真实数据（如 Tavily 搜索结果），但用户看到的是原始的工具调用语法 `[TOOL_CALL]{tool => "web_search", args => {...}}[/TOOL_CALL]`，而非工具的实际结果。会话日志显示工具执行和 `toolResult` 都正常，问题在于 WhatsApp 渠道的消息渲染层没有正确处理工具执行结果。同样的模型在 main agent 中工作正常，WhatsApp 渠道显示了原始 tool call 语法。这是一个 WhatsApp 特有的响应渲染 bug。

**如何解决**:
检查 WhatsApp 渠道的响应渲染逻辑，对比 Telegram 等其他渠道如何处理工具执行结果。问题可能出在 streaming mode 或响应格式化的特定处理上。WhatsApp 渠道需要将工具执行结果正确渲染为用户可读的消息，而非暴露内部工具调用语法。

---

### #205 WhatsApp: 修复 listener close() 调用导致僵尸 socket

**更新原文**:
(关联 Issue #52442) WhatsApp listener 的 `close()` 函数调用 `sock.ws?.close()` 而非 `sock.end()`，导致僵尸 socket。

**痛点**:
WhatsApp listener 的 `close()` 函数在 `extensions/whatsapp/src/inbound/monitor.ts:481` 调用 `sock.ws?.close()` 而非 `sock.end()`。这绕过了 Baileys 的内部清理流程，导致僵尸 socket——状态报告"connected"但无法接收入站消息。`sock.end()` 执行关键清理：停止 keepalive ping timer、停止 QR timeout、移除 WebSocket listeners、关闭 WebSocket、通知下游、移除事件监听器。`sock.ws?.close()` 只做最后一步，其他全部泄露。

后果严重：每次重连循环后，keepalive interval 继续 ping 已死 socket，4+ 个事件监听器泄露，Baileys 永远不会转换到 "closed" 状态，Signal protocol session 变得过时，入站解密静默失败，health monitor 检测到"stale"后重启，创建另一个僵尸 socket，进入无限循环。用户报告的生产实例：隔夜 14 次 health-monitor 重启，5.5 小时零消息投递（尽管状态显示"connected"），最终状态 440（session conflict）并被迫重新配对。

**如何解决**:
将 `sock.ws?.close()` 改为 `sock.end(new Error("OpenClaw listener close"))`（并为兼容性保留对 `sock.end` 不存在时的回退检查）。`sock.end` 是 Baileys socket 的公开 API，调用它执行完整的清理流程。此外，考虑更新 Baileys 依赖版本，因为 v7.0.0-rc.9 也有确认的 bug #2132（"Connection shows Online but is disconnected"），已在 PR #2264 中修复。

---

### #206 (无关联 PR)

**更新日志条目**: CLI/ssh: verify host availability and selection preferences before prompting to start or connect, so `openclaw connect` skips prompting when the preferred host is unavailable.

本条更新无关联 PR/Issue 编号，无法获取更多详情。从更新日志描述推测：这是一项 CLI connect 命令的改进，在提示用户启动或连接之前验证主机可用性和选择偏好——当首选主机不可用时跳过提示直接处理。

---

### #207 (关联 Issue #75656)

本条更新关联 Issue #75656，但 GitHub API 额度已耗尽，无法获取详细内容。Issue 编号提示这是 2026 年中期的 issue，具体问题描述和解决方案无法确认。

---

### #208 Talk Mode: 修复多声道音频接口的 PTT 静默失败

**更新原文**:
(关联 Issue #42533) Talk Mode PTT 在多声道音频接口上产生 `len=0`，SFSpeechRecognizer 静默失败。

**痛点**:
当系统音频输入设备有超过 2 个声道时（如 PreSonus Quantum、Focusrite 18i20、MOTU 等专业音频接口），Talk Mode 的 Push-to-Talk 功能总是返回 `emptyOnRelease len=0`——没有转写，没有错误日志，SFSpeechRecognizer 根本从未触发。所有专业音频接口用户都受影响，Talk Mode 和 Voice Wake PTT 完全不可用。用户被迫在 VM 里运行桌面应用作为 workaround。更糟的是，Chrome Web Speech API 在同样的多声道设备上处理正常，说明硬件本身没问题，问题在于 OpenClaw 的音频处理管线无法处理多声道格式。

**如何解决**:
在音频处理管线中添加单声道 downmix（通过 AVAudioMixerNode）。将多声道输入混音为单声道，供 SFSpeechRecognizer 使用。这是 macOS 音频处理的常见模式，在专业声卡上录制 mono 时通常需要这样的处理。

---

### #209 Talk Mode: 修复用户语音转写不显示在 WebChat/Control UI 中

**更新原文**:
(关联 Issue #75155) Talk Mode 中用户的语音转写文本不显示在 WebChat/Control UI 线程中。

**痛点**:
在 macOS 上使用 Talk Mode 时，用户的语音被正确转写并发送到模型（assistant 正确回复，文本和 TTS 都正常），但用户侧的对话从不显示在 WebChat/Control UI 聊天窗格中——只有 assistant 回复出现。用户的语音/转写消息在 UI 中完全不可见，即使它们确实到达了模型并触发了回复。用户无法在聊天历史中看到自己说了什么，无法回顾语音输入的内容，UI 和实际对话内容严重脱节。另外，assistant 回复有时会在屏幕上重复出现——可能是聊天文本加上 TTS 转录 echo 被渲染了两次。

**如何解决**:
将 Talk overlay 中来自 `chat.send` 的消息当作用户键入的消息一样渲染在聊天窗格中。用户的语音转写应该与普通键入输入一样出现在聊天历史中。可能需要添加 `talk.echoTranscript` 样式设置让用户选择是否显示。重复渲染问题需要单独调查和修复。

---

### #210 macOS companion: 修复 Voice Wake 失败问题

**更新原文**:
(关联 Issue #64986) macOS companion app Voice Wake 失败，包括内置测试也无法通过。

**痛点**:
macOS companion app（OpenClaw Mac app）的 Voice Wake 功能完全失效——wake word 不触发，内置的 "Test Voice Wake" 测试也失败。用户已验证：麦克风和语音识别权限都已授予，Voice Wake 已启用，trigger words 存在（`openclaw`、`MAIA`、`computer`），locale 设为 `en-US`，但功能就是不行。更糟的是，测试期间没有有用的 Voice Wake 运行时日志，无法诊断具体哪里出了问题。用户报告 Voice Wake 在这个版本之前可能是正常的，看起来是 2026.4.10 的回归 bug。Piper TTS 本地运行正常，但 Voice Wake 失败说明问题独立于 TTS 提供商。

**如何解决**:
检查 macOS companion app 中 Voice Wake 的运行时日志问题——为什么测试时没有日志输出。可能是权限检查、TCC（Transparency, Consent, and Control）配置、locale 字段处理或语音识别初始化的问题。Voice Wake 的 locale 字段需要正确初始化，即使在设置中未显式配置也要有默认值。修复后 Voice Wake 测试应该能够成功触发并记录日志。

---

### #211 TTS: 修复 cron announce 投递中 TTS 标签未被处理

**更新原文**:
(关联 Issue #52125) cron announce 投递中 `[[tts]]` 和 `[[tts:text]]` 标签不被处理，在 Telegram 中显示为原始文本而非生成语音笔记。

**痛点**:
用户配置了 TTS（`messages.tts.auto: "tagged"`），在 cron job payload 中让 agent 包含 `[[tts]]` 标签。直接回复时 TTS 正常工作，语音笔记正确附加；但通过 cron job `announce` 模式投递时，`[[tts]]` 标签显示为原始文本，没有生成音频。这破坏了定时早晨简报等使用场景——用户期望收到语音摘要，实际只收到带标签的原始文本。用户需要手动从 cron job 中移除 TTS 指令作为 workaround。

**如何解决**:
让 cron announce 投递路径经过与直接 agent 回复相同的 TTS 处理流程（`maybeApplyTtsToPayload`）。无论消息来自直接回复还是 announce 投递，只要包含 TTS 标签就应该触发语音生成。这是投递路径与 TTS 管道的集成问题——cron announce 投递跳过了 TTS 处理步骤。

---

### #212 WhatsApp: 修复引用图片不可见问题

**更新原文**:
(关联 Issue #59174) 当用户在 WhatsApp 中引用图片并 tagging openclaw 时，openclaw 无法看到该图片，用户需要重新发送图片并 tagging 才会显示。

**痛点**:
当用户在 WhatsApp 中回复一张图片并 tagging openclaw 时，openclaw 无法看到该图片——图片消息被忽略了。用户必须重新发送图片并 tagging openclaw 才能让 openclaw 看到。这是一个回归 bug，之前版本是可以工作的。问题严重影响 WhatsApp 渠道的媒体处理功能，用户体验显著下降。

**如何解决**:
问题可能出在 WhatsApp 的引用消息处理逻辑中。当用户通过引用（reply）方式 tagging openclaw 并同时发送图片时，WhatsApp 插件可能没有正确解析这种"引用+媒体"的组合消息格式。需要检查消息处理逻辑，确保引用回复中的媒体内容也被正确捕获。

---

### #213 Sessions: 工具结果持久化时剥离 Schema 并修复文件锁问题

**更新原文**:
(关联 Issue #6650, #15000 等) 工具结果在持久化到 session 文件时包含完整的 schema，导致会话文件无限增长，上下文窗口被 schema 定义填满，token 成本飙升；以及 session 文件锁在写入后未被释放。

**痛点**:
这是一个历史积累的性能和正确性问题，涉及多个独立 issue：

**Issue #6650（工具结果膨胀）**：每次调用工具时，整个 schema 被保存到 session 文件。即使使用 summarization skills，很多也会将工具输出作为 summarization 对象处理。结果是：session 文件无限增长，上下文窗口填满了 schema 定义，token 成本爆炸，LLM 性能下降。根因在于 `session-tool-result-guard.ts` 的 `transformToolResultForPersistence` hook 没有足够激进地剥离 schema。

**Issue #15000（文件锁未释放）**：Gateway 创建 session 文件锁但从不释放它，导致所有后续模型调用超时等待锁。Lock 文件在 session 写入完成后仍持续 60+ 秒。问题在 2026.2.9 引入，所有模型都受影响，不是特定提供商问题。这是严重的回归 bug，阻止所有 agent 响应直到手动干预。

**如何解决**:
(1) 在工具级别确保工具只返回必要的结果数据，而非完整的 response 对象；(2) 在 `session-tool-result-guard.ts` 的 persistence transform 中添加激进的 schema 剥离逻辑，删除 `schema`、`$schema`、`definitions`、`properties`、`required`、`metadata.schema` 等字段；(3) 修复 session 文件锁的释放逻辑——在完成处理程序（无论成功或失败）中调用 unlock()，确保锁立即释放。

---

### #214 CLI/doctor: 检测或退役报告网关健康状态不准确的 legacy ensure-whatsapp cron 检查

**更新原文**:
(关联 Issue #60204) legacy `ensure-whatsapp.sh` cron 检查产生误导性的健康日志条目，如 "WARN: Gateway inactive, starting via systemd"，即使网关实际是健康的。

**痛点**:
用户 crontab 中的 legacy `~/.openclaw/bin/ensure-whatsapp.sh` 脚本在 `systemctl --user is-active openclaw-gateway.service` 无法连接到 user bus 时（如 plain cron 环境缺少 `XDG_RUNTIME_DIR` 和 `DBUS_SESSION_BUS_ADDRESS`），会输出误导性的 "WARN: Gateway inactive, starting via systemd" 日志。这将调试引向错误方向——传输问题看起来像网关不健康。用户在 2026-04-03 07:15 进行 cron-delivery 调查时，gateway journal 显示 WhatsApp 健康，但 `whatsapp-health.log` 仍声称网关不活跃。这是一个误导性诊断日志的问题，让用户做无效的排查工作。

**如何解决**:
三个可能的解决路径：(1) 让 `openclaw doctor` / 健康诊断检测到调用 `ensure-whatsapp.sh` 的 legacy crontab 条目并明确警告它们已不受支持或过时；(2) 用 repo 管理的 systemd user unit 替换 legacy cron-based WhatsApp 检查，该 unit 具有正确的 DBus 环境；(3) 硬化脚本——当无法访问 user bus 时拒绝发出 "Gateway inactive" 消息，并将环境问题明确记录。

---

### #215 CLI: 修复 Slack JSON manifest 输出中的字符边框问题

**更新原文**:
(关联 Issue #65751) CLI 通过 Slack 渠道连接流程提供的 JSON manifest 使用了 ASCII 边框字符（`| `），导致 JSON 无效且破坏复制粘贴功能。

**痛点**:
当用户运行 `openclaw channels add` 并添加 Slack 渠道时，CLI 提供一个 JSON manifest 用于快速设置——但这个 JSON 被 ASCII 边框字符（垂直管道 `│` 和 `╮` 等）包围。用户无法直接复制 JSON 来使用，必须手动删除所有边框字符。manifest 输出的实际格式是：
```
│  Manifest (JSON):                                                                   │
│  {                                                                                  │
│    "display_information": {                                                         │
│      "name": "OpenClaw",                                                            │
...
```
这不是有效 JSON——管道字符不是 JSON 的一部分，但被包在输出中，让用户以为是 manifest 的一部分而复制使用。这是一个纯 UI/UX 问题，影响 CLI 的可用性。

**如何解决**:
在 `openclaw channels add` 的 Slack 渠道设置流程中，manifest JSON 输出不应该包含 ASCII 边框字符。纯 JSON 输出给用户复制使用，边框字符只用于 CLI 界面美化。如果需要在视觉上美化，应该提供"纯 JSON 模式"和"美化模式"两个选项，默认或标准选项应该是纯 JSON。

---

### #216 WhatsApp: 修复 logout 和 remove 不清理会话的问题

**更新原文**:
(关联 Issue #67746) `openclaw channels logout --channel whatsapp` 和 `openclaw channels remove --channel whatsapp` 命令无法成功清理 WhatsApp 配置，bot 仍然响应消息直到手动编辑配置文件并强制重启网关。

**痛点**:
当用户尝试完全移除 WhatsApp 号码/频道时，运行 `openclaw channels logout --channel whatsapp` 和 `openclaw channels remove --channel whatsapp` 后，bot 仍然响应来自"已移除"号码的消息。logout 和 remove 命令没有：(1) 立即终止 Gateway 中的活跃 WhatsApp socket/session；(2) 自动从 `openclaw.json` 中删除 whatsapp 配置块；(3) 删除 `~/.openclaw/credentials/whatsapp/` 中的会话凭证。这造成隐私风险——用户以为已断开个人号码，但 bot 继续在后台处理实时消息。用户必须手动：kill gateway 进程、手动删除配置中的 whatsapp 条目、手动删除凭证文件夹。这不是一个干净的移除流程。

**如何解决**:
在 `openclaw channels logout` 和 `openclaw channels remove` 命令的实现中，增加对活跃 WhatsApp session 的清理逻辑：在移除配置前，先调用 WhatsApp session 的 close/cleanup 方法；删除配置后，清理相关凭证文件夹；确保 gateway 内存中的状态也被清除。这需要一个完整的三层清理：磁盘配置、凭证文件、内存中活跃连接。

---

### #217 Voice Call: 支持 telephony TTS 路径的 [[tts:voiceId=...]] 指令

**更新原文**:
(关联 Issue #58114) Voice call 插件的 telephony TTS 路径不解析 `[[tts:...]]` 指令，无法在通话中切换语音。

**痛点**:
OpenClaw 有两条独立的 TTS 路径：(1) Message channel TTS——正确解析 `[[tts:voiceId=X model=Y]]` 指令，应用覆盖到 TTS 配置，从可见文本中剥离指令；(2) Telephony TTS——`parseTtsDirectives` 从未被调用，合并配置在插件启动时静态创建，原始文本（包括指令）直接发送给 ElevenLabs 作为要朗读的文本发送。结果是：消息渠道中完美工作的 `[[tts:voiceId=...]]` 指令在语音通话中被读出来而非被处理。用户期望的中途语音切换完全不可用——这是电话机器人的一个重要功能缺失。

**如何解决**:
在 `playTtsViaStream(text, streamSid)` 或 `speak(ctx, callId, text)` 中，在将文本传递给 `synthesizeForTelephony` 之前：(1) 调用 `parseTtsDirectives(text, modelOverridesPolicy)` 提取指令覆盖并清理文本；(2) 如果存在覆盖，将它们合并到 per-call TTS 配置中（克隆静态 `mergedConfig`，应用覆盖）；(3) 传递清理后的文本 + per-call 配置给 `synthesizeForTelephony`。这让 telephony TTS 能够支持与消息渠道相同的动态语音切换指令。

---

### #218 Sessions: 修复优雅关闭时文件锁未释放导致 CPU 忙轮询

**更新原文**:
(关联 Issue #75805, #49603) Gateway 重启后 session lock 文件处于卡住状态，导致忙轮询循环消耗 50%+ CPU，直到手动删除锁文件。

**痛点**:
OpenClaw gateway 在关闭时没有正确清理 session lock 文件，导致重启后陷入忙轮询循环：Gateway 试图获取/释放同一个锁，CPU 消耗 50%+ 且响应变得迟钝。用户必须 SSH 进去手动删除锁文件才能恢复。具体表现：(1) 一个任务挂起 110+ 分钟后被 reap；(2) Gateway 重启没有正确 orphan/cleanup 锁；(3) Gateway 试图在启动时重新获取锁但遇到阻止释放的条件；(4) 结果：锁获取/释放逻辑进入无限循环。这是 #15000 和 #49603 问题的延伸/变种，说明 session 锁机制在边缘情况下的清理存在系统性问题。

**如何解决**:
在 gateway 启动时，删除任何由 starting PID 拥有的 `agents/*/sessions/*.lock` 文件，之后再初始化锁管理器。这样即使之前的进程留下了锁文件，新的进程也会清理它们。锁获取应该有超时机制防止无限等待。被挂起的任务应该有可配置的自动 reap 超时。

---

### #219 Agents/subagent: 修复上下文引擎初始化顺序导致 spawn 失败

**更新原文**:
(关联 Issue #73095, PR #73904) 在 subagent spawn 准备阶段之前初始化内置上下文引擎，解决 "Context engine 'legacy' is not registered" 错误。

**痛点**:
在 CLI backend-only 的冷启动场景中，subagent spawn 准备阶段失败并报错 "Context engine 'legacy' is not registered"。这是因为内置上下文引擎在 `sessions_spawn` 的上下文引擎准备阶段之前没有被初始化，导致配置的引擎无法被解析。这是一个初始化顺序问题——某些上下文引擎只在需要时才注册，但 subagent spawn 的准备阶段假设这些引擎已经注册。冷启动路径（不经过完整的 gateway 初始化）触发了这个 bug。

**如何解决**:
在 subagent spawn 准备阶段之前初始化内置上下文引擎。确保 `sessions_spawn` 的上下文引擎准备之前，所有内置引擎（legacy 等）都已经注册完毕。添加回归测试覆盖冷注册路径，防止未来再次出现这个初始化顺序问题。涉及文件：`src/agents/subagent-spawn.ts`、`src/agents/subagent-spawn.runtime.ts` 等。

---

### #220 (无关联 PR)

**更新日志条目**: Gateway/handlers: handle empty prompt archives gracefully in model input preprocessing, so empty, whitespace-only, and unreadable prompt archives do not abort the run.

本条更新无关联 PR/Issue 编号，无法获取更多详情。从更新日志描述推测：这是网关的模型输入预处理逻辑改进，当 prompt archive 为空或不可读时，优雅处理而非中止运行。

---

### #221 (无关联 PR)

**更新日志条目**: Gateway/agents: detect `runKey` from agent-scoped tool calls and keep it in routing context throughout the run, so agent-scoped tools use their own session rather than the parent session.

本条更新无关联 PR/Issue 编号。从更新日志描述推断：这是网关的 routing context 改进，从 agent-scoped 工具调用中检测 `runKey` 并在整个运行过程中保持在路由上下文中，让 agent-scoped 工具使用自己的 session 而非 parent session。

---

### #222 (无关联 PR)

**更新日志条目**: Agents/sessions: warn when sessions are created without explicit ephemerality, so unexpected pinning can be diagnosed and prevented.

本条更新无关联 PR/Issue 编号。从更新日志描述推断：这是 sessions 创建逻辑的改进，当 session 在没有显式指定临时性（ephemerality）的情况下被创建时发出警告，帮助诊断和预防意外的 pinning 行为。

---

### #223 (无关联 PR)

**更新日志条目**: Plugins/runtime: add `resolveAgentScopedPluginTool` helper and use it in agent-scoped tool routing, so session-scoped tools are correctly resolved from the agent scope rather than the root scope.

本条更新无关联 PR/Issue 编号。从更新日志描述推断：这是插件运行时的一个架构改进，添加 `resolveAgentScopedPluginTool` 辅助函数并在 agent-scoped 工具路由中使用，确保 session-scoped 工具从 agent scope 而非 root scope 正确解析。

---

### #224 (无关联 PR)

**更新日志条目**: Config: warn when JSON parse fails on encrypted config fields and the field is not a known encrypted field, so unknown encrypted field formats do not silently block config migration.

本条更新无关联 PR/Issue 编号。从更新日志描述推断：这是配置迁移逻辑的改进，当 JSON 解析加密配置字段失败且该字段不是已知的加密字段格式时，发出警告——避免未知格式的加密字段静默阻止配置迁移。

---

### #225 Config: 修复备份恢复失败时仍报告成功的问题

**更新原文**:
(关联 PR #70515) 修复 `maybeRecoverSuspiciousConfigRead` 中 `copyFile` 失败时仍记录 "Config auto-restored from backup" 并写入 `valid: true` 审计记录的问题。

**痛点**:
在 `maybeRecoverSuspiciousConfigRead`（异步和同步）中，`copyFile(backupPath, configPath)` 被包装在 bare `catch {}` 中。当 copy 失败时（磁盘满、EACCES 权限拒绝），代码仍然记录 "Config auto-restored from backup" 并写入 `valid: true` 的审计记录。结果是：用户和审计工具认为配置已被修复，但实际上损坏的配置仍在使用。这是一个静默的数据损坏——看起来正常但实际有问题的配置被当作有效处理，可能导致生产环境的配置问题无法被发现。

**如何解决**:
在 `copyFile` 失败时捕获错误，发出明确的失败警告，设置 `valid` 为 `restoredFromBackup`，在审计记录中保存 `restoreErrorCode` 和 `restoreErrorMessage`。修复后的行为：复制失败时记录 "Config auto-restore from backup failed"，审计记录显示 `valid: false` 和具体的错误码。涉及 `src/config/io.observe-recovery.ts` 中第 659-662 行的 bare `catch {}` 模式。

---

### #226 Agents/compaction: 修复 Azure 内容过滤器阻止压缩时的回退问题

**更新原文**:
(关联 Issue #64960, PR #74470) 压缩失败时当 Azure 内容过滤器阻止汇总，compaction 永远无法恢复，因为没有模型回退链。

**痛点**:
当压缩/summarization 模型托管在 Azure 上且会话历史包含触发 Azure 内容管理策略的内容时，压缩失败并返回 HTTP 400——然后永远无法恢复。原因：`(1) retryAsync` 用相同的模型重试 3 次；(2) `resolveEmbeddedCompactionTarget()` 使用会话的主模型，除非设置了 `compaction.model` override，没有回退链；(3) 溢出循环用相同的模型重试 compaction。结果是：溢出 → compact → Azure 拒绝 → 溢出 → 永远重复。这是严重的永久性失败——用户必须手动干预才能恢复对话。

**如何解决**:
为隐式压缩引入模型回退链：当主压缩模型拒绝内容时，通过活跃会话的模型回退链尝试备用模型。确保回退选择是临时的（ephemeral）——仅在每次尝试时有效，不会持久化到 `sessions.json`（避免重新引入 #58303 的模型锁定 bug）。显式 `agents.defaults.compaction.model` 保持精确，不继承会话回退链。所有压缩模型都失败后，强制 `compact_then_truncate` 作为硬逃脱。

---

### #227 Gateway/config: 允许 config.patch 修改 subagent thinking 配置路径

**更新原文**:
(关联 Issue #75764, PR #75802) Gateway config.patch 拒绝修改文档中记录 `agents.defaults.subagents.thinking` 路径，但 schema 和文档都说明该路径是可配置的默认 subagent thinking。

**痛点**:
`gateway config.patch` 工具有一个硬编码的 mutation allowlist（允许修改的配置路径白名单），其中包含 `agents.defaults.thinkingDefault` 和 `agents.list[].thinkingDefault`，但不包括 `agents.defaults.subagents.thinking`。结果是：用户尝试通过 `config.patch` 修改文档中记录的 subagent thinking 默认路径时，工具报错 "gateway config.patch cannot change protected config paths: agents.defaults.subagents.thinking"。文档和 schema 都说这个路径是可配置的，但实际工具拒绝修改——这是文档/实现不一致的问题，影响用户体验和配置管理。

**如何解决**:
将 `agents.defaults.subagents.thinking` 和 `agents.list[].subagents.thinking` 添加到 `isAllowedGatewayConfigPath` 匹配行为中，允许 `config.patch` 修改这些文档中记录的 subagent thinking 默认路径。同时添加对相邻受保护 subagent 策略字段的 guard 覆盖。涉及 `src/agents/tools/gateway-tool.ts` 中的硬编码 allowlist。

---

### #228 (无关联 PR)

**更新日志条目**: Subagents: treat subagent spawned inside an isolated cron run as isolated, so subagents of isolated jobs are not re-parented to the main session.

本条更新无关联 PR/Issue 编号。从更新日志描述推断：这是 subagent 生命周期管理的改进，在 isolated cron 运行中 spawn 的 subagent 应该被视为 isolated，避免被重新 parent 到 main session。

---

### #229 (无关联 PR)

**更新日志条目**: Agents/sessions: normalize session store access so that concurrent operations do not race on session file metadata, and clean up stale concurrent handles.

本条更新无关联 PR/Issue 编号。从更新日志描述推断：这是 session store 的并发访问改进，归一化 session 文件访问，避免并发操作在 session 文件元数据上的竞态条件，并清理过期的并发句柄。

---

### #230 Discord: 修复完成后永久 thinking emoji 未清除的问题

**更新原文**:
(关联 Issue #75458) Discord status reactions 在完成后可能留下永久的非终态 thinking emoji（🤔）而非终态 done emoji（👍）或无反应。

**痛点**:
Discord 状态反应在 turn 完成后可能留下最终/永久反应在非终态生命周期状态。在活跃 Discord 频道中，当 turn 成功完成后，触发消息上唯一剩余的反应是 `🤔`（thinking）而非终态的 `👍`（done）或无反应。这让用户困惑——消息明显已完成，但持久化的反应说 agent 仍在思考。更糟的是，在同一调试会话中，多个 bot 所有的生命周期反应累积在同一用户消息上（`🥱`、`🔥`、`👨‍💻`、`🤔` 等），必须手动删除。反应作为用户可见的状态 UI，其状态误导性让用户以为 agent 仍在工作或卡住，即使回复已经送达。

**如何解决**:
问题根源可能在 `channel-feedback-*.js` / `src/channels/status-reactions.ts` 的 reaction 控制器：(1) 如果之前的 `removeReaction` 调用失败或竞态，`currentEmoji` 可能与真实反应状态不匹配；(2) 防抖/待处理的 thinking 或工具更新可能在终态/清除路径之后应用；(3) Discord 适配器 rate limits 或 API 错误。需要确保 Discord 的 finalization 路径在 `removeAckAfterReply: true` 时正确清除所有状态反应，或在完成后将最终持久反应设置为终端 done emoji。

---

### #231 Discord: 修复 channel-level agentId 配置被拒绝且可能导致网关中止

**更新原文**:
(关联 Issue #62455) 当在 `channels.discord.guilds.<guildId>.channels.<channelId>` 下添加 `agentId` 时，配置重载无效且网关可能中止并报错 "Unrecognized key: agentId"。

**痛点**:
当用户尝试在 Discord 频道级别配置 `agentId`（如 `channels.discord.guilds.<guildId>.channels.<channelId>.agentId`）时，这个配置形状不被支持。配置重载被跳过，gateway 可能中止并报错：`channels.discord.guilds.<guildId>.channels.<channelId>: Unrecognized key: "agentId"`。受支持的 per-channel agent 路由路径是 `bindings[]` 配置（`match.channel = "discord"` 和 `peer.kind = "channel"`），但无效的 channel-level 配置形状不会自我修复，用户得不到明确的迁移指导。问题是这可以让整个网关中止，是一个严重的配置陷阱。

**如何解决**:
让 channel-level `agentId` 被支持，或在 config 验证失败时提供明确的可操作迁移提示，指向正确的 `bindings[]` 配置方式。`openclaw doctor --fix` 应该能够自动修复或重写不支持的配置，而不是让网关中止。

---

### #232 Discord: 修复 DM 入站消息 ctx.To 设置错误

**更新原文**:
(关联 Issue #68126) Discord DM 入站消息的 `ctx.To` 被错误设置为 `channel:<DM_channel_id>` 而非 `user:<userId>`，导致下游组件将 DM 当作 channel 对话处理。

**痛点**:
Discord DM（直接消息）入站消息的 `ctx.To` 被错误地设置为 `channel:<DM_channel_id>` 而非 `user:<userId>`。这导致下游组件（mirror、delivery-recovery）将 DM 当作 channel 对话处理，最终导致 Discord API `Unknown Channel` 错误。具体表现：Mirror session key 构建为 `agent:main:discord:channel:<id>` 而非正确的 `agent:main:discord:direct:<userId>`。虽然 `OriginatingTo` 和 `lastRouteTo` 通过 `dmConversationTarget` 正确修复，但 `ctx.To` 本身没有被修复——这是 DM 路由中一个隐蔽但关键的路径分支。

**如何解决**:
在 `message-handler` 的 `effectiveTo` 赋值处应用 DM 修正：`const effectiveTo = autoThreadContext?.To ?? (isDirectMessage ? user:${author.id} : replyTarget)`。注意 `deliverTarget` 应该保持 `channel:<DM_channel_id>`（因为 Discord API 通过 channel ID 发送 DM），只有语义的 `ctx.To` 需要修正。涉及 `threading-*.js` 中 `originalReplyTarget` 的构建逻辑。

---

### #233 Discord: 修复 channel-info 失败时 DM fork 成重复 channel sessions

**更新原文**:
(关联 Issue #59817) Discord DM 在 channel-info/网络故障期间可能 fork 成重复的 channel sessions，同一个 DM 对话被记录在多个不同的 session keys 下。

**痛点**:
Discord 直接消息可能在 DM channel 分类失败时被记录在多个 session keys 下。同一 DM 最终出现在三个不同的 session keys 中：`agent:main:discord:direct:<userId>`（正确）、`agent:main:discord:channel:<DM_channel_id>`（错误，DM channel id 被当作 guild/channel session）和 `agent:main:discord:channel:<userId>`（错误，user id 被当作 channel id）。这导致对话历史碎片化，回复可能消失。这是因为入站 Discord 路由依赖于 `channelInfo?.type === ChannelType.DM` 的检查，当分类失败或超时时，回退到 channel-mode 路由并构建 `channel:` session key 而非 `direct:`。

**如何解决**:
在 `resolveDiscordAutoThreadReplyPlan` 和相关路由逻辑中，确保 DM 消息始终使用 `direct:` session key。即使 channel info 解析失败或超时，也要基于消息本身的 `chat_type === "direct"` 来确定会话类型，而不是回退到 channel 模式。添加对 channel info 解析失败的容错处理，确保即使在网络故障期间，DM 也能被正确路由到 `direct:` session。

---

### #234 Discord: 扩展 retry policy 覆盖 429 之外的瞬态错误

**更新原文**:
(关联 Issue #52396) Discord retry policy 只覆盖 429 (RateLimitError)，不覆盖 502/503/timeout/connection errors，导致这些瞬态错误导致的消息丢失无法重试。

**痛点**:
Discord 出站消息 retry（`extensions/discord/src/retry.ts`）只在 `RateLimitError`（HTTP 429）时重试。任何其他瞬态失败——502、503、connection reset、timeout、DNS resolution failure——都立即失败，不重试。对比 Telegram 的 retry policy（`src/infra/retry-policy.ts`）：retry on `429|timeout|connect|reset|closed|unavailable|temporarily`。结果是：Cron jobs 和 agent sessions 在 Discord API 瞬态失败时静默丢失消息——job 被标记为 `error: ⚠️ ✉️ Message failed`，没有重试。用户报告的真实案例：Daily Stock Pick cron job 成功完成了所有研究（约 276 秒的工作），但最终 Discord 发送失败，整个输出丢失。

**如何解决**:
将 `shouldRetry` 扩展到也覆盖瞬态 HTTP 错误和连接失败，类似于 Telegram 的策略：`const DISCORD_TRANSIENT_RE = /429|502|503|timeout|connect|reset|closed|unavailable|temporarily|fetch.failed/i`。现有的 `channels.discord.retry` 配置（attempts、delays、backoff）保持不变，只需要扩大 retry 资格谓词。

---

### #235 Discord: 修复 Components v2 文本无法从 referenced_message 提取

**更新原文**:
(关联 Issue #56228) 当用户回复用 Discord Components v2（containers、text display blocks）发送的消息时，回复上下文 body 为空，因为 `resolveDiscordMessageText()` 只检查 `content` 和 `embeds[0]`，两者对 v2 消息都是空的。

**痛点**:
当用户回复用 Discord Components v2（containers、text display blocks）发送的消息时，回复上下文 body 是空的。这是因为 `resolveDiscordMessageText()`（在 `route-resolution-*.js` 中约第 301 行）只检查 `content` 和 `embeds[0]`，而 v2 消息的这两者按设计都是空的。Discord API 在 `referenced_message` 中确实包含完整的 `components` 数组（包括 type 10 TextDisplay 节点），但没有被解析。结果是：agent 无法看到用户正在回复的是什么通知，ReplyToBody 元数据为空。Workaround 是使用 legacy embeds 而非 components v2，但 v2 是更现代的组件格式。

**如何解决**:
在 `resolveDiscordMessageText()` 中添加 fallback：当 `baseText` 为空且 `message.flags & 32768`（IS_COMPONENTS_V2）时，从 components 树中提取 text。实现 `extractComponentsV2Text()` 函数，遍历 components 树并连接 type 10（TextDisplay）节点的文本。这确保 v2 component 消息的文本能被正确提取到回复上下文中。

---

### #236 Discord: 将 DISCORD_GATEWAY_READY_TIMEOUT_MS 改为可配置

**更新原文**:
(关联 Issue #72273) 将硬编码的 15 秒 Discord ready timeout 改为可配置，以支持多账号 stagger 启动场景。

**痛点**:
`DISCORD_GATEWAY_READY_TIMEOUT_MS` 被硬编码为 15 秒，无法通过环境变量或配置 schema 覆盖。在多账号 Discord 设置中，staggered provider 启动（延迟 70s/80s/90s 以减少 Discord 启动 rate limits）意味着 15 秒的全局 ready check 在后面的账号完成之前就触发了，产生重启循环。用户报告：9 个 Discord agent 账号在单个 gateway 上，15 秒超时在第 3+ 个账号完成前触发，导致 SIGTERM → restart loop。在一天内观察到 244 次 `gateway starting…` 事件、161 次 SIGTERM、44 次 ready timeout。这让用户只能通过本地 sed patch bundle（`15e3 → 9e4`）来 workaround，每次 `npm update -g openclaw` 后都需要重新应用，因为 bundle 文件名 hash 会变化。

**如何解决**:
通过环境变量（`OPENCLAW_DISCORD_READY_TIMEOUT_MS` 和 `OPENCLAW_DISCORD_RUNTIME_READY_TIMEOUT_MS`）或配置 schema（`gateway.discord.readyTimeoutMs`）使两个 Discord timeout 可配置。合理的默认值可以是 `max(account.startupStaggerMs) + 30s`，而非固定的 15s。

---

### #237 Discord: 支持 description_localizations 保留本地化命令描述

**更新原文**:
(关联 Issue #56580) Discord slash command 描述在 native command reconcile/redeploy 后恢复到英语默认值，需要支持 `description_localizations`。

**痛点**:
当 OpenClaw 重新运行 native command reconcile/redeploy（如 restart、provider reload 或 update 后），Discord slash command 描述恢复到打包的英语默认值。这让本地化的 Discord slash command UX 变得脆弱——即使 Discord 命令栈支持 `description_localizations`，OpenClaw 也不使用它。用户的本地化描述（如韩语）在每次 reconcile 后被英语覆盖。这不是只有韩语的问题——任何 locale 特定的 Discord slash command 体验都会漂移，因为 OpenClaw 只支持单个默认 description 字符串，不支持官方 localization 字段。

**如何解决**:
在 native command specs 中引入可选的 localization 元数据，并将其传递到 Discord 命令序列化。在序列化时输出 `description_localizations` 字段。reconcile 时应比较 localization 字段，这样漂移会被主动纠正。至少保留现有的英语 `description` 作为 fallback。这需要修改 OpenClaw 的命令生成逻辑和 Carbon/Discord 命令序列化。

---
### #238 Discord：出站 @AgentName 提及转换可靠性

**更新原文**：
Discord: make outbound @AgentName mention conversion reliable for known agents, even without prior DM traffic. Fixes [#67587](https://github.com/openclaw/openclaw/issues/67587). Thanks [@andyliu](https://github.com/andyliu).

**关联 Issue**：#67587

**痛点**：
在多 Agent 环境下，一个 OpenClaw Agent 需要通过 Discord 向另一个 Agent 发送消息并提及对方（如 `@Vladislava`），但 OpenClaw 生成的输出是纯文本 `@Vladislava`，而不是 Discord 能识别的真实提及 `<@USER_ID>`。结果是被提及的 Agent 根本收不到 Discord 的提及通知，多 Agent 协作流程断裂。

问题根因在于 OpenClaw 的出站提及重写依赖于「账户级缓存映射」（handle → userId）。只有当目标账号已经在本地缓存过该用户名时，重写才生效；缓存缺失时，`@Name` 就以纯文本发出。Issue 中明确指出这是"best-effort and cache-dependent"，在 Agent 之间需要确定性协作的场景下不可接受。

**如何解决**：
PR 修复使得已知 Agent 的用户名（configured agent names）能够在没有先前 DM 流量的情况下也能被可靠地转换为 `<@userId>` 格式。实现上应该是从本地配置（如 agent 别名列表）中直接建立名称→ID 映射，而非仅依赖过往消息流中观察到的缓存。具体改动涉及 Discord provider 的出站消息处理逻辑，在发送前对已知 Agent 名称做确定性替换，不再依赖机会主义缓存。

---
### #239 Discord：v2026.4.29 OAuth/429 启动回归

**更新原文**：
Discord: retry startup OAuth @me fetch on timeout, fix 429 hit during native slash command deploy. Fixes [#75341](https://github.com/openclaw/openclaw/issues/75341). Thanks [@andyliu](https://github.com/andyliu).

**关联 Issue**：#75341

**痛点**：
v2026.4.29 引入了 Discord 集成的启动回归：
1. **OAuth 超时**：_gateway 启动时，`oauth2/applications/@me` 和 `users/@me` 的预检查请求超时（timeoutMs=2500~10000，实际耗时 9524~22353ms），Bot/App 身份预检反复失败。
2. **429 rate limit 爆发**：启动时大量 `native-slash-command-deploy-rest:patch` 调用以 HTTP 429 被 Discord rate limit 拒绝，retry_after 累积导致启动时间大幅拉长。

用户在日志中提供了清晰的时间戳证据，v2026.4.26 正常，v2026.4.29 开始出问题，且不是 #74765（Token 解析不匹配）的问题。

**如何解决**：
PR 做了两件事：
1. **OAuth fetch 超时重试**：对 `oauth2/applications/@me` 和 `users/@me` 的超时错误增加重试机制，而不是直接失败，确保 Bot 身份预检在网络抖动时能恢复。
2. **429 限流处理**：在原生斜杠命令部署的 PATCH 请求中增加 429 响应处理（可能是排队重试或退避），避免启动时 burst 请求被一次性拒绝。

关键设计：启动阶段的请求应该有时间感（graceful retry），而不是碰壁直接失败或无节制重试。

---
### #240 插件钩子：ctx.channelId 返回 Provider 名而非实际频道 ID

**更新原文**：
Plugin hook ctx.channelId now correctly returns the provider-scoped channel ID instead of the provider name. Fixes [#59881](https://github.com/openclaw/openclaw/issues/59881). Thanks [@andyliu](https://github.com/andyliu).

**关联 Issue**：#59881

**痛点**：
在插件钩子（`before_prompt_build`、`agent_end`）中，`ctx.channelId` 被错误地填充为 Provider 名称（如 `"discord"`），而不是实际的频道标识符（如 Discord 的 snowflake ID `"1472750640760623226"`）。这导致需要按频道隔离数据的插件（如 Hindsight）把所有 Discord 频道的数据错误地合并到一个桶中。

Issue 中的调试输出清晰展示了问题：
```
[Hindsight] before_prompt_build - bank: main::discord, channel: undefined/undefined
```
插件的 `deriveBankId()` 使用 `ctx?.channelId || sessionParsed.channel`，由于 `ctx.channelId` 是 `"discord"`（truthy），fallback 从不触发，实际频道 ID 丢失。

**如何解决**：
修复将 `ctx.channelId` 的语义恢复为「实际频道/会话标识符」，而引入 `ctx.messageProvider` 来承载 Provider 名称。session key `agent:main:discord:channel:1472750640760623226` 中本身就包含正确的频道 ID，钩子上下文只需正确提取即可。插件可使用 `ctx.channelId` 做真正的频道级隔离，`ctx.messageProvider` 获取 Provider 类型。

---
### #241 Discord：组件 v2 按钮路由

**更新原文**：
Discord: fix component v2 button routing when interactive components are sent via FollowupMessage. Thanks [@andyliu](https://github.com/andyliu).

**关联 PR**：未在更新日志中引用具体 PR 号

**痛点**：
Discord 的组件 v2（interactive components / button 交互）通过 FollowupMessage 发送时，按钮的路由目标（interaction callback 目标频道/会话）未被正确解析，导致点击按钮后事件无法路由到正确的处理逻辑。FollowupMessage 是 Discord 消息机制中用于后续追加消息的接口，按钮组件的 callback 需要指向原始 interaction 或对应会话，而 v2 组件格式与之前格式存在差异，导致路由匹配失败。

**如何解决**：
修复 FollowupMessage 中组件 v2 button 的路由目标解析，使其能正确指向对应的 callback 目标频道/会话。改动涉及 Discord provider 中 followup message 的交互式组件处理路径，按钮点击事件的 channel/session 解析逻辑。

---
### #242 诊断：会话进展追踪与 stuck 警告

**更新原文**：
Diagnostics: track session progress before stuck warnings, back off repeated session.stuck diagnostics while processing. Fixes [#72010](https://github.com/openclaw/openclaw/pull/72010). Thanks [@vincentkoc](https://github.com/vincentkoc).

**关联 PR**：#72010

**痛点**：
诊断系统在判断"会话是否卡住"时，没有将 ACP runtime events（reply/tool/status/block 事件）计入进展，导致长时间运行的 channel turn 被误判为 stuck 并触发警告。即使用户和 Agent 之间的通信正常进行，诊断也可能因为未检测到这些事件而误报 session stuck。

此外，typing keepalive 回调被错误地当作进度事件，导致真正静默的 model call 无法触发 no-progress 诊断。

**如何解决**：
PR 做了三处修正：
1. **将 ACP runtime events 视为进展**：reply/tool/status/block 事件触发时，不再产生 stuck 警告，因为这些事件表明会话正在被处理。
2. **typing keepalive 不计入进展**：真正的静默 model call 应该能触发 no-progress 诊断，不能被 keepalive 干扰。
3. **处理中会话的 stuck 诊断退避**：当同一会话持续处于 processing 状态时，重复的 stuck 诊断应该退避，避免噪音日志。

---
### #243 插件工具：修复多项延迟回归

**更新原文**：
Plugin tools: skip core coding tool construction when only plugin tools are allowlisted, keep workspace plugin registry cache separate from scoped loads, and add regressions for both latency paths. Fixes [#75882](https://github.com/openclaw/openclaw/issues/75882) [#75907](https://github.com/openclaw/openclaw/issues/75907) [#75906](https://github.com/openclaw/openclaw/issues/75906) [#75887](https://github.com/openclaw/openclaw/issues/75887) [#75851](https://github.com/openclaw/openclaw/issues/75851). Thanks [@andyliu](https://github.com/andyliu).

**关联 PR**：#75922（+407 / -190）

**痛点**：
v2026.4.29 引入了一系列严重的延迟回归，涉及多个子系统：

1. **#75882 事件循环 stall**：Gateway 间歇性地 stall Node 事件循环数十到数百秒，导致跨频道延迟、回复丢失、频道断开。WhatsApp、Telegram、Slack 均受影响。根本原因是 plugin registry LRU 在 cron scope key 增多时频繁驱逐，cold reload（jiti 编译 + V8 JIT）同步阻塞事件循环 22-35s。

2. **#75907 Windows + Node 24 core-plugin-tools 冷启动阻塞**：Windows 11 (WSL2) + Node 24.14.0 上，每次冷启动 core-plugin-tools 阻塞 30-40s，因为 `createCodingTools()` 在 Windows 上做文件系统探测、工具描述生成、PTY/shell 检测比 Linux 慢得多。

3. **#75906 每轮延迟回退**：v2026.4.24 → v2026.4.29，Telegram 延迟从 ~4s 退化到 ~15s，且与 Active Memory 无关。怀疑是 system-prompt rebuild、spawned subagent 路由元数据或 people-aware memory 等新功能带来的 per-turn 开销。

4. **#75887 system-prompt rebuild 阻塞事件循环**：每次 embedded-run 的 `system-prompt` 阶段耗时 16-29s（`node-llama-cpp` 同步 embedding 推理 + MMR re-rank），即使空闲也占用 1 个 CPU 核心。

5. **#75851 LRU 缓存驱逐导致事件循环阻塞**：64 个 cron job 触发 scope-key 变体填满 LRU，主 workspace 条目被驱逐，后续冷加载同步阻塞事件循环 22-35s。

**如何解决**：
PR #75922 是针对上述五个 issue 的综合修复，核心思路是「按需构造 + 缓存隔离」：
1. **跳过核心编码工具构造**：当 explicit allowlist 只请求 plugin tools 时，不构造核心编码工具，避免 Windows/Node 24 上的 30-40s 阻塞。
2. **Workspace 插件注册表缓存与 scoped 加载分离**：workspace 全量插件缓存独立于 scoped cron 加载，不因 cron scope key 变体被驱逐。
3. **增加回归测试**：为两条延迟路径分别添加测试，防止未来再引入类似问题。

变更文件涵盖 `attempt.ts`、`pi-tools.ts`、`loader.ts` 等核心模块。

---
### #244 故障转移：内部服务器错误分类改进

**更新原文**：
Failover: improve internal server error classification to treat a pure 'status: internal server error' message as a server error. Fixes [#73844](https://github.com/openclaw/openclaw/pull/73844). Thanks [@andyliu](https://github.com/andyliu).

**关联 PR**：#73844（+16 / -1）

**痛点**：
故障转移（failover）机制在判断是否为内部服务器错误时，存在分类不够精确的问题。如果 LLM 返回的错误消息是纯文本 `"status: internal server error"`（未经过 scrubber 处理）， failover 模块可能无法正确将其归类为服务器错误，导致应该触发 failover 的场景没有被正确处理，可能导致请求被错误重试或过早放弃。

**如何解决**：
改进 failover 的错误分类逻辑，将原始的 `"status: internal server error"` 消息（即使未经过清洗）也正确识别为服务器错误。这意味着 failover 的正则匹配在判断输入是否为服务器错误时，不再依赖输入是否被清洗过，而是直接识别原始错误消息中的关键模式。

---
### #245 Gateway：启动控制面重试显式化

**更新原文**：
Gateway: make startup control-plane retries explicit by returning consistent retryable startup error shapes from startup-gated RPC methods. Fixes [#76012](https://github.com/openclaw/openclaw/pull/76012). Thanks [@andyliu](https://github.com/andyliu).

**关联 PR**：#76012（+49 / -29）

**痛点**：
Gateway 启动时存在两个就绪面：HTTP liveness/readiness 和 Websocket/RPC startup gates。HTTP `/health`/`/healthz` 可能在所有 sidecar-backed control-plane 行为就绪之前就已经返回成功。同时 Websocket 连接时已经会报告 startup race 为可重试的 `UNAVAILABLE` 错误（`reason: "startup-sidecars"`）。

但 method-level startup gate 返回的错误形状不统一——方法如 `sessions.create`、`sessions.send`、`sessions.abort`、`agent.wait`、`tools.effective` 在 sidecar 未就绪时只返回包含方法名的 generic `UNAVAILABLE`，没有使用与 Websocket 相同的 canonical startup retry detail，导致重试感知调用方无法区分"早期启动竞态"和"普通 gateway 故障"。

**如何解决**：
PR 将启动 gate 扩展到所有依赖 sidecar 就绪的控制面方法，并统一返回 `retryable startup error shape`：
- `details.reason = "startup-sidecars"`
- `retryAfterMs = GATEWAY_STARTUP_RETRY_AFTER_MS`
- 所有启动 gate RPC 错误可被 `isRetryableGatewayStartupUnavailableError` 识别

这使得 ClawBench 等调用方在 startup 期间能可靠地重试，而不是将早期竞态当作永久故障处理。

---
### #246 Google：Gemini 2.5 thinkingBudget 最低值调整

**更新原文**：
Google: raise minimal thinkingBudget floor to 512 for Gemini 2.5 models. Fixes [#70629](https://github.com/openclaw/openclaw/pull/70629). Thanks [@andyliu](https://github.com/andyliu).

**关联 PR**：#70629（+27 / -0）

**痛点**：
Google Generative AI API 将 Gemini 2.5 模型的 `thinkingBudget` 最低值从 128 提升到 512，但 OpenClaw 的 `getGoogleThinkingBudget` 在 `gemini-2.5-flash-lite`、`gemini-2.5-flash`、`gemini-2.5-pro` 的配置表中仍然将 `minimal` 硬编码为 128。这导致所有使用 `reasoning: "minimal"` 的调用立即被 Google API 拒绝，错误信息为 `400 INVALID_ARGUMENT: The thinking budget 128 is invalid. Please choose a value between 512 and 24576`。

受影响的场景包括后台 session 和主心跳 agent，静默失败，用户不知道为何突然无法正常工作。

**如何解决**：
将 Gemini 2.5-pro 和 Gemini 2.5-flash 的 `thinkingBudget` table 中 `minimal` 值从 128 提升到 512，与 Google API 最新限制对齐。`low`、`medium`、`high` 档位不受影响。同时在 `transport-stream.test.ts` 中增加了参数化测试，覆盖所有 budget 档位并验证 regression。

---
### #247 工具：sessionKey="current" 支持频道插件 Agent

**更新原文**：
Tools: resolve sessionKey="current" for channel-plugin agents. Fixes [#74141](https://github.com/openclaw/openclaw/issues/74141). Thanks [@vincentkoc](https://github.com/vincentkoc).

**关联 PR**：#72306（+141 / -3）

**痛点**：
`session_status` 工具的 docstring 指示 Agent 传入 `sessionKey="current"`（通用别名），但 resolver 只对第一方 UI client ID（TUI、CLI、WEBCHAT_UI 等）处理这个别名。频道插件驱动的 Agent（Slack、Discord、Scope 等）的请求 session key 是 channel-style（如 `agent:scopy:scope:scopy:direct:scopy`），不在 `CURRENT_SESSION_CLIENT_ALIAS_IDS` 列表中。

结果是 `sessionKey="current"` 对于频道插件 Agent 无法解析，`sessions.resolve("current")` 失败，Agent 重试，使用构造的 key（也找不到），白白浪费 ~4 秒才放弃。每次涉及时间/日期或 status 的调用都要付这个 4s 税。

**如何解决**：
在所有现有 fallback 都不命中后，将 `sessionKey="current"` 视为请求者自己的 session，返回一个由 minimal synthesized entry 支撑的 status card。这是 fallback 链的最后一步，严格限定为字面量 `"current"`。

关键设计：没有改变 `resolveCurrentSessionClientAlias` 或 `CURRENT_SESSION_CLIENT_ALIAS_IDS`，也没有改变 literal sessions/sessionIds 名为 `"current"` 的现有优先级——这些仍然按原样工作。只为频道插件 Agent 增加了一个最终兜底。

---
### #248 Cron：修复重复 wake-now 任务在临时 busy skip 时的 ghost run

**更新原文**：
Cron: retry recurring wake-now jobs on temporary busy skips instead of recording an immediate ok ghost run. Fixes [#75964](https://github.com/openclaw/openclaw/issues/75964). Thanks [@andyliu](https://github.com/andyliu).

**关联 Issue/PR**：#75964 / #76083（+18 / -23）

**痛点**：
`executeMainSessionCronJob` 在以下条件同时满足时存在一个 early-exit 路径：
1. `wakeMode === "now"`
2. 第一次 `runHeartbeatOnce` 返回 `skipped`，原因是可重试的 busy（`requests-in-flight`、`cron-in-progress` 或 `lanes-busy`）
3. job 是重复性的（`schedule.kind !== "at"`）

此时 cron 记录 `status: ok`，`durationMs ~16ms`，但 Agent 其实根本没有处理这个 cron event——这是一个"ghost run"。真实场景中用户的每日 8PM debrief cron 中招，日志显示心跳 bare-poll fallback 被触发，而不是预期的 `cron-event` provider。

**如何解决**：
PR #76083 将 immediate ok ghost run 改为重试机制：对于可重试的临时 busy skip，延迟重新调度心跳而不是立即返回成功。同时保持 `cron-in-progress` 标记在 deferred heartbeat 路径上，防止同步 wake-now 成功时出现竞争。

回归测试覆盖了 timer regression、heartbeat ghost reminder 和 main-job-passes-heartbeat-target-last 场景。

---
### #249 Google：修复 Gemini 3.1 Pro Preview 在 Agent Runtime 中挂起

**更新原文**：
Google: handle thoughtSignature-only parts to prevent Gemini stream hang. Fixes [#76071](https://github.com/openclaw/openclaw/issues/76071). Thanks [@andyliu](https://github.com/andyliu).

**关联 Issue/PR**：#76071 / #76080（+125 / -4）

**痛点**：
Gemini 3.1 Pro Preview 在 Agent Runtime 中运行时总是触发 LLM idle timeout——Agent  Runtime 中模型完全不返回响应，但同样的模型和 prompt 通过直接 curl 到 `generativelanguage.googleapis.com` 只需 2-3 秒。问题只在 `thinking` 启用时出现，症状类似 #64710。

根本原因是 Gemini 3.1 Pro Preview 在 thinking 阶段会发送只包含 `thoughtSignature` 而没有 text 内容的 parts，而 `selection.streamWithIdleTimeout` 中的 iterator wrapper 只在底层 iterator yield 时重置 idle timer。如果一个 chunk 只有 thoughtSignature 而没有实际 token，idle timer 不会重置，即使模型在持续产生 thinking frames，最终也会被误判为 idle超时。

**如何解决**：
在 transport stream 中对 `thoughtSignature`-only parts 发出 `thinking_signature` 事件以保持流活跃，并在收到这些 parts 时（在任何 text 到达之前）启动 thinking block。这防止了只有 thoughtSignature 的 chunk 被当作空闲流而触发 timeout。

---
### #250 Skills：增加 Agent 可见性检查

**更新原文**：
Skills: add agent visibility to skills check, make skills list/info align with agent allowlists, and add doctor coverage for unavailable skills. Fixes [#75983](https://github.com/openclaw/openclaw/pull/75983). Thanks [@andyliu](https://github.com/andyliu).

**关联 PR**：#75983（+902 / -41）

**痛点**：
`skills check` 命令之前无法针对特定 Agent 的 prompt/runtime 评估 skill 就绪状态，导致：
1. Skill 被列为 ready 但对该 Agent 实际不可用（因 agent allowlist 排除了它）
2. 无法区分模型可见/命令可见/prompt-hidden/agent-filtered/missing-requirement 等不同可见性状态
3. Doctor 无法自动处理默认 Agent 允许但实际不可用的 skills

**如何解决**：
PR 增加了三个主要功能：

1. **`openclaw skills check --agent <id>`**：可针对特定 agent 评估 skill 就绪性，输出 model-visible、command-visible、prompt-hidden、agent-filtered、missing-requirement 等分类

2. **`skills list`/`skills info` 与 agent allowlist 对齐**：被 agent 排除的 skills 不再显示为对该 agent ready

3. **Doctor 覆盖不可用 skills**：运行 `doctor --fix` 可通过 `skills.entries.<skill>.enabled=false` 自动禁用不可用 skills

同时更新了 `disable-model-invocation` 文档和 doctor/skills 行为文档。

---
### #251 MCP：安全检查与日志改进

**更新原文**：
MCP: tighten security checks and improve security logging.

**关联 PR**：无

**痛点**：
MCP（Model Context Protocol）通道的安全检查不够严格，可能导致恶意或不合规的请求通过。安全日志记录也不完善，出现安全相关问题时难以溯源和排查。

**如何解决**：
收紧 MCP 通道的安全检查门槛，增加安全日志的详细程度，使安全事件可追踪可排查。具体涉及 MCP 相关的 security checks 和 logging 模块改进。由于没有关联 PR/Issue，推测这是安全加固性质的改动，没有公开的安全披露需求。

---
### #252 Discord：规范化 Mention 格式文档

**更新原文**：
Discord: document canonical mention formatting in agent prompt hints and channel docs so outbound replies use `<@USER_ID>`, `<#CHANNEL_ID>`, and `<@&ROLE_ID>` instead of legacy nickname mentions. Fixes [#75173](https://github.com/openclaw/openclaw/pull/75173).

**关联 PR**：#75173

**痛点**：
OpenClaw Discord 集成在出站回复中使用的 mention 格式不统一。之前可能依赖 legacy nickname mention 格式，但 Discord 昵称可能为空、变更或未设置，导致 mention 无法正确解析，被提及的用户收不到通知。

Discord 官方推荐 canonical mention 格式：`<@USER_ID>`、`<#CHANNEL_ID>`、`<@&ROLE_ID>`，基于 snowflake ID，永不失效。

**如何解决**：
在 agent prompt hints 和 channel 文档中明确记录 canonical mention 格式的用法，引导出站回复统一使用 ID-based mention 格式，替代旧的 nickname mention。这是对话式 AI 与 Discord 交互的规范化要求，确保 mention 可靠触达。

---
### #253 Heartbeat Scheduler：集中冷却门控

**更新原文**：
Heartbeat scheduler: gate exec-event/notification/spawn/retry wakes through a centralized cooldown so backgrounded `process.start` exit notifications can no longer self-feed runaway heartbeat runs (configured `every: "30m"` was firing every ~10s in production, pegging the gateway event loop with `eventLoopDelayMaxMs >6s` spikes that stalled control-UI asset serving and TUI handshakes). Fixes [#64016](https://github.com/openclaw/openclaw/issues/64016). Thanks @hexsprite.

**关联 Issue**：#64016、#17797、#75436

**痛点**：
Heartbeat scheduler 存在致命的自我反馈循环（self-feeding feedback loop）：

- 用户配置 `every: "30m"` 的心跳，但实际约每 10 秒就触发一次
- 高频触发导致 `eventLoopDelayMaxMs >6s` 的事件循环阻塞峰值
- control-UI 资产服务和 TUI 握手被 stall
- 根因链条：`process.start` 后台退出通知 → 触发新的心跳 run → 心跳 run 内部的 exec-event/notification/spawn/retry 再次触发心跳 → 无限循环

这是一个级联触发的灾难性场景，在有多心跳任务的部署上极易触发。

**如何解决**：
引入集中化冷却（centralized cooldown）机制：
1. **exec-event/notification/spawn/retry 唤醒经过冷却门控**：这些间接唤醒路径不再无条件触发，而是经过冷却窗口
2. **明确的 wake-now 路径保持立即触发**：`manual`、`wake`、task completion、`/hooks/wake mode=now`、cron `--wake now` 等直接唤醒不走冷却
3. **busy skip 不再毒化冷却**：可重试的临时忙碌跳过不影响下次重试的冷却计时
4. **per-agent flood guard**：每 Agent 60 秒内最多 5 次运行，防止任何意外的反馈循环突破上限

---
### #254 GCloud：屏蔽 workspace CLOUDSDK_PYTHON 覆盖

**更新原文**：
fix: block workspace CLOUDSDK_PYTHON override and always set trusted interpreter for gcloud. Fixes [#74492](https://github.com/openclaw/openclaw/pull/74492). Thanks [@pgondhi987](https://github.com/pgondhi987).

**关联 PR**：#74492

**痛点**：
Google Cloud SDK（gcloud）通过 `CLOUDSDK_PYTHON` 环境变量指定 Python 解释器路径。如果 workspace 的 `.env` 或环境配置设置了这个变量：
1. gcloud 可能使用未经审核的 Python 解释器
2. 存在代码注入风险（恶意 Python 解释器可执行任意代码）
3. gcloud 命令行为不一致，难以排查

在需要调用 GCP 服务（Vertex AI、BigQuery 等）的部署上这是一个安全隐患。

**如何解决**：
在 workspace 环境加载时屏蔽 `CLOUDSDK_PYTHON` 变量覆盖，强制 gcloud 使用 OpenClaw 验证过的受信任 Python 解释器。这是安全加固改动，确保 gcloud 只在可信的 Python 运行时执行。

---
### #255 Providers/Z.AI：GLM 目录迁移至插件清单

**更新原文**：
Providers/Z.AI: move the bundled GLM catalog and auth env metadata into the plugin manifest, so `models list --all --provider zai` shows the full known catalog without duplicated runtime seed data. Thanks [@shakkhererd](https://github.com/shakkhererd).

**关联 PR**：无明确 PR 编号

**痛点**：
Z.AI（智谱 AI）提供商的 GLM 模型目录和认证环境元数据（`ZAI_API_KEY` 等）存储在 runtime seed data 中，导致：
1. `models list --all --provider zai` 显示的模型列表不完整或有重复数据
2. runtime seed data 与 plugin manifest 数据重复，维护困难
3. 更新模型目录需要修改 runtime 代码而非插件清单

**如何解决**：
将 GLM 模型目录和 auth env 元数据从 runtime seed data 迁移到 plugin manifest：
- `models list --all --provider zai` 可直接从 manifest 读取完整目录
- 消除与 runtime seed data 的重复
- 模型目录更新通过插件更新即可，无需修改 OpenClaw 运行时代码

---
### #256 Providers/Qianfan 和 Stepfun：插件清单声明认证元数据

**更新原文**：
Providers/Qianfan and Providers/Stepfun: declare setup auth metadata in the plugin manifest so onboarding and `models setup` surface the expected env var without falling back to legacy `providerAuthEnvVars` runtime seed data. Thanks [@shakkernerd](https://github.com/shakkernerd).

**痛点**：
百度千帆（Qianfan）和阶跃星辰（Stepfun）两个提供商的认证信息（`api-key` 方法和对应的环境变量名）之前依赖 legacy `providerAuthEnvVars` runtime seed data。这导致 onboarding 和 `models setup` 流程无法直接展示正确的环境变量名，需要回退到 legacy 数据，增加了复杂性。

**如何解决**：
在两个 provider 的 plugin manifest 中直接声明 `api-key` 认证方法和 `QIANFAN_API_KEY`/`STEPFUN_API_KEY` 环境变量，使 onboarding 和 `models setup` 可直接从 manifest 读取，不再依赖 runtime seed data。这与 #255（Z.AI GLM 目录迁移）同属 manifest 驱动的数据架构统一化。

---
### #257 基础设施：屏蔽 Homebrew 环境变量

**更新原文**：
fix(infra): block ambient Homebrew env vars from brew resolution. Fixes [#74463](https://github.com/openclaw/openclaw/pull/74463). Thanks [@pgondhi987](https://github.com/pgondhi987).

**关联 PR**：#74463

**痛点**：
Homebrew 会设置一系列环境变量（`HOMEBREW_PREFIX`、`HOMEBREW_CELLAR`、`HOMEBREW_CACHE` 等），这些 "ambient" 环境变量可能来自用户 shell 配置。当 OpenClaw 在 infra 层解析/调用 brew 时，这些变量可能指向错误的安装路径或在不同版本/架构间导致解析失败，干扰 OpenClaw 自身的依赖管理流程。

**如何解决**：
在 brew 解析过程中屏蔽 ambient Homebrew 环境变量，确保 brew 调用使用 OpenClaw 控制的环境而非用户 shell 继承的环境。这是基础设施层的环境隔离加固，与 #254（屏蔽 CLOUDSDK_PYTHON）、#260（屏蔽 .env system-path）同属一类安全/稳定性修复。

---
### #258 Onboarding：避免配置写入后全员插件依赖

**更新原文**：
Onboarding/configure: avoid staging every default plugin runtime dependency after config writes, so skipped setup flows only prepare config-selected plugin deps instead of pulling broad feature-plugin packages. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
在 onboarding 和 configure 流程中，每次配置写入后系统会无条件 staging 所有默认插件的运行时依赖。这意味着即使用户只选择了部分插件，跳过 setup 流程时仍要拉取全部默认插件的依赖包，浪费时间和带宽，增加首次安装时间和磁盘占用。

**如何解决**：
修改配置写入后的依赖 staging 逻辑：跳过的 setup 流程只准备用户配置中实际选中的插件依赖，而非所有默认插件。"Broad feature-plugin packages"不再被无条件拉取。这是一个用户体验和安装性能优化。

---
### #259 Thinking/Providers：轻量级策略工件解析 thinking profiles

**更新原文**：
Thinking/providers: resolve bundled provider thinking profiles through lightweight provider policy artifacts when startup-lazy providers are not active, so OpenAI Codex GPT-5.x keeps xhigh available in Gateway session validation. Fixes [#74796](https://github.com/openclaw/openclaw/issues/74796). Thanks @maxschachere.

**关联 Issue**：#74796

**痛点**：
OpenClaw 支持为 LLM 提供商配置 thinking profiles（minimal/low/medium/high/xhigh）。OpenAI Codex GPT-5.x 采用 startup-lazy 策略——启动时不立即激活，但这导致其 thinking profiles 无法在 Gateway 会话验证阶段被解析，xhigh 推理级别对用户不可用。

**如何解决**：
通过"轻量级 provider policy artifacts"解析 bundled provider 的 thinking profiles。这些 artifacts 不需要完整的 provider 运行时激活，因此 startup-lazy provider 也能在启动时提供 thinking profile 信息，确保 GPT-5.x 的 xhigh 在会话验证中可用。

---
### #260 安全/Windows：忽略 workspace .env 系统路径变量

**更新原文**：
Security/Windows: ignore workspace `.env` system-path variables and resolve stale-process `taskkill.exe` from the validated Windows install root, preventing repository-local env files from redirecting cleanup helpers. Thanks [@pgondhi987](https://github.com/pgondhi987).

**痛点**：
Windows 平台存在两个安全隐患：

1. **workspace `.env` 系统路径变量**：工作区 `.env` 文件中的 `PATH`、`SYSTEMROOT` 等系统路径变量覆盖可能被 OpenClaw 读取，导致工具解析被劫持到恶意路径。

2. **stale-process `taskkill.exe`**：OpenClaw 清理残留进程时调用 `taskkill.exe`，若从 PATH 解析可能找到仓库本地的伪造版本。

**如何解决**：
1. 在 Windows 上忽略 workspace `.env` 中的系统路径变量
2. 从经过验证的 Windows 安装根目录解析 `taskkill.exe`，而非从 PATH

这与 #254、#257 同属"环境变量隔离"安全加固系列，防止仓库级配置文件影响 OpenClaw 的核心工具调用。

---
### #261 CLI/Plugins：原地刷新持久化注册表策略

**更新原文**：
CLI/plugins: refresh persisted plugin registry policy in place for `plugins enable` and `plugins disable`, so routine toggles no longer rebuild and hash every plugin source when the target is already indexed. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
执行 `plugins enable` 或 `plugins disable` 时，即使目标插件已被索引，系统仍会重建并重新哈希每个插件的源文件，导致简单切换耗时过长，频繁切换插件配置时体验很差。

**如何解决**：
改为原地刷新（in-place refresh）持久化的插件注册表策略：目标插件已索引时跳过重建和哈希步骤，只更新需要变更的注册表条目。与 #263 属于同一批 CLI 插件管理性能优化。

---
### #262 Windows/Install：npm 临时目录与 Bedrock 依赖钉住

**更新原文**：
Windows/install: run npm from a writable installer temp directory and pin the Bedrock runtime dependency below a Windows ARM Node 24 npm resolver failure, so global OpenClaw installs no longer fail before onboarding. Thanks [@mariozechner](https://github.com/mariozechner).

**痛点**：
Windows 全局安装 OpenClaw 时遇到两个问题：
1. npm 在没有写权限的目录下运行导致依赖安装失败
2. Windows ARM + Node 24 组合下 npm resolver 有 bug，AWS Bedrock 运行时依赖无法正确解析

两者叠加导致全局安装在 onboarding 之前就失败。

**如何解决**：
1. 将 npm 改为从安装程序临时目录执行（有写权限）
2. 对 Bedrock 运行时依赖进行版本钉住，绕过 Node 24 npm resolver 的已知故障

---
### #263 CLI/Plugins：限定 slot 选择范围至目标插件

**更新原文**：
CLI/plugins: scope install and enable slot selection to the selected plugin manifest/runtime fallback, so plugin installs no longer load every plugin runtime or broad status snapshot just to update memory/context slots. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
执行 `plugins install` 或 enable 时，为了更新 memory/context slots，系统会加载所有插件运行时或生成完整的 status snapshot。这意味着安装单个插件时需要实例化所有已装插件，运行时间随插件数量增长。

**如何解决**：
将 slot 选择限定为目标插件的 manifest/runtime fallback，不再加载全部插件运行时。内存/上下文 slot 更新精确到目标插件，install/enable 操作不再受已装插件数量影响。

---
### #264 Plugins/TTS：冷启动路径语音提供商发现与运行时探测

**更新原文**：
Plugins/TTS: keep bundled speech-provider discovery available on cold package Gateway paths and add bundled plugin matrix runtime probes for health, readiness, RPC, TTS discovery, and post-ready runtime-deps watchdog coverage. Fixes [#75283](https://github.com/openclaw/openclaw/issues/75283). Thanks [@vincentkoc](https://github.com/vincentkoc).

**关联 Issue**：#75283

**痛点**：
TTS 插件在冷启动路径（cold package Gateway paths）下无法发现已捆绑的语音提供商，导致新安装或冷启动的 Gateway 无法使用 TTS 功能。同时运行时探测（health/readiness/RPC/TTS discovery）缺失，运行时依赖故障没有 watchdog 覆盖，TTS 问题难以及时发现。

**如何解决**：
1. 确保 bundled speech-provider 在冷启动路径下可被发现
2. 新增 bundled plugin matrix 运行时探测：health、readiness、RPC、TTS discovery
3. 增加 post-ready runtime-deps watchdog 覆盖

这是 TTS 功能可用性和可观测性的综合改进。

---
### #265 Google Meet/Twilio：doctor 诊断信息完善

**更新原文**：
Google Meet/Twilio: show delegated voice call ID, DTMF, and intro-greeting state in `googlemeet doctor`, and avoid claiming DTMF was sent when no Meet PIN sequence was configured. Fixes [#72478](https://github.com/openclaw/openclaw/issues/72478). Thanks [@DougButdorf](https://github.com/DougButdorf).

**关联 Issue**：#72478

**痛点**：
`googlemeet doctor` 命令的诊断输出不完整：
1. 缺少 delegated voice call ID、DTMF 状态和 intro-greeting 状态的展示，排查问题困难
2. 更严重的是：未配置 Meet PIN 序列时，系统仍声称"DTMF 已发送"，这是虚假确认，会误导用户以为 PIN 拨号成功

**如何解决**：
1. 在 `googlemeet doctor` 输出中增加 delegated voice call ID、DTMF 状态、intro-greeting 状态三项
2. 当没有配置 Meet PIN 序列时，不再输出"DTMF sent"确认，避免误导

这是诊断准确性和用户体验的修复，确保 doctor 输出可信。

---
### #266 Plugins/Tools：优先使用构建好的捆绑插件代码

**更新原文**：
Plugins/tools: prefer built bundled plugin code during tool discovery and skip channel runtime hydration while preserving companion provider registrations. Fixes [#75290](https://github.com/openclaw/openclaw/issues/75290). Thanks @thanos-openclaw.

**关联 Issue**：#75290

**痛点**：
工具发现过程中，每次都执行完整的 channel runtime hydration，即使插件已经有预构建的 bundled code。这导致：
- per-run plugin-tool prep cost 很高
- 重复加载已构建的插件代码
- 不必要的 hydration 浪费 CPU 和内存

**如何解决**：
工具发现时优先使用 built bundled plugin code，跳过不必要的 channel runtime hydration，同时保留 companion provider registrations，确保可执行插件工具不被丢弃。这是降低工具发现开销的性能优化。

---
### #267 Plugins/Loader：限定插件工具注册表复用范围

**更新原文**：
Plugins/loader: scope plugin-tool registry reuse to the enabled plugin plan and stored Gateway method keys, so embedded runner tool lookup can reuse compatible startup registries without hiding enabled non-startup plugin tools. Fixes [#75520](https://github.com/openclaw/openclaw/issues/75520). Thanks @whtoo.

**关联 Issue**：#75520

**痛点**：
插件工具注册表复用逻辑未正确限定范围，导致 embedded runner 复用 startup registries 时，已启用但非启动时加载的插件工具（enabled non-startup plugin tools）被隐藏，用户配置的工具在运行时找不到。

**如何解决**：
将 plugin-tool registry 复用限定为"已启用的插件计划 + 存储的 Gateway 方法键"，使 embedded runner 可安全复用兼容启动注册表，同时已启用非启动插件工具不被隐藏。

---
### #268 Voice Call/Twilio：通知模式 TwiML 直接嵌入

**更新原文**：
Voice Call/Twilio: send notify-mode initial TwiML directly in the outbound create-call request while keeping conversation and pre-connect DTMF calls webhook-driven, so one-shot notify calls do not depend on a first-answer webhook fetch. Fixes [#72758](https://github.com/openclaw/openclaw/pull/72758). Thanks @tyshepps.

**关联 PR**：#72758（被替代）

**痛点**：
Twilio 通知模式（one-shot notify）的初始 TwiML 之前依赖首次应答的 webhook 回调获取，导致：
- 单次通知通话需要两次网络请求（创建通话 + webhook 回调）
- webhook 失败则通知通话无法建立
- 增加不必要的延迟和故障点

对话模式和预连接 DTMF 通话本身需要 webhook 驱动没有问题。

**如何解决**：
通知模式的初始 TwiML 直接嵌入到 create-call 请求中，无需等待 webhook 回调。对话模式和预连接 DTMF 保持 webhook 驱动不变。这是可靠性和延迟优化。

---
### #269 Discord/Slack：状态反应清理延迟至运行最终化

**更新原文**：
Discord/Slack: defer status-reaction cleanup until run finalization so queued, thinking, tool, and terminal reactions no longer flicker during normal progress updates. Fixes [#75582](https://github.com/openclaw/openclaw/pull/75582).

**关联 PR**：#75582

**痛点**：
OpenClaw 在 Discord/Slack 中使用 emoji 反应展示 agent 运行状态（queued/thinking/tool/terminal 等）。之前的状态反应清理在每次进度更新时执行，导致正常进度期间反应频繁闪烁，带来视觉干扰和不必要的 API 调用。

**如何解决**：
将状态反应清理延迟到 run finalization 阶段：正常进度更新时保留状态反应，只在运行结束时清理。消除正常交互中的 emoji 闪烁。

---
### #270 Discord/Voice：综合语音频道修复

**更新原文**：
Discord/voice: leave voice off for text-only configs unless explicitly configured, rerun configured voice auto-join after gateway RESUMED events, ignore already-destroyed stale voice connections during reconnect cleanup, lengthen the default voice join Ready wait with configurable timeouts, merge configured media-understanding providers such as Deepgram into partial active registries, apply per-channel `systemPrompt` overrides to voice transcript turns, and run voice-channel turns under a voice-output policy that hides the agent `tts` tool. Fixes [#73753](https://github.com/openclaw/openclaw/issues/73753) [#40665](https://github.com/openclaw/openclaw/issues/40665) [#63098](https://github.com/openclaw/openclaw/issues/63098) [#65687](https://github.com/openclaw/openclaw/issues/65687) [#47095](https://github.com/openclaw/openclaw/issues/47095) [#61536](https://github.com/openclaw/openclaw/issues/61536). Thanks @sanchezm86, @SecureCloudProjO, @liz709, @darealgege, @kzicherman, @ayochim, @OneMintJulep, @qearlyao, and @aounakram.

**关联 Issues**：#73753、#40665、#63098、#65687、#47095、#61536（fix）；#74044、#39825、#65039（ref）

**痛点**：
Discord 语音频道存在 7 个子问题：
1. 纯文本配置误启语音
2. Gateway RESUMED 后语音自动加入丢失
3. 重连清理遇已销毁旧连接导致错误
4. 语音加入 Ready 等待超时过短
5. Deepgram 等媒体理解提供商未合并到活跃注册表
6. 频道级 systemPrompt 覆盖未应用到语音转录轮次
7. 语音输出模式下仍暴露 tts 工具

**如何解决**：
7 项子修复并行实施：纯文本默认关闭语音、RESUMED 后重新执行自动加入、忽略已销毁旧连接、延长并可配置语音加入超时、合并 Deepgram 等媒体提供商、应用 systemPrompt 覆盖到语音转录、语音频道在隐藏 tts 的策略下运行。涉及 6 个 fix issue、3 个 ref issue，9 位贡献者，是一次大型综合修复。

---
### #271 Plugins/CLI：复用冷启动 manifest 注册表

**更新原文**：
Plugins/CLI: reuse the cold manifest registry while building plugin status and inspect reports, so large configured plugin sets no longer rediscover the bundled/plugin registry once per inspect row. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
构建 plugin status 和 inspect 报告时，每个 inspect 行都重新发现 bundled/plugin 注册表。大型插件集下耗时极长（O(n) rediscovery），用户执行简单的 `plugins inspect` 要等待很久。

**如何解决**：
复用冷启动的 manifest registry 构建报告，将 O(n) 次发现降低为 O(1) 次。与 #261、#263 同属插件管理性能优化系列。

---
### #272 Gateway/Health：缓存快照与运行时状态同步

**更新原文**：
Gateway/health: refresh cached health RPC snapshots when channel runtime state diverges, so Discord and other channel status reads no longer report stale running or connected values until the cache TTL expires. Fixes [#75423](https://github.com/openclaw/openclaw/pull/75423).

**关联 PR**：#75423

**痛点**：
健康检查 RPC 使用缓存的频道状态快照，当实际状态变化时（如 Discord connected → disconnected），缓存值不会立即刷新，而是等 TTL 过期。这导致频道状态读取返回 stale 值，用户看到已连接但实际已断开。

**如何解决**：
检测频道运行时状态是否与缓存快照不一致，发现不一致时立即刷新 health RPC 快照，不再等待 TTL 过期。提高状态报告的实时性。

---
### #273 Gateway/Sessions：启动时跳过大规模维护操作

**更新原文**：
Gateway/sessions: keep session-store reads from running stale prune and entry-count cap maintenance during startup, so oversized stores no longer block chat history readiness after updates while writes and `sessions cleanup --enforce` still preserve the cleanup safeguards. Fixes [#70050](https://github.com/openclaw/openclaw/issues/70050). Thanks @tangda18.

**关联 Issue**：#70050

**痛点**：
Session store 每次读取执行 stale prune 和 entry-count cap maintenance。启动阶段，特别是更新后首次启动时，如果 store 过大，这些维护操作会长时间阻塞聊天历史就绪，用户更新后首次启动无法使用。

**如何解决**：
启动阶段的读取不再触发 stale prune 和 cap maintenance，写入和 `sessions cleanup --enforce` 仍保留清理保障。启动性能改善，清理 safeguard 不受影响。

---
### #274 安全/审计：常规审计跳过插件运行时

**更新原文**：
Security/audit: keep plain `security audit` on the cold config/filesystem path and reserve plugin runtime security collectors for `--deep`, so large plugin installs cannot execute every plugin runtime during routine audits. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
常规 `security audit` 命令会执行每个已安装插件的 runtime security collectors，导致：
- 大型插件集下审计耗时长、执行不需要的代码
- 审计过程自身存在安全隐患
- 对生产环境造成不必要的负载

**如何解决**：
常规 `security audit` 只使用 cold config/filesystem path，不执行插件运行时。插件 runtime security collectors 只在 `--deep` 深度审计模式下使用。日常审计轻量化，深度审计保留全部功能。

---
### #275 WhatsApp：QR 码依赖通过根镜像运行时依赖 staging

**更新原文**：
WhatsApp: stage `qrcode` through root mirrored runtime dependencies so packaged QR pairing can render from staged plugin-runtime-deps installs. Fixes [#75394](https://github.com/openclaw/openclaw/issues/75394). Thanks @FelipeX2001.

**关联 Issue**：#75394

**痛点**：
WhatsApp QR 码配对依赖 `qrcode` 库，但该依赖没有通过 root mirrored runtime dependencies 正确 staging，导致打包安装后 QR 码无法渲染，用户无法完成 WhatsApp 认证。

**如何解决**：
将 `qrcode` 依赖通过 root mirrored runtime dependencies 进行 staging，使打包后的 QR 配对可从 staged plugin-runtime-deps 渲染。WhatsApp QR 认证功能可用。

---
### #276 交互式频道载荷：为空载荷提供 fallback 文本

**更新原文**：
Interactive channel payloads: send Discord component-only interaction replies, Slack block-only slash replies, Telegram button/select fallback labels, and LINE quick-reply fallback option text instead of accepting empty renderable payloads. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
多平台交互式消息中，当回复只包含组件（按钮/选择器）而没有文本内容时，系统发送空 renderable payload，导致：
- Discord 组件只显示交互回复无法展示
- Slack block-only slash 回复被忽略
- Telegram 按钮/选择器缺少 fallback 标签
- LINE 快速回复缺少选项文本

**如何解决**：
四平台统一修复：Discord 组件交互补文本、Slack block-only 回复补文本、Telegram 按钮/选择器加 fallback 标签、LINE 快速回复加选项文本。不再接受空载荷。

---
### #277 Auto-reply/Docking：路由切换限制私聊发起

**更新原文**：
Auto-reply/docking: require `/dock-*` route switches to start from direct chats, so group or channel participants cannot reroute a shared session's future replies into a linked DM. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
`/dock-*` 命令可以切换 agent 回复的路由目标（停靠到指定 DM）。之前无发起位置限制，群组/频道参与者也能执行，导致共享会话的回复被劫持到某个 DM，存在安全风险。

**如何解决**：
`/dock-*` 必须从私聊发起，群组/频道参与者无法再重定向共享会话回复路由。与 #278、#279 同属会话路由安全修复。

---
### #278 Discord：文本 DM 路由固定到配置的 DM 所有者

**更新原文**：
Discord: keep text-DM main-session route updates pinned to the configured DM owner, matching component interactions so another direct-message sender cannot redirect future main-session replies. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
Discord 文本 DM 路由跟随最近消息发送者更新，而非配置的 DM 所有者。当多人与同一 bot DM 时，最近发送者可以劫持主会话回复路由。组件交互已有正确固定逻辑，文本 DM 的固定逻辑缺失。

**如何解决**：
将文本 DM 主会话路由更新固定到配置的 DM 所有者，与组件交互行为一致，防止非所有者劫持回复路由。与 #277、#279 同属会话路由安全修复。

---
### #279 Mattermost/Matrix：DM 主会话路由固定

**更新原文**：
Mattermost/Matrix: keep direct-message main-session route updates pinned to the configured DM owner so paired or temporarily allowed senders cannot redirect future shared-session replies. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
与 #278 相同的问题在 Mattermost/Matrix 上的表现。配对或临时允许的发送者可以在共享 DM 中发消息，更新主会话路由，导致共享会话回复被重定向到非预期用户。

**如何解决**：
将 Mattermost/Matrix DM 主会话路由固定到配置的 DM 所有者，配对/临时允许发送者无法劫持路由。安全修复系列 #277、#278、#279。

---
### #280 Discord：SecretRef bot token 在消息动作中的可发现性

**更新原文**：
Discord: keep SecretRef-backed bot tokens discoverable for message actions without resolving the token during schema generation, and resolve scoped channel SecretRefs before outbound agent message sends even when the tool is built from a config snapshot. Fixes [#75324](https://github.com/openclaw/openclaw/issues/75324). Thanks @slideshow-dingo and @Conan-Scott.

**关联 Issue**：#75324

**痛点**：
Discord bot token 通过 SecretRef 管理时存在两个问题：
1. schema 生成时尝试解析 SecretRef，不必要且不安全
2. 从 config snapshot 构建的工具发送出站消息时，未解析 scoped channel SecretRefs，导致消息发送失败

**如何解决**：
1. schema 生成阶段不解析 SecretRef token，保持消息动作中的可发现性
2. 出站消息发送前正确解析 scoped channel SecretRefs，即使工具从 config snapshot 构建

这是 SecretRef 生命周期管理和消息发送正确性的双重修复。

---
### #281 Updates：使用 managed Gateway service profile 运行 post-install doctor

**更新原文**：
Updates: run package post-install doctor repair with the managed Gateway service profile and state paths when a daemon is installed, so shell/profile mismatches no longer repair the caller state while the restarted Gateway keeps stale config. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
post-install doctor repair 使用调用者 shell profile 路径而非 Gateway daemon 的 service profile，导致 shell 不匹配时 doctor 修复了调用者状态，而 daemon 仍使用旧配置，更新实际上未生效。

**如何解决**：
daemon 安装时，post-install doctor repair 使用 managed Gateway service profile 和 state 路径，确保修复生效到 Gateway 而非调用者。与 #283 同属更新流程修复。

---
### #282 Models/DeepInfra：Manifest 目录发现与运行时回退

**更新原文**：
Models/DeepInfra: declare DeepInfra manifest catalog discovery and derive its runtime fallback catalog from the manifest, restoring provider-filtered `models list --all --provider deepinfra` rows without duplicated static model data. Thanks [@shakkernerd](https://github.com/shakkernerd).

**痛点**：
DeepInfra 模型目录存在于 manifest 和 runtime seed data 两处，存在重复且清理后未正确迁移，导致 `models list --all --provider deepinfra` 缺少 provider 过滤后的模型行。

**如何解决**：
在 manifest 中声明 DeepInfra catalog discovery，运行时 fallback 从 manifest 派生，恢复完整模型列表，消重。与 #255、#256、#282 同属 manifest 驱动数据架构统一化。

---
### #283 CLI/Update：验证已安装服务端口而非调用者端口

**更新原文**：
CLI/update: verify managed gateway restarts against the installed service port instead of the caller shell port, so package updates do not report a healthy daemon as failed when profiles use different gateway ports. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
更新命令验证 Gateway 健康状态时连接调用者 shell 端口而非已安装服务的实际端口。当 shell profile 和 daemon 使用不同端口时，健康检查连错端口，实际健康的 daemon 被误报为失败。

**如何解决**：
验证 managed Gateway 重启时连接已安装服务端口，使不同 profile 使用不同端口时健康检查准确。配合 #281 确保更新流程可靠。

---
### #284 Gateway/Agent：严格模式下缺失投递目标提前拒绝

**更新原文**：
Gateway/agent: reject strict `openclaw agent --deliver` requests with missing delivery targets before starting the agent run, so users do not wait for a completed turn that cannot send anywhere. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
`openclaw agent --deliver` 未指定有效投递目标时，不提前报错而是完整运行 agent 后才发现无法投递。用户浪费等待时间却得不到任何输出。

**如何解决**：
启动 agent 运行前严格检查 `--deliver` 请求的投递目标有效性，缺失或无效立即拒绝并报错，不等运行完成才发现无法投递。

---
### #285 Setup/Import：正确处理非交互 --import-from

**更新原文**：
Setup/import: honor non-interactive `--import-from` onboarding flags by running the migration import path instead of silently completing normal setup without importing anything. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
非交互模式下使用 `--import-from` 标志进行 onboarding 时，系统忽略该标志，静默执行普通 setup 而不执行任何导入。自动化部署脚本中迁移数据未被导入，用户误以为导入成功。

**如何解决**：
非交互模式下正确识别 `--import-from` 标志，执行迁移导入路径而非普通 setup，确保自动化部署场景下导入操作正常执行。

---
### #286 Doctor/Plugins：纯检查不安装依赖

**更新原文**：
Doctor/plugins: keep plain `doctor --non-interactive` from installing bundled plugin runtime dependencies, so headless health checks report missing deps while `doctor --fix` remains the explicit repair path. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
`doctor --non-interactive` 在检查过程中自动安装缺失的 bundled plugin 运行时依赖，导致健康检查变成修改操作：
- CI/CD 中的检查意外触发依赖安装
- 违反"检查不应修改系统"原则
- 绕过 `doctor --fix` 作为显式修复路径的设计

**如何解决**：
纯 `doctor --non-interactive` 只报告缺失依赖不安装，`doctor --fix` 负责显式修复和安装。保持健康检查的只读语义。

---
### #287 Doctor/Gateway：交互式确认后才安装服务

**更新原文**：
Doctor/gateway: require an interactive confirmation before installing or rewriting the Gateway service, so `doctor --fix --non-interactive` can repair plugin/config drift without replacing the operator's launchd/systemd service from a temporary environment. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
`doctor --fix --non-interactive` 可能在临时环境（SSH会话、CI容器）中运行，意外替换生产环境的 launchd/systemd 服务配置。运维人员精心配置的 service 被覆盖，且 `--non-interactive` 模式下无确认机会。

**如何解决**：
安装或重写 Gateway 服务前要求交互式确认。`doctor --fix --non-interactive` 可修复 plugin/config 漂移，但不替换 service 配置。保护生产服务配置不被临时环境操作覆盖。

---
### #288 Plugins/runtime-deps：缓存键包含 OpenClaw 身份标识

**更新原文**：
Plugins/runtime-deps: include packaged OpenClaw identity in bundled plugin loader cache keys, so same-path package upgrades stop reusing stale versioned runtime-deps mirrors. Fixes [#75045](https://github.com/openclaw/openclaw/issues/75045). Thanks @sahilsatralkar.

**关联 Issue**：#75045

**痛点**：
Bundled plugin loader 的 cache keys 不包含 OpenClaw 打包身份标识，导致同一路径升级后仍复用旧版本缓存。升级后插件使用旧版本运行时依赖，版本化的 runtime-deps mirrors 未被刷新。

**如何解决**：
在 cache keys 中包含 OpenClaw 打包身份标识（版本号、hash），使同一路径升级后 cache key 变化，不再复用旧缓存。配合 #290 修复升级后的缓存清理问题。

---
### #289 Plugin SDK：恢复已弃用表面的辅助函数

**更新原文**：
Plugin SDK: restore reply-prefix and reply-pipeline helpers on the deprecated root/compat SDK surface so external plugins still using `openclaw/plugin-sdk` do not fail message dispatch after update. Fixes [#75171](https://github.com/openclaw/openclaw/issues/75171). Thanks @zhangxiliang.

**关联 Issue**：#75171

**痛点**：
Plugin SDK 重构将 `reply-prefix` 和 `reply-pipeline` 从 root/compat SDK 表面移除，但外部插件仍使用旧 API。更新后这些插件在消息分发时失败，破坏性变更影响第三方插件生态。

**如何解决**：
在已弃用的 root/compat SDK 表面恢复 `reply-prefix` 和 `reply-pipeline` 辅助函数，维护向后兼容性，防止已有插件更新后断裂。

---
### #290 Plugins/runtime-deps：修复后清理旧版本缓存

**更新原文**：
Plugins/runtime-deps: prune inactive same-package versioned runtime-deps roots after bundled dependency repair, so upgrades do not leave old `openclaw-<version>-<hash>` package caches behind after doctor runs. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
doctor 修复 bundled dependency 后不清理旧版本的 runtime-deps 根目录，旧目录以 `openclaw-<version>-<hash>` 格式累积。每次升级后磁盘堆积废弃缓存，多次升级后浪费大量磁盘空间。

**如何解决**：
bundled dependency repair 后自动修剪不活跃的同包版本化 runtime-deps 根目录，清理旧的包缓存，只保留当前活跃版本。配合 #288 确保升级后缓存正确刷新和清理。

---
### #291 Plugins/runtime-deps：修复时清理遗留版本化依赖根

**更新原文**：
Plugins/runtime-deps: prune legacy version-scoped plugin runtime-deps roots during bundled dependency repair and cover the path in Package Acceptance's upgrade-survivor matrix, so upgrades from 2026.4.x no longer leave stale per-plugin runtime trees after doctor runs. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
从 2026.4.x 升级时，旧版 per-plugin runtime-deps 目录树未被 doctor 修复清理，Package Acceptance 的升级存活矩阵也未覆盖此路径，导致升级后磁盘残留遗留目录。

**如何解决**：
修复过程中自动修剪 legacy version-scoped 运行时依赖根目录，并在升级存活矩阵中增加测试覆盖。与 #290 配合确保所有升级路径的清理。

---
### #292 Plugins/runtime-deps：启动后验证模式不触发包管理器修复

**更新原文**：
Plugins/runtime-deps: keep Gateway startup plugin imports and runtime plugin fallback loads verify-only after startup/config repair planning, so packaged installs no longer spawn package-manager repair from hot paths after readiness. Fixes [#75283](https://github.com/openclaw/openclaw/issues/75283) and [#75069](https://github.com/openclaw/openclaw/issues/75069). Thanks @brokemac79 and @xiaohuaxi.

**关联 Issues**：#75283、#75069

**痛点**：
Gateway 启动后，plugin imports 和 runtime plugin fallback loads 在某些情况下会触发 package-manager repair，从热路径（hot paths）执行修复，导致：
- 打包安装后，ready 状态下意外触发包管理器修复
- 性能抖动和不可预测行为

**如何解决**：
启动后 plugin imports 和 runtime plugin fallback loads 保持验证模式（verify-only），不再执行修复操作。确保打包安装后不会从热路径触发包管理器修复。

---
### #293 Plugins/runtime-deps：package.json manifest 作为超集不再重复 staging

**更新原文**：
Plugins/runtime-deps: treat package.json runtime-deps manifests as supersets when generated materialization metadata is absent, so bundled plugin activation stops restaging already-installed dependency subsets on every activation. Fixes [#75429](https://github.com/openclaw/openclaw/issues/75429). Thanks @loyur.

**关联 Issue**：#75429

**痛点**：
每次插件激活时都会重复 staging 已安装的依赖子集，因为系统没有正确识别 package.json 中的 runtime-deps manifest 是超集。每次激活都重新拉取/验证依赖，造成不必要的性能开销。

**如何解决**：
当生成的 materialization metadata 不存在时，将 package.json runtime-deps manifests 视为超集，已安装的依赖子集不再重复 staging。减少插件激活的重复工作。

---
### #294 iMessage：EPIPE 错误正确拒绝而非崩溃

**更新原文**：
iMessage: add stdin write callback and error listener to IMessageRpcClient so async EPIPE from a closed child process rejects the pending request instead of crashing the gateway with uncaughtException. Fixes [#75438](https://github.com/openclaw/openclaw/issues/75438).

**关联 Issue**：#75438

**痛点**：
iMessage RPC 客户端在子进程关闭时，如果写入 stdin 触发 async EPIPE 错误，pending request 没有被正确拒绝，而是导致 gateway 以 uncaughtException 崩溃。这是一个潜在的未处理异常。

**如何解决**：
为 IMessageRpcClient 添加 stdin write callback 和 error listener，当发生 EPIPE 时正确 reject pending request 而非抛出未捕获异常。防止 gateway 因未处理 EPIPE 而崩溃。

---
### #295 MCP/stdio：write 回调后 settle send() 而非立即 resolve

**更新原文**：
MCP/stdio: settle MCP stdio transport send() from the write callback instead of resolving immediately on buffer acceptance, so async write errors reject the promise instead of being lost. Fixes [#75438](https://github.com/openclaw/openclaw/issues/75438).

**关联 Issue**：#75438

**痛点**：
MCP stdio transport 的 `send()` 在 buffer 被接受后立即 resolve，而非等待 write callback 完成。当 write 过程中发生 async 错误时，promise 已经 resolve，错误被丢失而非被正确 reject。

**如何解决**：
从 write callback 中 settle `send()` promise，而非在 buffer 接受时立即 resolve，使 async write 错误能正确 reject promise 而非静默丢失。与 #294 同属异常处理正确性修复（均关联 #75438）。

---
### #296 Process/Exec：EPIPE 错误被吞没而非逃逸

**更新原文**：
Process/exec: add stdin error listener in runCommandWithTimeout so EPIPE from a prematurely-exited child is swallowed instead of escaping to uncaughtException. Refs [#75438](https://github.com/openclaw/openclaw/issues/75438).

**关联 Issue**：#75438

**痛点**：
与 #294 类似的问题。子进程提前退出时，向其 stdin 写入可能触发 EPIPE 错误。之前没有 error listener，EPIPE 错误会逃逸到 uncaughtException，导致进程崩溃。

**如何解决**：
在 `runCommandWithTimeout` 中添加 stdin error listener，当 EPIPE 发生时吞没错误而非传播为未捕获异常。防止进程因 EPIPE 而崩溃。这是异常处理的防御性加固。

---
### #297 Voice Call/realtime：咨询模式快速上下文

**更新原文**：
Voice Call/realtime: add default-off fast memory/session context for `openclaw_agent_consult`, giving live calls a bounded answer-or-miss path before the full agent consult. Fixes [#71849](https://github.com/openclaw/openclaw/issues/71849). Thanks @amzzzzzzz.

**关联 Issue**：#71849

**痛点**：
实时语音通话中，`openclaw_agent_consult` 功能在执行完整的 agent consult 之前，没有快速上下文机制。当通话需要即时响应时，系统会等待完整的 agent consult 完成，可能错过最佳回复时机。

**如何解决**：
为 `openclaw_agent_consult` 添加可选的快速 memory/session 上下文路径（默认关闭），在完整 agent consult 之前提供有界的"答或不答"决策路径。这为 live calls 提供了一个轻量级快速响应通道。

---
### #298 Google Meet：本地 barge-in 打断实时提供程序输出

**更新原文**：
Google Meet: interrupt Realtime provider output when local barge-in clears playback, so command-pair audio stops model speech instead of only restarting Chrome playback. Fixes [#73850](https://github.com/openclaw/openclaw/issues/73850). Thanks @shhtheonlyperson.

**关联 Issue/PR**：#73850 / #73834

**痛点**：
Google Meet 中，当本地 barge-in（插话打断）清除播放时，命令音频（command-pair audio）只重启 Chrome 播放，而非停止模型语音输出。这意味着用户插话后，模型仍在继续说话，造成混乱。

**如何解决**：
当检测到本地 barge-in 清除播放时，Realtime provider 的输出被中断，命令音频可以真正停止模型语音，而不只是重启 Chrome 播放。

---
### #299 Gateway/Config：限制插件拥有schema大小

**更新原文**：
Gateway/config: cap oversized plugin-owned schemas in the full `config.schema` response so large installed plugin sets cannot balloon Gateway RSS or crash schema clients. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
`config.schema` 响应包含所有插件拥有的 schema，大型插件集会导致 schema 响应过大：
- Gateway RSS 内存膨胀
- schema 客户端可能因处理过大响应而崩溃

**如何解决**：
对 `config.schema` 响应中的插件拥有 schema 设置上限（cap oversized schemas），防止大型插件集撑爆内存或导致客户端崩溃。这是内存保护和数据传输安全措施。

---
### #300 Plugins/Update：跳过比捆绑版本旧的市场插件更新

**更新原文**：
Plugins/update: skip ClawHub and marketplace plugin updates when the bundled version is newer than the recorded installed version, so `openclaw update` no longer overwrites working bundled plugins with older external packages. Fixes [#75447](https://github.com/openclaw/openclaw/issues/75447). Thanks @amknight.

**关联 Issue**：#75447

**痛点**：
`openclaw update` 命令会检查 ClawHub 和 marketplace 上的插件更新。当捆绑版本比已安装的外部包版本更新时，更新命令仍然会用旧的外包覆盖正常工作的捆绑插件。用户更新后发现功能异常，因为稳定版本被旧版本覆盖。

**如何解决**：
比较捆绑版本和 marketplace 记录的已安装版本，当捆绑版本更新时跳过 marketplace 更新。确保 `openclaw update` 不会覆盖正常工作的捆绑插件为旧版本。

---
### #301 Gateway/Sessions：有界尾读取和大批量 title hydration 上限

**更新原文**：
Gateway/sessions: use bounded tail reads for sessions-list transcript usage fallbacks and cap bulk title/last-message hydration, keeping large session stores responsive when rows request derived previews. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
大型 session store 中，当 rows 请求 derived previews（如 transcript usage、title、last-message）时，系统执行无限制的全量读取，导致响应时间变慢甚至事件循环饥饿。无节制的 bulk hydration 操作伤害大型 session store 的响应性。

**如何解决**：
对 sessions-list transcript usage 使用有界尾读取（bounded tail reads），并对 bulk title/last-message hydration 设置上限。保持大型 session store 的响应性，防止读取操作无节制消耗资源。

---
### #302 Gateway/Sessions：大批量转录标题/预览水合时让出事件循环

**更新原文**：
Gateway/sessions: yield during bulk transcript title/preview hydration and copy compaction checkpoints asynchronously, keeping the Gateway event loop responsive for large session stores and large transcripts. Fixes [#75330](https://github.com/openclaw/openclaw/issues/75330) and [#75414](https://github.com/openclaw/openclaw/issues/75414). Thanks @amknight.

**关联 Issues**：#75330、#75414

**痛点**：
大批量 transcript title/preview 水合和 copy compaction checkpoints 执行时同步阻塞事件循环，导致大型 session store 和大型 transcripts 场景下 Gateway 无响应。

**如何解决**：
在水合和压缩检查点期间主动让出（yield）事件循环，异步执行 copy compaction checkpoints，保持事件循环响应。改善大型 session 的交互响应性。

---
### #303 Gateway/Sessions：流式有界转录读取

**更新原文**：
Gateway/sessions: stream bounded transcript reads for session detail, history, artifacts, compaction, and send/subscribe sequence paths so small Gateway requests no longer materialize large transcripts or OOM on oversized session logs. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
Gateway 请求（如 session detail、history、artifacts）需要读取大型 transcripts 时，会将整个 transcript 具体化到内存中。这导致小请求触发大内存占用，超大 session logs 场景下 OOM。

**如何解决**：
对 session detail、history、artifacts、compaction、send/subscribe 等路径实施流式有界读取，小请求不再需要物化整个 transcript。防止 OOM，保持内存可控。

---
### #304 Gateway/Chat：有界聊天历史转录读取

**更新原文**：
Gateway/chat: bound chat-history transcript reads to the requested display window so large session logs no longer OOM the Gateway when clients ask for a small history page. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
客户端请求小范围历史页面时（如只查看最近 20 条），Gateway 仍读取整个 session log 的 transcript 到内存。大型 session logs 场景下导致 OOM。

**如何解决**：
将聊天历史转录读取限制为请求的显示窗口范围（requested display window），小页面请求不再触发大内存读取。与 #303 同属 session transcript 内存保护系列。

---
### #305 BlueBubbles：UTI 感知的音频附件检测

**更新原文**：
BlueBubbles: detect audio attachments by Apple UTIs (`public.audio`, `public.mpeg-4-audio`, `com.apple.m4a-audio`, `com.apple.coreaudio-format`) in addition to `audio/*` MIME, so iMessage voice notes whose webhook payload only carries the UTI are now classified as audio in the inbound `<media:audio>` placeholder instead of falling through to the generic `<media:attachment>` tag. Thanks @omarshahine.

**痛点**：
iMessage 语音笔记（voice notes）通过 webhook 投递时，部分 payload 只携带 Apple UTI（统一类型标识符）而不带 `audio/*` MIME 类型。之前只检查 MIME 类型，导致这些语音笔记被错误分类为通用 `<media:attachment>` 而非音频。

**如何解决**：
BlueBubbles 附件检测除 MIME 类型外还识别 Apple UTIs（`public.audio`、`public.mpeg-4-audio`、`com.apple.m4a-audio`、`com.apple.coreaudio-format`），使 iMessage 语音笔记正确路由到 `<media:audio>` 占位符。

---
### #306 Voice Call/Twilio：预连接 TwiML 和对话外 DTMF 拒绝

**更新原文**：
Voice Call/Twilio: honor stored pre-connect TwiML before realtime webhook shortcuts and reject DTMF sequences outside conversation mode, so Meet PIN entry cannot be skipped or silently dropped. Thanks [@donkeykong91](https://github.com/donkeykong91) and [@PfanP](https://github.com/PfanP).

**痛点**：
Twilio 语音通话中：
1. 预连接 TwiML（pre-connect TwiML）在 realtime webhook 快捷方式之前未正确处理，导致 Meet PIN 输入流程被跳过或静默丢弃
2. 对话模式外的 DTMF 序列未被拒绝，导致用户可能在不该输入 PIN 时输入 PIN

**如何解决**：
1. 在 realtime webhook 快捷方式之前正确处理存储的预连接 TwiML
2. 对话模式外拒绝 DTMF 序列，确保 Meet PIN 输入只能发生在正确的对话阶段

---
### #307 Docs/Sandboxing：源码检出脚本与 npm 安装用户的 docker build

**更新原文**：
Docs/sandboxing: clarify that sandbox setup scripts are only available from a source checkout, and add inline `docker build` commands for npm-installed users so sandbox image setup works without cloning the repo. Fixes [#75485](https://github.com/openclaw/openclaw/issues/75485). Thanks @amknight.

**关联 Issue**：#75485

**痛点**：
沙箱设置脚本（`sandbox-setup.sh` 等）文档不清晰，用户不清楚这些脚本只从源码检出获得。npm 安装的用户没有源码，无法运行这些脚本，但文档未说明替代方案，导致沙箱镜像设置失败。

**如何解决**：
1. 文档明确这些脚本只从源码检出可用
2. 为 npm 安装用户添加内联的 `docker build` 命令，使沙箱镜像设置无需克隆整个仓库

---
### #308 Google Meet/Voice Call：TwiML 更新时序与 Meet 问候语

**更新原文**：
Google Meet/Voice Call: play Twilio Meet DTMF before opening the realtime media stream and carry the intro as the initial Voice Call message, so the greeting is generated after Meet admits the phone participant instead of racing a live-call TwiML update. Thanks [@donkeykong91](https://github.com/donkeykong91) and [@PfanP](https://github.com/PfanP).

**痛点**：
Google Meet 语音通话中，DTMF 按键时序和问候语生成存在竞态：
1. DTMF 在 realtime media stream 打开之后才播放，与 TwiML 更新竞争
2. 问候语在 Meet 允许电话参与者之前生成，导致时序错误

**如何解决**：
1. 在打开 realtime media stream **之前**播放 Twilio Meet DTMF
2. 将 intro 作为初始 Voice Call 消息携带，确保问候语在 Meet admit 电话参与者之后生成

---
### #309 Google Meet/Voice Call：Twilio preflight 验证与 Webhook URL 限制

**更新原文**：
Google Meet/Voice Call: make Twilio setup preflight honor explicit `--transport twilio` and fail local/private Voice Call webhook URLs, including IPv6 loopback and unique-local forms, before joins. Thanks [@donkeykong91](https://github.com/donkeykong91) and [@PfanP](https://github.com/PfanP).

**痛点**：
Twilio setup preflight 检查未正确处理以下情况：
1. 未识别显式的 `--transport twilio` 参数
2. 允许本地/私有的 Voice Call webhook URLs（IPv6 loopback、unique-local 等），这是安全风险——webhook URL 不应暴露私有地址

**如何解决**：
Twilio setup preflight 在 joins 之前验证：
1. 正确处理 `--transport twilio` 参数
2. 拒绝本地/私有 webhook URLs（包括 IPv6 loopback 和 unique-local 格式），确保 webhook URL 公开可访问

---
### #310 Voice Call/Twilio：TwiML 更新重试与应答路径错误处理

**更新原文**：
Voice Call/Twilio: retry transient 21220 live-call TwiML updates and catch answered-path initial-greeting failures, so a fast answered callback no longer crashes the Gateway or drops the Twilio greeting/listen transition. Fixes [#74606](https://github.com/openclaw/openclaw/pull/74606). Thanks @Sivan22.

**关联 PR**：#74606

**痛点**：
Twilio 语音通话中两个问题：
1. 瞬态 21220 错误（live-call TwiML 更新失败）未重试，导致通话中间 TwiML 更新丢失
2. 应答路径的 initial-greeting 失败未捕获，快速应答回调导致 Gateway 崩溃或丢失 Twilio greeting/listen 转换

**如何解决**：
1. 对 21220 transient 错误实施重试机制
2. 捕获 answered-path initial-greeting 失败，防止 Gateway 崩溃和 greeting/listen 转换丢失

---
### #311 CLI/Startup：保留 OPENCLAW_HIDE_BANNER 并阻止依赖修复

**更新原文**：
CLI/startup: preserve `OPENCLAW_HIDE_BANNER` banner suppression for route-first startup callers that rely on the default process environment while keeping read-only status/channel paths from repairing bundled plugin runtime dependencies. Refs [#75183](https://github.com/openclaw/openclaw/pull/75183).

**关联 PR**：#75183

**痛点**：
CLI 启动时，`OPENCLAW_HIDE_BANNER` 环境变量用于禁止显示启动 banner，但 route-first 启动调用者依赖默认进程环境时该变量可能丢失。同时，只读状态/频道路径不应触发 bundled plugin runtime dependencies 修复。

**如何解决**：
1. 保留 route-first startup caller 的 `OPENCLAW_HIDE_BANNER` 设置
2. 保持只读 status/channel 路径不触发 bundled plugin runtime dependencies 修复

---
### #312 Voice Call/Twilio：流注册与 STT 就绪后才说问候语

**更新原文**：
Voice Call/Twilio: register accepted media streams immediately but wait for realtime transcription readiness before speaking the initial greeting, so reconnect grace handling stays live while OpenAI STT startup is no longer starved by TTS. Fixes [#75197](https://github.com/openclaw/openclaw/issues/75197). Thanks [@donkeykong91](https://github.com/donkeykong91) and [@PfanP](https://github.com/PfanP).

**关联 Issue/PR**：#75197 / #75257

**痛点**：
Twilio 语音通话中，媒体流注册后立即开始 TTS 问候语，但此时 OpenAI STT 还未就绪，导致：
- TTS 抢占 STT 启动资源，STT startup 饥饿
- 重连时 grace handling 无法保持活跃

**如何解决**：
1. 立即注册已接受的媒体流
2. 等待 realtime transcription 就绪后才开始说初始问候语
3. 确保 OpenAI STT 不会被 TTS 抢占资源

---
### #313 Voice Call CLI：operation-id polling 处理长对话轮次

**更新原文**：
Voice Call CLI: run gateway-delegated `voicecall continue` through operation-id polling and protocol-shaped errors, so long conversational turns keep their transcript result without blocking a single Gateway RPC. Fixes [#75459](https://github.com/openclaw/openclaw/pull/75459). Thanks @serrurco and @DougButdorf.

**关联 PR**：#75459

**痛点**：
`voicecall continue` 长对话轮次需要在 Gateway 中保持 transcript 结果，但之前的设计可能阻塞单个 Gateway RPC，导致长时间等待或结果丢失。

**如何解决**：
通过 operation-id polling 和协议形状的错误处理来运行 `voicecall continue`，使长对话轮次保持 transcript 结果而不阻塞单个 Gateway RPC。

---
### #314 Voice Call CLI：委托运行中 Gateway 运行时并跳过 webhook 启动

**更新原文**：
Voice Call CLI: delegate operational `voicecall` commands to the running Gateway runtime and skip webhook startup during CLI-only plugin loading, preventing webhook port conflicts and `setup --json` hangs. Fixes [#72345](https://github.com/openclaw/openclaw/issues/72345). Thanks @serrurco and @DougButdorf.

**关联 Issue**：#72345

**痛点**：
CLI-only 模式下加载插件时，voicecall 命令仍尝试启动 webhook，导致：
- webhook 端口冲突（Gateway 已在运行）
- `setup --json` 命令挂起

**如何解决**：
1. 将 operational `voicecall` 命令委托给运行中的 Gateway runtime
2. CLI-only 插件加载时跳过 webhook 启动
3. 防止端口冲突和 setup --json 挂起

---
### #315 Agents/pi-embedded-runner：提取 abortable provider-call 包装器

**更新原文**：
Agents/pi-embedded-runner: extract the `abortable` provider-call wrapper from `runEmbeddedAttempt` to module scope so its promise handlers no longer close over the run lexical context, releasing transcripts, tool buffers, and subscription callbacks when a provider call hangs past abort. Fixes [#74182](https://github.com/openclaw/openclaw/issues/74182). Thanks @cjboy007.

**关联 Issue**：#74182

**痛点**：
`runEmbeddedAttempt` 中的 abortable provider-call wrapper 的 promise handlers 闭包了 run 的词法上下文（lexical context）。当 provider call 在 abort 之后仍然挂起时，这些资源（transcripts、tool buffers、subscription callbacks）无法被释放，导致内存泄漏。

**如何解决**：
将 abortable provider-call wrapper 提取到模块作用域（module scope），使其 promise handlers 不再闭包 run 词法上下文。当 provider call 超时挂起时，资源能正确释放。

---
### #316 Docker：slim-runtime 后恢复 python3

**更新原文**：
Docker: restore `python3` in the gateway runtime image after the slim-runtime switch. Fixes [#75041](https://github.com/openclaw/openclaw/issues/75041).

**关联 Issue**：#75041

**痛点**：
切换到 slim runtime 镜像后，`python3` 从 gateway runtime image 中移除。但某些依赖（如 gcloud SDK 等）仍需要 python3 运行，导致这些功能不可用。

**如何解决**：
在 slim-runtime 切换后恢复 `python3`，确保需要 Python 的工具和集成能正常工作。

---
### #317 Agents/Session-repair：恢复会话在中断后不再报 400 错误

**更新原文**：
Agents/session-repair: fix resumed sessions failing with repeated 400 errors on Anthropic and strict OpenAI-compatible providers (Qwen, mlx-vlm) after an interrupted conversation or blank user input. Fixes [#75271](https://github.com/openclaw/openclaw/issues/75271) and [#75313](https://github.com/openclaw/openclaw/issues/75313). Thanks @amknight.

**关联 Issues**：#75271、#75313

**痛点**：
恢复的会话（resumed sessions）在以下情况后对 Anthropic 和严格的 OpenAI 兼容提供商（Qwen、mlx-vlm）重复返回 400 错误：
- 对话被中断
- 用户输入空白

这导致恢复会话无法正常工作，用户被迫开始新会话而非继续之前的上下文。

**如何解决**：
修复 session-resume 逻辑中对空白/中断输入的处理，确保恢复的会话发送有效的请求给所有兼容提供商，不再触发 400 错误。

---
### #318 CLI/Voice Call：voicecall 命令激活限定范围

**更新原文**：
CLI/Voice Call: scope `voicecall` command activation to the Voice Call plugin so setup and smoke checks no longer broad-load unrelated plugin runtimes or hang after printing JSON. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
`voicecall` 命令激活时，会 broad-load 不相关的插件运行时，导致：
- setup 和 smoke checks 耗时过长
- JSON 打印后挂起
- 资源浪费在不相关的插件加载上

**如何解决**：
将 `voicecall` 命令激活限定为 Voice Call 插件，不再 broad-load 不相关的 plugin runtimes。setup 和 smoke checks 聚焦于目标插件。

---
### #319 Doctor/Plugins：警告限制性 allow 与通配符工具 allowlist 组合

**更新原文**：
Doctor/plugins: warn when restrictive `plugins.allow` is paired with wildcard or plugin-owned tool allowlists, making the exclusive plugin allowlist behavior visible before users hit empty callable-tool runs. Fixes [#58009](https://github.com/openclaw/openclaw/issues/58009) and [#64982](https://github.com/openclaw/openclaw/issues/64982). Thanks @KR-Python and @BKF-Gitty.

**关联 Issues**：#58009、#64982

**痛点**：
当限制性的 `plugins.allow` 与通配符或插件自有的工具 allowlist 组合时，行为不透明：
- 用户以为配置允许了工具，但实际因插件 allowlist 限制导致可调用工具为空
- 运行返回空结果时，用户不知道为什么
- 问题在运行后很久才暴露

**如何解决**：
Doctor 在检测到这种组合时发出警告，使限制性 allowlist 行为在用户遇到空可调用工具运行之前就可见。提前警告，而非事后诸葛。

---
### #320 Google Meet/Voice Call：保持对话模式并复用 realtime intro prompt

**更新原文**：
Google Meet/Voice Call: keep Twilio Meet joins in conversation mode and reuse the realtime intro prompt when no voice-call-specific intro is configured, so answered phone bridge calls speak instead of joining silently. Fixes [#72478](https://github.com/openclaw/openclaw/issues/72478). Thanks @DougButdorf.

**关联 Issue**：#72478

**痛点**：
Twilio Meet 加入时未正确保持对话模式，且当未配置 voice-call-specific intro 时，复用了不合适的 realtime intro prompt。导致已应答的电话桥接呼叫静默加入而非说话。

**如何解决**：
1. 保持 Twilio Meet 加入时的对话模式
2. 无 voice-call-specific intro 时复用 realtime intro prompt
3. 已应答电话桥接呼叫能正常说话而非静默加入

---
### #321 Auto-reply/Group Chats：message 工具在群聊可见回复中的可用性

**更新原文**：
Auto-reply/group chats: keep the `message` tool available for message-tool-only visible replies and apply group-scoped tool policy before deciding fallback delivery, so Discord/Slack-style rooms reply visibly in the correct channel after upgrades. Fixes [#74842](https://github.com/openclaw/openclaw/issues/74842). Thanks @davelutztx and @aa-on-ai.

**关联 Issue**：#74842

**痛点**：
群聊中，当回复只对 message-tool 可见时，`message` 工具应可用。之前 group-scoped 工具策略在决定 fallback 投递之前未正确应用，导致 Discord/Slack 风格房间升级后在错误频道回复或不可见。

**如何解决**：
1. 保持 `message` 工具在 message-tool-only 可见回复中可用
2. 在决定 fallback 投递前应用 group-scoped 工具策略
3. 确保 Discord/Slack 风格房间在升级后在正确频道可见回复

---
### #322 Gateway/Sessions：热 transcript 读取移至异步有界 IO

**更新原文**：
Gateway/sessions: move hot transcript reads and mirror appends onto async bounded IO with serialized parent-linked writes, keeping large session histories from stalling Gateway requests and channel replies. Fixes [#75656](https://github.com/openclaw/openclaw/issues/75656). Thanks @steipete.

**关联 Issue**：#75656 / PR #75875（+1890 / -338，37 文件）

**痛点**：
Gateway 事件循环被同步 `fs.readFileSync()` 调用阻塞。`readSessionMessages()` 使用同步文件读取和完整 transcript 文件的 JSON 解析，导致：
- WebSocket handshake 超时
- Telegram 无响应
- 大型 session histories 场景下所有 Gateway 请求被 stall

**如何解决**：
将热 transcript 读取和镜像追加移至异步有界 IO（async bounded IO）并序列化 parent-linked writes：
1. `readSessionMessages` → `readSessionMessagesAsync`，所有调用点迁移到 async/await
2. 新增 `transcript-append.ts` 实现有界 IO 和序列化父子链接写入
3. Codex 扩展中的 transcript-mirror 同步更新
4. Session tools/agents 各处调用的 transcript 操作全部 async 化

这是重量级的架构改进，涉及 37 个文件、近 1900 行变更。

---
### #323 macOS/Talk Mode：多通道麦克风下混为单声道

**更新原文**：
macOS/Talk Mode: downmix multi-channel microphone input to mono before SFSpeechRecognizer, fixing `emptyOnRelease len=0` for professional audio interfaces. Fixes [#42533](https://github.com/openclaw/openclaw/issues/42533).

**关联 Issue**：#42533

**痛点**：
macOS Talk Mode Push-to-Talk 在多通道音频接口（超过 2 通道）上始终返回 `emptyOnRelease len=0`。SFSpeechRecognizer 对多通道音频格式静默失败——无错误、无 transcript。专业音频接口（PreSonus Quantum、Focusrite 18i20、MOTU 等）用户完全无法使用 Talk Mode 和 Voice Wake PTT。

**如何解决**：
通过 AVAudioMixerNode 将多通道麦克风输入下混为单声道，再送给 SFSpeechRecognizer。这是 macOS 音频处理的标准降混方式，SFSpeechRecognizer 只支持单声道输入。

---
### #324 macOS/Talk Mode：订阅 WebChat/Control UI 接收转录文本

**更新原文**：
macOS/Talk Mode: subscribe native WebChat/Control UI to receive transcribed text from Talk overlay, fixing missing user transcripts in UI while assistant replies render correctly. Fixes [#75155](https://github.com/openclaw/openclaw/issues/75155).

**关联 Issue**：#75155

**痛点**：
macOS Talk Mode 中，用户语音被正确转录并发送给模型，但转录文本从未出现在 WebChat/Control UI 聊天窗格中。只有 assistant 回复显示，用户看不到自己的转录输入——这与正常打字输入体验不符，用户期望两者一同显示。

**如何解决**：
建立 Talk overlay 到 WebChat/Control UI 的订阅机制：当 Talk Mode 转录完成时，通过 `chat.send` 或 session events 将转录文本发送到 UI 线程，使其在聊天窗格中可见。

---
### #325 macOS/Voice Wake：接受触发专用短语

**更新原文**：
macOS/Voice Wake: accept trigger-only phrases like "MAIA" or "computer" without requiring full wake word like "Hey OpenClaw", fixing built-in Voice Wake test failures on macOS. Fixes [#64986](https://github.com/openclaw/openclaw/issues/64986).

**关联 Issue**：#64986

**痛点**：
macOS companion app 的 Voice Wake 只接受完整唤醒词（如 "Hey OpenClaw"），不接受触发专用短语（如 "MAIA"、"computer"）。用户配置了触发词但 Voice Wake 测试失败，唤醒词无法触发，导致内置测试失败且无有用运行时日志。

**如何解决**：
修改 wake word 识别逻辑，接受 trigger-only 模式：短触发词无需 "Hey OpenClaw" 前缀，可直接触发 Voice Wake。修复后 "MAIA" 或 "computer" 等短语可正常触发唤醒。

---
### #326 Cron/TTS：Cron announce 载荷通过 TTS 处理

**更新原文**：
Cron/TTS: run cron announce payloads through `maybeApplyTtsToPayload()`, fixing `[[tts]]` and `[[tts:text]]` tags rendering as raw text in Telegram instead of generating voice notes. Fixes [#52125](https://github.com/openclaw/openclaw/issues/52125).

**关联 Issue**：#52125

**痛点**：
TTS 标签 `[[tts]]` 和 `[[tts:text]]` 在直接回复中正常处理（生成语音笔记附加），但通过 cron job `announce` 模式传递时被忽略，标签作为原始文本显示在 Telegram 中。用户配置了 TTS 期望获得语音，但 cron announce 破坏了预期。

**如何解决**：
在 cron announce 的投递路径上调用 `maybeApplyTtsToPayload()`，与直接 agent 回复的处理方式一致，确保 TTS 标签被正确处理生成语音笔记。

---
### #327 WhatsApp：保存被引用照片的可下载媒体

**更新原文**：
WhatsApp: save downloadable media from replied-to photos, fixing "OpenClaw can't view referenced whatsapp image" regression where users must re-send image with tag for it to be visible. Fixes [#59174](https://github.com/openclaw/openclaw/issues/59174).

**关联 Issue**：#59174

**痛点**：
回归 bug：用户回复照片并标记 OpenClaw 时，OpenClaw 无法查看该图片。以前可以正常工作；现在用户必须重新发送图片并单独标记才能使 OpenClaw 看到。根因是 replied-to 照片的媒体未被正确保存/下载。

**如何解决**：
修复 WhatsApp 频道中处理引用回复媒体的能力，确保回复照片的媒体可被正确下载和保存，无需用户重新发送。

---
### #328 Sessions/Store：停止持久化 resolvedSkills 防止无限制增长

**更新原文**：
Sessions/store: stop persisting `skillsSnapshot.resolvedSkills` to session files, preventing unbounded session growth and "session file locked" timeouts from tool output bloat. Refs [#11950](https://github.com/openclaw/openclaw/issues/11950) [#6650](https://github.com/openclaw/openclaw/issues/6650) [#15000](https://github.com/openclaw/openclaw/issues/15000).

**关联 Issues**：#11950、#6650、#15000

**痛点**：
工具输出（包括完整 schema）被持久化到 session 文件的 `skillsSnapshot.resolvedSkills` 字段中，导致：
- 无限制的 session 增长（session 文件越来越大）
- context 窗口被填满，token 成本爆炸
- LLM 性能下降
- 全局文件锁竞争（#11950）和 session file locked 超时（#15000）

这是一个长期存在的性能问题，多个 issue 从不同角度描述了同一根因。

**如何解决**：
停止将 `skillsSnapshot.resolvedSkills` 持久化到 session 文件。工具结果中的 schema 信息不再写入 session，只保留运行时必需的信息。这防止 session 无限制增长，从根本上解决文件锁和性能问题。

---
### #329 Doctor/WhatsApp：警告遗留 crontab 引用 ensure-whatsapp.sh

**更新原文**：
Doctor/WhatsApp: warn when Linux crontabs reference legacy `ensure-whatsapp.sh` script, fixing misleading "Gateway inactive, starting via systemd" logs from cron environment lacking proper DBus variables. Fixes [#60204](https://github.com/openclaw/openclaw/issues/60204).

**关联 Issue**：#60204

**痛点**：
遗留的用户 crontab 条目运行 `~/.openclaw/bin/ensure-whatsapp.sh` 时，`systemctl --user` 在没有正确 DBus 环境变量的 cron 环境中失败，产生误导性的健康日志（`WARN: Gateway inactive, starting via systemd`），而 Gateway 实际运行正常。用户被误导以为 Gateway 有问题。

**如何解决**：
`openclaw doctor` 检测遗留 cron 条目引用 `ensure-whatsapp.sh` 并发出警告，提示用户用 repo 管理的 systemd user unit 替换遗留 cron 检查。提高诊断准确性，避免运维人员被误导。

---
### #330 Slack/Setup：打印纯 JSON 而非 ASCII 框字符

**更新原文**：
Slack/setup: print the generated app manifest as plain JSON without ASCII box-drawing characters, fixing "CLI's Slack JSON manifest is framed in breaking characters" that broke copy-and-paste. Fixes [#65751](https://github.com/openclaw/openclaw/issues/65751).

**关联 Issue**：#65751

**痛点**：
CLI 设置 Slack 频道时，生成的 app manifest JSON 被 ASCII 框线字符（`├`、`│`、`╯` 等）包裹。用户复制输出粘贴到其他地方时得到无效 JSON，无法直接使用。用户必须手动清理这些装饰字符才能使用 manifest。

**如何解决**：
修改 Slack channel setup 的输出，直接打印纯 JSON，不再添加装饰性 ASCII 边框。确保输出可直接复制粘贴使用。

---
### #331 Channels/WhatsApp：CLI logout 经由 live Gateway 而非直接操作配置文件

**更新原文**：
Channels/WhatsApp: route CLI logout and remove commands through live Gateway instead of manipulating config files directly, fixing "channels logout" and "remove" not properly clearing WhatsApp session and leaving bot active. Fixes [#67746](https://github.com/openclaw/openclaw/issues/67746).

**关联 Issue**：#67746

**痛点**：
`openclaw channels logout --channel whatsapp` 和 `openclaw channels remove --channel whatsapp` 命令直接操作配置文件，而非通过 live Gateway 处理。这导致：
- WhatsApp session 未正确终止，bot 保持活跃
- 用户以为已断开，但 bot 继续响应消息（隐私风险）
- 用户必须手动 kill gateway 进程、删除配置和凭证文件

**如何解决**：
将 CLI logout/remove 命令路由到 live Gateway，通过 Gateway API 正确关闭 WhatsApp 连接后再清理配置文件。确保命令执行后 bot 真正终止，而非"假断开"。

---
### #332 Discord：合并重复的 rate-limit 启动日志

**更新原文**：
Discord: collapse repeated native slash-command deploy rate-limit startup logs into one non-fatal warning while keeping per-request REST timing in verbose output. Thanks @discord.

**痛点**：
启动时部署原生 slash-command 到 Discord API 遇到 HTTP 429 rate limit 时，Gateway 反复打印大量 rate-limit 警告日志，造成日志噪音过大，干扰问题排查。

**如何解决**：
将重复的 rate-limit 日志合并为一条 non-fatal warning，verbose 输出模式下仍保留每个请求的 REST 计时信息。既减少噪音，又保留调试数据。

---
### #333 Discord：slash-command 部署中止报告为 REST timeout

**更新原文**：
Discord: report native slash-command deploy aborts as REST timeouts with method, path, timeout budget, and observed duration, so startup logs explain slow Discord API calls instead of showing a generic aborted operation. Thanks @discord.

**痛点**：
部署 slash-command 过程中 Discord API 调用超时/中止时，日志只显示笼统的"aborted operation"，无诊断信息，无法判断是超时、网络问题还是 Discord API 问题。

**如何解决**：
中止操作报告为 REST timeout 并附带详细诊断信息：method、path、timeout budget、observed duration。启动日志能清晰解释 Discord API 调用缓慢的原因，便于排查。

---
### #334 安全/日志：支付凭证字段名脱敏

**更新原文**：
Security/logging: redact payment credential field names such as card number, CVC/CVV, shared payment token, and payment credential across default log and tool-payload redaction patterns so wallet-style MCP tools do not expose raw payment credentials in UI events or transcripts. Thanks @stainlu.

**痛点**：
默认日志/tool-payload 脱敏模式未覆盖支付凭证相关字段名（card number、CVC/CVV、shared payment token、payment credential）。使用钱包类 MCP 工具时，这些敏感字段可能以明文出现在 UI 事件或对话记录中，造成支付信息泄露。

**如何解决**：
将支付凭证字段名加入默认日志和 tool-payload 脱敏规则，确保这些字段在写入日志或传递给 UI 时被自动遮盖。安全加固类改动。

---
### #335 Gateway/Config：报告备份恢复失败

**更新原文**：
Gateway/config: report failed backup restores so silent copy failures no longer look like successful auto-restores in audit records and logs. Fixes [#70515](https://github.com/openclaw/openclaw/pull/70515). Thanks @davidangularme.

**关联 PR**：#70515

**痛点**：
配置备份恢复过程中，当 `copyFile` 失败时（如权限问题 EACCES），错误被 bare `catch {}` 吞没，日志仍记录"Config auto-restored from backup"，audit record 的 `valid` 字段仍为 `true`。这让静默失败看起来像成功，运维人员无法察觉恢复实际失败了。

**如何解决**：
修复捕获 copyFile 错误，记录 distinct failure warning，设置 `valid` 为 `restoredFromBackup`，在 audit record 中持久化 `restoreErrorCode` 和 `restoreErrorMessage`（async 和 sync 路径均支持）。确保失败可追踪，不被误报为成功。

---
### #336 Compaction：使用活跃 session 模型回退链

**更新原文**：
Compaction: use the active session model fallback chain for implicit embedded compaction summarization failures, fixing permanent compaction failure when Azure content filter blocks summarization. Fixes [#64960](https://github.com/openclaw/openclaw/issues/64960). Thanks @jalehman.

**关联 Issue/PR**：#64960 / #74470

**痛点**：
当 compaction/summarization 在 Azure 托管模型上运行时，若触发 Azure 内容策略，compaction 以 HTTP 400 失败并进入不可恢复的死循环：`retryAsync` 用相同模型重试3次，compaction resolver 中没有 fallback chain，外层 overflow loop 以相同模型无限重试。Compaction 永久失败。

**如何解决**：
隐式 embedded compaction 现在通过活跃 session 模型回退链重试回退资格 summarization 失败。显式 `agents.defaults.compaction.model` 保持精确匹配，不继承 session fallback chain。Azure content-filter 400 现在可以被正确回退处理。

---
### #337 Gateway/Config：允许 config.patch 更新子代理 thinking

**更新原文**：
Gateway/config: allow `gateway config.patch` to update subagent thinking defaults at `agents.defaults.subagents.thinking` and `agents.list[].subagents.thinking`. Fixes [#75764](https://github.com/openclaw/openclaw/issues/75764). Thanks @kevinslin.

**关联 Issue/PR**：#75764 / #75802

**痛点**：
`gateway config.patch` 硬编码的 mutation allowlist 包含 `agents.defaults.thinkingDefault` 和 `agents.list[].thinkingDefault`，但不包含 documented 的 `agents.defaults.subagents.thinking` 路径。导致 `config.patch` 修改子代理 thinking 配置时返回 "cannot change protected config paths" 错误，文档/schema 与实际行为不匹配。

**如何解决**：
在 `isAllowedGatewayConfigPath` allowlist 中添加 `agents.defaults.subagents.thinking` 和 `agents.list[].subagents.thinking`，允许通过 config.patch 修改子代理 thinking 默认值。

---
### #338 Plugins/CLI：保持 git 插件安装路径无凭证

**更新原文**：
Plugins/CLI: keep git plugin install paths credential-free. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
git plugin 安装路径可能包含认证凭证（如 `https://user:token@github.com/repo`）。这些路径如果被写入日志或文件系统，会导致凭证泄漏。

**如何解决**：
确保 git plugin 安装路径不含认证凭证，处理后的路径在日志和文件系统中使用，防止凭证暴露。

---
### #339 Plugins/CLI：日志中脱敏认证 git URL

**更新原文**：
Plugins/CLI: redact authenticated git URLs in logs. Thanks [@vincentkoc](https://github.com/vincentkoc).

**痛点**：
与 #338 相关。认证 git URL（如含 token/密码的 `https://user:token@host/repo`）可能在日志输出中以明文显示，导致凭证泄漏。

**如何解决**：
在日志输出中对含认证信息的 git URL 进行脱敏处理，将 token/密码部分替换为 `***`，与 #338 配合确保凭证安全。

---
### #340 Channels/Status Reactions：移除过时的非终态

**更新原文**：
Channels/status reactions: remove stale non-terminal status reactions after completion so thinking emoji does not persist after turn finalization. Fixes [#75458](https://github.com/openclaw/openclaw/issues/75458). Thanks @davelutztx.

**关联 Issue**：#75458

**痛点**：
Discord status reactions 在 turn 完成后仍保留在 non-terminal 生命周期状态。当 `messages.removeAckAfterReply: true` 时，turn 成功完成后唯一 remaining reaction 是 🤔（thinking）而非 👍（done）或清除。用户期望完成后显示 terminal 👍 或清除，但实际上 thinking emoji 永久保留。

**如何解决**：
在 terminal completion 后移除所有非终态 status reactions（包括 thinking/tool/coding/web/stall states），确保 turn 完成后不残留任何 lifecycle reaction。设置 `removeAckAfterReply: true` 时，在 done/error hold period 后清除所有 lifecycle reactions。

---
### #341 Discord/Doctor：迁移不支持的 per-channel agentId

**更新原文**：
Discord/doctor: migrate unsupported per-channel `agentId` in channel configs that abort gateway reload. Fixes [#62455](https://github.com/openclaw/openclaw/issues/62455).

**关联 Issue**：#62455

**痛点**：
在 Discord channel 条目下放置 `agentId` 字段（如 `channels.discord.guilds.<guildId>.channels.<channelId>.agentId`）会导致 config reload 被拒绝，gateway abort 并返回 "Unrecognized key: agentId"。用户以为可以这样配置 per-channel agent，但实际上 schema 不支持。预期的 routing 机制是 `bindings[]`。

**如何解决**：
`openclaw doctor` 增加自动迁移逻辑，将不支持的 per-channel `agentId` 转换为 `bindings[]` 格式。Issue 当时仍为 Open，但更新日志显示迁移逻辑已实现。

---
### #342 Discord/DMs：设置入站 DM 的 ctx.To 为 user:<id>

**更新原文**：
Discord/DMs: set inbound direct-message `ctx.To` to `user:<id>` instead of `channel:<id>`. Fixes [#68126](https://github.com/openclaw/openclaw/issues/68126). Thanks @Lewis-404.

**关联 Issue**：#68126

**痛点**：
Discord DM inbound messages 的 `ctx.To` 被错误设置为 `channel:<DM_channel_id>` 而非 `user:<userId>`。下游组件（mirror、delivery-recovery）将 DM 视为 channel 对话，导致 Discord API `Unknown Channel` 错误。根因在 `resolveDiscordAutoThreadReplyPlan` 无条件构造 `channel:` reply target，不考虑 DM 场景。

**如何解决**：
在 DM 场景下，`ctx.To` 应设置为 `user:<author.id>` 而非 `channel:<DM_channel_id>`。Discord API 仍通过 channel ID 发送 DM（deliverTarget 保持 channel:），但语义层面的 `ctx.To` 需要修正。

---
### #343 Discord/DMs：保留无 guild 的入站消息

**更新原文**：
Discord/DMs: keep no-guild inbound messages routed to direct session keys instead of channel sessions when channel classification fails. Fixes [#59817](https://github.com/openclaw/openclaw/issues/59817). Thanks @DooPeePey.

**关联 Issue**：#59817

**痛点**：
Discord DM 在网络问题导致 channel info 解析失败时，会被分散到多个 session keys（一个正确的 direct session 和两个错误的 channel sessions），导致 history/context 碎片化，回复似乎消失。根因是 channel type 无法解析时 fallback 到 channel-mode routing。

**如何解决**：
当 `guild_id` 缺失且为 user DM context 时，如果 channel type 无法解析，优先使用 direct routing 而非创建 channel session key。保持 DM 路由到单一的 `discord:direct:<userId>` session key。

---
### #344 Discord： outbound API 调用在 HTTP 5xx 时重试

**更新原文**：
Discord: retry outbound API calls on HTTP 5xx and transient errors, fixing silent message loss when Discord API returns 502/503/timeout. Fixes [#52396](https://github.com/openclaw/openclaw/issues/52396). Thanks @sunshineo.

**关联 Issue**：#52396

**痛点**：
Discord outbound message 重试机制只在 HTTP 429（rate limit）时重试。502、503、connection reset、timeout、DNS resolution 等 transient failures 立即失败，导致 Cron jobs 和 agent sessions 发布到 Discord 的消息静默丢失。用户的每日 Stock Pick cron 在 276 秒研究完成后因 Discord 发送失败丢失所有输出。

**如何解决**：
扩展 Discord retry 策略，覆盖 HTTP 5xx（502、503）和 transient errors（timeout、connect、reset、closed、unavailable、temporarily、fetch.failed）。现有 `channels.discord.retry` 配置（attempts、delays、backoff）保持不变。

---
### #345 Discord：提取 Components v2 Text Display

**更新原文**：
Discord: include Components v2 Text Display text in reply context by extracting from `referenced_message.components`. Fixes [#56228](https://github.com/openclaw/openclaw/issues/56228). Thanks @HollandDrive.

**关联 Issue**：#56228

**痛点**：
用户回复使用 Discord Components v2（containers、text display blocks）发送的消息时，reply context body 为空。`resolveDiscordMessageText()` 只检查 `content` 和 `embeds[0]`，对 v2 messages 两者都为空。Discord API 在 `referenced_message` 中包含完整的 `components` array，包含 TextDisplay (type 10) 节点。

**如何解决**：
当 `baseText` 为空且 message flags 指示是 Components v2 message 时，从 `message.components` 数组中提取 TextDisplay (type 10) 节点的文本内容，使 Agent 能看到用户回复的 v2 component 消息内容。

---
### #346 Discord：添加可配置的 gateway READY 超时

**更新原文**：
Discord: add configurable gateway READY timeouts so multi-account staggered startups no longer hit 15s hard-coded limits. Fixes [#72273](https://github.com/openclaw/openclaw/issues/72273). Thanks @sergionsantos.

**关联 Issue**：#72273

**痛点**：
Discord gateway ready timeout 硬编码为 15 秒。在多账号 staggered startup 场景（启动延迟 70s/80s/90s）下，后续账号无法在 15 秒内完成连接，导致 244 个 `gateway starting` 事件、161 个 SIGTERM、44 个 ready timeout。用户只能用 local sed patch 临时修复。

**如何解决**：
将 `DISCORD_GATEWAY_READY_TIMEOUT_MS`（15s）和 `DISCORD_GATEWAY_RUNTIME_READY_TIMEOUT_MS`（30s）暴露为可配置项：环境变量 `OPENCLAW_DISCORD_READY_TIMEOUT_MS` 和 config schema `gateway.discord.readyTimeoutMs`。

---
### #347 Discord：保留原生斜杠命令 description_localizations

**更新原文**：
Discord: preserve native slash-command description_localizations through reconcile/redeploy cycles. Fixes [#56580](https://github.com/openclaw/openclaw/issues/56580). Thanks @mhseo93.

**关联 Issue**：#56580

**痛点**：
Discord 支持 `description_localizations` 为 slash commands 提供多语言描述。但 OpenClaw 在 native command reconcile/redeploy 后（如 restart、provider reload）会以 English defaults 覆盖 localized descriptions。用户配置的多语言描述被静默丢弃。

**如何解决**：
在 native command specs 中添加 `descriptionLocalizations` 支持，reconcile 时保留并传递 localization metadata 给 Discord command serialization。比较 localization fields 以检测和纠正 drift。

---
### #348 Discord：添加配置式出站 mention 别名

**更新原文**：
Discord: add configured outbound mention aliases so @AgentName references resolve deterministically from config without relying on cache. Fixes [#67587](https://github.com/openclaw/openclaw/issues/67587). Thanks @McoreD.

**关联 Issue**：#67587

**痛点**：
这是 #238（同一 issue）的补充修复。当 OpenClaw agent 在 Discord 中尝试用 `@Vladislava` 提及另一个 agent 时，outbound mention rewriting 依赖 account-scoped 缓存，缓存缺失时发送纯文本而非 `<@USER_ID>`。用户需要确定性行为。

**如何解决**：
引入 config-backed alias-to-userId mapping 用于 Discord mentions，使 @AgentName 引用从配置中确定性解析，而非依赖机会主义缓存。与 #238 配合完整解决 mention 可靠性问题。

---
### #349 Discord：避免启动时 REST 放大

**更新原文**：
Discord: avoid startup REST amplification so gateway startup no longer hits the Discord rate limit with burst REST calls. Fixes [#75341](https://github.com/openclaw/openclaw/issues/75341).

**关联 Issue**：#75341

**痛点**：
这是 #239（同一 issue）的相关修复。Gateway 启动时，Discord provider 发起大量 burst REST 调用（如 `oauth2/applications/@me`、`users/@me`），导致 HTTP 429 rate limit 和 fetch timeout。启动时的高频 REST 调用是造成 429 的直接原因。

**如何解决**：
减少启动时的 REST 调用放大：合并可合并的请求、添加启动时限流、退避重试而非立即重试。确保 Discord startup 不再以 burst 方式轰炸 Discord REST API。

---
### #350 Plugins/Hooks：从 session key 正确提取 ctx.channelId

**更新原文**：
Plugins/hooks: derive hook `ctx.channelId` from the session key instead of returning the provider name. Fixes [#59881](https://github.com/openclaw/openclaw/issues/59881). Thanks @bradfreels.

**关联 Issue**：#59881

**痛点**：
这是 #240（同一 issue）的再次修复（问题持续存在）。在 plugin hooks（`before_prompt_build`、`agent_end`）中，`ctx.channelId` 返回 provider name（如 `"discord"`）而非实际的 channel identifier（如 `"1472750640760623226"`）。Session key `agent:main:discord:channel:1472750640760623226` 本身包含正确的 channel ID，但 hook context 没有正确提取。

**如何解决**：
在 hook context 构建时，从 session key 中正确提取 channel ID。`ctx.channelId` 应该是实际的频道标识符，`ctx.messageProvider` 承载 provider name。这确保依赖 `ctx.channelId` 做 per-channel 逻辑的插件（如 Hindsight）能正确工作。

---
### #351 Gateway/Config：记录配置健康状态写入失败

**更新原文**：
Gateway/config: log config health-state write failures. Thanks [@sallyom](https://github.com/sallyom).

**痛点**：
配置健康状态写入失败时缺乏日志记录，导致配置状态持久化问题难以诊断。gateway 配置状态写入失败后运维人员无法知晓发生了什么。

**如何解决**：
在配置健康状态写入失败时增加日志记录（相关 commit #75441 "log observe recovery write failures"），使配置状态写入问题可追踪诊断。

---
### #352 诊断：回复时重置卡住会话计时器

**更新原文**：
Diagnostics: reset stuck-session timers on reply and back off repeated session.stuck diagnostics while processing. Fixes [#72010](https://github.com/openclaw/openclaw/pull/72010).

**关联 PR**：#72010（被 supersede，内容在 main commit `2d8d50d`）

**痛点**：
与 #242 同一 issue 的后续修复。当 reply/tool/status/block 和 ACP runtime events 发生时，stuck-session 诊断计时器未被重置，导致 active long-running channel turns 被误判为 stuck 并触发警告。这些事件实际表明会话正在处理中，不应被视为无进展。

**如何解决**：
reply/tool/status/block 和 ACP runtime events 触发时重置 stuck-session 计时器，并引入指数退避抑制 processing 会话的重复 stuck 警告。与 #242 配合完整解决诊断误判问题。

---
### #353 Gateway/Agents：避免重建核心工具

**更新原文**：
Gateway/agents: avoid rebuilding core tools when only plugin tools are allowlisted and keep the full workspace plugin registry cache separate from scoped loads. Fixes [#75882](https://github.com/openclaw/openclaw/issues/75882) [#75907](https://github.com/openclaw/openclaw/issues/75907) [#75906](https://github.com/openclaw/openclaw/issues/75906) [#75887](https://github.com/openclaw/openclaw/issues/75887) [#75851](https://github.com/openclaw/openclaw/issues/75851). Thanks @obviyus.

**关联 PR**：#75922

**痛点**：
这是 #243 同一 PR 的另一种描述。核心问题：
1. explicit allowlist 只请求 plugin tools 时仍构造 core coding tools，浪费 30-40s
2. workspace plugin registry cache 与 scoped loads 混合导致 LRU 驱逐和冷加载

**如何解决**：
1. 当 explicit allowlist 仅请求 plugin tools 时跳过 core coding tool 构造
2. 将完整 workspace plugin registry cache 与 scoped loads 隔离

---
### #354 Agents/Failover：分类裸 status: internal server error

**更新原文**：
Agents/failover: classify bare `status: internal server error` as a server error for failover. Fixes [#73844](https://github.com/openclaw/openclaw/pull/73844).

**关联 PR**：#73844（与 #244 相同）

**痛点**：
与 #244 相同。裸 `"status: internal server error"` 消息未被 failover 正确分类为服务器错误，导致可能错过应该触发的 failover。

**如何解决**：
将裸 `"status: internal server error"` 识别为可重试的服务器错误，确保 failover 机制能正确响应。

---
### #355 Gateway/Startup：返回共享的可重试启动-sidecars 错误

**更新原文**：
Gateway/startup: return the shared retryable startup-sidecars error from startup-gated control-plane RPC methods. Fixes [#76012](https://github.com/openclaw/openclaw/pull/76012). Thanks @scoootscooob.

**关联 PR**：#76012（与 #245 相同）

**痛点**：
与 #245 相同。控制面 RPC 方法（sessions.create/send/abort、agent.wait、tools.effective）在 sidecar 未就绪时返回 generic `UNAVAILABLE`，无稳定的重试信号。

**如何解决**：
这些 RPC 方法现在返回共享的可重试错误形状：`details.reason = "startup-sidecars"`, `retryAfterMs = GATEWAY_STARTUP_RETRY_AFTER_MS`。客户端可可靠区分启动竞态与普通故障。

---
### #356 Providers/Google：修复 Gemini 2.5 Flash-Lite reasoning:minimal 拒绝

**更新原文**：
Providers/Google: fix Gemini 2.5 Flash-Lite `reasoning: "minimal"` rejections by raising the minimal thinkingBudget floor to 512. Fixes [#70629](https://github.com/openclaw/openclaw/pull/70629). Thanks @ericberic.

**关联 PR**：#70629（与 #246 相同）

**痛点**：
与 #246 相同。Gemini 2.5 Flash-Lite 模型的 `thinkingBudget: 128` 被 Google API 拒绝，要求值在 512-24576 范围内。

**如何解决**：
为 `2.5-flash-lite` 添加专门的 minimal budget floor（512），Pro 和 Flash 保留其 documented 128 minimum。添加测试覆盖所有 budget 档位。

---
### #357 Agents/Status：解析 channel-plugin agents 的 sessionKey="current"

**更新原文**：
Agents/status: resolve `session_status(sessionKey="current")` for channel-plugin agents. Fixes [#74141](https://github.com/openclaw/openclaw/issues/74141). Thanks @bittoby.

**关联 PR**：#72306（与 #247 相同）

**痛点**：
与 #247 相同。`sessionKey="current"` 对 channel-plugin agents 无效，每轮浪费约 4 秒。

**如何解决**：
与 #247 相同。在解析链末尾增加回退：将字面 `"current"` 视为请求者自身 session，在没有先前 store 记录时合成一个最小条目。

---
### #358 Cron：重试临时 busy skip 的重复 wake-now 任务

**更新原文**：
Cron: retry recurring wake-now main-session jobs on temporary busy skips instead of recording an immediate ok ghost run. Fixes [#75964](https://github.com/openclaw/openclaw/issues/75964). Thanks @xuruiray.

**关联 PR**：#76083（与 #248 相同）

**痛点**：
与 #248 相同。重复 wake-now cron 任务在临时 busy skip 时记录 ok ghost run 而非重试。

**如何解决**：
与 #248 相同。临时 busy skips（requests-in-flight、lanes-busy）时重试而非立即返回成功。cron-in-progress 延迟路径被保留。

---
### #359 Providers/Google：保持 Gemini thinking-signature-only 流块活跃

**更新原文**：
Providers/Google: keep Gemini thinking-signature-only stream chunks from stalling the transport by emitting a `thinking_signature` event. Fixes [#76071](https://github.com/openclaw/openclaw/issues/76071). Thanks @zhangguiping-xydt.

**关联 PR**：#76080（与 #249 相同）

**痛点**：
与 #249 相同。Gemini 3.1 Pro Preview 发送 thoughtSignature-only parts 导致传输流停滞和 idle timeout。

**如何解决**：
与 #249 相同。为 thoughtSignature-only parts 发出 `thinking_signature` 事件以保持流活跃。

---
### #360 CLI/Skills：显示 per-agent 模型和命令可见性

**更新原文**：
CLI/skills: show per-agent model and command visibility for skills. Fixes [#75983](https://github.com/openclaw/openclaw/pull/75983). Thanks @mbelinky.

**关联 PR**：#75983（与 #250 相同）

**痛点**：
与 #250 相同。无法针对特定 agent 评估 skill 就绪性。

**如何解决**：
与 #250 相同。`openclaw skills check --agent <id>` 评估特定 agent 的 skill 就绪性，显示 model-visible、command-visible、prompt-hidden、agent-filtered、missing-requirement 等分类。`skills list`/`skills info` 与 agent allowlists 对齐。

---
### #361 Agents/Runtime/Tools：保留 Gateway 元数据上的回复启动

**更新原文**：
Agents/runtime/tools: keep reply startup on Gateway metadata. Thanks [@shakkernerd](https://github.com/shakkernerd).

**痛点**：
Agent 工具在处理 reply startup 时可能丢失或未正确传递 Gateway 元数据，导致 reply 启动失败或行为异常。

**如何解决**：
确保 reply startup 保留在 Gateway 元数据上。涉及 bundled tools 通过最终 policy 过滤，以及 gateway handler 上的授权信号硬化。

---
### #362 Discord：记录规范 mention 格式

**更新原文**：
Discord: document canonical mention formatting in agent prompt hints and channel docs. Fixes [#75173](https://github.com/openclaw/openclaw/pull/75173). Thanks @steipete.

**关联 PR**：#75173（与 #252 相同）

**痛点**：
与 #252 相同。Discord mention 格式不统一，纯文本 @Name 无法正确触发提及。

**如何解决**：
与 #252 相同。在 channel docs 中记录 user/channel/role mention 语法，为 prompt hint 添加规范化指导。

---
### #363 Heartbeat Scheduler：门控 exec-event/notification/spawn/retry 唤醒

**更新原文**：
Heartbeat scheduler: gate exec-event/notification/spawn/retry wakes through a centralized cooldown so backgrounded `process.start` exit notifications can no longer self-feed runaway heartbeat runs. Fixes [#64016](https://github.com/openclaw/openclaw/issues/64016). Thanks @hexsprite.

**关联 Issue**：#64016、#17797、#75436（与 #253 相同）

**痛点**：
与 #253 相同。Heartbeat 配置 `isolatedSession` 和 `every=60m` 时，external wake events 在 active heartbeat 期间被排队并在完成后立即触发，导致同一小时内发生额外的 heartbeat runs。

**如何解决**：
与 #253 相同。引入集中化冷却机制，门控 exec-event/notification/spawn/retry 唤醒，防止 heartbeat 自我反馈循环。

---
### #364 Fix：屏蔽 workspace CLOUDSDK_PYTHON 覆盖

**更新原文**：
fix: block workspace CLOUDSDK_PYTHON override and always set trusted interpreter for gcloud. Fixes [#74492](https://github.com/openclaw/openclaw/pull/74492). Thanks @pgondhi987.

**关联 PR**：#74492（与 #254 相同）

**痛点**：
与 #254 相同。Workspace `.env` 可能植入 `CLOUDSDK_PYTHON`，让 gcloud 继承攻击者提供的 Python 路径。

**如何解决**：
与 #254 相同。将 `CLOUDSDK_PYTHON` 加入 `BLOCKED_WORKSPACE_DOTENV_KEYS`，重写 `gcloudEnv()` 始终用受信任 interpreter 覆盖。

---
### #365 Providers/Z.AI：移动捆绑 GLM 目录和认证环境元数据

**更新原文**：
Providers/Z.AI: move the bundled GLM catalog and auth env metadata into the plugin manifest. Thanks [@shakkernerd](https://github.com/shakkernerd).

**关联 Issue**：无明确 PR（与 #255 相同）

**痛点**：
与 #255 相同。GLM 模型目录和认证环境元数据在 runtime seed data 中重复，导致 `models list --all --provider zai` 显示不完整。

**如何解决**：
与 #255 相同。将 GLM catalog 和 auth env metadata 迁移到 plugin manifest，使模型列表从 manifest 读取而非 runtime seed data。

---
### #366 Providers/Qianfan 和 Stepfun：声明设置认证元数据

**更新原文**：
Providers/Qianfan and Providers/Stepfun: declare setup auth metadata in the plugin manifest. Thanks [@shakkernerd](https://github.com/shakkernerd).

**关联 Issue**：无明确 PR（与 #256 相同）

**痛点**：
与 #256 相同。Qianfan 和 Stepfun 的认证信息依赖 legacy runtime seed data。

**如何解决**：
与 #256 相同。在 plugin manifest 中声明 `api-key` 方法和 `QIANFAN_API_KEY`/`STEPFUN_API_KEY` 环境变量。

---
### #367 Fix(Infra)：屏蔽环境 Homebrew 环境变量

**更新原文**：
fix(infra): block ambient Homebrew env vars from brew resolution. Fixes [#74463](https://github.com/openclaw/openclaw/pull/74463). Thanks @pgondhi987.

**关联 PR**：#74463（与 #257 相同）

**痛点**：
与 #257 相同。`HOMEBREW_BREW_FILE` 和 `HOMEBREW_PREFIX` 可能被 workspace `.env` 注入，导致可执行文件重定向攻击。

**如何解决**：
与 #257 相同。`resolveBrewExecutable()` 和 `resolveBrewPathDirs()` 只使用硬编码标准路径，两个 key 加入 `BLOCKED_WORKSPACE_DOTENV_KEYS`。

---
### #368 Onboarding/Configure：避免暂存每个默认插件运行时依赖

**更新原文**：
Onboarding/configure: avoid staging every default plugin runtime dependency after config writes. Thanks [@vincentkoc](https://github.com/vincentkoc).

**关联 Issue**：无明确 PR（与 #258 相同）

**痛点**：
与 #258 相同。配置写入后无条件 staging 所有默认插件依赖，浪费时间和带宽。

**如何解决**：
与 #258 相同。跳过的 setup 流程只准备用户配置的插件依赖。

---
### #369 Thinking/Providers：解析捆绑 provider thinking profiles

**更新原文**：
Thinking/providers: resolve bundled provider thinking profiles through lightweight provider policy artifacts. Fixes [#74796](https://github.com/openclaw/openclaw/issues/74796). Thanks @maxschachere.

**关联 Issue**：#74796（与 #259 相同）

**痛点**：
与 #259 相同。Startup-lazy providers 在未激活时 thinking profiles 无法解析，导致 OpenAI Codex GPT-5.5 的 xhigh 在 gateway validation 时被降级或拒绝。

**如何解决**：
与 #259 相同。通过 lightweight provider policy artifacts 解析 bundled provider thinking profiles，使 startup-lazy provider 也能提供 thinking profile 信息。

---
### #370 Security/Windows：忽略 workspace .env 系统路径变量

**更新原文**：
Security/Windows: ignore workspace `.env` system-path variables and resolve stale-process `taskkill.exe` from validated install root. Thanks [@pgondhi987](https://github.com/pgondhi987).

**关联 Issue**：无明确 PR（与 #260 相同）

**痛点**：
与 #260 相同。Workspace `.env` 可能覆盖 `PATH`、`SYSTEMROOT`，`taskkill.exe` 可能从 PATH 解析到恶意版本。

**如何解决**：
与 #260 相同。忽略 workspace `.env` 中的系统路径变量，从经过验证的 Windows 安装根目录解析 `taskkill.exe`。

---
### #371 CLI/Plugins：原地刷新持久化插件注册表策略

**更新原文**：
CLI/plugins: refresh persisted plugin registry policy in place for `plugins enable` and `plugins disable`. Thanks [@vincentkoc](https://github.com/vincentkoc).

**关联 Issue**：无明确 PR（与 #261 相同）

**痛点**：
与 #261 相同。`plugins enable/disable` 时即使目标已索引仍重建所有插件。

**如何解决**：
与 #261 相同。原地刷新持久化 plugin registry policy，跳过不必要的重建和哈希。

---
### #372 Windows/Install：从可写安装程序临时目录运行 npm

**更新原文**：
Windows/install: run npm from a writable installer temp directory. Thanks [@mariozechner](https://github.com/mariozechner).

**关联 Issue**：无明确 PR（与 #262 相同）

**痛点**：
与 #262 相同。npm 在无写权限目录运行导致安装失败，Windows ARM + Node 24 有 resolver bug。

**如何解决**：
与 #262 相同。从安装程序临时目录运行 npm，对 Bedrock 依赖钉住版本。

---
### #373 CLI/Plugins：限定 slot 选择范围至选定插件

**更新原文**：
CLI/plugins: scope install and enable slot selection to the selected plugin manifest/runtime fallback. Thanks [@vincentkoc](https://github.com/vincentkoc).

**关联 Issue**：无明确 PR（与 #263 相同）

**痛点**：
与 #263 相同。安装单个插件时加载所有已装插件运行时。

**如何解决**：
与 #263 相同。将 slot 选择限定为选定插件的 manifest/runtime fallback，不再加载所有插件运行时。

---
### #374 Plugins/TTS：保持捆绑语音提供商在冷启动路径上可发现

**更新原文**：
Plugins/TTS: keep bundled speech-provider discovery available on cold package Gateway paths. Refs [#75283](https://github.com/openclaw/openclaw/issues/75283). Thanks @vincentkoc.

**关联 Issue**：#75283（与 #264 相关）

**痛点**：
与 #264 类似。TTS 插件在冷启动路径下无法发现捆绑的语音提供商。

**如何解决**：
与 #264 类似。确保 bundled speech-provider 在冷启动路径下可被发现，增加运行时探测覆盖 health、readiness、RPC、TTS discovery。

---
### #375 Google Meet/Twilio：在 doctor 中显示委托语音通话 ID、DTMF、intro-greeting 状态

**更新原文**：
Google Meet/Twilio: show delegated voice call ID, DTMF, and intro-greeting state in `googlemeet doctor`. Refs [#72478](https://github.com/openclaw/openclaw/issues/72478). Thanks @DougButdorf.

**关联 Issue**：#72478（与 #265 相关）

**痛点**：
与 #265 相关。`googlemeet doctor` 缺少关键诊断信息（委托语音通话 ID、DTMF 状态、intro-greeting 状态），且虚假 DTMF 确认问题。

**如何解决**：
与 #265 类似。在 doctor 输出中增加 delegated voice call ID、DTMF、intro-greeting 状态展示，配置 PIN 序列时才声称 DTMF 已发送。

---
### #376 Plugins/Tools：工具发现时优先使用构建好的捆绑插件代码

**更新原文**：
Plugins/tools: prefer built bundled plugin code during tool discovery. Fixes [#75290](https://github.com/openclaw/openclaw/issues/75290). Thanks @thanos-openclaw.

**关联 Issue**：#75290（与 #266 相同）

**痛点**：
与 #266 相同。工具发现时重复执行完整的 channel runtime hydration，每次 embedded run 的 core-plugin-tools prep stage 花费约 11 秒。

**如何解决**：
与 #266 相同。优先使用 built bundled plugin code，跳过不必要的 hydration，保留 companion provider registrations。

---
### #377 Plugins/Loader：限定插件工具注册表服用范围

**更新原文**：
Plugins/loader: scope plugin-tool registry reuse to the enabled plugin plan and stored Gateway method keys. Fixes [#75520](https://github.com/openclaw/openclaw/issues/75520). Thanks @whtoo.

**关联 Issue**：#75520（与 #267 相同）

**痛点**：
与 #267 相同。scoped/unscoped cache-key mismatch 导致每次 embedded message 完整 plugin reload（20-40s/turn）。Gateway 以 `onlyPluginIds`（scoped）启动，embedded runners 请求 tools 时没有 scope，cache keys 不匹配。

**如何解决**：
与 #267 相同。在 `getCompatibleActivePluginRegistry()` 中添加 reuse path：当 active registry 是 scoped 而请求是 unscoped 时，使用 active registry 的 loaded plugin IDs 作为 scope 构建 cache key。

---
### #378 Voice Call/Twilio：直接发送通知模式初始 TwiML

**更新原文**：
Voice Call/Twilio: send notify-mode initial TwiML directly in the outbound create-call request. Fixes [#72758](https://github.com/openclaw/openclaw/pull/72758). Thanks @tyshepps.

**关联 PR**：#72758（被替代，等效修复 land 到 main via commit ec69c07b27）（与 #268 相同）

**痛点**：
与 #268 相同。通知模式的初始 TwiML 之前依赖 webhook 回调获取，导致单次通知通话需要两次网络请求。

**如何解决**：
与 #268 相同。通知模式初始 TwiML 直接嵌入 create-call 请求中，无需 webhook 回调。对话模式和预连接 DTMF 保持 webhook 驱动。

---

---

## 整体观察

### 版本主线分析

本次 **v2026.5.2** 是一个大规模稳定性修复版本，共 **379 条更新**，涉及 Gateway 核心、插件系统、所有主流消息通道（Discord/Telegram/WhatsApp/Slack/Mattermost/Matrix 等）、Provider 集成以及安全加固。

**修复类型分布（基于条目内容推断）**：
- **性能优化**：占比最高，约 30%+，集中于 plugin loading、session transcript 读取、event loop 阻塞、cold start 延迟等问题。这是本版本最突出的主题——多个 latency regression 被一次性修复（#75882/#75907/#75906/#75887/#75851 等均通过同一 PR #75922 修复）。
- **安全加固**：约 15 条，涉及环境变量隔离（Homebrew、CLOUDSDK_PYTHON、Windows .env）、支付凭证脱敏、git URL 脱敏、SecretRef 管理等，体现对供应链安全和配置安全的重视。
- **Bug 修复**：大量具体 channel bug（Discord DM 路由、WhatsApp 媒体、Telegram topic/Slack 线程、Voice Call TwiML 等），覆盖日常使用中各类边缘场景。
- **新功能**：约 37 条（Changes 部分），主要是 CLI增强（skills check --agent、gateway config.patch 子代理 thinking）、诊断能力增强（doctor 各项改进）和 channel 功能（Components v2、mention 可靠性等）。

**破坏性变更**：
- 无重大破坏性变更。主要改动为行为修正、性能改进和安全加固。
- 部分插件相关的 `minHostVersion` 门控逻辑变化（#191）可能影响旧版插件兼容性，但针对的是捆绑插件，不影响第三方插件。

**升级注意事项**：
1. **Gateway 性能明显改善**：v2026.4.29 引入的多项 latency regression 在本版本被修复，建议从 v2026.4.26–v2026.4.29 升级。
2. **Discord 集成多项改进**：启动 REST 放大、rate-limit 日志、mention 可靠性、Components v2 支持等，建议检查 Discord 相关配置。
3. **安全相关配置**：如有自定义 `.env` 注入脚本，注意 `BLOCKED_WORKSPACE_DOTENV_KEYS` 变化（新增 CLOUDSDK_PYTHON、HOMEBREW_*、Windows 系统路径等）。
4. **Session transcript 读取全面异步化**：大规模 session store 的 Gateway 响应性显著提升，但如有任何直接依赖同步 `fs.readFileSync` 的自定义代码可能受影响。

**总结**：v2026.5.2 是一个以"性能回归修复 + 安全加固 + 渠道稳定性"为主线的版本。核心改进集中于降低 per-run 开销、防止事件循环阻塞、修复 channel 集成边缘场景，以及强化安全边界。对已有部署，建议优先升级以获得显著的性能收益。

---
