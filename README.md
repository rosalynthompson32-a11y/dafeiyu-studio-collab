# 大肥鱼工作室 · 多 Agent 协作工作流

> **Claude Code 监工主持 × DSH（DeepSeek Harness）执行主力 × GPT 审查终测 × Qwen 长图文制作** —— 人类只当验收官的多 Agent 协作框架
>
> 人提需求 → 主持与审查席议方案 → 派活（随单附验收命令）→ 执行者实时干活并自检 → 审查席实跑终测 → 问题回环 → 交付人类
>
> 两条协作通道：人在电脑前用本机直连（Headless / Web UI 直播 / 双线并行）；人不在电脑前用飞书群聊 @ 派活
>
> 📌 2026-10 升级：工作流进入「五步常态流 ＋ 终测闸门」，DSH 侧完成预设/记忆/路由全面升级 —— 详见 [docs/常态工作流.md](docs/常态工作流.md)、[docs/架构-DSH增强全景.md](docs/架构-DSH增强全景.md)、[docs/全功能总览.md](docs/全功能总览.md)

---

## 为什么做这套东西

一个人同时使用多个 AI（终端里的 Claude Code、独立跑的 DeepSeek Harness、桌面 Qwen Code）时，常见的痛点是：

- 多个 AI 各自为战，成果散落各处，没有统一的工作区；
- 谁干了什么、卡在哪一步，全靠事后猜测；
- AI 干完的活没人复核，出错了要等人类发现；
- 各 AI 能力各有所长，靠人脑记「什么活丢给谁」容易错配。

本项目把多方协作固化成一套可复用的工作流：

| 痛点 | 解法 |
|---|---|
| 成果散落 | 统一工作区 `大肥鱼工作室\<任务名>\`，一任务一文件夹 |
| 过程黑盒 | 实时日志机制 + Web UI 直播，每一步动作可见 |
| 无人复核 | 审查独立成席：执行者自检 ＋ 审查席（GPT）实跑终测，只认证据 |
| 分工不清 | 多席角色表 + 按能力派活：人类验收、Claude Code 主持、DSH 主力执行、GPT 审查、Qwen 长图文 |

---

## 角色分工（多席制 · 2026-10）

| 角色 | 职责 | 不做什么 |
|---|---|---|
| **人类（验收官）** | 提需求、看直播、最终拍板 | 不检查细节、不催进度 |
| **Claude Code（主持/总调度）** | 议方案、派活（显式展示启动命令）、播报进展、转派改单、汇报结论 | 不替执行者干活 |
| **DSH / DeepSeek Harness（执行主力）** | 产出文件、自检、写 `_实时日志.md`、批量后台任务；文档/报告/呈现/渲染类默认归此席 | 不自封终审 |
| **GPT（审查席，只读）** | 方案复核、实跑终测与复测（只认实跑＋证据）；缺判据的派单当场退回 | 不改文件 |
| **Qwen（长图文执行岗）** | 识图、长文本理解、PDF/PPT/Excel 制作、图文排版 | 不接通用代码工程与系统类 |
| **GLM（代码设计席，候补）** | 多文件工程/算法/协议等精细化设计（默认仍归执行主力，按需分派） | 不自检自评 |

### 按能力派活

| 活 | 派给谁 |
|---|---|
| 议方案、主持、终报 | Claude Code |
| 文档/报告/呈现/批量/后台长活/渲染 | DSH |
| 方案审查、实跑终测与复测 | GPT |
| 识图、长文、PDF/PPT/Excel | Qwen |
| 多文件工程/算法/协议（精细化） | GLM（或由主持酌定） |
| 截图/UI 视觉判断（无长图文席时） | DSH（modlens vision），任务里附图片绝对路径，执行者输出所见结论，主持复核 |

### 审查分工

```
执行者干活 → 执行者自检（留痕于 _实时日志.md） → 审查席实跑终测（只认实跑＋证据） → 问题回环 → 汇报人类
```

> 回环两层：小瑕疵（写法/格式/命名/次项缺漏）由审查席出改法清单 → 主持转原会话改 → 复测，不上报人类；
> 实质问题（方向错/崩坏/缺件/动了不该动的）呈人类 → 人类与审查席议改 → 主持落单返工。
>
> 为什么不让执行者起 claude 子进程初审：实测会永久挂起（headless 模式 pwsh 管道问题），
> 且独立 claude 进程与主持会话之间没有通信通道，审查结果无人接收——故审查独立成席。

---

## 现行工作流：五步常态流（2026-10）

1. **议方案** —— 主持＋审查席按需求先议（轻件一句话级；重件两分钟门禁）。
2. **派活** —— 主持按型定向，**随单附验收命令**；缺判据的派单由审查席当场退回（**生产方不得自带考卷**）。
3. **执行** —— 各席按型接活；全程写实时日志。
4. **实跑终测** —— 审查席只认实跑＋证据；复测范围写死（只测改动项；若改动触及已过项，该方向全量重测）。
5. **问题回环** —— 小瑕疵原会话改＋复测不上报；实质问题呈人类议改。

完整口径（类型定向判据、红线三条、终测闸门两档、收撤清单）见 [docs/常态工作流.md](docs/常态工作流.md)。

## 架构升级：DSH 全面增强（2026-10）

执行主力的 DSH 已完成预设／记忆／路由全面升级，形态为一套**可复刻装配**：

- **增强层**：38 个插件组成的一体化 agent 工作台 ＋ 6 个本地扩展件（带源码）＋ 皮肤氛围层；
- **内核机制**：多智能体预设（席位制）、跨会话长期记忆（宿主／客户端／来源接入）、多模型路由、MCP 工具接入、技能管理；
- **治理与发行**：双层治理（治理索引＋派单规程）、验收与集成分离、发版四件套（清单／扫描／校验／打包）＋ 发行台账与回退行。

全景与复刻步骤见 [docs/架构-DSH增强全景.md](docs/架构-DSH增强全景.md)。

---

## 两条协作通道

### 通道一：本机直连（人在电脑前）

监工在工作区启动 DSH、派活、播报、终审——见下文「三种运行模式」。

### 通道二：群聊 @ 协作（人不在电脑前）

各 agent 以机器人身份接入同一飞书群，**群聊里 @ 谁谁干活**：

- **@Claude Code**：派活主持、复核、汇报
- **@DSH**：批量执行
- **@Qwen**：识图、长文、制作

约定：私聊 = 各自单独完成；群聊 @ = 协作（完成后 @ 监工反馈 + @ 人类汇报）；验收以群内可见产出为准。

---

## 工作区约定

```
D:\大肥鱼工作室\                  ← 协作根目录
├── <任务名>\                     ← 每个任务一个下级文件夹
│   ├── <产出文件>
│   ├── _实时日志.md              ← 执行者每完成一步追加一行
│   └── _dsh工作轨迹.md           ← 协作轨迹归档（可选）
```

产出文件禁止散落在根目录或其它位置。

---

## 三种运行模式（本机直连）

### 模式一：Headless（适合重任务）

监工在工作室目录启动 DSH，任务直接写入任务文件夹，最终汇报经 stdout 回到监工。

```bash
cd /d/大肥鱼工作室 && /d/DeepSeek-Harness/node_modules/.bin/dsh \
  --profile headless \
  --patch /d/DeepSeek-Harness/patch-headless.yml \
  "<任务描述：含任务文件夹名、产出要求、输出路径>"
