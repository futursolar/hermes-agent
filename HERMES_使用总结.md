# Hermes Agent 使用总结

> 个人实战总结，非官方文档。基于 VPS + Telegram 的生产环境使用经验。

---

## 1. 环境与部署

### 我的架构

```
Telegram (iOS) ←→ VPS (GCP/Linux) ←→ Hermes Agent
                       ↓
                  按需调用: gh CLI / curl / Python 脚本 / MCP
```

- **部署方式**: 官方一键安装 (`curl ... | bash`)，纯 Linux 环境
- **模型**: 主对话用 DeepSeek V4，识图切 Grok (xAI OAuth)，委派子任务用 Flash 模型省 token
- **通信**: 主要通过 Telegram Bot，偶尔 SSH 上去用 CLI

### 安装要点

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc
hermes setup   # 配置模型、API Key、消息平台
```

> 如果有旧版 OpenClaw 配置，`hermes claw migrate` 可自动迁移。

---

## 2. 三层记忆体系

Hermes 有三层"记忆"，协同工作：

| 层级 | 对应功能 | 存什么 | 举例 |
|------|---------|--------|------|
| **持久记忆** | `memory` 工具 | 长期不变的配置、偏好、环境 | "用户偏好中文交流"、"SoSoValue API Key 路径" |
| **会话搜索** | `session_search` | 跨会话的历史对话 | "上周我们讨论的那个币叫什么？" |
| **技能系统** | `skills` | 可复用的工作流程 | Alpha 日报生成流程、SoSoValue 数据采集 |

### 记忆使用原则

- **memory 存高频指针**，不要存临时任务状态
- **session_search 查细节**，关键词检索即可
- **skill 存工作流**，做过 5 次以上的复杂任务就去固化

---

## 3. Skills 技能系统 — 最核心的功能

### 为什么 Skills 重要

当你反复做同一类任务（比如每日 Alpha 扫描、SoSoValue 日报、代码审查），把流程写成技能后：
- 不用每次重复"你先做 A，再做 B"
- 技能包含已验证的命令、API 端点、注意事项
- 技能自更新：使用时发现问题可以直接修补

### 我的技能集

```
crypto/
  alpha-discovery     — 每日链上 Alpha 发现与报告
  sosovalue           — SoSoValue 数据 API 调用
  sosovalue-daily     — SoSoValue 日报生成宏
  sodex               — SoDEX DEX 交易
  hertzflow           — HertzFlow 永续合约协议

okx-*                 — OKX Agentic Wallet 系列 (钱包/交易/桥接/DEX/安全等)

devops/
  rclone-cloud-backup — rclone 自动备份到云
  vps-proxy-setup     — Shadowsocks 代理搭建

github/               — GitHub 工作流 (PR/Issue/Code Review)

social/
  agent-reach         — 多平台社交媒体读取
