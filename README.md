# 大肥鱼工作室 · 多 AI 协作工作流

> **DSH（DeepSeek Harness）执行 × Claude Code 监工** —— 一个人类只当验收官的多智能体协作框架
>
> 人提需求 → Claude 派活并全程播报 → DSH 实时干活 → DSH 自检 → Claude 终审 → 交付人类

---

## 为什么做这套东西

一个人同时使用两个 AI（终端里的 Claude Code、独立跑的 DeepSeek Harness）时，常见的痛点是：

- 两个 AI 各自为战，成果散落各处，没有统一的工作区；
- 谁干了什么、卡在哪一步，全靠事后猜测；
- AI 干完的活没人复核，出错了要等人类发现。

本项目把三方协作固化成一套可复用的工作流：

| 痛点 | 解法 |
|---|---|
| 成果散落 | 统一工作区 `大肥鱼工作室\<任务名>\`，一任务一文件夹 |
| 过程黑盒 | 实时日志机制 + Web UI 直播，每一步动作可见 |
| 无人复核 | 双层审查：DSH 自检（初审）+ Claude 亲读文件（终审） |
| 分工不清 | 固定角色表：人类验收、Claude 监工、DSH 执行 |

---

## 角色分工

| 角色 | 职责 | 不做什么 |
|---|---|---|
| **人类（验收官）** | 提需求、看直播、最终拍板 | 不检查细节、不催进度 |
| **Claude Code（监工）** | 派活（显式展示启动命令）、播报进展、终审验收、汇报结论 | 不替 DSH 干活 |
| **DSH（执行者）** | 产出文件、自检初审、写 `_实时日志.md` | 不再调用 claude CLI 子进程初审 |

### 审查分工（双层）

```
DSH 干活 → DSH 自检（初审，留痕于 _实时日志.md） → Claude 亲读文件复核（终审） → 通过才汇报人类
```

> 为什么不让 DSH 起 claude 子进程初审：实测会永久挂起（headless 模式 pwsh 管道问题），
> 且独立 claude 进程与监工会话之间没有通信通道，审查结果无人接收。

### 视觉分工

| 内容类型 | 谁看 |
|---|---|
| 文本、文件、代码、页面快照 | Claude 直接看 |
| 截图、图片、UI 界面 | 派给 DSH（modlens vision），任务里附图片绝对路径，它输出所见结论，Claude 复核 |

---

## 工作区约定

```
D:\大肥鱼工作室\                  ← 协作根目录
├── <任务名>\                     ← 每个任务一个下级文件夹
│   ├── <产出文件>
│   ├── _实时日志.md              ← DSH 每完成一步追加一行
│   └── _dsh工作轨迹.md           ← 协作轨迹归档（可选）
```

产出文件禁止散落在根目录或其它位置。

---

## 三种运行模式

### 模式一：Headless（适合重任务）

Claude 在工作室目录启动 DSH，任务直接写入任务文件夹，最终汇报经 stdout 回到监工。

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

人类在浏览器里亲眼看 DSH 干活（Think 过程、每次工具调用、失败重试、自检表），Claude 用 agent-browser 通过 CDP 自动化派活并当监工。

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

想让人类边看边让 Claude 派活时：`dsh web` 服务常驻 3080 当看板 + headless 任务照常在后台派发，两条线共享会话库。

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
- Claude 仍负责 tail `_实时日志.md` 播报 + 终审复核。

**进程回收**：任务完成后主动问人类是否停止进程（dsh web 服务、浏览器等），防止 token 消耗。

---

## 实战案例

### 案例一：html协作试验（Web UI 直播首测）

- **任务**：修复 `计算器.html` 的两个提示级问题（keydown 缺 `e.isComposing` 检查、使用说明未同步）
- **结果**：55 秒 · 11 步 · 产出正确，监工终审通过
- **意外收获**：现场暴露 DSH 的 Bash/Glob 工具处理中文路径会崩溃（exit 3221225794），DSH 自行诊断并改用 PowerShell 绕过——headless 模式下这个问题永远看不见

### 案例二：半导体专利改进（headless 模式）

- **任务**：改进 4 份半导体专利草稿
- **结果**：4 篇全部改进完成，监工终审抓出说明书/权利要求不一致、实用新型客体风险、专利号格式错误等问题，出具了审核报告

### 案例三：双线并行首测（2026-08-27）

- **任务**：短任务（3 文件）+ 2 分钟分步长任务（4 步、每步间隔 30 秒），验证「web 服务看板 × headless 后台」同时运行
- **结果**：两个任务均通过终审；运行中会话需刷新页面才出现在 UI 树（刷新后自动打开最新会话），完成后带自动标题常驻

---

## 踩坑记录（含解法）

| # | 现象 | 根因 | 解法 |
|---|---|---|---|
| 1 | `NO_ADAPTER: no adapter registered for provider "deepseek-modlens"` | settings.yaml 用户层在运行时覆盖 cordis 配置，headless 栈不加载 modlens | patch 里 `- id: settings, disabled: true` |
| 2 | `$.model missing required value` | cordis patch 的 config 是**整体替换**而非浅合并 | patch 写全 provider / model / reasoningEffort |
| 3 | headless 下 DSH 起 claude 子进程初审时永久挂起（20+ 分钟无事件） | pwsh 工具生成子进程的管道问题 | 审查分工改为：初审 DSH 自检、终审监工 Claude |
| 4 | Web UI 会话里 Bash/Glob 工具遇中文路径崩溃（exit 3221225794 = DLL 初始化失败） | Web UI 服务进程环境与中文路径编码冲突 | 不干预，DSH 会自行改用 PowerShell；或任务里给短路径 |
| 5 | claude CLI 子进程偶发失败（stdout 为空、exit 非零） | 环境变量 `ANTHROPIC_DEFAULT_*` 被设成 deepseek 模型名 | `env -u ANTHROPIC_DEFAULT_* claude -p "..." --allowedTools "Read"` |
| 6 | 权限分类器拦截 `--dangerously-skip-permissions`、`-ExecutionPolicy Bypass`、`DSH_PERMISSION_MODE=danger-full-access` | 安全策略 | 用最小权限替代：`--allowedTools "Read"`；直接 `powershell -NoProfile -Command`；输出限制在 dsh 工作区内 |
| 7 | 想诊断 DSH 卡住原因但看不到内部状态 | 会话数据是 zstd 压缩的 JSONL | 用 web profile 的 `fzstd` 包解压 `~/.dsh/sessions/<cwd转义>/session-<id>/session.jsonl.zstd` |
| 8 | headless 下 DSH 的 bash 工具不可用（DSH 自报） | headless 栈环境限制 | 无需干预，DSH 自行改用 PowerShell（sleep 用 `Start-Sleep`） |
| 9 | 双线并行时运行中的 headless 会话不在 UI 会话树里 | 侧栏不自动轮询，服务端查询按需触发 | 刷新页面即出现（刷新后自动打开最新会话） |

---

## 关键设计原则

1. **透明度优先**：派活过程显式可见——先向人类展示完整启动命令，再执行；人类随时能在日志或 Web UI 里看到「下命令 → 干活 → 返回 → 复核」全过程。
2. **角色不越界**：监工不替执行者干活，执行者不自封终审；人类是唯一验收人。
3. **真实自检**：DSH 的自检必须真实读文件、逐项核对，有 bug 就修，修完再查。
4. **最小权限**：所有子进程调用按需授权，宁可用受限工具组合，不用一键放行开关。

---

## 技术栈

- [DeepSeek Harness (dsh)](https://www.deepseek.com/) — 执行者，DeepSeek 官方 agent 框架（cordis 内核，web profile + headless profile）
- [Claude Code](https://claude.com/claude-code) — 监工，派活、播报、终审
- [agent-browser](https://github.com/vercel-labs/agent-browser) — Chrome DevTools Protocol 自动化，Web UI 直播模式的"手"
- Chrome（remote-debugging-port 9222）— 直播窗口，人类用眼睛验收现场
