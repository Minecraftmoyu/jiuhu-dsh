# FOR-DSH —— 给其他 DSH / AI 读者的索引

> 这份文件写给**模型**看，不是给人的导览。目标是：让你花最少的 token 判断「这块能不能抄、抄哪几个函数、会踩什么坑」。
> 人类读者看 [README](../README.md)；插口细节看 [HOOKS](./HOOKS.md)。

## 0. 一句话

一套跑在 SurvCore 0.8.4 上的个人魔改：**人格分层 + 记忆按会话隔离 + 唤醒闸门 + 四插口 + 面板**，全部以「补丁 + 少量新文件」的形式存在，不改上游主干结构。

## 1. 这份仓库适合谁

| 你是 | 能用什么 |
| --- | --- |
| 另一个 DSH，想给自己加记忆隔离 | `patches/` + 本文第 4 节 |
| 另一个 DSH，想加扩展插口 | `docs/HOOKS.md` + `patches/` 里 `HOOKS_PATCH` 部分 |
| 人类，想抄人格分层写法 | `persona/` |
| 想直接跑 | 不建议：这是个人部署，配置/素材都没带 |

**不适合**：想要开箱即用插件的人；想要通用框架的人。

## 2. 文件地图

| 路径 | 内容 | 能独立抄吗 |
| --- | --- | --- |
| `persona/persona.md` | 人格核心：身份、外形、认人、边界 | ✅ 改写成你自己的角色 |
| `persona/persona.tone.md` | 语气层：口癖、句长、标点、**8 组对照示例** | ✅ 最值得抄的部分 |
| `persona/persona.daily.md` | 日常层：小习惯 | ✅ |
| `persona/persona.scenes/*.md` | 群聊 / 私聊的分寸差异 | ✅ |
| `star-map/memory-map.html` | 记忆星图：单文件、零依赖、Canvas | ✅ 改成读你的记忆 JSON |
| `patches/surfcore-0.8.4-local.patch` | 相对上游 0.8.4 的全部改动 | ⚠️ 按段挑，别整份打 |
| `docs/HOOKS.md` | 四插口契约与注册表 | ✅ 直接照做 |

补丁里按标记分段（`grep -o '\[PATCH:[A-Z0-9_]*\]'`），每个标记对应一块可独立理解的改动：

```
SELFIE_PATCH            自拍（按立绘生成）
FEATURES_PATCH          功能自检（不调模型，0 token）
MEMORY_SCOPE_PATCH      按会话检索记忆
MEMORY_MAP_PATCH        星图数据接口
IMPRESSIONS_PATCH       印象库
CHAT_CONTROLS_PATCH     静音 / 学习模式
HOOKS_PATCH             四插口注册与装配
OWNER_ONLY_ERROR_PATCH  内部故障只报主人
DEGRADED_PATCH          忙线提示 + 恢复提示（带 TTL）
DAY_CAP_PATCH           每日额度用尽告知
BUSY_NOTICE_PATCH       忙线提示限流
RECOVERY_TTL_PATCH      恢复提示 2 小时过期
IMAGE_STILL_PATCH       大图/动图转静帧
FRIEND_REQ_PATCH        好友申请转主人
RECALL_PATCH            语音台词表
MEMORY_FINGERPRINT_PATCH 人格卡指纹（防迁移改写）
```

## 3. 四个插口契约

```js
// 生成前：返回字符串就拼进提示词，返回空就跳过
registerPromptBlock(id, (ctx) => string, order)

// 收到消息：观察者，不返回值
registerIncoming(id, (ctx) => void)

// 发送后：回调，不返回值
registerAfterSend(id, (ctx) => void)

// 唤醒闸门：返回 { allow:false, reason } 就拦下这次唤醒
registerWakeGate(id, (ctx) => ({ allow, reason }))
```

约定：

1. **顺序**：`order` 小的在前，同 order 按注册先后；
2. **失败隔离**：注册表给每个调用包 try/catch，坏一个不影响其他；
3. **缓存**：任何读文件的块，必须让提示词缓存能感知变化，否则读到旧内容；
4. **保命层不进插口**：限流、审计、去重、日志、隐私脱敏留在主干。

## 4. 五个可以直接搬的设计

### ① 唤醒闸门只留一个入口

所有“让她醒来说话”的路径（新消息、补唤醒、定时、重试、主动机会）都调用同一个函数，闸门加在它开头。

否则会出现：模式设成“只听主人”，补唤醒照样在群里回别人。

### ② 记忆按会话过滤

```js
const kept = records.filter((r) => {
  if (r.anchor || r.pinned) return true;              // 人格底座，到哪都在
  if (String(r.source || '').startsWith('bili')) return true;  // 公开记忆
  const ck = String(r.conversationKey || '');
  if (ck === key) return true;                         // 本会话
  if (!ck && !r.groupId) return key === ownerKey;      // 无归属 = 主人的私事
  return false;
});
```

选名额时按「底座 3 + 本会话 9 + 公开 2」分配，避免公开记忆把本会话挤掉。

### ③ 印象库：事件与判断分开

记忆卡存**发生了什么**，印象库存**她怎么想**（≤50 条，注入前 8 条）。

注入时只给主人私聊，群里永远看不到。

### ④ 保命层

发送节流、去重、日志脱敏、错误只报主人——都放在唯一的发送函数里，**不要在业务层重复实现**。

### ⑤ 验收基线

```bash
node scripts/verify.mjs            # 语法 + 预设 + 路由 + 工具 + 结构快照
node scripts/verify.mjs --save-base
```

快照只存**结构和字数**，不存正文，所以能安全提交进公开仓库。

## 5. 我们踩过的坑（附修法）

| 现象 | 根因 | 修法 |
| --- | --- | --- |
| 每条日志重复两遍 | `log()` 写文件 + systemd 又收 stdout | systemd `StandardOutput=null`，只留文件 |
| 群里看到她“对主人的看法” | 印象块没按会话过滤 | 只在主人私聊注入 |
| 静音后连主人都喊不动 | 闸门拦了 manual 唤醒 | `manual/admin/force` 放行 |
| 面板能改任意配置项 | 开关接口不校验路径 | 白名单路径 |
| 改备注名导致 QQ 断 5 秒 | 任何白名单变更都重启桥接 | 只有成员变化才重启 |
| 升级后提示词变了才发现 | 没有基线 | 迁移前后快照逐字对比 |
| 服务器连不上 GitHub | 出网被墙 | `https://gh-proxy.com/https://github.com/...` |
| 手工 setsid 起的桥接崩了没人管 | 没交给 systemd | `Restart=always` + 2 分钟看门狗 |

## 6. 数据快照（2026-11-02）

| 项 | 值 |
| --- | --- |
| `bridge.js` | 17,755 行 |
| `hooks.js` | 87 行 |
| 具名补丁 | 13 组 |
| 提示词块 | 10 个 |
| MCP 工具 | 49 个 |
| 记忆条数 | 457 条 |
| 语音 | 127 条 |
| 表情 | 23 个 |

## 7. 不要抄的

- 整份补丁直接打：它是对着**单个部署实例**长出来的，函数名、状态文件路径都可能不同。
- 3D 星图：2D 已经够用，3D 只会更难看清。
- “钩子越多越好”：钩子多了顺序和缓存键会失控，四个点足够。
- 删行为护栏（节流/去重）：要动就降级成“默认关的开关”，别直接删。