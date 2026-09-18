# CatAgent

全局 agent 人设与协作规范（AGENTS.md），供 Kimi Code / Claude Code 等支持 AGENTS.md 注入的 agent 工具使用。

## 来源声明

本配置 **fork 自 [onevcat 的 gist](https://gist.github.com/onevcat/7c5838349c7264d6019bebe39df30405)**（原版为 Claude Code 的 AGENTS.md），本地修改后使用，**不跟踪 gist 后续更新**——本目录下的 `AGENTS.md` 即唯一维护源。

## 相对原版的修改

| 修改点 | 原版 | 本版 |
|---|---|---|
| 服务对象 | onevcat 及其个人域名 | EdwinQQQ（git: edwinQQQ <exzero@126.com>），GitHub: [EggmanQi](https://github.com/EggmanQi) |
| `/grill-me` 命令 | 直接引用 Claude Code 特有命令 | 改为行为等价描述（质疑者角色：反向假设 + 边界条件追问），注明 Kimi Code 无此命令 |
| 其余内容（猫娘人设、核心原则、代码风格、Git 安全等） | — | 原样保留 |

## 安装（全局生效）

```sh
# Kimi Code / Claude Code 等通用位置（跨工具共享）
cp CatAgent/AGENTS.md ~/.agents/AGENTS.md

# 如需仅 Kimi Code 生效，改用：
# cp CatAgent/AGENTS.md ~/.kimi-code/AGENTS.md
```

新会话启动时自动加载；修改后无需重启已开启的会话，新会话即可生效。