```

patch 文件（`D:\DeepSeek-Harness\patch-headless.yml`）：

```yaml
# headless 栈不加载 modlens 插件，把 provider 从 deepseek-modlens 覆盖回官方 adapter
- id: agent-default-model
  config:
    provider: deepseek-official
    model: deepseek-v4-flash
    reasoningEffort: high
# settings.yaml 里 agent-default-model 指向 deepseek-modlens，会在运行时投影覆盖上面的配置；
# 禁用 settings 插件让配置回落到 cordis 树
- id: settings
  disabled: true
```

**实时可见玩法**：任务文本要求 DSH「每完成一个关键步骤立即向 `<任务文件夹>/_实时日志.md` 追加一行 `## HH:MM 动作描述`」→ 后台启动 DSH → 监工每 30–60 秒 tail 日志向人类播报。

### 模式二：Web UI 直播（适合想看全过程）

人类在浏览器里亲眼看 DSH 干活（Think 过程、每次工具调用、失败重试、自检表），监工用 agent-browser 通过 CDP 自动化派活。

```bash
# 1. 启动带调试端口的 Chrome（独立用户目录，保留登录态）
powershell -NoProfile -Command "Start-Process 'chrome.exe' -ArgumentList '--remote-debugging-port=9222','--user-data-dir=%USERPROFILE%\AppData\Local\Google\Chrome-Automation'"

# 2. 打开 DSH Web UI
cd <agent-browser 目录>
AGENT_BROWSER_HOME=<agent-browser 目录> ./bin/agent-browser --cdp 9222 open http://127.0.0.1:3080

# 3. 找输入框、填入任务、发送
AGENT_BROWSER_HOME=<agent-browser 目录> ./bin/agent-browser --cdp 9222 snapshot -i   # 找 textbox ref
AGENT_BROWSER_HOME=<agent-browser 目录> ./bin/agent-browser --cdp 9222 fill @eN "<任务>"
AGENT_BROWSER_HOME=<agent-browser 目录> ./bin/agent-browser --cdp 9222 click @e<发送按钮>
```

