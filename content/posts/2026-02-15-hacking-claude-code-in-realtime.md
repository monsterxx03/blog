---
title: "Hacking Claude Code in Realtime"
date: 2026-02-15T19:25:07+08:00
categories:
  - tech
tags:
  - claude code
  - ai
  - linko
---

做了个新玩具 [linko](https://github.com/monsterxx03/linko), 本质上是个透明 mitm proxy, 但给他加上了可视化 ai agent 和 llm api 之间对话的能力.

限制:

- 目前只支持 MacOS (透明代理的实现用了MacOS 上的pfctl, linux 的支持很简单, 有空可以加下)
- 支持的 llm api 只有 anthropic 的接口格式, 任意兼容的第三方接口应该都可以(测试过minimax 和 deepseek)

原理上和 charles 之类的工具的主要区别在"透明", 不需要客户端指定`http_proxy`(有些工具不支持), 通过防火墙强行劫持https流量, 来做流量解密.

## 安装

```bash
brew tap monsterxx03/tap
brew install linko
```

或者直接去 [ releases ](https://github.com/monsterxx03/linko/releases) 下 binary.

## 怎么玩

### 生成 CA 证书

```bash
linko gen-ca
```

会在 `~/.config/linko/certs/` 下生成 `ca.crt` 和 `ca.key`.

### 信任证书

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/.config/linko/certs/ca.crt
```

### 启动 MITM proxy

```bash
sudo linko mitm
```

proxy 默认在 9890 端口. 需要 sudo 是因为要用 pfctl 把系统流量重定向过来.

默认会对系统中所有发往 443 端口的https 流量做MITM, 可以用 `--whitelist` 参数限定只对特定host 做MITM.

e.g.:

    sudo linko mitm --whitelist api.minimaxi.com,*.deepseek.com

验证:

```bash
curl -v https://api.minimaxi.com
```

如果看到证书是 "Linko MITM CA" 签的, 说明已经正常工作了.

### 打开管理界面

浏览器访问 `http://localhost:9810`.

MITM tab 是通用的 MITM 流量解析监控页, 这边和llm api没什么关系, 任意https 流量都可以从这边看 request, response.

![mitm](/posts/images/mitm.png)

## 配合 Claude Code 用

Claude Code 用的库默认不识别自签名证书, 要手动关掉:

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 claude
```

然后你就可以在管理界面(LLM tab)看到 Claude Code 发的请求和收的响应了.

默认的系统提示词:

![cc-system](/posts/images/cc-system.png)

默认发送的tools:

![cc-tools](/posts/images/cc-tools.png)

tools 定义:

![cc-tool-def](/posts/images/cc-tool-def.png)

触发工具调用时:

![cc-tool-use](/posts/images/cc-tool-use.png)

细看 ClaudeCode 的交互还挺有趣的, 比如怎么告诉llm 目前有哪些skill, plan mode 执行后发生了什么, 触发subagent 执行后 system prompt 的变化.

有空可以写一下, 一边用Claudecode 写代码, 一边把它和llm 之前的交互流程重现出来的过程很有趣, terminal 里那个转圈动画后隐藏了不少.

## 其他agent 工具

理论上本机直连 anthropic 格式接口的工具都能使用.

测试了opencode, openclaw, 都能正常解析结果, openclaw 也需要用`NODE_TLS_REJECT_UNAUTHORIZED` 禁用证书检查, 具体看README.

没有买 anthropic 官方订阅, 实际只测试了 Minimax CN(api.minimaxi.com) 和 Deepseek (api.deepseek.com).

如果只是关注 agent 和 LLm 之间的交互行为, 用哪家的接口其实都一样.

PS: cursor 不支持, 即使用自定义model 和api key, llm 调用也不是直连的, 会从 cursor server 端走一圈,
cursor 和 server 的交互走的connect rpc + protobuf, 如果能逆向出pb文件, 理论上也是能解析的.

## 关于开发

ClaudeCode + MiniMax2.1 写了大概 90% 的代码, 2.5 刚出, 还没试过复杂任务, 目前没感觉出和2.1有什么区别.
剩下的代码有的是 TraeCN, 有的用ClaudeCode + deepseek3.2, deepseek 的思维链实在是太长了, 过于啰嗦, 效果不如MiniMax2.1, api 还用不了新模型, 不知道怎么样.

开发过程中的几个难点(目前还都是人工解决的, debug 靠人, 代码 ai 来):

- 用pfctl 做透明流量重定向时如何避免死循环(MacOS的pf版本太低, 很多新语法不支持, LLM 口胡浪费了我好多 token...)
- MiniMax 的 SSE 响应做了br 压缩, 如何在 mitm 中处理流式压缩的SSE response body.

前端是 ts + react, ts 语法是一行没学过, 生成的前端代码一行没看, 过程中主要出问题的地方是浅拷贝导致的前端组件没触发重新渲染, 来回了很多次, 但还是坚持不去学ts和react让AI搞定了.