```

### Skill 使用技巧

- 做复杂任务前先 `/skill-name` 加载技能
- 技能加载后会注入到当前对话上下文，提供专有指令
- 技能内容可以在对话中用 `skill_manage` 实时更新
- 任务成功后主动 offer 保存为 skill

---

## 4. Cron 定时任务

### 我的定时任务

| 任务 | 频率 | 说明 |
|------|------|------|
| Alpha 日报 | 每天 08:00 | 多链 Alpha 发现 + 叙事分析 + 聪明钱追踪 |
| SoSoValue 日报 | 每天 08:00 | 新闻、宏观、赛道行情、V4A-Flash 预警 |
| 叙事推特化 | 每天 08:00 | Agent-Reach 抓取 Twitter 行业叙事 |
| 云端备份 | 每天 04:00 | rclone 备份关键配置到云存储 |

### Cron 配置要点

- 用 `cronjob(action='create')` 自然语言描述即可
- `deliver` 参数控制输出投递到哪里（Telegram/本地文件/全平台）
- `skills` 参数可绑定技能，cron 运行时自动加载
- `context_from` 可实现任务链：A 收集数据 → B 分析 → C 分发

---

## 5. 消息平台集成

### Telegram 接入

```bash
hermes gateway setup   # 配置 Telegram Bot Token
hermes gateway start   # 启动网关
```

- 支持多平台同时在线（Telegram、Discord、Slack 等）
- `/sethome` 设置默认输出频道
- `/status` 查看连接状态

### 移动端使用心得

- iOS Telegram + Termius (SSH 到 VPS) 是最佳组合
- 日常对话全部走 Telegram，只在需要 VPS 操作时开 Termius
- 图片直接在 Telegram 发，识图自动触发

---

## 6. Token 优化

### RTK (Rust Token Killer)

Hermes 内置 token 优化工具，自动压缩常见命令输出：

```bash
rtk cargo build     # 编译输出压缩 80-90%
rtk cargo test      # 测试失败仅显示错误
rtk git diff        # diff 输出压缩 80%
rtk gh pr view      # PR 信息精简
rtk ls              # 树形紧凑显示
```

**原则**: 能用 `rtk` 前缀的都加上，尤其在 cron 任务中能省大量 token。

### 上下文压缩

- `/compress` 手动触发压缩
- 自动压缩阈值可在 `config.yaml` 调整
- 长对话后主动压缩，避免 token 浪费

---

## 7. 委托与并行

### delegate_task

复杂任务拆成子任务并行执行：

```python
delegate_task(
    goal="分析这个合约的安全性",
    context="合约地址: 0x...",
    toolsets=["terminal", "web"]
)
```

- 适合: 代码审查、安全分析、多链数据收集
- 不适合: 需要用户交互的任务（子 agent 不能用 `clarify`）
- 子 agent 无记忆，必须传递完整上下文

### execute_code

3+ 次工具调用且有中间处理逻辑时，用 `execute_code` 写成 Python 脚本：

```python
from hermes_tools import terminal, read_file, write_file
# 循环、条件、数据处理
```

- 所有工具调用压缩为一次 API 调用，零中间 token 消耗
- 尤其适合批量 API 请求 + 数据处理

---

## 8. 外部 API 集成

### 通过 Skill 封装

每个外部 API 都封装成一个 skill，包含：
- API 端点文档
- 认证方式
- 已验证的调用脚本
- 已知坑点

### 我的集成

| 服务 | 集成方式 | 用途 |
|------|---------|------|
| SoSoValue | Python 脚本 + skill | 日报数据源 |
| RootData | MCP Server | 项目/VC 信息查询 |
| SoDEX | EIP-712 签名脚本 | DEX 交易 |
| xAI (Grok) | OAuth | 识图 |
| Twitter | twitter-cli | 推文读取/叙事追踪 |
| GitHub | gh CLI | 代码管理 |

### MCP 集成

```yaml
# config.yaml
mcp_servers:
  rootdata:
    command: npx
    args: [...]
```

接入后 MCP 工具自动注册，跟内置工具一样调用。

---

## 9. 踩坑记录

### 坑 1: 记忆污染
❌ 在 memory 里存临时任务状态（"今天跑了 XX 日报"）  
✅ memory 只存持久化事实，临时状态靠 session_search

### 坑 2: Skill 不更新
❌ 发现 skill 有错但手动绕过去，下次继续踩坑  
✅ 发现问题立刻 `skill_manage(action='patch')` 修复

### 坑 3: Token 浪费在编译输出
❌ `cargo build` 直接跑，几千行输出全部进上下文  
✅ `rtk cargo build` 压缩 90%

### 坑 4: Cron 任务无 deliver
❌ 创建 cron 忘了设 deliver，结果存在本地没人看  
✅ 明确设置 deliver 目标（Telegram 或其他）

### 坑 5: 子 Agent 上下文不足
❌ `delegate_task(goal="查一下那个币")` — 子 agent 不知道是哪个币  
✅ 传递完整上下文：合约地址、链、相关文件路径

### 坑 6: Twitter 中文内容搜索盲区
`search_twitter()` 对中文推文索引不全，部分含完整 CA 的推文搜不到。  
✅ 对中文社区为主的项目，额外用 `twitter-cli` 手动搜索补充。

---

## 10. 推荐工作流

### Alpha 发现 → 验证 → 交易的完整链路

```
08:00 cron: Alpha 日报生成
  ↓
人工过一遍候选列表
  ↓
怀疑某个币 → session_search 查历史讨论
  ↓
需要更多数据 → OKX OnchainOS / SoSoValue / RootData
  ↓
确认交易 → OKX Agentic Wallet / SoDEX
  ↓
复盘 → 更新 skill（新的分析维度/数据源）
```

### 代码项目协作

```
收到需求
  ↓
/plan → 生成实施计划
  ↓
delegate_task → 分拆给子 agent 并行开发
  ↓
requesting-code-review → 自动安全扫描 + 质量检查
  ↓
gh pr create → 提交 PR
  ↓
任务完成 → skill_manage 保存工作流
```

---

## 11. 配置文件速查

```bash
hermes model           # 切换模型
hermes tools           # 管理工具开关
hermes config set ...  # 修改配置项
hermes config get ...  # 查看配置
hermes doctor          # 诊断问题
hermes update          # 升级
```

配置文件位置: `~/.hermes/config.yaml`  
环境变量: `~/.hermes/.env`  
技能目录: `~/.hermes/skills/`  
日志: `~/.hermes/logs/`

---

> 📌 持续更新中。Hermes 本身也在快速迭代，建议关注 [官方更新日志](https://github.com/NousResearch/hermes-agent/releases)。
