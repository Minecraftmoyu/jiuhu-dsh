# 本地补丁清单（PATCHES）

> 自动生成：`node scripts/patch-list.mjs --write`
> 升级上游后跑 `node scripts/patch-list.mjs`，对照标签是否还在；少了的说明被上游覆盖了。

共 **13** 组标记、**13** 处。

| 标记 | 处数 | 说明 | 位置 |
| --- | --- | --- | --- |
| `BUSY_NOTICE_PATCH` | 1 | 忙线提示：同一会话 30 分钟最多一条 | src/bridge.js:9646 |
| `DAY_CAP_PATCH` | 1 | 每日唤醒额度用完时：别让对方以为被无视，发一句「今天不聊了」，每个会话每天只发一次。 | src/bridge.js:9617 |
| `DEGRADED_PATCH` | 1 | ── 出错时的「我在忙」与恢复后的「我回来了」────────────────────────── | src/bridge.js:9599 |
| `FEATURES_PATCH` | 1 | ── 功能自检（纯代码，不调模型 → 0 token）──────────────────────────── | src/bridge.js:13236 |
| `FRIEND_REQ_PATCH` | 1 | ── 好友/加群申请：转给主人决定（方案 A）──────────────────────────── | src/bridge.js:13354 |
| `IMAGE_STILL_PATCH` | 1 | 大 GIF / 超大图直接塞给模型有两个问题：体积超限被拒、以及模型本来也只能看一帧。 | src/bridge.js:10165 |
| `MEMORY_FINGERPRINT_PATCH` | 1 | 9.5) 人格卡指纹：0.8.x 的层迁移曾在重启时改写 persona.md（我们丢过内容）， | src/bridge.js:13316 |
| `MEMORY_MAP_PATCH` | 1 | ── 记忆星图 ───────────────────────────────────────────────── | src/bridge.js:7891 |
| `MEMORY_SCOPE_PATCH` | 1 | ── 按会话检索记忆 ──────────────────────────────────────────────── | src/bridge.js:13699 |
| `OWNER_ONLY_ERROR_PATCH` | 1 | 内部故障只报主人 | src/bridge.js:9674 |
| `RECALL_PATCH` | 1 | ── 语音台词表：jy-xx 是本地模型生成的台词，她发语音时拿不到原文， | src/bridge.js:13529 |
| `RECOVERY_TTL_PATCH` | 1 | 恢复提示（2 小时 TTL，避免久后还提「刚才」） | src/bridge.js:9660 |
| `SELFIE_PATCH` | 1 | ── 自拍（按定妆立绘生成，不是随机脸）────────────────────────────── | src/bridge.js:13546 |
