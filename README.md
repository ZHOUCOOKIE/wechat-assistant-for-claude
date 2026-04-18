# wechat-assistant-for-claude

一个给 Claude（Claude Code / Cowork mode）使用的 Skill，让 Claude 通过 computer-use 操控 Windows 桌面版微信，在群聊或私聊里帮你发送 / 回复消息。

> A Claude Skill that drives the Windows desktop WeChat client via computer-use to send and reply to messages on your behalf. Skill content is in Simplified Chinese because both the target app (微信) and the intended audience are Chinese-speaking.

---

## 这个 Skill 能做什么

- 在指定群聊或私聊中发一条消息
- 先读最近几条聊天记录，再按照话题、关系和语气生成一条"贴切"的回复
- 跟进对方的新消息，延续对话

## 为什么要单独写一个 Skill

桌面版微信没有官方 API，Claude 只能靠"截图 + 视觉识别 + 模拟鼠标键盘"来操作。我在 Windows 上实际跑的时候撞上了一堆具体的坑 —— 中文输入会丢字、启动流程里 `Wechatappex` 会触发假授权错误、新版 UI 由 `msedgewebview2.exe` 渲染导致定位错位、托盘/最小化状态识别不对、聊天对象错认、坐标漂移等等。一个个排查清楚之后，把能稳定工作的全流程和解法沉淀进这份 Skill，这样 Claude 下次不用从零再踩一遍。

## 平台支持

- ✅ **Windows** — 我本人实测并调通的全流程，覆盖新版微信 4.x（Weixin.exe，UI 用 WebView2）和旧版 WeChat 3.x
- ❓ **macOS / Linux** — 我没测过。这份 Skill 里有一部分（比如 `Wechatappex` 假授权、WebView2 渲染）是 Windows 特有的问题，在 macOS 上可能根本不存在；但 UI 布局、进程名、快捷键、前台焦点机制都不一样，所以即使能跑也不一定按这份 Skill 描述的步骤。欢迎 macOS 用户试一下并反馈实际情况

## 安装方式

### Claude Code / Cowork mode

把 `SKILL.md` 放到你的 skills 目录下，作为一个子文件夹：

```
<你的 .claude 目录>/skills/wechat-reply-assistant/SKILL.md
```

例如：

- Cowork mode: `~/.claude/skills/wechat-reply-assistant/SKILL.md`
- Claude Code 项目级: `<项目>/.claude/skills/wechat-reply-assistant/SKILL.md`

重启 Claude 后，该 Skill 会自动出现在可用 skill 列表中，触发关键词示例见下。

### 触发关键词（Skill 的 description 里已写）

只要你在对话里说下面这类话，Claude 就会自动调用这个 Skill：

- "帮我发个微信给 XX"
- "回复一下 XX"
- "在 XX 群里说一下 ..."
- "跟进一下 XX 的消息"

甚至不用明确提"微信"二字 —— 只要上下文是聊天工具且提到"发给 XX / 在群里说 / 回复一下"，也会触发。

## 前置条件（使用前务必看）

1. **已登录的 Windows 桌面版微信**。Skill 不会替你扫码登录，遇到扫码页会停下来让你本人扫。
2. **computer-use MCP 可用**，并会在任务开始时一次性向你申请以下应用权限：
   - `微信`（新版微信 4.x 所有子进程都叫 `Weixin.exe`，会被 resolver 统一映射回"微信"，申请这一个就够 —— **不要**再追加 `Wechatappex` / `msedgewebview2.exe` / `Weixin` 等变体，resolver 会驳回）
   - `任务管理器`
   - `文件资源管理器`（托盘区域属于 Explorer 进程，找托盘里的微信图标需要它；也是解 `Wechatappex` 假授权错误的关键）
   - `clipboardWrite` 剪贴板写入（中文消息必须走剪贴板，`type` 直接打中文会丢字）
3. **建议独占前台使用**。Skill 工作时你不要再抢鼠标/键盘，以免点到错误位置。

## 关键设计点（简述）

完整内容见 [`SKILL.md`](./SKILL.md)。这里只列最容易踩坑、最值得注意的几条：

- **不要一上来就 `open_application("微信")`**。如果微信已经在后台跑，直接 `open_application` 会触发"新实例登录"流程，把你已登录的会话搅乱。正确顺序是：先判断进程在不在跑 → 在跑就从任务栏/托盘激活已有实例，不在跑才启动新实例。
- **`Wechatappex` 假授权错误的解法是 `open_application("文件资源管理器")`**。微信启动流程里某些中转层会以 `Wechatappex` 的身份占据前台，导致 `left_click` 被 frontmost-app 检查拦下来。打开文件资源管理器把前台焦点抢走就好，不是真的缺授权，**不要**去追加 `Wechatappex` 相关的授权（resolver 一定会驳回）。
- **中文消息走剪贴板**。`write_clipboard` + `ctrl+v` 发送，不要用 `type` 直接打中文。
- **发送前先读 5–10 条最近聊天记录**。消息要像"本人会说的话"，呼应对方的最近话题、沿用对话里已有的命名/语气/emoji 风格，不要模板化客套话，不要凭空编事实。
- **发送操作用 `computer_batch` 一次做完**减少轮次。发完再 `screenshot` 确认消息真的在对话区出现了。

## 隐私与边界

- Skill 只在用户明确让它发消息时才发。读聊天历史是为了写好这一条，不会把历史内容转述给其他人或写进其他文档。
- 不会替你主动联系第三方（除非你明确点名）。
- **不做**：删除消息、撤回消息、修改群设置、踢人、转账、发红包。这类敏感操作必须你本人来。
- 对话中出现的敏感信息（身份证、银行账号、密码、病情等）不会被引用或记录。

## 局限性

- 只在 Windows 上实测过；macOS / Linux 是否开箱能用未知，可能没有 Windows 那些坑，也可能有别的。
- 没有官方 API，完全依赖视觉 + 模拟输入，偶尔会因为微信版本 UI 改动需要重新校对坐标识别逻辑 —— 发现不工作时先 `screenshot` 对比，再看是哪一步定位方式失效了。
- 不处理"发语音 / 视频通话 / 发图片 / 发文件 / 发红包"等富媒体操作，只覆盖文本消息。

## 贡献

欢迎 issue 和 PR。如果你在不同 Windows / 微信版本上遇到了本 Skill 未覆盖的坑，附上现象、错误信息和你的解法，这是这份 Skill 最有价值的积累方向。

## License

[MIT](./LICENSE) © 2026 ZHOU JUNAN
