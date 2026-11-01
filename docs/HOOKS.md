# 四个固定插口（HOOKS）

> 来源：群里给 Moming 的建议（加钩子、不加架构、收到消息/生成前/生成后/发送前 四个点、保命层不动）
> 我们落地时只做一件事：**插口固定 4 个，功能只注册不修改主干**。

## 四个插口

| 插口 | 时机 | 返回值 | 典型用途 |
| --- | --- | --- | --- |
| `incoming` | 收到一条消息 | 无 | 记最后说话人、学说法观察 |
| `beforeGenerate` | 组装提示词时 | 字符串（空则跳过） | 记忆、印象、欠事、笔记、心情 |
| `afterGenerate` | 模型输出后 | 改写后的文本 | 截断、去重、安全检查 |
| `afterSend` | 消息发出后 | 无 | 欠事捕获、记账、恢复标记 |

另有 `wakeGate`（唤醒闸门，返回 `{allow:false, reason}` 即拦截）作为决策层的横切点——它不算业务块，但同样走注册表。

## 现有注册

`src/bridge.js` 的 `ensureHooksRegistered()` 里注册了 10 个提示词块 + 1 个闸门：

```
order 0    persona          人格卡（核心/语气/日常/场景 由上层拼好）
order 10   admin            管理员规则
order 20   bili-brief       B 站记忆简报
order 30   memory-card      记忆卡（全局摘要）
order 40   conversation-memory  按会话检索的旧事
order 50   impressions      印象库（只给主人私聊）
order 60   open-loops       欠事
order 70   self-notes       自我笔记
order 80   mood             心情
order 90   recovery         恢复提示（2 小时 TTL）
```

闸门：`chat-control`（静音 / 学习模式）。

查看当前注册：`GET /api/hooks?token=<consoleToken>`

## 新增一个块（不碰 bridge.js）

```js
// 在 ensureHooksRegistered() 里加一行即可
registerPromptBlock('weather', () => weatherBlock(), 55);
```

顺序由 `order` 决定，同 order 按注册先后。**任何读取文件的块，必须让提示词缓存能感知变化**（bridge 里对应状态指纹），否则会读到旧内容。

## 失败隔离

`src/hooks.js` 里每个调用都包 try/catch：单个块抛异常只记 `[hooks:*] 失败`，不影响其它块和主流程。

## 不搬的东西（保命层）

限流、审计、去重、日志、隐私脱敏、发送节流——这些**不放进插口**，仍在 bridge 主干：

- 它们必须对**每一次**发送生效，走插口反而可能被漏注册；
- 它们出错要能立刻发现，藏在模块里会变难查。
