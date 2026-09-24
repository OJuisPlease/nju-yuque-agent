# 项目现状（交接用）

> **更新时间**：2026-09-23 22:15（+08:00）
>
> 这份文档只写「项目本身现在是什么样」。**易变的东西**（commit、单元状态、积压数量）
> 请以机器上的实际输出为准，别照抄这里的数字。
>
> 这台机器/账号/凭证的信息**不在仓库里**——`docs/deploy.md` 开头写着「具体那台机器的
> 信息（IP / 账号 / 凭证位置）由项目负责人单独交接，不要写进公开仓库」。
> 开发机本地另有一份 `.local/access.md`（已 gitignore），里面有访问方式。

## 1. 这是什么

`yuque-agent`：让 LLM 接管语雀知识库的常驻 agent。它不是聊天机器人，而是
「感知（轮询语雀 diff）→ 判断（LLM 按提示词处理申请/投稿）→ 留痕（`runs/*/session.jsonl`）
→ 交付（QQ 通知 + `plan.json` 下载口）」这一条流水线。

生产机上跑**两个**交付通道：

| 单元 | 作用 |
|---|---|
| `yuque-agent-qq.service` | 轮询 + 归档 + 通知泵 + QQ 网关（收 `/help` `/status` `/run` `/archive` 等命令、把结论与运行中播报发回去） |
| `yuque-agent-plan.service` | `yqa serve-plan`：带密钥的 `plan.json` / 申请清单下载口，给下游 cac 手动取件 |
| `yuque-agent.service` | 裸轮询（不带 QQ）—— 与第一个**二选一**，`Conflicts=` 在 systemd 层面互斥。本机启用的是 `-qq` |

## 2. 代码与仓库

* **上游 = `Aalas1111/nju-yuque-agent`**（合作者）。生产机检出 `/opt/yuque-agent` 的
  `origin` 就是它，单元里的 `Documentation=` 也指向它。
* `OJuisPlease/nju-yuque-agent` 是**镜像**：只同步，不在它上面堆自己的提交。
* 截至上面那个时间点，**两个远端的 `main` 是同一个 commit**（一条线，不再分叉）。
* **待合并**：分支 `agent-ops-contract`（在镜像仓）—— `AGENTS.md` + `scripts/deploy.sh`。
  合并后生产机上才有 `scripts/deploy.sh`。
* 服务器上**只允许快进**，禁止 rebase / 手工 merge。

## 3. 目录与文件

```
/opt/yuque-agent/                          # 代码检出（root 拥有；.venv 里是 py3.12）
  deploy/*.service                         # 三个 systemd 单元的权威副本
  docs/deploy.md                           # 部署真相（目录、单元、验收清单）
  docs/handoff.md                          # 对下游的冻结契约（申请 JSON / 通知事件）
  AGENTS.md                                # 干活规矩（先读它）
/var/lib/yuque-agent/workspace/<repo>/     # 工作区，一个语雀库一个目录
  qqbot.json                               # 通知映射 + 四份权限名单（600）
  state.json                               # 快照，每轮轮询都写
  runs/<run_id>/                           # 每次 run 的 payload / session / result
  outbox/notify/{pending,done,unrouted,failed}/
/var/lib/yuque-agent/ops.log               # 部署记录（deploy.sh 追加）
/home/yuque/.yuque/                        # 凭证目录（600，属主 yuque）
```

## 4. 权限模型（四份独立名单）

`qqbot.json` 的 `inbound` 有**四份名单，各自独立生效**，命中其一即可（就高不就低）：

| 名单 | 收什么 id | 给什么 |
|---|---|---|
| `allow` | 个人 openid（群里是 `member_openid`） | 只读命令 |
| `admins` | 个人 openid | 再加 `/run` `/archive` |
| `user_groups` | 群 `group_openid` | 整群是用户（谁发言都算） |
| `admin_groups` | 群 `group_openid` | 整群是管理员（**高风险**，见 `docs/qqbot.md` §4.1） |

生产机上目前：个人 1 人 / 个人管理员 1 人，用户群 0 个 / 管理员群 1 个
（管理员群意味着群里任何人都能 `/run` `/archive`，`yqa qq doctor` 会一直提醒这条）。

## 5. 运行状态（会变，以机器为准）

* 轮询活着：`state.json` 每分钟都被写一次。
* **没有变更就不会产生 run**：`runs/` 里最后一次是 9-21 上午，之后语雀侧没有 diff，
  所以没花 token。这属于设计行为，不是故障。
* 通知积压：`pending / done / unrouted / failed` 全为 0。
* 已知缺口（不是 bug，是没配）：
  1. **通知投不出去**：`notify.members` 与 `default_target` 都空、`unmapped=skip`，
     来了申请只会进 `outbox/notify/unrouted/`，没人收到提醒。
  2. 没有配 `user_groups`（普通用户群）。
  3. root 密码在交接过程中明文出现过，**建议轮换**。

## 6. 凭证（三处，谁也不许手删）

| 位置 | 作用 |
|---|---|
| `~yuque/.yuque/qqbot.json` | 扫码登录落点（600） |
| 同目录 `qqbot.json.bak` | 每次写入同步的备份，主文件丢失/损坏时自动恢复 |
| `~yuque/.yuque/agent.env` 的 `YQA_QQ_APPID/YQA_QQ_SECRET` | systemd `EnvironmentFile`，优先级最高——文件被删也不掉线 |

语雀 token 在 `~yuque/.yuque/auth.json`；LLM / plan 密钥在同一个 `agent.env` 里。
**真丢了先别叫人扫码**：先看 `.bak` → `agent.env`；再不行看 `docs/qqbot.md` 的恢复办法。

## 7. 怎么干活

先读 [`AGENTS.md`](../AGENTS.md)（那是给人也给代理的规矩，每条都对应一次真实事故）：
一条上游 + feature 分支交付、不许在生产机跑测试、常驻进程必须来自 `deploy/` 的单元、
一次只允许一个写者。更新生产机只有一条路：`scripts/deploy.sh`（带锁、只快进、
沙箱 HOME 跑测试、装单元、重启、写 `ops.log`）。
