# Scraping API for Python: What a Request Actually Costs, Which Plan Fits, and a Working `requests` Call

A scraping API for Python is worth paying for when your own `requests` + BeautifulSoup stack starts getting blocked. On ScraperAPI, the cheapest paid tier is **Hobby at 100,000 credits**, and the Python integration is a single `requests.get()` against the API endpoint with your key, plus optional parameters like `render=true` or `country_code=us`. No SDK to install.

The part most write-ups skip: **a credit is not a request.** A standard page costs 1 credit, but Google costs 25, JS rendering adds 10, and premium proxies plus rendering together cost 25. Your $49 of "100,000 requests" can turn into 4,000 real pages if you point it at the wrong targets. Below is the full plan table, the per-parameter credit math, what happens when you run out, and where this API is the wrong purchase.

> **Quick pick:** Most Python developers should start on the free trial (5,000 credits, 7 days) and then **Hobby** if they're scraping a handful of sites with occasional rendering, or **Business** if they need country-level geotargeting and 100 concurrent threads. Hobby caps geotargeting at US & EU only.
> 👉 [Start the 5,000-credit free trial on ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons)

## All ScraperAPI plans, credits, and concurrency

ScraperAPI lists seven paid tiers plus a free trial. Every plan is billed monthly with a discount for annual prepay; the credit allowances below are monthly.

