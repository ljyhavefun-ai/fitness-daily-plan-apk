# 每日健身计划 — Android APK 导出

> 给 Claude Code：这是一份从 Muse 全栈空间导出的完整代码 + 数据包，目标是把它改写成一个可独立运行的 Android APK。原空间不动，本目录是只读快照（2026-11-27 导出，当时 live v37）。
>
> **路线已改（2026-11-27 用户定）：WebView 套壳，UI 原样不动。先读 `WEBVIEW-SHELL.md`，那是唯一正确的路线，本文件其余部分的“改写”目标已作废。不要原生重实现任何界面。**

## 这是什么

「每日健身计划」是一个给单人（用户李峻宇，60 kg / 170 cm，目标 2027 年 8 月前增肌到 70 kg）用的训练记录与计划系统：

- 每周推/拉/腿 6 天轮换（周一推、周二拉、周三腿 + 核心、周四推、周五拉、周六腿 + 核心、周日休），每肌群每周 2 次。
- 训练时按计划逐组记录实际重量 / 次数 / RIR / 动作变形 / 关节感受 / 组间休息，系统按 ATI 规则实时给出结构化裁决（增减负重、延长休息、减组、停动作等），裁决只生成待确认建议，用户接受后才改动后续组。
- 计划按版本管理（v1–v6），含周工作组、动作清单、配重、休息与证据清单。

原实现：Muse 全栈空间，React 19 + TypeScript + Drizzle + SQLite（服务端动作在 `server/src/actions.ts`，迁移在 `drizzle/`，客户端在 `client/`）。见 `fitness-space/`。

## 目录结构

```
fitness-apk-export/
├── README.md                  # 本文件
├── .gitignore
├── fitness-space/             # 原空间代码快照（React 19 + TS + Drizzle SQLite）
│   ├── client/                # React 客户端
│   ├── server/src/actions.ts  # 服务端动作（计划/会话/逐组记录/ATI 裁决/统计/Drive 上传）
│   ├── drizzle/               # 迁移 0001–0009（数据模型以此为准）
│   ├── space.json             # 空间配置
│   └── package.json / bun.lock / bunfig.toml / tsconfig.base.json
├── data/                      # 导出数据（只读快照，见 data/README-data.md）
│   ├── sessions.json          # 9 场会话 + 111 条逐组记录
│   ├── planner.json           # 当前 Planner 全量（当前计划 v5、器械快照、动作库、事件）
│   ├── plan-versions.json     # 8 个计划版本 v1–v6
│   ├── evidence.json          # 44 条原书证据 E01–E44
│   ├── principles.json        # ATI 体系 + 7 条训练原则 + RIR 标尺
│   └── stats.json             # 汇总统计
├── docs/                      # 设计文档（ATI v2、健身体系重设计背景、交互逻辑、空间现状、设计师提示词 v2）
└── references/                # 参考资料（ATI 文档、证据技能、健身增肌知识库）
    └── books-BIBLIOGRAPHY.md  # 9 本书书目（全文不在仓库）
```

## 数据说明

- 导出时间 2026-11-27，范围 2026-11-11 至 2026-11-26：**8 场已完成 / 92 组 / 26,203 kg**，另有 1 场进行中（2026-11-18 腿日，id 9，11/14 组）。
- 逐场对照、容量口径、舍入说明与全部已知数据缺口见 `data/README-data.md`，先读它再动手——数据有缺口（RIR/动作/关节感受大量未记录、休息计时不规律、睡眠营养未结构化），不要编造补齐。
- 隐私：`data/` 已去掉全部 `driveLink`，只留 `driveFileName` 与 `driveStatus`；不要把任何 Google Drive 链接放回仓库。
- 数据模型以 `fitness-space/drizzle/` 迁移与 `data/*.json` 的实际字段为准；`actualReps` 等字段在不同场次有字符串与数字两种形态，导入时做数值兼容。

## 目标：改成 Android APK（已改为 WebView 套壳路线，见 WEBVIEW-SHELL.md）

**不要按下面这节的旧写法做原生重写。** 正确路线是 `WEBVIEW-SHELL.md`：WebView 跑原样网页 + JS Bridge + 原生 SQLite，UI 一个像素也不许改。下面这节里“本地存储替代后端”“保留核心规则”“Drive 上传改为可选”仍然有效，但实现方式以 `WEBVIEW-SHELL.md` 为准。

- **本地存储替代 Muse 空间后端**：用 Android 本地数据库（Room/SQLite 均可）实现原 Drizzle 模型——计划（plan）→ 训练会话（session）→ 逐组记录（set_log），含原始计划快照（orig*）、当前目标（target*）、实际（actual*）、RIR、变形、关节感受、ATI 裁决（proposal / aiRule / aiEvidence 等）。
- **保留核心规则**（不要简化成普通记账 App）：
  - ATI 裁决规则 A–G 与 14 个裁决动作（见 `data/principles.json` 的 `ati.actions` 与 `data/evidence.json` 的 `ruleEvidence`）、8 步统一决策循环；
  - RIR 五档标尺（3+ / 2 / 1 / 0 / 0 且变形），RIR 与完成次数矛盾时以完成次数为准；
  - 恢复分层 GREEN / BLUE / AMBER / RED、准备度权重（近期表现 30 / 睡眠 20 / 疲劳 15 / 酸痛 10 / 营养 15 / 心理 10）、休息基础表、安全边界 6 条；
  - 容量口径：已完成、未跳过、非热身的力量组，实际重量 × 实际次数；热身组与自重组不计容量。
- **Drive 上传改为可选**：原空间每场结束后上传一份 Markdown 健身日志到 Google Drive（`driveFileName` / `driveStatus` 字段即由此来）。APK 里改为可选项：默认本地保存日志，Drive 上传只在用户明确开启并授权后才做；没有 Drive 时一切功能正常。
- **9 本书全文不在仓库里**（版权原因）：`data/evidence.json` 与 `data/principles.json` 只含证据摘要、章节页码定位与引用，不含书的全文；`references/books-BIBLIOGRAPHY.md` 只有书目。不要从网上找盗版全文放进仓库；APK 需要引用时只引用书名/章节/页码。
- 不做联网账号体系、不做社交；先保证离线可用：建计划 → 开训 → 逐组记录 + ATI 裁决 → 结算复盘 → 看统计，这条主链路完整可用。

## 已知限制

- 动作库 52 条（ACTIVE 39 / CONDITIONAL 11 / PROVISIONAL 2），器械快照 ATI-EQ-1.1（2026-11-14 用户现场确认），见 `data/planner.json` 的 `equipmentSnapshot` 与 `data/principles.json` 的 `v2Layers`。
- 证据来源状态 `SOURCE_LIMITED`：章节与关键片段已定位核验，非整书逐页审计；裁决文案里的证据引用按原样保留，不要改写成更肯定的表述。
