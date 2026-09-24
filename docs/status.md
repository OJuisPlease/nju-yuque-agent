# 项目现状（交接用）

> **更新时间**：2026-09-24 13:45（+08:00）
>
> 这份文档只写「项目本身现在是什么样」。**易变的数字**（commit、单元开关、
> 积压条数）请以机器上的实际输出为准，别照抄这里。
>
> 这台机器 / 账号 / 凭证的信息**不在仓库里**——`docs/deploy.md` 开头写着
> 「具体那台机器的信息（IP / 账号 / 凭证位置）由项目负责人单独交接，
> 不要写进公开仓库」。开发机本地另有一份 `.local/access.md`（已 gitignore），
> 里面有访问方式、常用命令与踩过的坑。
>
> 干活之前先读 [`AGENTS.md`](../AGENTS.md)（每条规矩都对应一次真实事故）。

## 1. 这是什么

`yuque-agent`：让 LLM 接管语雀知识库的常驻 agent。不是聊天机器人，而是一条流水线：
**感知**（轮询语雀 diff，没变化就不唤醒 LLM）→ **判断**（按提示词处理申请/投稿）
→ **留痕**（`runs/<run_id>/session.jsonl`）→ **交付**（QQ 通知 / 命令 + `plan.json` 下载口）。

## 2. 现在跑着什么

| systemd 单元 | 作用 | 说明 |
|---|---|---|
| `yuque-agent-qq.service` | 轮询 + 归档 + 通知泵 + QQ 网关 | **本机启用的那个**；不监听端口，自己外连 QQ 网关 |
| `yuque-agent-plan.service` | `yqa serve-plan`：申请清单下载口 | 听 `0.0.0.0:8787`，**公开、无鉴权、打开即下载**（理由见 `docs/deploy.md` §10 与 `AGENTS.md` §2.2） |
| `yuque-agent.service` | 裸轮询（不带 QQ） | **故意停用**：与前一个是 `Conflicts=` 二选一（都轮询同一工作区、都写 `state.json`） |

## 3. 代码线与交付流程

* **上游 = `Aalas1111/nju-yuque-agent`**（合作者）：生产机检出的 `origin` 就是它。
* `OJuisPlease/nju-yuque-agent` 是**负责人侧的镜像**：只同步，不在上面堆提交。
* 流程：从上游 `main` 开 feature 分支 → 推到镜像 → 开 PR → 上游合并
  （已合并过的 PR #3 就是这个流程：`AGENTS.md` + `scripts/deploy.sh`）。
* 生产机**只允许快进**：`scripts/deploy.sh` 会先 `fetch` 再 `merge --ff-only`，
  工作区不干净或非快进都会**硬失败**。
* 截至更新时间，两个远端的 `main` 是同一个 commit。

## 4. 目录（生产机）

```
/opt/yuque-agent/                    # 代码检出（root 拥有；.venv 是 py3.12）
  deploy/*.service                   # 三个单元的权威副本（与 /etc 对齐，有守卫测试）
  scripts/deploy.sh                  # 唯一的部署方式
/var/lib/yuque-agent/
  workspace/lqogh0_jsjysq/           # 工作区：语雀库 lqogh0/jsjysq
    qqbot.json                       #   通知映射 + 四份权限名单（600）
    state.json                       #   快照，每分钟被轮询更新
    runs/<run_id>/                   #   每次 run 的 payload / session.jsonl / result
    outbox/notify/{pending,done,unrouted,failed}/
  ops.log                            # 部署记录（deploy.sh 追加）
/home/yuque/.yuque/                  # 凭证目录（600，属主 yuque）
```

## 5. 权限模型：四份独立名单

`qqbot.json` 的 `inbound` 有四份名单，**各自独立生效**，命中其一即可（就高不就低）：

| 名单 | 收什么 id | 给什么 |
|---|---|---|
| `allow` | 个人 openid（群里是 `member_openid`） | 只读命令 `/help` `/status` `/pending` |
| `admins` | 个人 openid | 再加 `/run` `/archive` |
| `user_groups` | 群 `group_openid` | 整群是用户（群里谁发言都算） |
| `admin_groups` | 群 `group_openid` | 整群是管理员（**高风险**：`/archive` 会改知识库结构） |

判定与命令行为见 [`docs/qqbot.md`](qqbot.md) §4。`/help` 按身份给清单：
管理员（含管理员群里的人）看到全部，普通用户只看到自己能用的。
命令回复**只讲事实**，不带配置指南。

## 6. 运行状态与已知缺口

* **轮询是活的**：`state.json` 每分钟都被写一次。
* **没有变更就不产生 run**（0 token 是设计行为）。截至更新时间 `runs/` 有 9 条，
  最新一条是 2026-09-24 01:17。
* **通知投递不通（当前最该补的）**：`notify.members` 与 `default_target` 都空、
  `unmapped=skip` → agent 产出的通知只会堆进 `outbox/notify/unrouted/`
  （截至更新时间已积压 8 条），**没有人会收到提醒**。要么配 `default_target`
  （兜底发到某个群/某人），要么按人名配 `members`。
* 没有配 `user_groups`（普通用户群）；管理员群 1 个。
* root 密码在交接过程中明文出现过，**建议轮换**（合作者用公钥，不受影响）。

## 7. 凭证（三处，谁也不许手删）

| 位置 | 作用 |
|---|---|
| `~yuque/.yuque/qqbot.json` | 扫码登录落点（600） |
| 同目录 `qqbot.json.bak` | 每次写入同步的备份；主文件丢失/损坏时**自动恢复** |
| `~yuque/.yuque/agent.env` 的 `YQA_QQ_APPID/YQA_QQ_SECRET` | systemd `EnvironmentFile`，**优先级最高**——文件被删也不掉线 |

语雀 token 在 `~yuque/.yuque/auth.json`；LLM key / plan key 也在同一个 `agent.env`。
真丢了先别叫人扫码：按 `.bak` → `agent.env` 的顺序找；`docs/qqbot.md` 里有登录与恢复流程。

## 8. 相关文档

| 文档 | 给谁 |
|---|---|
| [`AGENTS.md`](../AGENTS.md) | **先读这个**：怎么改、怎么部署、什么绝对不许做 |
| [`docs/deploy.md`](deploy.md) | 运维：目录布局、三个单元、防火墙、上线验收清单 |
| [`docs/handoff.md`](handoff.md) | 对下游的冻结契约（申请 JSON / 通知事件） |
| [`docs/design.md`](design.md) | 维护者：架构、周期定义、工具分级、提示词写法 |
| [`docs/qqbot.md`](qqbot.md) | QQBot 接入方：扫码登录、通知投递、命令能力边界、排查 |
| `.local/access.md` | 本机文件（已 gitignore）：生产机怎么连、凭证在哪、常用命令 |