| Plan | Credits / month | Concurrent threads | Geotargeting | Monthly price | Buy |
| --- | --- | --- | --- | --- | --- |
| Free trial | 5,000 (7 days) | — | US & EU | $0, no card | [Start free trial](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |  |
| Startup | 1,000,000 | 50 | US & EU | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |  |
| Business | 3,000,000 | 100 | Country-level | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |  |
| Scaling | 5,000,000 | 200 | Country-level | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |  |
| Professional | 10,500,000 | 300 | Country-level | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |  |
| Advanced | 21,500,000 | 500 | Country-level | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |  |
| Enterprise | 22,000,000+ | 500+ | Country-level | [Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |  |

Two things to read off this table before anything else. Geotargeting is the real tier boundary: Hobby and Startup only cover US and EU proxies, and `country_code` for Japan, Brazil, or India requires **Business** or above. And analytics history is capped at the last 30 days on Hobby and Startup, unlimited from Business up.

A note on the price column: the official pricing page renders plan prices in the checkout flow rather than as static text, so treat the credit and concurrency figures here as the decision-relevant numbers and confirm the current dollar amount on the pricing page before you pay. Credits per dollar improve as you move up, which is why the jump from Hobby to Business is usually about capability, not volume.

## The credit table that decides your bill

This is the part that makes budget estimates wrong. ScraperAPI's own documentation prices each request type and each added capability separately, and they stack.

| Request type or parameter | Credit cost |
| --- | --- |
| Standard page | 1 |
| E-commerce (Amazon, Walmart, eBay) | 5 |
| Google, Bing (incl. subdomains) | 25 |
| LinkedIn | 30 |
| Bot-protection bypass (Cloudflare, DataDome, PerimeterX, Akamai) | +10 |
| `render=true` (JS rendering) | +10 |
| `premium=true` (residential/mobile proxies) | +10 |
| `screenshot=true` | +10 |
| `premium=true` + `render=true` | 25 total |
| `ultra_premium=true` | 30 |
| `ultra_premium=true` + `render=true` | 75 |
| `country_code`, `session_number`, `device_type`, `output_format`, `autoparse`, `keep_headers` | Free |

Do the arithmetic on a realistic Python job. A price-monitoring script hitting 20,000 Amazon product pages with JS rendering on uses 15 credits each (5 for Amazon, +10 for rendering), or 300,000 credits. That's three times Hobby's allowance for 20,000 pages. The same 20,000 pages on plain product blogs with no rendering is 20,000 credits, comfortably inside Hobby.

Two habits save real money here. Every response returns its cost in the `sa-credit-cost` header, so log it per request instead of trusting your estimate. And the `max_cost` parameter caps what a single scrape can consume, which stops one pathological URL from eating a day's budget. The dashboard's Domain Cost Estimator tells you a URL's price before you build the crawler around it.

### Do failed requests and retries cost credits?

A 429 from exceeding your concurrency limit or credit balance is refused, not queued, and the documentation states there's no extra charge for going over — the request is simply turned away. That means a badly paced crawler that hammers the endpoint and collects 429s burns wall-clock time, not credits. Worth knowing because in most scraping APIs the retry path is where budgets quietly die; here the failure mode is throttling, so the fix is a backoff in your Python loop rather than a bigger plan.

## How to call it from Python

ScraperAPI's documented Python integration is REST, not an SDK. You pass the target URL and your key as query parameters to the API endpoint and get HTML back. The shortest working version:

python
import requests

API_KEY = "YOUR_API_KEY"
TARGET = "https://example.com/"

response = requests.get(
    "https://api.scraperapi.com",
    params={
        "api_key": API_KEY,
        "url": TARGET,
    },
    timeout=70,
)
print(response.status_code)
print(response.text[:500])


Adding capability is a matter of adding parameters to that dict:

| Parameter | What it does | Extra credits |
| --- | --- | --- |
| `render=true` | Runs the page's JavaScript, returns post-render HTML | +10 |
| `country_code=us` | Routes through a proxy in that country | 0 |
| `premium=true` | Residential or mobile IPs | +10 |
| `session_number=123` | Reuses the same IP across requests; expires 15 min after last use | 0 |
| `screenshot=true` | Returns a screenshot (implies rendering) | +10 |
| `autoparse=true` | Returns parsed JSON on supported sites | 0 |
| `device_type=mobile` | Mobile user agent | 0 |
| `keep_headers=true` | Sends your own headers through | 0 |

A geo-locked page with rendering, reusing one sticky session:

python
response = requests.get(
    "https://api.scraperapi.com",
    params={
        "api_key": API_KEY,
        "url": "https://example.com/regional-pricing",
        "render": "true",
        "country_code": "de",
        "session_number": "42",
    },
    timeout=70,
)


Note the interaction the docs flag: `session_number` cannot be combined with `premium` or `ultra_premium`. If your target needs residential IPs *and* a sticky session, you have to pick one.

### Fitting it into Scrapy and asyncio

Because it's a plain HTTP endpoint, the integration pattern is a proxy or a downloader middleware rather than an SDK object. With Scrapy the simplest route is pointing your request at the API endpoint with the target URL as a parameter, so concurrency is governed by Scrapy's own settings and by your plan's thread limit. Keep those two numbers aligned: setting `CONCURRENT_REQUESTS = 64` on Hobby's 20 threads just generates 429s.

For asyncio code, `httpx` or `aiohttp` against the same endpoint works the same way. Cap your own in-flight requests to the plan's concurrency and let a semaphore do the throttling.

If you're comparing integration styles across tools: some scraping APIs ship dual sync/async clients and expose fields like `credits_used` and `attempts` directly on the response object, and some SDKs default to two automatic retries with exponential backoff on 408, 429, and 5xx. Those are useful patterns to expect, but ScraperAPI's Python path is the REST call above. The credit accounting comes back in a response header, not in a typed object.

## Where ScraperAPI fits against other options

Three tools come up repeatedly when Python developers shop this category. What follows is configuration and fit, not a scoreboard.

**Scrapeless** ships an official Python SDK (`pip install scrapeless`) with a broader surface: a browser session product that returns a WebSocket endpoint you connect to with Playwright or pyppeteer, a Web Unlocker, crawlers, a Google SERP API, and prebuilt connectors for e-commerce and travel sites. Browser sessions take parameters like `session_ttl=180` and `proxy_country='US'`, and the proxy network covers 195+ countries. Default request timeout is 30000 ms. If you need to *drive* a browser session rather than fetch rendered HTML, this shape is closer to that.

**Context.dev** targets LLM pipelines. Its Python SDK returns markdown, HTML, JSON against a schema you define, query-relevant highlights, or base64 screenshots from one `client.web.scrape()` call. Defaults are two automatic retries with backoff and a 1-minute timeout, Python 3.9+, with an `aiohttp` backend option for async throughput. If the output is going into a RAG index or an LLM prompt rather than a parser, its output formats are the differentiator.

**ScrapingBee** is the one with a dedicated Scrapy middleware, which matters if your project is already a Scrapy spider and you'd rather not hand-roll the request layer.

ScraperAPI's advantage in that group is the transparency of its credit model: a published per-parameter cost table, a per-request cost header, a pre-scrape cost estimator, and a `max_cost` cap. You can compute the bill before you commit. Whether its per-credit pricing beats the others depends on your target mix, so run your own URLs through each cost estimator rather than comparing headline credit counts.

## Who shouldn't buy this

If your targets are static HTML and your request volume is modest, `requests` plus BeautifulSoup plus a handful of free proxies costs nothing and takes an afternoon. A scraping API earns its fee when you hit Cloudflare or DataDome, need JS rendering, or need country-level IPs — the things you can't fix with better parsing code.

On ScraperAPI specifically, skip it if:

- **You need country-level geotargeting on a small budget.** That's Business and up. Hobby and Startup are US & EU only, so a project scraping Japanese or Brazilian retail sites has no cheap tier here.
- **You're scraping Google heavily.** At 25 credits per SERP request, Hobby's 100,000 credits buys 4,000 searches. A dedicated SERP API is usually the cheaper instrument for that job.
- **Your workload is browser automation, not page fetching.** Clicking through a checkout flow or filling forms needs a session-based browser product, not a fetch endpoint.

## Picking a plan

Match the plan to the credit math, not the request count:

- **Evaluating it at all** → the 7-day, 5,000-credit trial, no card required. Point it at your five hardest URLs first; if those come back clean, the rest will.
- **Standard pages, US/EU targets, under 100,000 credits** → Hobby. 20 threads is enough for a single machine's worth of crawling.
- **Same workload with rendering or premium proxies on a slice of requests** → still Hobby, but compute your blended credit cost per URL before assuming the allowance covers your list.
- **Country-level geotargeting, or more than 20 concurrent requests** → Business. This is the first tier where `country_code` works globally and analytics history is unlimited.
- **Steady volume above 3M credits** → Scaling and up, where pay-as-you-go becomes available so an overage doesn't stop your pipeline mid-month.

Two things to confirm before you enter a card. First, check whether auto-upgrade is on: it's opt-in on Hobby through Professional and moves you to the next plan automatically when credits run out, which is convenient and also a spend you didn't explicitly approve that month. Second, price your actual target list through the Domain Cost Estimator — the credit table above gives you the multipliers, but only your URL mix tells you the real number.

👉 [Check the current Hobby price and start with the free trial credits](https://www.scraperapi.com/?fp_ref=coupons)
