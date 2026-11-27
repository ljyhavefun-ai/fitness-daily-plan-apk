# 每日健身计划 — APK 路线：WebView 套壳，UI 原样不动

> 给 Claude Code：这是唯一正确的 APK 路线，取代 README.md 里“改写成独立 APK”的旧目标。
> 上一版按旧目标做了原生重实现，UI 全变了，用户已否决。本文件定死：**UI 一个像素也不许改**。

## 目标

把 `fitness-space/` 里的现有 React 网页应用原样装进 Android APK，用 WebView 跑。
用户在手机上打开 APK，看到的必须和 Muse 空间里一模一样：同一套界面、同一套颜色、同一套流程。

## 红线（违反任何一条即不合格）

1. **不许重写任何界面。** `client/src/App.tsx`（223,381 字节）、`client/src/theme.css`、`client/src/main.tsx`、`client/src/index.html` 一字不改。禁止用原生组件、React Native、Flutter 或任何方式重新实现任何一屏。
2. **不许改视觉。** 风格 Y 保持原样：墨绿黑底 `#0B1210` + 青柠 `#B9F27D`、毛玻璃、圆角、字号、间距、图标，全部以打包后的网页渲染为准。
3. **不许简化规则。** ATI 裁决、RIR 五档、恢复分层、休息规则、容量口径、安全边界，全部按 `server/src/actions.ts` 现有逻辑保留（见 README.md 的“保留核心规则”一节）。
4. **不许改流程。** 建计划 → 开训 → 逐组记录 + ATI 裁决 → 结算复盘 → 看统计，主链路一字不动；任何按钮、弹窗、文案不得增删改。
5. **不许联网账号、不做社交。** 离线可用是底线。

## 架构：WebView + 本地网页 + JS Bridge + 原生 SQLite

```
APK
├── WebView：加载本地打包好的网页（见“构建”）
│   └── client/dist/index.html（App.tsx + theme.css 原样编译产物）
├── JS Bridge（替代原来的服务端 RPC）
│   └── 网页里的 api.xxx(args) 调用，原样转发给原生层
└── 原生 SQLite：表结构按 drizzle/ 迁移 0001–0009 一比一建表
```

- 网页与原生之间只传数据（JSON），不传界面。界面永远由网页自己渲染。
- 首次启动时把 `data/` 里的 JSON 导入本地数据库（见 README.md“数据说明”与 `data/README-data.md`，数据缺口不许编造补齐）。

## 构建（网页部分）

- 用原来的构建方式产出 `client/dist/`：`bun ./client/build.mjs`（入口 `client/dist/index.html`，见 `space.json`）。
- 把整个 `client/dist/` 原样打进 APK 的 assets，WebView 直接加载本地 `index.html`，不走网络。
- **注意：`@hatch/space-sdk` 不在仓库里。** `package.json` 里它是 `file:/opt/hatch/skills/spaces/ts-runtime/dist/space-sdk.tgz`，仓库里只有 `bun.lock`，没有这个包。不要去找它、不要下载它、不要假装它存在。
  用你自己的构建配置（Bun.build 或等价工具，配 Tailwind 插件）复现同一份输出：入口 `client/src/main.tsx`、产物 `client/dist/index.html` + 资源文件。构建产物必须和原来一致，构建方式可以换。

## 必须做的 5 处垫片（只允许动这 5 处，且不得改变任何视觉输出）

App.tsx 只在以下 5 个点依赖 Muse 运行时，全部用最小垫片替换。垫片只能替换“能力”，不能改 App.tsx 的任何一行界面代码。

| 原依赖 | 在 APK 里换成什么 | 要求 |
|---|---|---|
| `createActionClient`（`client/src/api.ts`） | WebView JS Bridge 客户端 | 保持 `api.xxx(args)` 的调用签名和返回数据形状完全一致；原来 POST 到 `./actions` 的 `{action, args}`，现在走 bridge 调原生 |
| `spaceQueryClient`（`main.tsx`） | 普通 react-query `QueryClient` | 等价替换；`main.tsx` 里 `<QueryClientProvider>` 包裹结构不变 |
| `SafeAreaTopScrim`（App.tsx:4983） | CSS 安全区 | `theme.css` 已引入 `tailwindcss-safe-area`，用 `env(safe-area-inset-*)` 实现同等效果；顶部毛玻璃条颜色仍是 `var(--bg)`；APK 不得再加原生标题栏 |
| `actionFields` 宏（App.tsx:20，`{...FIELDS.xxx.yyy}` 散布在各输入框上） | 本地同名实现 | 它只是从 zod 输入 schema 生成表单字段属性（label、校验等）。照 `server/src/actions.ts` 里的 zod schema 生成一份取值完全相同的实现；App.tsx 里所有 `{...FIELDS...}` 的展开结果必须与原来一致 |
| `main.tsx` 的 `.hatch-space-root` 外层 div | 保留 | `className="hatch-space-root"` 和 `data-hatch-space-root` 属性都留着；原来 SDK 靠它加安全区与 viewport 样式，APK 里用等效 CSS 复现（不要删这个 div，不要改它的 class） |

## 后端移植对照表（`server/src/actions.ts` → 原生层）

