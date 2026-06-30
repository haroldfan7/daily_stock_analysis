# GitHub Actions 个人部署清单

本文面向希望 Fork 本项目并用 GitHub Actions 每日自动分析自选股的用户。所有 API Key、Webhook、Token 都应放在 GitHub Actions Secrets 或 Variables 中，不要写入代码、`.env`、workflow 或文档。

## 1. 创建自己的仓库

1. 在 GitHub 打开 `ZhuLinsen/daily_stock_analysis`。
2. 点击右上角 `Fork`，创建到自己的账号或组织下。
3. 打开自己的 fork，进入 `Settings` -> `Secrets and variables` -> `Actions`。
4. 在 `Actions` 页签启用 workflow：`I understand my workflows, go ahead and enable them`。

默认 workflow 为 `.github/workflows/00-daily-analysis.yml`，名称是 `每日股票分析`。它已经包含：

- 工作日定时触发：北京时间 18:00，对应 UTC `10:00`。
- 手动触发：`workflow_dispatch`。
- 手动输入 `mode`：`full`、`market-only`、`stocks-only`。
- 手动输入 `force_run`：跳过交易日检查，适合首次连通性测试。

## 2. 推荐配置：DeepSeek OpenAI-compatible

如果暂定使用 DeepSeek 的 OpenAI-compatible 接口，推荐先配置下面这些值。

| 名称 | 建议位置 | 示例值 | 说明 |
| --- | --- | --- | --- |
| `OPENAI_API_KEY` | Secret | 不填写到仓库 | DeepSeek 或兼容网关的 API Key |
| `OPENAI_BASE_URL` | Variable 或 Secret | `https://api.deepseek.com` | 兼容 OpenAI 的 API 入口；私有网关地址可放 Secret |
| `OPENAI_MODEL` | Variable | `deepseek-v4-flash` | 默认分析模型 |
| `GENERATION_BACKEND` | Variable | `litellm` | 可不配；workflow 默认走 LiteLLM |
| `GENERATION_FALLBACK_BACKEND` | Variable | `litellm` | 可不配；workflow 默认值为 `litellm` |

如需显式指定 LiteLLM 路由，也可以额外设置：

```env
LITELLM_MODEL=openai/deepseek-v4-flash
```

如果改用项目内置 DeepSeek provider，而不是 OpenAI-compatible 入口，则使用 `DEEPSEEK_API_KEY` 或 `DEEPSEEK_API_KEYS`，并参考 `docs/LLM_CONFIG_GUIDE.md` 中的 DeepSeek 渠道示例。

## 3. 企业微信机器人推送

企业微信机器人只需要一个最小 Secret：

| 名称 | 建议位置 | 说明 |
| --- | --- | --- |
| `WECHAT_WEBHOOK_URL` | Secret | 企业微信群机器人 Webhook URL |

可选项：

| 名称 | 建议位置 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `WECHAT_MSG_TYPE` | Variable 或 Secret | `markdown` | 企业微信消息类型 |
| `NOTIFICATION_REPORT_CHANNELS` | Variable | `wechat` | 只把日报推送到企业微信 |
| `REPORT_SHOW_LLM_MODEL` | Variable | `true` | 是否在报告底部显示模型名 |

## 4. STOCK_LIST 示例

`STOCK_LIST` 是必填项，推荐放在 Repository Variables；如果你希望隐藏持仓偏好，也可以放在 Repository Secrets。workflow 会先读取 `vars.STOCK_LIST || secrets.STOCK_LIST`，再注入运行时 `STOCK_LIST`。

### A 股

```env
STOCK_LIST=600519,300750,000001,002594,688981
```

### 港股

```env
STOCK_LIST=hk00700,hk09988,hk03690,hk01810,hk00005
```

### 美股

```env
STOCK_LIST=AAPL,MSFT,NVDA,TSLA,GOOGL
```

### A 股 + 港股 + 美股混合

```env
STOCK_LIST=600519,300750,hk00700,hk09988,AAPL,NVDA,TSLA
```

说明：

- A 股可直接使用 6 位代码。
- 港股可使用 `hk` 前缀加 5 位代码，例如 `hk00700`。
- 美股使用 ticker，例如 `AAPL`、`NVDA`。
- README 中也展示了 `600519,hk00700,AAPL` 这种混合格式。

## 5. 推荐 Secrets 和 Variables

最小可运行配置：

| 名称 | 类型 | 是否必填 | 说明 |
| --- | --- | --- | --- |
| `OPENAI_API_KEY` | Secret | 是 | 使用 DeepSeek OpenAI-compatible 时的 API Key |
| `OPENAI_BASE_URL` | Variable 或 Secret | 是 | `https://api.deepseek.com` |
| `OPENAI_MODEL` | Variable | 是 | 例如 `deepseek-v4-flash` |
| `WECHAT_WEBHOOK_URL` | Secret | 是 | 企业微信机器人 |
| `STOCK_LIST` | Variable 或 Secret | 是 | 自选股列表 |

建议增强配置：

| 名称 | 类型 | 是否必填 | 说明 |
| --- | --- | --- | --- |
| `SERPAPI_API_KEYS` | Secret | 推荐 | 金融新闻搜索补强 |
| `TAVILY_API_KEYS` | Secret | 可选 | 通用新闻搜索 |
| `BOCHA_API_KEYS` | Secret | 可选 | 中文新闻搜索 |
| `TUSHARE_TOKEN` | Secret | 可选 | A 股数据补强 |
| `FINNHUB_API_KEY` | Secret | 可选 | 美股数据补强 |
| `LONGBRIDGE_APP_KEY` / `LONGBRIDGE_APP_SECRET` / `LONGBRIDGE_ACCESS_TOKEN` | Secrets | 可选 | 港股/美股实时行情补强 |
| `MARKET_REVIEW_REGION` | Variable | 可选 | 默认 `cn`，可设为 `cn,hk,us` |
| `ANALYSIS_TIMEOUT_MINUTES` | Variable | 可选 | 默认 `30` |

## 6. 手动 Run workflow 测试

1. 进入自己的 fork。
2. 打开 `Actions`。
3. 选择左侧的 `每日股票分析`。
4. 点击 `Run workflow`。
5. `mode` 首次建议选 `stocks-only`，减少变量。
6. 勾选 `force_run`，跳过交易日检查。
7. 点击绿色 `Run workflow`。

测试通过后，再用 `mode=full` 跑一次完整分析。若企业微信收到报告，说明模型、股票列表和推送渠道都已经连通。

## 7. 常见排查

- Actions 日志提示没有股票列表：确认 `STOCK_LIST` 配在 Repository Variables 或 Repository Secrets，而不是写进代码。
- 模型鉴权失败：确认 `OPENAI_API_KEY` 是 Secret，`OPENAI_BASE_URL` 只填到兼容入口，不要追加 `/chat/completions`。
- 没收到企业微信消息：确认 `WECHAT_WEBHOOK_URL` 是完整机器人 URL，并检查群机器人是否启用关键词或安全限制。
- 手动运行没有执行：确认使用自己的 fork，并且已经在 Actions 页启用 workflow。
