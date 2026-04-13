<p align="center">
  <img src="https://img.alicdn.com/imgextra/i3/O1CN01Kpgs1i1duSbfODzCm_!!6000000003797-2-tps-240-240.png" width="120" alt="DingTalk">
</p>

# 我花 2 小时给 Claude Code 写了个钉钉 skill，现在 AI 自己给老板汇报工作

> 作为一个被钉钉消息淹没的打工人，我做梦都想让 AI 替我回消息、写周报、发日报。上周末 2 小时撸出来一个 Claude Code skill，现在它真的能自己在钉钉群里汇报进度、答复问题、发日报。代码完全开源，60 秒就能装好。

---

## 🤔 我在解决什么问题

每天打开钉钉，满屏都是：

- "这个需求做得怎么样了？"
- "今天的数据同步了吗？"
- "那个 bug 修好了没？"
- "@你 能看一下这个吗？"

我每回一条消息，要做三件事：
1. 切出去执行任务（查代码、跑脚本、看日志）
2. 把结果复制出来
3. 切回钉钉格式化成 Markdown 发出去

**这就是典型的上下文切换税。** 一天下来，光是切屏 + 复制粘贴就耗掉 2 小时。

## 💡 那干脆让 AI 直接在钉钉里干活呢？

思路很直接：

```
我在钉钉群 @机器人：查一下今天发了多少邮件
   ↓
机器人收到消息
   ↓
调用 claude -p "查一下今天发了多少邮件"  ← Claude Code 直接在工作目录执行
   ↓
把结果格式化成 Markdown 发回群里
```

**核心洞察：Claude Code 本身就是个强大的命令行工具，能读文件、跑脚本、查数据库。只要给它一个"耳朵"（钉钉）和一个"嘴巴"（钉钉），它就能在群聊里直接干活。**

## ⚙️ 最终成果：DingTalk Bridge

GitHub 地址：**https://github.com/mguozhen/dingtalk-bridge**

### 能力矩阵

| 功能 | 说明 |
|------|------|
| 📤 发送消息 | 从命令行或 Python 代码往钉钉群发 Markdown / 纯文本 |
| 📥 接收 @消息 | 24/7 监听群聊，被 @ 时自动响应 |
| 🤖 自动执行 | 收到消息 → 调用 `claude -p` → 返回结果到群里 |
| 💪 崩溃自愈 | 指数退避重连（5s → 60s），macOS LaunchAgent 守护 |
| 🔐 零硬编码 | 所有凭据通过环境变量或 config.json 管理 |
| 🧪 27 个回归测试 | 全部 mock，跑一下 0.15 秒看结果 |

### 实际运行效果

#### 场景 1：让 AI 自己生成日报

钉钉群里我 @ 机器人说：

> @MailBot 统计一下今天 duongwise5521793 和 duong337942 两个账号发了多少邮件，打开率和点击率分别是多少

机器人 5 秒后回复：

```markdown
**Done**

**Daily Email Report — 2026-04-13**

📧 **发送量**
- duongwise5521793-gmail: 10 封
- duong337942.warrior54-gmail: 14 封
- Total: 24 封

📊 **核心指标**
- 打开率: 37.5% (9/24)
- 点击率: 16.7% (4/24)
- 回复: 1 条（Kyle from ABC Dental）

⚠️ **风险信号**
- duongwise5521793 收到 Gmail daily limit warning，建议明天减至 8 封
```

**我没写一行代码生成这个报告。** Claude Code 自己去读了 `sent_emails.db`，自己去查了 `tracking.db`，自己判断了风险，然后格式化成 Markdown 发回群里。

#### 场景 2：老板问进度，AI 自己答

老板 @ 我问：

> 那个媒体 pitch 做到哪一步了？

我 @ 机器人转发一下：

> @MailBot 媒体 pitch 做到哪一步了

机器人回复（内部它读了 `outreach_tracker.md`）：

