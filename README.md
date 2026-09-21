# 洛克王国：世界重要消息助手 MVP

这是给 Fly 和少量朋友使用的轻量提醒器。它查询「洛克魔法书」数据源，只在关注商品出现或重要活动列表发生变化时，使用每个人自己的 Server酱 SendKey 单独推送；平时没有重要变化就保持安静。

当前完成的是可安全运行的 MVP 骨架：真实接口路径和鉴权方式已核实，代码、去重、测试、GitHub Actions 与 Secrets 配置已就绪。由于本地没有 API Key，**线上响应字段尚未实测**；项目不会把文档里的占位 JSON 当成真实结构。首次接入真实 Key 时先执行“结构探测”，只保存键和类型，再确认适配器。

## 已核实的外部事实

- 远行商人：`GET https://wegame.shallow.ink/api/v1/games/rocom/merchant/info`。
- 活动信息：`GET https://wegame.shallow.ink/api/v1/games/rocom/activities/info`，文档说明数据来自小程序 `getInitInfo` 的 `otherActivities`。
- 两个接口都属于基础认证接口，文档列出 `X-API-Key`。API Key 需在「洛克魔法书」申请；第三方项目公告称该上游已开始收费，因此本项目不会代购或假设免费。
- Server酱 Turbo 的 SendKey 以 `SCT` 开头，接口为 `POST https://sctapi.ftqq.com/{SendKey}.send`；免费会员每天最多 5 条，免费版不支持一个 SendKey 群发多人。这个 MVP 因此让每个人单独注册，并使用各自 SendKey。
- Server酱³ 的 SendKey 以 `sctp` 开头，代码也兼容它；它主要推送到独立 App，不等同于 Turbo 微信通道。
- GitHub Actions 定时任务最短间隔为 5 分钟，但高峰期可能延迟甚至丢弃排队任务；计划任务只在默认分支运行。当前排在北京时间每小时 `13`、`43` 分，避开整点并控制免费额度。
- 公共仓库连续 60 天无活动时，GitHub 会自动停用计划任务；需要在仓库中重新启用或产生有效活动。
- 公共仓库的标准 GitHub-hosted runner 免费；私有仓库的 GitHub Free 当前含每月 2,000 分钟。每 30 分钟一次约 1,440 次/月，短任务通常按至少 1 分钟计，目标是留在免费额度内，但仍应在 Billing 中设置预算上限。

来源：

- [洛克魔法书 API 文档：远行商人](https://rocom.apifox.cn/466047211e0)
- [洛克魔法书 API 文档目录](https://rocom.apifox.cn/llms.txt)
- [Server酱 SendKey 与免费额度](https://sct.ftqq.com/docs/getting-started/sendkey/)
- [Server酱通道、群发和价格规则](https://sct.ftqq.com/docs/getting-started/channels/)
- [GitHub Actions schedule 限制](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)
- [GitHub Actions 计费与免费额度](https://docs.github.com/en/actions/concepts/billing-and-usage)

## 本地安全试跑（无需任何密钥）

Windows PowerShell：

```powershell
$env:PYTHONPATH = "src"
python -m unittest discover -s tests -v
python -m rock_world_assistant --mock --dry-run
```

`tests/fixtures/*.json` 都带有 `SYNTHETIC TEST DATA` 标记，是合成测试数据，不是实际 API 响应。

## 配置关注内容

编辑 `config/settings.json`：

- `watch_items`：普通字符串按“包含”匹配；以 `re:` 开头时按正则匹配。
- `include_keywords`：只有标题、简介或状态包含关键词的活动才算重要。
- `exclude_keywords`：排除测试服等不想看的活动。
- `notify_on_any_change`：设为 `true` 会监控所有活动变化，容易增加消息量。
- `notify_on_first_run`：默认 `false`，首次只建立活动基线，不推送旧活动。
- `notify_errors`：是否在数据源连续失败三次后给该收件人发故障提醒。

添加朋友时复制一段 recipient，并给每个人设置唯一的英文小写 `id`。配置文件不放任何密钥。

## 首次真实接口核对

你需要自己在「洛克魔法书」获取 API Key；如果页面要求付费，由你决定是否购买。本项目不会代为授权或购买。

本地临时设置环境变量后执行：

```powershell
$env:ROCOM_API_KEY = "在你自己的终端临时填写"
$env:PYTHONPATH = "src"
python -m rock_world_assistant --dry-run --probe-shape runtime/probe-shape.json
```

`runtime/probe-shape.json` 只保存字段名、类型和数组长度，不保存字段值，也被 `.gitignore` 排除。检查日志中的商品/活动数量与结构后，再去掉 `--dry-run`。不要把 Key 粘贴到聊天、代码、配置文件或日志里。

## GitHub Secrets

远程仓库建立后，在 `Settings → Secrets and variables → Actions` 添加：

1. `ROCOM_API_KEY`：洛克魔法书 API Key。
2. `SERVERCHAN_SENDKEYS_JSON`：每人独立 SendKey 的 JSON 映射，例如：

```json
{"fly":"SCT_xxx","friend_a":"SCT_yyy"}
```

这只是格式示例，仓库中不要出现真实值。收件人 id 必须与 `config/settings.json` 一致。日志不会打印 SendKey。

先从 Actions 页面手动运行，并保持 `dry_run=true`；核对无误后再以 `dry_run=false` 手动跑一次。定时触发没有 inputs，会按正式模式运行。

## 去重、状态和失败行为

- 每位收件人单独记录已成功通知的事件；一人的推送失败不会阻塞其他人。
- 关注商品按商品身份、时间范围和北京时间轮次去重；活动按重要字段快照去重。
- 状态原子写入 `runtime/state.json`，Actions 使用固定 concurrency group，避免两个运行同时改状态。
- 工作流把状态提交回默认分支；状态只含哈希、时间和错误计数，不含密钥。
- 没有状态变化时不会产生提交，避免每次轮询刷提交历史。
- API 或响应结构异常时采取“失败关闭”：不会把未知结构误当成空列表，也不会误更新去重基线。
- 推送失败不会标记为已送达，下次会重试。Server酱没有幂等键，因此若“推送已成功，但 Actions 在提交状态前崩溃”，下一次仍可能重复一条；仅靠 Actions + Server酱无法彻底消除这个极小窗口。
- 北京时间由 Python 的 `Asia/Shanghai` 时区处理；GitHub 调度仍可能延迟，所以通知时间不是硬实时保证。

## 尚未替你执行

- 未申请、购买或验证洛克魔法书 API Key。
- 未注册或验证任何人的 Server酱 SendKey。
- 未进行真实推送。
