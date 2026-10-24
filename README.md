# python requests 403 cloudflare：被拦截的真正原因、免费修复方法和抓取 API 的真实成本账

用 Python requests 抓一个在浏览器里打开完全正常的网站，返回的却是 403——这种情况十有八九是 Cloudflare 在拦你，而且单纯改 User-Agent 大概率没用。这篇文章把拦截的底层原因讲清楚，给你几条免费的修复路线，再算一笔用抓取 API（以 ScraperAPI 为例）绕过 Cloudflare 的真实成本账，帮你判断到底该自己折腾还是直接花钱省事。

> **选购速览**：如果你的爬虫已经写好、只想找一个稳定的“过盾取页”层，ScraperAPI 是省事的选择——注册即有 7 天试用 5,000 credits（无需信用卡），之后有每月 1,000 credits 的免费计划；付费入门档 Hobby 每月 $49。👉 [领取 5,000 免费 credits 试用](https://www.scraperapi.com/signup?fp_ref=coupons)

## 为什么 requests 会被 Cloudflare 返回 403

核心原因是你发出的请求在 Cloudflare 眼里“长得不像浏览器”。具体来说，问题通常出在这几层：

1. **TLS 指纹对不上**。requests 库建立 TLS 连接时的指纹（JA3 哈希）和任何真实浏览器都不匹配。Cloudflare 不需要看你的内容，光看握手特征就能判定这是 Python 脚本。
2. **默认 headers 太扎眼**。requests 默认的 User-Agent 是 `python-requests/x.x`，headers 的数量、顺序也和浏览器不一致，全是自动化特征的死穴。
3. **没有 cookie 处理**。Cloudflare 的挑战流程会先下发一组 cookie，真实浏览器会带着这些 cookie 完成二次访问，裸 requests 做不到。
4. **浏览器端 JavaScript 挑战**。部分站点开启了需要在真实浏览器里执行的 JS 挑战，这类验证纯 HTTP 请求路径根本过不去——这也是为什么换 headers 收效有限。

换 headers、用 Session 维持 cookie 这些方法能解决一部分拦截，但对开了严格模式或 JS 挑战的站点基本无解。有个细节很能说明 Cloudflare 检测的精细程度：它对某些 UA 的匹配是大小写敏感、且锚定字符串开头的，`Mozilla/5.0 Python-urllib/3.14` 能通过而 `Python-urllib/3.14` 不行——规则精确到这种程度，靠猜参数很难赌赢。

## 免费路线能走多远

先试免费方案，能省则省。按成本从低到高：

**第一步：把请求伪装得像浏览器。** 换完整的浏览器 headers（不只 UA，还有 Accept、Accept-Language 等），用 `requests.Session()` 保持 cookie，加超时和重试。对防护等级较低的站点，这一步就够。

**第二步：换能伪造 TLS 指纹的库。** `curl_cffi` 支持 `impersonate="chrome"` 模拟 Chrome 的 TLS 指纹，2026 年有开发者在被 Cloudflare 保护的 API 上用它成功，而老牌的 `cloudscraper` 在同一个站点上失败。如果第一步无效，直接试这条。

**第三步：接受现实，上代理或抓取 API。** 站点开启了 JS 挑战、或者你的云服务器 IP 段本身就在 Cloudflare 的预封锁名单里，本地脚本再怎么伪装也过不去——Cloudflare 对数据中心 IP 段的封锁是按 IP 信誉来的，这一层你在代码里解决不了。

免费路线的隐性成本是维护：Cloudflare 的检测规则一直在变，今天能跑的脚本下个月可能就挂，你得持续跟进。如果抓取任务是长期跑的生产业务，这段时间成本也要算进去。

## 用抓取 API 过盾：ScraperAPI 的真实成本账

ScraperAPI 的用法很直接：把目标 URL 发给它的 API 端点，它通过自己的代理池完成请求、渲染和处理反爬，把 HTML 或解析后的 JSON 返回给你，你现有的 requests 代码只需要改一行请求地址。Cloudflare、Turnstile、Datadome、PerimeterX、Akamai 都有对应的 bypass 机制。

它的计费单位是 credits，这里的账是很多人没算清楚的：**普通页面 1 credit，但启用 Cloudflare Bypass 每次要加 10 credits**。如果目标页面还需要 JS 渲染（`render=true`，也是 10 credits），一次请求就是 21 credits。其他常见成本：Amazon/Walmart/eBay 每次抓取 5 credits，Google/Bing 搜索 25 credits，LinkedIn 30 credits；`premium=true` 加 10，`ultra_premium=true` 加 30。

拿一个具体场景算：抓 10 万个开了 Cloudflare 防护的普通网页（需要 bypass + 渲染），每次 21 credits，共 210 万 credits。对照下表，Hobby 的 10 万 credits 明显不够，Startup（100 万）也不够，得看 Business（300 万，$299/月）——这笔账决定了你该买哪一档。

**全部套餐如下（月付价 / 年付折算月价，年付享 10% off）：**

| 套餐 | 月付 | 年付折算月价 | Credits/月 | 并发线程 | 购买入口 |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [注册免费计划](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [查看 Hobby 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [查看 Startup 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [查看 Business 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [查看 Scaling 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [查看 Professional 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [查看 Advanced 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 定制报价 | 定制报价 | 22,000,000+ | 500+ | [联系销售获取报价](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

所有付费套餐功能一致：JS 渲染、高级住宅和移动 IP、验证码与反爬处理、自动重试、粘性会话（`session_number`，持续 15 分钟）、无限带宽、结构化数据端点（Amazon、Google 等）、DataPipeline、Crawler，以及 LangChain、n8n、Zapier 和 MCP server 集成。差别只在 credits 数量、并发数、地理定位精度（Hobby 和 Startup 只支持 US & EU，Business 起支持国家级）和分析面板保留时长。

几条写在工作流程里能省钱的规则：

- 每个请求的响应头里有 `sa-credit-cost` 字段，直接告诉你这次花了多少；正式跑量前可以用 dashboard 里的 Domain Cost Estimator 查任意 URL 的精确成本，或者调用 `urlcost` 端点。
- `max_cost` 参数可以给单次请求设花费上限，防止一次意外请求烧掉一堆 credits。
- **失败不扣费**：官方文档明确只对成功请求（200 和 404 状态码）收费，以及你在 70 秒内主动取消的请求。被 Cloudflare 拦下来的失败尝试不计费，这对过盾场景很友好。
- 额度用尽默认暂停到下个周期刷新；Hobby 到 Professional 可以开 auto-upgrade 自动升档（通常单价更优），Scaling 以上支持 Pay-As-You-Go 按固定费率继续用，并且能设月度消费上限——这个上限是支出封顶，不会被预扣钱。
- 超过套餐并发上限时直接返回 429 拒绝请求，不排队也不额外收费。

退款方面，官方政策是 **7 天无理由退款**（"no questions asked"），联系客服即可办理，取消订阅也随时可以、不收费用。👉 [开通后不满意 7 天内可全额退款](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## 它过 Cloudflare 到底稳不稳：第三方实测数据

这里得说句实话：ScraperAPI 的 Cloudflare bypass 功能存在、有明确加价，但第三方实测的成功率并不算顶尖。2026 年 6 月有一份针对 6 款抓取 API 的基准测试（24 个 URL、144 次请求，测试方法和原始数据公开在 GitHub），ScraperAPI 在 Cloudflare 防护类目 5 个站点只通过了 2 个，总分 10/24（42%）；同场测试中 Anakin 和 Firecrawl 在 Cloudflare 类目都是 5/5，以反爬为卖点的 ZenRows 也只有 2/5。另一篇 2026 年的评测标题写得更直白，认为它适合基础站点、反爬表现一般。

需要注意这份基准是 Anakin 自己做并自测的，存在利益关联，数字当参考看即可。但多个独立来源的方向一致：**如果你的核心需求就是过 Cloudflare，ScraperAPI 不一定是最强的那一档，它的优势在功能齐全（结构化端点、DataPipeline、批量异步）和计费透明，而不是单点突破率。** 目标站点防护特别凶的话，建议先用免费试用对着你的真实目标测一轮再决定付费档位，别只看营销页。

反过来，它也有明显合适的场景：你的目标大多是普通页面（1 credit）、偶尔有几个难啃的站点（加 10 credits 就能处理），而且你想保留自己写的爬虫逻辑、只把“取页”这一层外包出去。这种模式下它比端到端平台（比如 Apify 那种跑现成 Actor 的）更贴合已有代码。

## 常见问题

**requests 加了 User-Agent 还是 403，还有救吗？**
有，但取决于站点开了哪层防护。只查 headers 的站点，补全 headers + Session + cookie 就能过；开了 TLS 指纹检测的，换 curl_cffi 这类能模拟浏览器握手的库；开了 JS 挑战的，纯 Python 库基本没戏，要么上无头浏览器，要么用抓取 API。

**ScraperAPI 的 bypass 是自动的吗？要加参数吗？**
它对验证码和反爬的处理属于付费功能的一部分，但 Cloudflare、Datadome 等针对性 bypass 会额外消耗 credits（每次 +10）。具体启用方式以官方文档的 supported parameters 为准，跑量前先用 urlcost 端点确认单次成本最稳妥。

**免费计划够测试吗？**
注册后 7 天内有 5,000 credits 试用（无需信用卡），足够对着真实目标验证 bypass 效果；之后的免费计划每月 1,000 credits、5 个并发，做小项目或持续验证够用。测试额度不够还可以联系客服申请加量。

**年付和月付差多少？**
年付统一 10% off：Hobby 从 $49 降到 $44.10/月，一年省 $58.8；Business 从 $299 降到 $269.10/月，一年省 $358.8。用量稳定的话年付划算；还在验证阶段就先月付，反正 7 天内不满意能退。

## 最后怎么选

按你的情况对号入座：

- 只抓几十几百个普通页面、偶尔被拦 → 先把 headers 和 Session 补全，再试 curl_cffi，零成本解决；
- 目标站点开了 JS 挑战、或脚本跑在云服务器上怎么改都被拦 → 用抓取 API，先拿 5,000 credits 的试用对着真实目标测成功率；
- 长期跑量、以普通页面为主 → ScraperAPI 的 credits 模式成本可预算，从 Hobby 起步，开 auto-upgrade 兜底；
- 核心需求就是高难站点的高通过率 → 先用试用对比几家再决定，ScraperAPI 未必是最强选项，但它“失败不扣费 + 7 天无理由退款”的组合让你试错成本接近零。

下单前确认两件事就够了：用 Domain Cost Estimator 查一遍你的目标 URL 单次消耗多少 credits；算一下月用量乘以单价落在哪一档最划算。👉 [领取 5,000 免费 credits 开始测试](https://www.scraperapi.com/signup?fp_ref=coupons)
