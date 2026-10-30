# Tachi (agent 项目与博文系列)

## 2026-10-30: Tachi 是什么、博文系列进度

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: tachi, agent, monsterxx03/tachi, 系列文章, 自研 agent, go

Tachi 是用户自己从零写的 Go agent（github.com/monsterxx03/tachi，2026 年 4 月开工），博客已成系列：`2026-05-23-tachi.md`（从 0 开始做 agent：TUI、channel 模式等）→ `2026-10-30-tachi-agent-networking.md`（组网：mDNS/Gossip/A2A）。写新篇时可沿用"上一篇讲 X，这篇讲 Y"的开头。

---

## 2026-10-30: Tachi 组网设计要点（写作时可复用的事实）

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: 组网, mdns, gossip, a2a, ed25519, node id, 身份, TOFU, pin, 端口 9990

组网三件事：协议（A2A 交接任务）、发现（mDNS + gossip）、身份（ed25519）。节点 id = sha256(pubkey)[:16] 的 32 位 hex，由公钥推出而非随机；远端面走 mTLS 且要求客户端证书（自签），鉴权"只识别不授权"，首次见新钥匙记 TOFU（`peer-identities.json`，UNVERIFIED→pin）。默认监听 `0.0.0.0:9990`。

---

## 2026-10-30: Tachi 发现层参数与根规则

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: 候选队列, 发现, 根规则, contact_ttl, ttl, nudge, 64 台, mDNS 服务名, _tachi._tcp

根规则：「发现永远不决定信任」——配置 peers / mDNS 公告 / gossip 转述三种来源只产生"去拨一次"的候选，验证（双向 TLS + 读 Agent Card）后才入目录。mDNS 服务名 `_tachi._tcp.local`（libp2p/zeroconf/v2），公告只带 id/v/scheme。gossip 是 `POST /node/v1/gossip` 挂在已有 A2A mux 上，push-pull 交换 {self.addrs, peers[id,addr,age]}；contact_ttl 10 分钟、目录 ttl 1 小时、稳态 3 分钟×扇出 2、变化后 nudge 10 秒；上限 64 候选 / 5 次尝试 / 8 地址 / 64 KiB。

---

## 2026-10-30: 用户的三台机器与使用场景

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: 机器, tailscale, nas, fnos, ugreen, vps, hostkvm, macbook, mpb-m1, 100.x

常开三台：MacBook（节点名 mpb-m1）、家里绿联 NAS 刷 fnOS（ugreen-nas）、香港小 VPS（hostkvm），三台无共同局域网，靠 Tailscale `100.x` 地址互联（gossip 形状检查只认 CGNAT 段）。典型用法：把"只有某台机器能做"的活（墙外查余额/抓周报/收 Discord、NAS 上找文件看磁盘）委派过去，只在桌面说一句。

---

## 2026-10-30: 模型侧只有两个组网工具，且默认禁链式转委托

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: DelegateTask, RemoteTask, 工具, 链式委派, 转委托, 权限, 远端一轮

模型只多两个工具：`DelegateTask`（交活）与 `RemoteTask`（查/等/答/取消），节点是参数而非一节点一工具；委派工具是运行级附加件、不进注册表，故 cron 与后台运行天然够不到。默认不允许链式转委托（A→B→C），远端一轮不生成会话标题、权限问询一律拒绝。

---