```markdown
**Done**

**Media Outreach Status — 2026-04-13**

📬 **累计投递**: 71 家顶级科技媒体
- 第一批：TechCrunch, VentureBeat, Wired 等 41 家
- Retell 对标批：SiliconANGLE, VentureBeat, DestinationCRM 等 6 家  
- 垂直行业批：Small Business Trends, Dental Economics, HVACR Business 等 24 家

📈 **回复情况**
- 已回复: 3 家（Kyle Wiggers/TC 要补充材料，Emilia David/VB 感兴趣，其余在跟进）
- 待跟进: 15 家在 72 小时窗口内
- 今日新增目标: Jack Clark (Anthropic) 的周刊

📌 **下一步**
- 24h 内跟进 Kyle Wiggers 的补充材料请求
- 锁定 2 家日本 DX 媒体完成本地化 pitch
```

整个过程我没动一下手。

#### 场景 3：深夜报警

凌晨 3 点，监控脚本发现 SMTP 连不上。机器人自己在群里发：

```markdown
**Alert**

🚨 **SMTP 异常** — 2026-04-13 03:17

账号: duong337942.warrior54-gmail
错误: 550-5.4.5 Daily limit exceeded
影响: 今日剩余 6 封邮件发送失败

**自动处置**:
- ✅ 已切换至 Reserve 池 quyduong79981111
- ✅ 已标记 duong337942 为今日冷却
- ⚠️ 需人工确认明天是否恢复
```

我第二天醒来看到，问题已经自愈了。

## 🛠️ 技术实现：3 个核心模块

### 1. 发送消息（OpenAPI）

钉钉的 OpenAPI 需要两步：先拿 accessToken，再调 `groupMessages/send`。

```python
def send_markdown(title, text):
    conv = _get_conv_info()   # 读 data/conv.json
    token = _get_access_token()  # 用 APP_KEY + SECRET 换 token
    
    payload = {
        "robotCode": conv["robotCode"],
        "openConversationId": conv["openConversationId"],
        "msgKey": "sampleMarkdown",
        "msgParam": json.dumps({"title": title, "text": text}),
    }
    # POST to api.dingtalk.com/v1.0/robot/groupMessages/send
```

**踩坑记录**：`openConversationId` 不是从后台拿的，是机器人第一次被 @ 时钉钉推给你的。所以首次使用必须先让机器人收一条消息，自动保存到 `conv.json`。

### 2. 接收 @消息（Stream SDK）

钉钉推荐 Stream 模式：长连 WebSocket，有消息推给你，不用暴露公网回调。

```python
class BridgeHandler(dingtalk_stream.ChatbotHandler):
    async def process(self, callback):
        text = callback.data["text"]["content"]
        webhook_url = callback.data["sessionWebhook"]
        
        # 非阻塞：丢到子线程执行，handler 立刻返回 ACK
        threading.Thread(
            target=execute_prompt,
            args=(text, webhook_url),
            daemon=True,
        ).start()
        
        return AckMessage.STATUS_OK, "OK"
```

**关键细节**：handler 必须立刻返回 ACK，否则钉钉会认为你处理失败并重试。执行 Claude CLI 可能要 30 秒，所以必须异步。

### 3. 24/7 存活（魔鬼藏在细节里）

这是最痛的部分。钉钉 WebSocket **30 分钟不活动会自动断开**，而 `dingtalk_stream` SDK 的默认 keepalive 是没处理好的。我的做法：

```python
# 20 秒一次手动 ping，远低于 30 分钟阈值
async def _keepalive(self, ws, ping_interval=20):
    while True:
        await asyncio.sleep(ping_interval)
        try:
            await ws.ping()
        except websockets.exceptions.ConnectionClosed:
            break

client.keepalive = types.MethodType(_keepalive, client)
```

叠加：

