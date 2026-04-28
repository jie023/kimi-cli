# 只读模式功能清单

## 一、进入/退出只读模式

| 功能 | 方式 | 说明 |
|------|------|------|
| 启动时进入 | `--readonly` CLI 参数 | 命令行启动时直接以只读模式运行 |
| 启动时进入 | `default_readonly=true` 配置 | `kimi-cli` 配置文件中设置默认只读 |
| 运行中进入 | `/readonly` 斜杠命令 | 当前会话切换为只读模式 |
| 运行中退出 | `/execute` 斜杠命令 | 解除只读模式，并批量执行待修改清单 |

## 二、工具拦截策略

| 工具 | 只读模式行为 |
|------|-------------|
| `WriteFile` | **全部拦截**，记录到待修改清单 |
| `StrReplaceFile` | **全部拦截**，记录到待修改清单 |
| `AgentTool`（子代理） | **全部拦截**，禁止启动子代理 |
| `Shell` | **智能拦截**：<br>- 后台任务：一律拦截<br>- 前台命令：检测是否安全，危险命令拦截，只读命令（`ls`、`cat`、`grep`、`git status` 等）放行 |

## 三、Shell 危险命令检测

检测维度：

- 重定向符 `>` / `>>`
- 文件操作命令：`rm`、`mv`、`cp`、`mkdir`、`touch`、`chmod` 等
- Git 危险操作：`git push`、`git commit`、`git merge`、`git reset` 等
- `sed -i` 原地编辑
- 安装/构建命令：`pip install`、`npm install`、`cargo build`、`make` 等
- **.NET 文件操作 API**：`WriteAllText`、`File.Create`、`StreamWriter` 等
- **PowerShell Cmdlet**：`Set-Content`、`Out-File`、`New-Item`、`Remove-Item` 等
- **Python -c 文件写入**：`open(`、`os.remove`、`shutil` 等
- **Node.js -e**：一律拦截
- **其他脚本解释器**：`bash -c`、`php`、`ruby`、`perl` 等
- **网络下载**：`curl -o`、`wget --output-document`、`Invoke-WebRequest` 等

## 四、待修改清单（Pending Edits）

| 命令 | 功能 |
|------|------|
| `/pending` | 查看当前待修改清单 |
| `/pending-edit <序号>` | 将指定项的完整参数注入对话上下文，便于修改 |
| `/pending-remove <序号>` | 删除指定序号的待修改项 |
| `/pending-clear` | 清空全部待修改清单 |

**数据模型：**

- `tool_name`: 被拦截的工具名
- `params`: 工具调用的原始参数
- `description`: 人类可读的操作描述
- `timestamp`: 记录时间戳

## 五、批量执行流程

1. 用户发送 `/execute`
2. 系统解除只读模式
3. 将待修改清单按顺序注入 system message
4. AI 依次执行每个操作
5. 执行完成后清单自动清空

## 六、AI 行为引导

- `ReadonlyModeInjectionProvider` 动态注入提示词
- 明确告诉 AI：**直接调用工具**，系统会自动拦截并记录
- 禁止 AI 反复重试被拦截的操作
- 禁止 AI 只在文字中描述修改计划而不调用工具

## 七、状态持久化

- `SessionState.readonly`：会话级别的只读状态持久化
- `SessionState.pending_edits`：待修改清单持久化
- `StatusUpdate.readonly_mode`：UI 状态同步
