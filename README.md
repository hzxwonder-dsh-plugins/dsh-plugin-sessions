# DSH 会话引用与对话模式

`dsh-plugin-sessions` 为 DeepSeek Harness Web 提供会话引用复制，以及无需选择工作目录的对话模式。支持版本：`0.1.5-rc.2`，Node.js 24。

## 使用

- 点击左侧 **新会话** 直接开始对话：Host 会在聊天记录之外注册一个名为 **对话** 的工作区，指向 `~/.dsh/plain-sessions`，新建的会话就运行在该目录下，因此输入卡片始终可用，不需要先选择工作目录。
- **对话** 分组固定显示在左侧工作区列表末尾，对话会话集中其中，与项目工作区并列。
- 工作区行展开后，行内的 **新建会话** 按该工作区创建会话；此时输入卡片的工作区标签显示该工作区，可直接输入。
- 鼠标移到输入卡片的工作区标签上时，标签右侧出现 ⊗ 退出控件。点击后放弃当前空白工作区会话并返回对话；对话模式本身不显示该控件。
- 在左侧会话的 `…` 菜单中点击 **复制会话引用**，或点击当前会话标题旁的复制按钮。将引用粘贴到另一个会话，连同你的问题一起发送，Harness 会读取对应会话并将快照加入模型上下文。

引用使用 Harness 原生格式 `@[会话名称](dsh-session:编码后的ID)`，按 ID 精确定位，因此同名会话可正确区分。引用范围是当前 Harness 实例可访问的会话，包含已归档会话和分叉会话；引用读取使用 Host 的统一会话接口。SSH 会话的内容以当前 Harness 实例中保存的聊天记录为准。

原生解析器每条消息最多接受 3 个会话引用，并限制上下文大小；长会话会按预算提供快照及省略信息。引用不能指向正在发送该条消息的会话自身。复制动作只读取会话，不发送模型请求。浏览器拒绝剪贴板访问时，会显示可手动复制的引用框。

## 安装

在目标部署中将本目录作为本地插件加入所用 profile：

```sh
dsh plugin --profile web add /absolute/path/to/dsh-plugin-sessions
```

对话工作区由 Host 插件注册，芯片标签与退出控件位于原生会话输入卡片，需要启用版本校验的兼容适配：

```sh
node scripts/adapt-workspace.mjs /absolute/path/to/runtime
node scripts/adapt-workspace.mjs /absolute/path/to/runtime --apply
```

先停止当前部署，完成安装及适配后启动，并刷新浏览器。适配器检查版本与所有替换位置，再修改 `@deepseek-ai/dsh-client-ui-workspace` 和 `@deepseek-ai/dsh-client-ui-conversation` 的 `lib/client.js`；原始文件和校验值分别保存在同目录的 `client.js.dsh-sessions-backup.json`。官方运行时升级后需要重新验证适配器。

可在 profile 补丁中配置对话目录与名称：

```yaml
- id: dsh-plugin-sessions
  config:
    plainRoot: /absolute/path/to/plain-sessions
    title: 对话
```

Host 在启动和每次 `/api/dsh-sessions` 的 `chat` 请求中确保该工作区存在：目录按 `0700` 创建并解析真实路径，已注册同一路径时复用，不改变既有顺序。客户端只接管 `uiWorkspace.startSession`：无工作区参数的调用进入对话，带工作区参数的调用保留该工作区自己的新会话。

## 验证

在此部署布局中执行：

```sh
npm test
node scripts/integration-check.mjs
```

集成检查在临时 profile 和端口 3397 启动真实 Harness，使用本地确定性模型适配器验证对话工作区注册与目录创建、会话运行在对话目录、重复请求复用、重启后保持在工作区列表末尾，以及引用内容确实进入模型输入、引用来源持久化、归档/分叉会话及重启后的读取。检查结束清理临时 profile。

## 回滚

停止部署，在所用 profile 中移除插件，再恢复原生会话卡片：

```sh
node scripts/adapt-workspace.mjs /absolute/path/to/runtime --restore
```

恢复前会校验当前适配文件和备份。若文件已被其他修改覆盖，脚本会保留文件并报错。工作区注册表与聊天记录继续保留。
