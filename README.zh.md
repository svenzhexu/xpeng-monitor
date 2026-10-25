# XPeng Monitor 小鹏数据监控

[English](README.md) | [中文](README.zh.md)

小鹏汽车综合数据监控工具，以 [Claude Code](https://claude.com/claude-code) Skill 的形式构建，通过专用 Node.js 脚本持续集成各市场、各维度的小鹏数据。目前已覆盖**中国市场交付周期**与**欧洲市场纯电动车日更销量**两大模块，其他数据持续补充中。

## 目前已支持

| 模块 | 市场 | 数据 | 来源 |
|------|------|------|------|
| 交付周期 | 🇨🇳 中国 | 各车型配置版本的交付等待时间（单位：**周**） | 小鹏官方商城 API（`store.xiaopeng.com`） |
| 日更上牌销量 | 🇪🇺 欧洲 | 纯电动车（BEV）每日上牌/交付销量，最近 12 个月环比对比 | [eu-evs.com](https://eu-evs.com) |

> **范围说明：** 欧洲模块仅覆盖**披露日更数据的少数欧洲国家**，并不能代表 XPeng 在整个欧洲市场的全貌，不应直接解读为「欧洲总销量」。

## 工作方式

- 所有 HTTP 请求一律通过 [`scripts/`](scripts/) 目录下的 Node.js 脚本发起——数据源的反爬策略、登录态、cookie 管理全部封装在脚本内，请勿用 curl/Python 等方式直接请求数据源。
- 脚本仅依赖 Node.js 内置模块（`https`、`fs`、`path`），**无需 `npm install`**，装好 Node.js 即可使用。
- 所有信号（正常数据、`LOGIN_REQUIRED`、`ERROR|...`）统一输出到 **stdout**，调用方只需检查 stdout。
- 瞬时网络错误（socket hang up、ECONNRESET、超时等）已在内置指数退避重试，对调用方透明。

## 安装为 Claude Code Skill

将本仓库克隆到 Claude Code 的 skills 目录：

```bash
git clone https://github.com/<your-username>/xpeng-monitor.git ~/.claude/skills/xpeng-monitor
```

之后在 Claude Code 中直接自然语言提问即可，例如：

- "小鹏 G6 交付周期多久？"、"G7 等几周？"
- "小鹏欧洲销量怎么样？"
- 或直接输入 `/xpeng-monitor`

Claude 会根据问题自动命中对应模块，并以 markdown 表格形式展示结果。

## 模块一 · 中国市场交付周期

### 脚本列表

| 脚本 | 用途 | 用法 |
|------|------|------|
| `fetch_car_series.js` | 获取全部车型列表 | `node scripts/fetch_car_series.js` |
| `fetch_car_versions.js` | 获取指定车型的配置版本列表 | `node scripts/fetch_car_versions.js <carSeriesCode>` |
| `fetch_delivery_period.js` | 查询指定配置的交付周期 | 见下表 |

`fetch_delivery_period.js` 会根据车型代码自动选择 API 策略：

| 模式 | 用法 | 适用车型 | 输出 |
|------|------|---------|------|
| Case 1 单版本 | `node scripts/fetch_delivery_period.js <carVersionCode> <carSeriesCode>` | GX、M03、P7 系列 | `OK\|min\|max\|carName` |
| Case 2 单版本 | `node scripts/fetch_delivery_period.js <carVersionCode> <carSeriesCode>` | 其他车型 | `OK\|min\|max\|carName` |
| Case 2 批量（推荐） | `node scripts/fetch_delivery_period.js <carSeriesCode> --all` | 其他车型 | `versionCode\|OK\|min\|max\|carName`（每行一个版本） |

### 示例

```bash
$ node scripts/fetch_car_series.js
1|G6|G6
2|GX|GX
3|M03_2026|M03
...

$ node scripts/fetch_delivery_period.js GX --all
1585MAX|OK|4|6|GX 1585 四驱 Max 仰望绿
665MAX|OK|6|8|GX 665 Max 星云白
```

最终以表格呈现：

| 车型 | 配置版本 | 车辆名称 | 最短交付周期 | 最长交付周期 |
|------|---------|---------|------------|------------|
| GX | 1585 四驱 Max | GX 1585 四驱 Max 仰望绿 | 4周 | 6周 |
| GX | 665 Max | GX 665 Max 星云白 | 6周 | 8周 |

说明：

- 交付周期单位为**周**，不要理解为"天"或做任何转换。
- 数据以**官网配置页口径**为准（`listDeliveryPeriodByVersion` 接口），该接口失败时自动回退到 SKU 自带的旧快照字段。

## 模块二 · 欧洲市场日更上牌销量

### 脚本用法

```bash
node scripts/xpeng_eu_daily.js [year] [--report] [--email <email> --password <password>]
```

- `year` 可选，默认当前年份
- `--report` 输出 4 个预格式化的 markdown 表格模块（**默认入口**）：
  - ① 月度环比对比（1–D 同范围公平对比）
  - ② 全月总销量
  - ③ 各车型销量明细
  - ④ 工作日销量对比（当前月 vs 上月，1 日 ~ T-2，按第 n 个工作日对齐，剔除周末与未上报完整的最近两天）
- 不加 `--report` 时输出单行紧凑 JSON，含 `daily` / `monthly_full` / `monthly_partial` 三层数据，供灵活分析（某车型长期趋势、单日异常定位、工作日/周末分析、同比对比等）
- 跨年范围自动处理（12 个月窗口通常跨两个自然年，脚本自动获取并合并）

### 登录流程

eu-evs.com 会将未登录 / 被判定为 bot / 会话失效的请求 302 重定向（`/bots`、`/login`、`/onlyNamed` 等）。脚本将数据页的任何 302 都视为需要登录，向 stdout 输出 `LOGIN_REQUIRED`（退出码 2）。

- 传入 `--email` / `--password` 即可自动执行登录（内部处理 CSRF token）
- 登录成功后会话 cookie 保存到 `scripts/.eu-session.json`，后续运行自动复用，凭据只需提供一次
- 尚未注册可前往 <https://eu-evs.com/register> 注册；建议**不要使用常用密码**，可设置独立的临时密码

### 输出信号

| stdout 信号 | 含义 | 处理方式 |
|------------|------|---------|
| JSON / markdown 报告 | 成功 | 原样展示 |
| `LOGIN_REQUIRED` | 数据页 302，需要登录 | 提供 `--email` / `--password` 重新运行 |
| `ERROR\|LOGIN_FAILED` | 邮箱或密码错误 | 重新获取凭据 |
| `ERROR\|HTTP_<code>_year_<y>` | 非预期状态码 | 5xx 稍后重试；4xx 说明请求有问题 |
| `ERROR\|PARSE_FAILED_year_<y>` | 页面结构解析失败 | 数据源页面结构可能已变更 |
| `ERROR\|no_data_for_year_<y>` | 指定年份无数据 | 建议换一年 |
| 其他 `ERROR\|<msg>` | 未列举的异常 | 将信息原样转述 |

## 项目结构

```
xpeng-monitor/
├── SKILL.md              # Skill 定义（触发条件、执行流程、约束）
├── README.md             # English docs
├── README.zh.md          # 本文件
└── scripts/
    ├── fetch_car_series.js        # 中国：获取全部车型列表
    ├── fetch_car_versions.js      # 中国：获取配置版本列表
    ├── fetch_delivery_period.js   # 中国：查询各版本交付周期
    ├── xpeng_eu_daily.js          # 欧洲：最近 12 个月日更数据
    └── diag_*.js                  # 诊断辅助脚本
```

## 免责声明

本项目仅供个人研究/监控使用。数据分别归属于小鹏汽车与 eu-evs.com，请遵守各数据源的服务条款。因统计口径差异，数据可能与官方月度公告不完全一致。
