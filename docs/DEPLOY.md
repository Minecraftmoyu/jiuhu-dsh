# 部署指南（DEPLOY）

> 从零到「能在 QQ 上聊天」的 10 步。
> 目标环境：**Linux**（Ubuntu / Debian 实测），Node 22，SurfCore **v0.8.4**，SnowLuma（OneBot v11）。
> Windows 未验证。

---

## 0. 你要准备什么

| 东西 | 说明 |
| --- | --- |
| 一台能跑 Node + 常驻服务的机器 | 2 核 2G 够用 |
| 一个 **QQ 小号** | 给机器人用，别用大号 |
| SnowLuma 或 NapCat | 把 QQ 接成 OneBot v11（HTTP 3000 / WS 3001） |
| DSH | 本项目的 agent 运行时，需能本地 HTTP 访问 |
| 模型 API | DeepSeek 或兼容网关 |

---

## 1. 拿代码

```bash
git clone https://github.com/Mieksido/surfcore.git /opt/surfcore
cd /opt/surfcore
git checkout v0.8.4
```

## 2. 打补丁

```bash
git apply --check /path/to/surfcore-0.8.4-local.patch   # 先干跑
git apply /path/to/surfcore-0.8.4-local.patch
```

补丁会新增 3 个文件（上游没有）：

```
src/hooks.js        四类插口 + 登记表
src/peak-hours.js   峰谷计费
src/self-model.js   自我模型
```

## 3. 放人格卡

```bash
mkdir -p characters/char-jiuhu1
cp -r /path/to/repo/persona/* characters/char-jiuhu1/
```

人格四层：`persona.md`（核心）/ `persona.tone.md`（语气）/ `persona.daily.md`（日常）/ `persona.scenes/`（群聊、私聊）。

## 4. 写配置

```bash
cp /path/to/repo/config.example.json /opt/surfcore/config.json
```

**必须改这几项**：

| 字段 | 说明 |
| --- | --- |
| `ownerQQ` | 你自己的 QQ（唯一的主人，只有这个号能叫「主人」） |
| `admin.qq` | 管理员 QQ（一般同 ownerQQ） |
| `allow.private` | 允许私聊的 QQ 列表 |
| `allow.groups` | 允许的群号列表 |
| `consoleToken` | 控制台令牌，**换成随机长字符串** |
| `dsh.baseUrl` | DSH 地址，默认 `http://127.0.0.1:18080` |
| `snowluma.wsUrl` / `httpUrl` | OneBot 地址，默认 `ws://127.0.0.1:3001`、`http://127.0.0.1:3000` |

> `config.example.json` 里所有 QQ 号都是 `1000000001` 之类的占位，别直接拿去跑。

## 5. 装依赖

```bash
npm install --omit=dev
```

## 6. 启动

```bash
node src/bridge.js
```

看到这些日志就对了：

```
桥接模式已确定为 reserved2
DSH 已就绪（模式: reserved2）
[B站记忆] 已刷新：N 条
```

## 7. 自检

```bash
TOKEN=$(python3 -c "import json;print(json.load(open('config.json'))['consoleToken'])")
curl -s "http://127.0.0.1:3100/api/features?token=$TOKEN" | python3 -m json.tool | head -40
```

12 项检查全绿才算通。

## 8. 验收基线（改了代码之后跑）

```bash
node scripts/verify.mjs            # 语法 / 预设 / 路由 / 工具 / 结构快照
node scripts/verify.mjs --save-base
```

## 9. 交给 systemd（可选但强烈建议）

```ini
[Unit]
Description=SurfCore bridge
After=network.target

[Service]
WorkingDirectory=/opt/surfcore
ExecStart=/usr/bin/node /opt/surfcore/src/bridge.js
Restart=always
RestartSec=5
StandardOutput=null
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now surfcore-bridge
```

## 10. 面板（可选）

`panel/server.mjs` 是零依赖单文件服务，只监听 `127.0.0.1:3200`：

```bash
node panel/server.mjs
```

浏览器访问 `http://127.0.0.1:3200/`（外部访问请走 SSH 隧道，别开公网）。

---

## 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 启动即退出 | `config.json` 缺 `consoleToken` 或格式错 | 用 `node -e "JSON.parse(...)"` 查一下 |
| `EADDRINUSE 3100` | 已经有一个 bridge 在跑 | `pkill -f bridge.js` 后重启 |
| 日志每条重复两遍 | systemd 收 stdout + 自己写文件 | 单元里加 `StandardOutput=null` |
| 收不到消息 | OneBot 地址/令牌不对，或号没在线 | 先 `curl http://127.0.0.1:3000/get_login_info` |
| 她回复很慢 | 模型延迟或额度用尽 | 看 `/api/features` 的「本月花费记账」 |
| 群里她不说话 | 群不在白名单 / 被静音 / 额度用完 | 面板「白名单」「静音与学习」页查 |

## 安全提醒

- `config.json` 里有 `consoleToken`、OneBot 令牌，**永远不要提交到仓库**
- 面板和控制台都只监听 `127.0.0.1`，不要直接暴露公网
- 机器人的 QQ 用小号，别用你自己的主号