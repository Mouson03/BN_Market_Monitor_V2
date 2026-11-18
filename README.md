# binance-price-monitor

币安标的价格监控：价格满足条件时通过 PushPlus 推送微信提醒，由 **外部定时器 + GitHub Actions** 驱动。

## 触发架构

GitHub 自带的 `schedule` 实测极不可靠（2026-11-18 应触发 15 次、实际只触发 2 次，约 11 小时无监控），因此**主触发已改为外部定时器**：

| 触发时刻（北京时间） | 单次运行 | 覆盖区间 |
|---|---|---|
| 07:00 | 350 分钟 | 07:00 – 12:50 |
| 11:00 | 350 分钟 | 11:00 – 16:50 |
| 15:00 | 350 分钟 | 15:00 – 20:50 |
| 19:00 | 350 分钟 | 19:00 – 00:50 |

- 外部服务 [cron-job.org](https://cron-job.org) 用 `POST /repos/{owner}/{repo}/actions/workflows/monitor.yml/dispatches` 触发（cron `0 7,11,15,19 * * *`，时区 `Asia/Shanghai`）
- 每次触发后，脚本在容器内连续循环 **350 分钟、每 60 秒查一次价**（GitHub 单 job 硬上限 6 小时，故留 10 分钟余量）
- 相邻场次**重叠约 2 小时**，因此**会同时看到两场运行在跑，属正常**；漏掉一次触发也不会断档
- 单次运行 350 分钟 × 4 次/天，覆盖 07:00 – 次日 00:50，不会跑进凌晨

GitHub 自带 `schedule` 已在 workflow 里注释掉，需要兜底时取消注释即可。

## 工作原理

- **数据源**：币安公开行情 API（优先 `data-api.binance.vision`，无地区限制，6 个域名轮询重试）
- **推送**：命中条件后按频率规则去重，通过 PushPlus 推送到微信
- **状态**：`state.json` 记录每条规则的提醒状态，由 workflow 自动提交回仓库（`commit_state` 带 rebase + reset 重试，适配多场重叠并发）

## 配置（config.yaml）

```yaml
pushplus_token: "你的PushPlus token"

rules:
  - symbol: SMHBUSDT        # 币安交易对
    alert_type: price_below # price_below=低于时提醒 / price_above=高于时提醒
    threshold: 460          # 触发阈值
    frequency: daily        # once=仅提醒一次 / daily=每日最多提醒一次
```

改配置 = 修改 `config.yaml` 提交即可，支持多条规则，每条独立去重。当前监控 SMHBUSDT 低于 **460** 和低于 **435** 两档。

## 本地调试

```bash
pip install -r requirements.txt
python monitor.py --test             # 发送一条测试推送
python monitor.py --once             # 只检测一轮
python monitor.py --loop-minutes 5   # 循环监控 5 分钟
```

## 手动触发

Actions → price-monitor → **Run workflow**：
- `mode = loop`（默认）：连续监控 350 分钟
- `mode = single`：只查一次，用于快速验证链路

## 维护提醒

- **cron-job.org 的 PAT 有到期日**（最长 1 年），过期后 dispatch 会返回 401，连续失败 25 次任务会被自动禁用 —— 到期前需更新 cron-job.org 里的 `Authorization` 头
- 若 workflow 文件改名，需同步修改 cron-job.org 的 URL
- 回滚：删掉 cron-job.org 的任务，并取消 workflow 里 `schedule` 的注释