- ✅ 指数退避重连（5s → 10s → 20s → 40s → 60s 封顶）
- ✅ `BaseException` 全抓（包括 KeyboardInterrupt 以外的所有）
- ✅ macOS LaunchAgent `KeepAlive=true`（进程挂了 launchd 自动拉起）

跑了快一周，没断过。

## 🚀 如何 60 秒装上

### 前置：创建钉钉机器人

1. 去 https://open-dev.dingtalk.com/
2. 创建「企业内部应用」→ 启用「机器人」能力
3. 消息接收模式选 **Stream 模式**
4. 复制 App Key 和 App Secret
5. 把机器人拉进群里

### 安装

```bash
# 克隆到 Claude Code skills 目录
git clone https://github.com/mguozhen/dingtalk-bridge.git \
  ~/.claude/skills/dingtalk-bridge

# 一键安装
bash ~/.claude/skills/dingtalk-bridge/scripts/install.sh
```

安装脚本会：
- 装 Python 依赖（`dingtalk_stream`, `websockets`）
- 问你要 App Key/Secret
- 生成 `config.json`（chmod 600，安全存储）
- 可选：注册 macOS LaunchAgent，24/7 守护运行

### 启动

```bash
python3 ~/.claude/skills/dingtalk-bridge/src/stream_bot.py
```

去钉钉群里 @ 它一下，conv_id 自动保存。之后你在 Claude Code 里说「发钉钉消息：xxxx」它就会自动调用这个 skill。

## 📊 一些数据

| 指标 | 数值 |
|------|------|
| 代码行数 | ~500 行 Python |
| 测试覆盖 | 27 个回归测试，0.15 秒跑完 |
| 安装时间 | 从 `git clone` 到跑起来 < 60 秒 |
| 依赖 | 仅 `dingtalk_stream` + `websockets` |
| 内存占用 | ~30MB（空闲状态） |
| 每天能处理的消息 | 无上限（钉钉限速是每分钟 20 条） |

## 🎯 适合哪些场景

✅ **开发团队**：让 AI 代答技术问题，直接读代码库
✅ **运营团队**：让 AI 自动生成日报、周报、数据快照
✅ **产品团队**：让 AI 实时查竞品数据、市场数据
✅ **个人用户**：让 AI 成为你在钉钉上的分身

❌ **不适合**：
- 需要处理图片/视频的场景（目前只支持文本）
- 对延迟要求 < 1s 的场景（Claude CLI 启动本身需要几秒）

## 🌟 如何参与

项目完全开源（MIT License），欢迎 PR。

**GitHub:** https://github.com/mguozhen/dingtalk-bridge

**支持的功能请求：**
- 图片消息发送（via markdown 内嵌图床）
- 多群组管理（当前只支持单群）
- 企业微信 / 飞书版本（原理完全一样）

**已提交的分发平台：**
- ✅ Clawhub: `clawhub install dingtalk-bridge`
- 🕐 claude-skill-registry（PR 审核中）
- 🕐 daymade/claude-code-skills（PR 审核中）

## 💭 为什么开源这个

一年前我还在手动处理每一条钉钉消息。现在 AI 替我处理了 80%，剩下 20% 是真的需要我决策的。

这不是玩具。这是一个**实实在在给我省 2 小时 / 天的生产力工具**。

之所以开源，因为我觉得：
1. 技术不复杂，不值得闭源
2. 每个被钉钉淹没的打工人都应该有这个选择
3. 我希望看到大家基于这个做出更多神奇的东西（飞书版？企微版？Slack 版？）

如果你用上了，给个 ⭐️ 让我知道一下。如果你有问题，提 Issue 我基本都会回。

---

**GitHub 仓库：** https://github.com/mguozhen/dingtalk-bridge
**一键安装：** `clawhub install dingtalk-bridge`
**License：** MIT（完全免费商用）

*用 2 小时写的代码，换 2 小时 / 天的自由。这笔账怎么算都划算。*
