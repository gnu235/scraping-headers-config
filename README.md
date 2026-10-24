# web scraping headers：哪些头真的决定成败、怎么配置才不被封，以及什么时候该交给托管抓取 API

你的爬虫返回 403，浏览器打开同一个 URL 却一切正常，问题多半出在请求头（headers）上。反爬系统在响应体开始之前就会检查你的请求头集合：缺了哪个头、值对不对、顺序像不像真浏览器，任何一处不匹配都可能让请求被打上标记。这篇文章讲清楚三件事：哪些 headers 对 web scraping 真正有影响、如何在 Python 里正确发送和调试它们，以及当你手动调优到极限时，像 ScraperAPI 这类托管抓取服务能接手什么。

> **快速结论**：需要绕过 Cloudflare、Datadome 这类防护站点，或者不想继续手动维护 header 指纹的话，ScraperAPI 的 Scaling 套餐（$475/月，500 万 credits，200 并发）是官方标注最受欢迎的档位；小规模项目从免费额度起步即可——注册送 1,000 credits，7 天试用再给 5,000 credits，无需信用卡。👉 [用免费 credits 测试 ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons)

## 为什么被封的往往是 headers，而不是 IP

一个请求头就是一组「名称： 值」，用来携带客户端身份、语言偏好、压缩能力、Cookie 和安全上下文等元数据，规范语义见 MDN 的 HTTP headers 参考和 RFC 9110。反爬系统会拿你的请求头集合和真实 Chrome 或 Firefox 会话的指纹做比对——值、有无、大小写、顺序，任何一项对不上都可能触发标记。所以大多数被封的爬虫栽在请求头阶段，代理反而是第二层问题。

问题在于默认的 HTTP 库几乎自动暴露你。用 Python Requests 的默认设置去请求 httpbin.org/headers，你只会看到 4 个头：`Accept: */*`、`Accept-Encoding: gzip, deflate`、`Host` 和一个写着 `python-requests/2.26.0` 的 User-Agent。真实浏览器一次发十几个头，而且顺序固定。`python-requests/2.x` 这个字符串甚至是全网被封锁最多的 User-Agent 值。

判断请求头异常可以从三个角度看：

1. **缺失**：浏览器总会发送 `Accept`、`Accept-Language`、`Accept-Encoding` 和 `Sec-Fetch-*` 系列，库默认不发。
2. **错误值**：比如 User-Agent 声称是 Chrome 124，`Accept` 的值却是 Chrome 从来没发过的格式。
3. **错误顺序**：浏览器按稳定的顺序发头，多数库按字母序或自己的默认顺序排列。

修补的思路很明确：精确发送真实浏览器会发的那套头，顺序正确、值内部一致。照着这个方向做，成功率提升相当可观——一项 cURL 配置测试显示，从零自定义开始（成功率 32%），加上自定义 User-Agent 升到 79%，再加 Cookies 到 89%，补上内容协商头之后到 97%，平均延迟也从 2.8 秒降到 0.8 秒。

## 对抓取真正重要的请求头清单