任务文本要点：写绝对路径 `D:\大肥鱼工作室\<任务>\`；注明「跳过 claude CLI 初审」。

### 模式三：双线并行（Headless 后台干活 × Web UI 看板）

让人类边看边派活时：`dsh web` 服务常驻 3080 当看板 + headless 任务照常在后台派发，两条线共享会话库。

```bash
# 1. 起 Web UI 服务（会自动打开默认浏览器；人类浏览器直接看 http://127.0.0.1:3080）
cd /d/DeepSeek-Harness && npx dsh web

# 2. headless 照常后台派活（与模式一完全相同）
cd /d/大肥鱼工作室 && /d/DeepSeek-Harness/node_modules/.bin/dsh \
  --profile headless \
  --patch /d/DeepSeek-Harness/patch-headless.yml \
  "<任务>"
```

验证结论（2026-08-27 实测）：

- headless 会话会出现在 Web UI 会话树——服务端从磁盘快照读取；
- **运行中的会话侧栏不自动轮询，刷新页面后即出现**，刷新后 UI 自动打开最新会话，可看任务原文、思路、每次工具调用全程；
- 已完成的会话带自动标题常驻树中；
- headless 下 DSH 的 bash 工具不可用，它会自行改用 PowerShell（如 `Start-Sleep`）；
- 监工仍负责 tail `_实时日志.md` 播报 + 终审复核。

**进程回收**：任务完成后主动问人类是否停止进程（dsh web 服务、浏览器等），防止 token 消耗。

---

## 实战案例（进阶）

### 案例一：多智能体群聊协作回路（实测全通）

- **形态**：多个 agent 以机器人身份入同一群，**@ 谁谁干活**；派活 → 产出 → 验收全链路在群内可见，人不在电脑前也能主持。
- **硬骨头**：跨应用 `open_id` 隔离——白名单必须存"对方应用视角"的 id；卡片消息发送后不可编辑 → 关键节点改发**带 @ 的纯文本副本**作中继。
- **约定沉淀**：私聊＝各自单独完成；群 @＝协作（完成后 @ 主持反馈 ＋ @ 人类汇报）；验收以群内可见产出为准。

### 案例二：跨会话记忆系统 ＋ 可复刻发行线（大工程样板）

- **记忆**：宿主／客户端两层 ＋ 来源接入（摘要＋指针，原文不动）；跨会话、跨项目保留；可修剪、可改档。
- **发行线**：件集**三源并集实取**（link 依赖 ∪ bundles ∪ patch 挂载）、denylist 代码化（扫描/校验/打包三处同一规则）、守卫校验（改一字节→红 exit 1）、打包**逐条 sha256 对账**、发行台账「一行一次发行」＋回退行。
- **真实数据**：一次终版发行 **19 件／771 文件**逐条对账全绿；施工中途的"漂移"被守卫逮住 → 待稳定后重取件集（漂移处置行留痕）；异常序全程留档，可追溯每一次打包。
- **看点**：把多文件工程的版本治理做成了**可复跑、可举证、可回滚**的流水线。

### 案例三：确定性渲染管线（逐帧动画出片）

- **问题**：无头环境直接录动画不可复现——虚拟时钟不推 `requestAnimationFrame`，截帧全空。
- **解法**：**接管动画时钟**（自绘 `__draw(t)` 帧函数）＋ CDP 锁定视口逐帧截帧 ＋ ffmpeg 编码成片；用相位定格与像素级校验（`elementFromPoint` 验真）保证每一帧内容可信。
- **产出**：任意时长、任意帧率的**确定性**动画视频——可回归、可对比、可复现。

### 案例四：会话库解码器（对 zstd 多帧容器的逆向）

- **问题**：客户端把会话数据存成 zstd 压缩的多帧 JSONL 容器，官方没有任何导出工具。
- **解法**：逆向容器结构 → 写出解码器（`read-dsh-session.mjs`），完整还原系统提示词、消息流与工具调用轨迹。
- **用途**：调试、取证、迁移与归档——黑盒变白盒。

### 案例五：官方表格文档流水线（模板保真）

- **问题**：官方 `.docx` 表格结构复杂（内容控件 ＋ 嵌套表格 ＋ 自动编号），既要填全、又不能破坏原版式。
- **解法**：文档顺序遍历（穿透 `sdt` 与嵌套表）＋空槽克隆插入 ＋按官方样例**校准字号／字体／首行缩进**（正文小四＋首行 2 字符、封面黑体分级字号）＋编号防冲突（自带自动编号的段落剥除手打前缀）。
- **工程细节**：文件被占用时**后台守候、一解锁自动落盘**；全流程两段式可重跑（填充器 → 格式化器）。

---

## 踩坑记录（含解法）

| # | 现象 | 根因 | 解法 |
|---|---|---|---|
| 1 | `NO_ADAPTER: no adapter registered for provider "deepseek-modlens"` | settings.yaml 用户层在运行时覆盖 cordis 配置，headless 栈不加载 modlens | patch 里 `- id: settings, disabled: true` |
| 2 | `$.model missing required value` | cordis patch 的 config 是**整体替换**而非浅合并 | patch 写全 provider / model / reasoningEffort |
| 3 | headless 下 DSH 起 claude 子进程初审时永久挂起（20+ 分钟无事件） | pwsh 工具生成子进程的管道问题 | 审查分工改为：初审执行者自检、终审监工 |
| 4 | Web UI 会话里 Bash/Glob 工具遇中文路径崩溃（exit 3221225794 = DLL 初始化失败） | Web UI 服务进程环境与中文路径编码冲突 | 不干预，DSH 会自行改用 PowerShell；或任务里给短路径 |
| 5 | claude CLI 子进程偶发失败（stdout 为空、exit 非零） | 环境变量 `ANTHROPIC_DEFAULT_*` 被设成 deepseek 模型名 | `env -u ANTHROPIC_DEFAULT_* claude -p "..." --allowedTools "Read"` |
| 6 | 权限分类器拦截 `--dangerously-skip-permissions`、`-ExecutionPolicy Bypass`、`DSH_PERMISSION_MODE=danger-full-access` | 安全策略 | 用最小权限替代：`--allowedTools "Read"`；直接 `powershell -NoProfile -Command`；输出限制在 dsh 工作区内 |
| 7 | 想诊断 DSH 卡住原因但看不到内部状态 | 会话数据是 zstd 压缩的 JSONL | 用 web profile 的 `fzstd` 包解压 `~/.dsh/sessions/<cwd转义>/session-<id>/session.jsonl.zstd` |
| 8 | headless 下 DSH 的 bash 工具不可用（DSH 自报） | headless 栈环境限制 | 无需干预，DSH 自行改用 PowerShell（sleep 用 `Start-Sleep`） |
| 9 | 双线并行时运行中的 headless 会话不在 UI 会话树里 | 侧栏不自动轮询，服务端查询按需触发 | 刷新页面即出现（刷新后自动打开最新会话） |
| 10 | 群聊里 @ 另一个 agent 没反应 | 飞书 open_id 按应用隔离，白名单里存的是自己应用视角的 id | 白名单必须存对方应用视角的 open_id |
| 11 | 飞书卡片消息事后无法恢复编辑 | 平台限制，卡片发送后不可改 | 关键节点发带 @ 的纯文本副本作中继 |
| 12 | 验收时"空手卡"陷入反复询证、来回拉扯 | 验收探针未带真工件，只剩口头问答 | 验收命令必须附带**真工件与可复跑步骤**；判死线 ≥15 分钟 |
| 13 | 群聊互 @ 测试误触身份中继（@ 命令被当真人派单转发） | 机器人身份互借导致指令乱流 | 测试一律用**自身身份＋单人 @**；发现乱流即整线离线刹车 |

---

## 关键设计原则

1. **透明度优先**：派活过程显式可见——先向人类展示完整启动命令，再执行；人类随时能在日志或 Web UI 里看到「下命令 → 干活 → 返回 → 复核」全过程。
2. **角色不越界**：监工不替执行者干活，执行者不自封终审；人类是唯一验收人。
3. **真实自检**：执行者的自检必须真实读文件、逐项核对，有 bug 就修，修完再查。
4. **最小权限**：所有子进程调用按需授权，宁可用受限工具组合，不用一键放行开关。

---

## 技术栈

- [DeepSeek Harness (dsh)](https://www.deepseek.com/) — 执行主力，DeepSeek 官方 agent 框架（cordis 内核，web profile + headless profile）
- [Claude Code](https://claude.com/claude-code) — 主持/总调度，议方案、派活、播报、汇报
- GPT（Codex CLI，只读）— 审查席，方案复核、实跑终测与复测
- Qwen Code（DashScope）— 长图文执行岗，识图、长文、PDF/PPT/Excel
- GLM（智谱）— 代码设计席（候补），多文件工程/算法/协议
- 飞书开放平台 — 群聊 @ 协作回路
- [agent-browser](https://github.com/vercel-labs/agent-browser) — Chrome DevTools Protocol 自动化，Web UI 直播模式的"手"
- Chrome（remote-debugging-port 9222）— 直播窗口，人类用眼睛验收现场