服务端动作的 TypeScript 逻辑（ATI 决策、RIR、恢复模型、休息规则、容量统计、计划版本管理）是纯逻辑，全部原样保留，只换掉下面 4 个 Muse 运行时能力：

| 原能力 | APK 等价实现 |
|---|---|
| `ctx.db<typeof schema>()`（Drizzle + SQLite） | 原生 SQLite，表结构按 `drizzle/` 迁移 0001–0009 一比一建；查询逻辑按 `actions.ts` 逐句对齐 |
| `ctx.invalidateQueries()` | bridge 返回成功后，在网页侧触发 react-query 对应失效（与原来行为一致） |
| `ctx.agent.spawnTask`（Drive 上传，`startDriveUpload`） | `uploadToDrive` 动作改为**本地保存日志文件**（默认行为）；界面上的上传按钮、文案、成功/失败反馈一律保留原样；Google Drive 上传只在用户明确开启并授权后才做，没有 Drive 时一切功能正常 |
| `ctx.inference.complete`（AI 裁决，见下节） | 必须按“AI 裁决缺口”一节处理，不得偷偷改成别的规则 |

动作清单以 `actions.ts` 末尾的 `defineAction` 为准，包括但不限于：`getDay`、`savePlan`、`setPlanTestDay`、`deletePlan`、`updatePlanSetWeight`、`startSession`、`logSet`、`submitFeedback`、`resumeAdjudication`、`respondProposal`、`undoProposal`、`acknowledgeAlert`、`finishSession`、`markLegacySets`、`deleteSession`、`uploadToDrive`。

## AI 裁决缺口（必读，不许绕过）

这是唯一一个**不能**原样搬进 APK 的能力，必须正面处理：

- ATI 裁决（`runAtiDecision`，约 actions.ts:1293 起；另有 `atiReviewPrompt` 两处总结调用、计划生成处的几次调用）原来走 `ctx.inference.complete`，即 Muse 运行时的模型推理。APK 里没有这个运行时，离线状态下调不到模型。
- **处理方式，二选一，按顺序：**
  1. **用户自带模型接口（首选）：** APK 设置里留一个模型接口配置项（接口地址 + Key，用户自己填）。配好后，裁决请求按原来的提示词结构（上下文 10.3 结构 + 44 条证据检索 + 14 个可组合裁决动作）发给用户配置的模型，返回后走原来同一套确定性安全校验；校验不通过则按原逻辑重试/走保守缺省。Key 只存本机，不上传、不进仓库。
  2. **确定性保守缺省（未配置模型时的默认行为）：** 代码里已经写好了这条路，直接用它，不要自己发明新规则：无明确停止信号时“按原计划继续，休息按原定时间，不额外加时”；出现明确停止信号（关节刺痛、动作变形且无法纠正、RIR 0 且变形、持续酸痛 >72h）时“停止该动作当日剩余组”。裁决弹窗里显示的文案就用代码里已有的那两段（“本次 AI 裁决暂时不可用，已按保守缺省执行……”/“安全边界：出现明确停止信号……”），一字不改。
- **禁止：** 不许把 AI 裁决悄悄换成简化的硬编码规则（比如恢复旧的规则 A–G 程序）；不许在界面上加“AI 已替换为本地规则”之类的新提示；不许删裁决弹窗或改它的文案。用户在意的就是这个裁决流程原样可用，缺模型时宁可走保守缺省，也不要一个长得像但规则变了的假裁决。

## 数据

- 首次启动导入 `data/*.json`：sessions.json（9 场会话 + 111 条逐组记录）、planner.json、plan-versions.json、evidence.json、principles.json、stats.json。
- 导入后核对：已完成 8 场 / 92 组 / 26,203 kg（2026-11-11 至 2026-11-26），另有 1 场进行中（2026-11-18 腿日，id 9，11/14 组）。
- 字段形态以 `data/README-data.md` 为准（`actualReps` 等有字符串与数字两种形态，做数值兼容；RIR/动作变形/关节感受大量未记录，不许编造）。

## 验收（全部通过才算完成）

1. **逐屏截图对比：** APK 每个屏幕与 Muse 空间同一屏幕并排截图，逐项核对：布局、颜色（`#0B1210` / `#B9F27D`）、圆角、毛玻璃、字号、间距、图标。任何一处肉眼可见的不同都要列出来并改掉，直到一致。
   至少覆盖：首页、计划详情、开训、逐组记录、感受提交、ATI 裁决弹窗（接受/忽略/撤销）、结算复盘、统计、计划管理。
2. **主链路离线走通：** 断网状态下建计划 → 开训 → 逐组记录 → 提交感受 → 裁决弹窗出现 → 接受/忽略 → 结算 → 统计数字更新，全程可用。
3. **数据核对：** 导入后统计页显示 8 场 / 92 组 / 26,203 kg；逐组记录与 `data/sessions.json` 逐条对得上。
4. **裁决行为核对：** 未配模型时走保守缺省，弹窗文案与原代码一致；配了模型后走模型裁决 + 同一套安全校验；停止信号出现时相关动作当日剩余组被停掉。
5. **不回归：** Drive 上传按钮还在，点击后本地保存日志成功；没有 Drive 授权时不报错、不卡死。
