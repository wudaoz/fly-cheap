# 飞哪儿便宜 · fly-cheap

一个跑在 GitHub 上的机票低价看板：GitHub Actions 每天抓一次 Google Flights 的价格，存成仓库里的 JSON，GitHub Pages 把它展示出来——从你的城市出发，去哪里最便宜、哪个月最便宜、价格走势，以及低于心理价位 / 创新低 / 大幅下降的提醒记录。

只需要一个 GitHub 账号和一个免费的 [SerpApi](https://serpapi.com/) key。没有服务器，没有数据库，fork 之后改一行出发地就能用。

```
GitHub Actions（每天一次）
  collect.py ──► SerpApi ──► Google Flights 价格 ──► data/<collection>/*.json
  alerts.py  ──► 按规则判断低价 ──► data/alerts/*.json
  build_bundle.py ──► data/bundle.json ──► 提交回仓库 ──► 部署到 GitHub Pages
                                                              │
index.html（静态页面）◄── fetch('data/bundle.json') ◄─────────┘
```

## 快速开始

1. **Fork** 这个仓库（或 Use this template）。
2. 改 `settings.json` 里的出发地（见下一节），提交。
3. 到 [serpapi.com](https://serpapi.com/) 注册，拿到 API key（免费 250 次/月）。
4. 仓库 **Settings → Secrets and variables → Actions → New repository secret**，名字 `SERPAPI_KEY`，值填 key。
5. 仓库 **Settings → Pages → Build and deployment → Source** 选 **GitHub Actions**。
6. **Actions → collect → Run workflow** 手动跑第一次。两三分钟后页面就在 `https://<你的用户名>.github.io/fly-cheap/`，手机浏览器打开即可。

之后每天自动采集一次（默认 00:40 UTC，即北京/新加坡时间 08:40；改 `.github/workflows/collect.yml` 里的 cron）。

## 改出发地和其他设置：`settings.json`

```json
{
  "title": "飞哪儿便宜",
  "origin": "SIN",            // 出发机场或城市的 IATA 代码：SIN、HKG、PVG、TPE、KUL、BKK …
  "originName": "新加坡",      // 页面上显示的出发地名
  "currency": "SGD",          // 币种：SGD、CNY、HKD、USD、MYR …
  "gl": "sg",                 // Google 的国家代码（sg、cn、hk、tw、my …），影响运价来源和本地化
  "tzOffsetHours": 8,         // 本地时区，只影响运行编号的时间
  "monthsAhead": 5,           // 按月扫描几个月（当月起）
  "windowDays": 125,          // 价格板只看未来多少天内出发的
  "europe": true,             // 是否额外扫一次「→ 欧洲，未来 6 个月」
  "fixedRoute": null,         // 可选的固定日期航线，见下
  "thresholds": {}            // 心理价位（往返），例：{"KUL": 110, "BKK": 200}
}
```

**出发地**：改 `origin` 和 `originName` 即可，Explore 接口会返回从那里出发去全球各地的最低价。页面按目的地分成东南亚 / 东亚 / 南亚·中东 / 欧洲四组，这个分组和中文名在 `index.html` 开头的 `CITIES` 常量里；没收录的城市会归到「全部」里，用英文名显示。采集脚本只对收录在 `collect.py` 的 `DEST` 表里的目的地做提醒判断。

**固定日期航线**：某条具体日期的往返想天天盯着（比如假期回家），填 `fixedRoute`：

```json
"fixedRoute": {"dest": "CAN", "out": "2027-02-06", "back": "2027-02-11", "deadline": "2027-02-07",
               "label": "春节去广州", "window": "CNY 2027", "note": "页面上显示的一句说明"}
```

页面会多出一个版块，过了 `deadline` 自动停查。填 `null` 就没有这个版块。

**请求次数**：每天 = `monthsAhead` + (`europe` ? 1 : 0) + (`fixedRoute` ? 1 : 0)，默认 5 + 1 + 0 = 6 次，一个月约 180 次，免费额度 250 次够用；加上固定航线是 7 次 / 天，也够。再多就要缩月份或扩额度。

## 文件

| 文件 | 作用 |
|---|---|
| `settings.json` | 出发地、币种、月份数、固定航线、心理价位。改完提交即生效。 |
| `collect.py` | 采集脚本。读 `settings.json`，调 SerpApi，每个结果写成 `data/<collection>/<id>.json`。纯标准库。 |
| `alerts.py` | 提醒规则。读历史数据和心理价位，命中的写入 `data/alerts/`。 |
| `build_bundle.py` | 把 `data/` 合并成一个 `data/bundle.json`，页面只请求这一个文件。 |
| `index.html` | 仪表盘页面，单文件，无构建步骤。 |
| `.github/workflows/collect.yml` | 定时采集 + 提交数据 + 部署 Pages。改页面或规则的 push 只重新部署，不消耗 SerpApi 额度。 |
| `docs/data-model.md` | 数据文件的字段说明。 |

## 提醒规则（`alerts.py`）

对采集到的最便宜的 9 个目的地（加固定航线），满足任一条件记一条提醒：

- (a) 最低价 ≤ 心理价位；
- (b) 比上一次采集低 10% 以上；
- (c) 已有 ≥ 3 次记录，且创历史新低。

欧洲只用 (a)，以及「比上一次欧洲采集同城市降 10% 以上」。去重：同一目的地已经提醒过，且这次没有比上次提醒价再低 3% 以上，就不重复记录。

心理价位没写的目的地用缺省值（`alerts.py` 和 `index.html` 各有一份相同的表）：东南亚 KUL 110、PEN 120、BKK 200、HKT 170、SGN 170、DAD 220、HAN 230、PQC 210、MNL 210、PNH 240、DPS 230、CGK 200、REP 300、CNX 280、USM 330、KBV 260、CEB 260、LGK 150、BKI 150，其他 250；东亚 HKG 320、TPE 260、MFM 350、TYO 480、OSA 450、SEL 400，其他 450；中国大陆 420；南亚·中东 500；欧洲 ATH 600，VIE / MIL / ZRH / BCN / MUC / ROM 680，PAR / MAD 720，LON / AMS 780，其他 720。这些是按新加坡出发、SGD 定的，换了出发地或币种记得在 `settings.json` 里覆盖。

提醒只显示在页面上。想推到手机，在 workflow 的 `Evaluate alert rules` 之后加一步，把 `alerts.py` 的输出发给 [ntfy.sh](https://ntfy.sh/)、Telegram Bot 之类即可。

## 本地运行

```bash
export SERPAPI_KEY=...
python3 collect.py data        # 消耗 6–7 次请求
python3 alerts.py data
python3 build_bundle.py data
python3 -m http.server 8000    # 打开 http://localhost:8000/
```

Python ≥ 3.9，无第三方依赖。页面用 `fetch` 读 `data/bundle.json`，所以直接双击打开 `index.html` 不行，要起一个静态服务器。

## 已知限制

- 只有 Google Flights 的运价，通常只含随身行李，实际下单价以航司或代理为准。
- Explore 对距今 4–5 个月的月份偶尔返回空，脚本会在 `runs` 里记 `partial` 并跳过那个月。
- GitHub 的定时任务不保证准点，高峰期可能晚几十分钟；仓库 60 天没有人工提交时 GitHub 会暂停定时 workflow（会发邮件，到 Actions 页面点 Enable 就恢复）。
- 页面上的时间按浏览器所在时区显示。

## License

MIT

---

## English

**fly-cheap** is a cheap-flights board that runs entirely on GitHub. A daily GitHub Actions workflow calls SerpApi's Google Travel Explore / Google Flights engines (6–7 requests a day, within the free tier), stores every result as JSON in `data/`, evaluates alert rules, and deploys a static dashboard to GitHub Pages.

Setup: fork → set `origin` / `currency` in `settings.json` → add a `SERPAPI_KEY` repository secret → Settings → Pages → Source: *GitHub Actions* → run the `collect` workflow once. Everything configurable (origin, currency, months, an optional fixed-date route, target prices) is in `settings.json`; alert rules are in `alerts.py`; the data schema is in `docs/data-model.md`. No push notifications by default — pipe `alerts.py`'s output to ntfy.sh or a Telegram bot in the workflow if you want them. MIT licensed.
