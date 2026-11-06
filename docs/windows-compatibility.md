# dsh-superagent-prototype · Windows 兼容性问题报告

> 实测环境：Windows 11 · Node v24.21.0 · git 2.56.0 · gh 2.103.3 · DSH NEXT 2.0.21-next
> 实测结果：**打下面 3 处补丁后，全链路跑通**（Issue → 通知 → 工单 → worker 会话 → 改文件 → PR）
> 上游没有开 issue/discussion，所以先写成文件。

---

## 摘要

这个原型在 Windows 上**根本起不来**，卡在三个独立的 POSIX 假设上。三处都是一行级的改动，不是架构问题。

| # | 症状 | 根因 | 严重度 |
|---|---|---|---|
| 1 | `setup: The agent GitHub CLI cannot read its account. spawn env ENOENT` | 用 `env` 命令改 PATH | **阻断** |
| 2 | `setup: git does not read the github.com credential configuration` | `GIT_CONFIG_GLOBAL` 带 `\\?\` 前缀，git 忽略 | **阻断** |
| 3 | `failed to create session: tools.restrict() names unknown global tool "bash"` | worker 白名单写死 `bash` | **阻断** |

---

## Bug 1 · `spawn env ENOENT`

**位置**：`src/sources/github.ts` → `workerProbe`

```ts
// 现状
return (argv) => runAsWorker(subprocess, [ "env", `PATH=${path}`, ...argv ], cwd, signal);
```

Windows 没有 `env` 命令。

**修法**：`runAsWorker` 的第 5 个参数本来就是 `env`，不需要外部命令。

```ts
return (argv) => runAsWorker(subprocess, argv, cwd, signal, { ...process.env, PATH: path });
```

---

## Bug 2 · `gitTooOld`（`GIT_CONFIG_GLOBAL` 前缀）

**位置**：`src/app/setup.ts` → `verifyEnvironment`

**现象**：git 返回空，判定失败，但 git 版本完全够（2.56）。

**实测（关键数据）**：

| `GIT_CONFIG_GLOBAL` | 结果 |
|---|---|
| `D:\sa-state\worker-gitconfig` | exit 0，读到了 ✓ |
| `D:/sa-state/worker-gitconfig` | exit 0，读到了 ✓ |
| `\\?\D:\sa-state\worker-gitconfig` | **exit 1，静默忽略** ✗ |
| `\sa-state\worker-gitconfig` | exit 0，读到了 ✓ |

**根因**：`workerGitConfig(stateDir)` 若 stateDir 带 `//?/` 前缀（Windows 长路径），拼出来的就是 `\\?\...`，git 不认。

**修法**：把路径洗成普通形式再交给 git，比对时两边按同一规则归一。

```ts
const plain = (x: string) => String(x ?? "")
  .split("\\\\?\\").join("").split("//?/").join("").split("\\").join("/");
const gitCfg = plain(workerGitConfig(stateDir));
const env = { ...process.env, GIT_CONFIG_GLOBAL: gitCfg, GIT_CONFIG_NOSYSTEM: "1", ... };
```

**另外**：`verifyEnvironment` 拿 `file:<路径>` 做**字符串全等比较**，路径分隔符、盘符、大小写任一不同就误判。建议改成**分解后比相对路径**：

```ts
const norm = (x) => String(x ?? "").replace(/^file:/, "").replace(/^[a-z]:/, "")
  .replace(/^\/+/, "").split("\\").join("/").toLowerCase();
if (norm(a[0]) !== norm(b[0]) || a[1] !== b[1]) throw new SetupError("gitTooOld");
```

---

## Bug 3 · worker 工具白名单写死 `bash`

**位置**：`src/app/sessions.ts`（`registerTool` 的 restrict 调用）

```ts
// 现状
agent.ctx.tools.restrict({ allow: [ "skill", "bash", "read", "read_image", "job_output", "job_list", "job_kill" ] })
```

`WORK_TOOLS` 里**已经同时有 `bash` 和 `pwsh`**（说明作者考虑过），但这一处 allow 列表只写了 `bash`。Windows 上没有 `bash` 工具 → `restrict()` 抛错 → **worker 会话开不出来**。

**修法**：按平台二选一，或两个都给（不存在的会在 restrict 时被拒，所以只能二选一）：

```ts
const shellTool = process.platform === "win32" ? "pwsh" : "bash";
agent.ctx.tools.restrict({ allow: [ "skill", shellTool, "read", ... ] })
```

**顺带**：`noWait()` 只判断 `name === "bash"`，`pwsh` 不享受"5 秒转后台"的规则，建议一并加上。

---

## 附：stateDir 的三方约束（这是最坑的一点，值得写进文档）

`stateDir` 必须同时满足三个互相矛盾的要求：

| 谁 | 要求 |
|---|---|
| 插件的 `Config` schema | 必须以 `/` 开头（`z.string().regex(/^\//)`） |
| DSH 的 workspace API | 必须是**完整限定**路径，`/sa-state` 会被拒：<br>`TypeError: Workspace path is not fully qualified` |
| git 的 `GIT_CONFIG_GLOBAL` | **不能**带 `\\?\` 前缀 |

唯一可行的组合：**`stateDir` 用 `//?/D:/...`，在交给 git 之前单独洗一遍**。

**建议**：把 schema 的 `^/` 放宽成"绝对路径"（Windows 上接受 `D:\` / `D:/` / `//?/D:/`），或者在文档里明确写这一条。

---

## 还缺一样东西（不算 bug，是文档没写透）

README 里 "Create a GitHub user account for the agent" 是**必须**的，不是建议：

- agent 账号和操作者账号**是同一个**时，自己建的 Issue 不会给自己发通知 → `trigger` 恒为 0，看起来像"插件不工作"
- 换成另一个账号 @ 它，立刻收到，全流程走通

建议在 SETUP 里加一句明确的判据：**"如果你用同一个账号自测，你不会看到任何 trigger。"**

---

## 实测证据（可复现）

打上三处补丁后：

```
Issue #3（另一账号提，@agent）
  → trigger    actor=1564asdsad  human=true
  → decision   mandate.strength = "explicit"
  → ticket     opened
  → worker     own session + own directory
  → PR #4      docs: 补一节「这是什么」  +6/-0
```

PR 的 diff 只动了目标文件，内容逐条对应请求。