| Header | 作用 | 示例值 | 对抓取的影响 |
| --- | --- | --- | --- |
| `User-Agent` | 声明浏览器、引擎和操作系统 | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/126.0.0.0 Safari/537.36` | 高——库默认值是最常见的爬虫破绽 |
| `Accept-Language` | 偏好的语言 | `en-US,en;q=0.9` | 高——浏览器总会发，库基本不发 |
| `Sec-Fetch-Dest/Mode/Site/User` | 请求的来源和方式 | `document` / `navigate` / `none` / `?1` | 高——大多数爬虫完全省略的浏览器专属头 |
| `Sec-Ch-Ua`（client hints） | 结构化的浏览器品牌和版本 | `"Chromium";v="126", "Google Chrome";v="126"` | 高（Chromium 站点）——必须与 User-Agent 一致 |
| `Accept` | 接受的内容类型 | `text/html,application/xhtml+xml,...;q=0.8` | 中——缺失或写成 `*/*` 显得很脚本化 |
| `Accept-Encoding` | 支持的压缩格式 | `gzip, deflate, br, zstd` | 中——漏掉 `br`/`zstd` 会显得客户端过旧 |
| `Referer` | 从哪个页面跳转过来 | `https://www.google.com/` | 中——合理的 referer 有助于深链请求 |
| `Connection` | 保持 TCP 连接 | `keep-alive` | 低——但浏览器会发 |
| `Upgrade-Insecure-Requests` | 优先 HTTPS 升级 | `1` | 低——小的一致性细节 |

这张表里最容易被低估的是 `Accept-Language`。所有浏览器都会发送它，几乎所有默认 HTTP 库都不发，它的缺失本身就是最可靠的廉价自动化信号之一。哪怕只做最精简的配置，也应该带上它。

User-Agent 单独多说两句。它是最常被检查、也最常被伪造的头，有两种写法注定失败：一是数百万个请求复用同一个 UA；二是手动篡改真实 UA 的版本号，造出一个浏览器世界从未存在过的版本。另外要跟得上版本——到今天还声称自己是 Chrome 102 的请求，和完全不带 UA 一样显眼。

## 一致性：比「多发几个头」更重要的规则

单看每个头都对还不够，头与头之间必须互相印证。User-Agent 说自己是 Windows，`sec-ch-ua-platform` 就必须是 `"Windows"`；声称 Chrome 126，`sec-ch-ua` 的品牌列表就必须包含 `v="126"`。伪装 Chrome 却不发 client hints，或发了版本不符的 client hints，都会被感知 Chromium 的检测器发现矛盾。用的是 Firefox 或 Safari 配置的话，应该像它们一样整体省略 client hints。

几个具体的搭配规则：

- **Accept 与请求类型匹配**。Chrome 的页面导航发送长而特定的字符串（`text/html,application/xhtml+xml,...`），JS 里的 XHR/fetch 只发 `*/*` 或 `application/json`。伪装成页面加载却带着 API 客户端式的 `Accept: */*` 是典型错配。
- **Accept-Language 与代理地区一致**。德国住宅 IP 配 `de-DE,de;q=0.9,en;q=0.8` 比 `en-US,en;q=0.9` 更自然。
- **Accept-Encoding 别超配**。广告了 Brotli 或 zstd 却不会解码，轻则收到乱码字节，重则被服务器识破。多数库自动处理 gzip 和 deflate，声明 `br`/`zstd` 之前先确认支持。
- **Referer 按真实导航逻辑来**。直接输入 URL 的导航不发 Referer；逐链接爬取时应把 Referer 设为真实的前一个 URL，这对分页和深爬取都有帮助。

顺带一提，`Accept-Language` 这类头还能做点正经事：`Accept: application/json` 可以在端点同时支持时拿到结构化数据而非 HTML；`If-None-Match`/`If-Modified-Since` 能让未变化的页面返回 `304 Not Modified`，省去重传 500 KB 正文。但也要清楚边界——headers 不会改变源 IP、不会执行 JavaScript、也不会授予访问权限。出口由代理决定，会话状态靠 Cookie，渲染和 JS 执行得靠浏览器运行时。

## 从浏览器复制真实 headers，以及顺序这个坑

最可靠的 header 组来源是你自己的浏览器。步骤：右键页面选 Inspect 打开 DevTools，访问目标站，切到 Network 标签选 Fetch/XHR，点侧栏第一个请求，在 Headers 标签里复制 Request Headers。复制后清理掉 cookie 和 `x-client-data` 这类冗余或过于具体的字段。这套组合的好处是内部自洽——UA、client hints、Accept 全部互相印证。

但有一个技术坑很多教程不提：不少 HTTP client 不遵守你定义的 header 顺序。Python Requests 库并不总是按你设置的顺序发送 headers，GitHub 上的 Issue 5814 记录了这个问题和解法。如果你的库默认按字母序重排，那精心还原的浏览器顺序就白做了，得先确认你的 client 尊重设定的顺序。

复制过来的完整集合也不是照单全收。另一种流派主张最小起步集：一个真实的 UA、`Accept: text/html,application/xhtml+xml`、`Accept-Language: en-US,en;q=0.9`，只在文档要求或 A/B 测试证明有响应差异时再加字段。照抄整个浏览器 header 会引入过期 cookie、不支持的压缩格式和版本特定的 client hints。两种做法的共同底线是：**逐个目标网站测试**。每个请求头都属于三类之一——Required（缺了就拿不到正常响应，必留）、Situational（只在某些端点或条件下需要）、Noise（服务器根本不看，删掉）。

还有一种失败模式藏在传输层：session 默认值、重定向处理器、SDK 中间件、代理网关都可能在你不知情时移除或替换字段。Python 里可以用 `prepare_request()` 检查实际发出的请求，重定向问题查 `response.history`。另外多数客户端跨主机重定向时会移除 `Authorization` 头——这是安全行为，除非两端都受控，否则应保留它。

## 手动调优到极限之后：交给托管抓取 API

把免费的那层修完之后，剩下的差距要用别的层去填。在受保护目标上，专用抓取工具在 Scrapeway 的基准测试里能跑出约 97–99% 的成功率（Scrapfly 以约 99% 居首），而原生 HTTP 客户端在同一批站点上低得多。补齐请求头只能填上差距里「容易的那部分」，TLS 指纹、IP 信誉和 JavaScript 挑战是另外几层，每一层都得用对应的手段去处理。

这就是 ScraperAPI 介入的位置。它把代理轮换、JS 渲染、CAPTCHA 处理和请求头管理打包成一个 API 调用，所有付费套餐都包含 JS Rendering、Premium 住宅与移动 IP、JSON 自动解析、轮换代理池、自定义 Header/Session、CAPTCHA 与反爬检测、自动重试、无限带宽，以及 99.9% 在线率保证。你不需要再维护一份随时过期的 Chrome header 字典。

先看全套餐对比：

| 套餐 | 月付价 | 年付折合月价 | API Credits | 并发线程 | 购买入口 |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49 | $44.10 | 100,000 | 20 | [ 开通 Hobby 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [ 开通 Startup 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [ 开通 Business 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [ 开通 Scaling 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [ 开通 Professional 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [ 开通 Advanced 套餐](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 定制 | — | 22,000,000+ | 500+ | 联系销售定制 |

年付统一 10% 折扣。Geotargeting 有个按档位划分的差异：Hobby 和 Startup 只支持美国和欧洲的定位，Business 起才是国家级定位；分析面板也一样，前两档只保留最近 30 天数据，Business 起不限。

选档的判断依据其实就看两个数：月请求量和目标站点。每月 10 万次以内普通页面，Hobby 的 $49 够用；如果主要抓电商和搜索引擎，同样请求数消耗的 credits 是另一个数量级（下一节细说）。年付比月付每月省约 $5–200 不等，确定长期使用的话，Hobby 年付一年能省 $58.80。

## 真实成本：别用「请求数」估算账单

ScraperAPI 按 credits 计费，一个标准页面请求消耗 1 credit，但不同域名的单价差异大到足以推翻你的套餐预算。这是大多数对比文章不会算给你看的部分：

- 标准页面：1 credit
- Amazon：5 credits
- Google 和 Bing（含子域名）：25 credits
- LinkedIn：30 credits
- Cloudflare、Datadome、PerimeterX 防护站点：每次绕过额外加 10 credits

算一笔实际的账：Hobby 套餐的 100,000 credits，如果全部用来抓 Google 搜索结果，实际只有 4,000 次请求。如果抓的是 Cloudflare 防护页面（11 credits/次），大约 9,000 次。所以选套餐之前，先在 dashboard 里用 Domain Cost Estimator 查一下你的目标 URL 的确切成本，并给每次抓取设一个 `max_cost` 上限，防止单个失控请求烧掉额度。官方还提供完整的域名成本清单和完整的计费文档可供核对。

额度用完之后的行为也值得提前知道：默认是暂停抓取，等下期续费。Hobby 到 Professional 五档可以开启 auto-upgrade，credits 用尽自动升到下一档；Scaling 及以上可以用 pay-as-you-go，按固定的单 credit 价继续抓，并自己设定月度支出上限。并发超限不会产生费用，请求会直接返回 429 而不是排队。年付方案可以提前续费拿到新一整批 credits。

## 在 ScraperAPI 里发送自定义 headers：一个参数和一条限制

前文的手动调优技巧在 ScraperAPI 里依然适用，只是发送方式变了。加上 `keep_headers=true` 参数，API 会把你的自定义 headers（User-Agent、Referer、Cookies 等）转发给目标网站。官方排障文档对此的说法是：启用 `keep_headers=true` 并传入自己的 headers，可以帮助把请求伪装成正常流量，降低被封的概率。用 n8n 集成的话，填入任意 `customHeaders` 条目会自动开启 `keepHeaders`（注意 n8n 里目前只支持通过 HTTP Request Node 使用）。

有一条硬限制必须记住：**ultra_premium 请求会丢弃所有自定义 headers**。也就是说，如果你对某个目标启用最高防护档位（ultra premium），`keep_headers` 在该请求上不生效，header 层的定制让位于服务端的全套处理。遇到目标站点被 ultra premium 覆盖时，header 调优就不再是你的杠杆，此时该关注的是 country 参数（两位 ISO 国家码，如 `US`、`GB`、`DE`，支持 50+ 国家路由）和 `max_cost` 这类仍然生效的控制。

免费试用不要求绑卡，注册流程本身就能拿到额度跑通第一个请求。👉 [领取免费试用额度并跑通第一个请求](https://www.scraperapi.com/?fp_ref=coupons)

## 常见问题

**只改 User-Agent 够吗？**
多数情况下不够。UA 是单一影响最大的头，但反爬系统检查的是整组指纹——一个声称 Chrome 126 的 UA 配上缺失的 `Sec-Fetch-*` 和 `Accept-Language`，仍然很容易被识别。至少要带上 UA、`Accept`、`Accept-Language`、`Accept-Encoding` 和 client hints，并保证互相一致。

**照抄教程里的 header 值行不行？**
行不通。教程里的 UA 版本（Chrome 99、Chrome 124）在发布几个月后就成了显眼的旧版本，值与版本不匹配的组合同样可疑。最可靠的做法是从自己浏览器的 DevTools 复制当前请求头，并随浏览器更新定期轮换。

**headers 能解决所有 403 吗？**
不能。在强防护目标上，headers 只是 TLS 指纹、IP 信誉和 JS 挑战之外的其中一层。正确顺序是：先修好免费的 header 层，确认它能过哪些站点，再针对真正需要的目标上代理或托管服务——为一小部分站点给整个项目堆基础设施不划算。

**Python Requests 发的头顺序和我设置的不一样怎么办？**
这是 Requests 的已知行为（Issue 5814），取决于使用方式它并不总是尊重你定义的顺序。可以用 `prepare_request()` 检查实际发出的请求；如果顺序对目标站点很关键，考虑换用尊重插入顺序的 client，或把请求层交给会自动处理指纹的托管 API。

## 下单前怎么选

按情况对号入座就够了：每月几千到几万次普通页面抓取，Hobby 的 $49（年付折合 $44.10/月）起步即可，先用免费额度确认目标站点的 credit 单价再决定；主力抓 Google、Bing 或 Amazon 的，按 credits 换算实际请求量选档，别按原始请求数选；需要国家级 geotargeting 或不受限的分析数据，直接从 Business（$299）起跳，低两档没有这两个能力；月请求 22M 以上或需要自定义并发和 volume pricing 的，走 Enterprise 定制。

最后两个下单前的检查：一是在 dashboard 的 Domain Cost Estimator 里核实目标 URL 的确切 credit 成本，这决定了你的套餐能撑多少实际请求；二是确认你对 header 定制有真实需求的目标站点是否需要 ultra premium——那个档位下 `keep_headers` 不生效，预算分配会不一样。确认完之后，👉 [按年付方案开通 ScraperAPI，立省 10%](https://www.scraperapi.com/pricing/?fp_ref=coupons)
