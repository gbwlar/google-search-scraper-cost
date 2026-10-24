# google search scraper python: three working routes compared, plus the real cost per 1,000 Google searches before you commit

Scraping Google search results with Python comes down to three routes: a bare `requests` + BeautifulSoup script (free, breaks fast), an open-source browser-based scraper (free, but you'll fight CAPTCHAs and manage proxies yourself), or a SERP API like ScraperAPI that returns parsed JSON from a single HTTP call. This guide walks through what each route actually costs and breaks, then digs into ScraperAPI's Google Search API — its sync and async endpoints, the full plan table, and the credit math that decides what 1,000 Google searches really cost you.

> **Quick pick:** if you need reliable Google SERP data at any real volume — rank tracking, SEO monitoring, running keyword lists nightly — the [Hobby plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) at **$49/month** (100,000 credits) is the entry point, and there's a **7-day free trial with 5,000 credits, no credit card required**. 👉 [Check ScraperAPI's plans and start the free trial](https://www.scraperapi.com/pricing/?fp_ref=coupons). If you only need a few hundred searches, a local open-source scraper (covered below) costs nothing.

## Why your requests + BeautifulSoup script dies after a few pages

Google actively defends its search results page. The defenses include IP rate limiting, CAPTCHAs, browser fingerprinting checks, JavaScript validation, request header validation, and region-personalized results. A script that sends `"User-Agent": "Mozilla/5.0"` and parses `<h3>` tags looks exactly like what it is — a bot in a hurry.

The typical failure sequence: first you get HTTP 429 responses, then CAPTCHA pages instead of results, then your IP gets temporarily banned. Even when a request succeeds, Google serves different HTML structures to different sessions, so your CSS selectors break without warning.

This is worth understanding before picking a tool, because every route below is essentially an answer to one question: how do you get around those walls without maintaining the workaround yourself?

## The three routes to scrape Google with Python

**Route 1 — direct requests.** Free, fine for a one-off experiment, unusable beyond a handful of queries. You already know why.

**Route 2 — open-source browser scrapers.** Several actively maintained projects on GitHub take this approach: they drive a hardened, stealth-patched browser locally and scrape what renders. The trade-offs are consistent across them:

- **omkarcloud/google-scraper** runs on Camoufox (a hardened Firefox), outputs clean JSON, and covers a wide range of Google verticals — search, AI Overviews, News, Shopping, Trends, Patents, Finance — through 26 API endpoints you run on your own machine. It's free and has no result-count limit. The catch: Google shows an "unusual traffic" CAPTCHA roughly once every 30 searches, which requires a paid CAPTCHA-solving service key to handle automatically, and at higher volumes you need to supply your own sticky residential proxies because Google pauses the same IP after a few dozen searches. First run also takes several minutes while the browser warms up.
- **shaikhsajid1111/google-serp** uses undetected-Chrome via SeleniumBase with a modular config, but a single run caps out at 10 organic results with no pagination, it needs a local Chrome or Brave binary, and headless mode on Linux servers requires xvfb. It's also very new with essentially no community track record.
- **Scrapy + proxy aggregator setups** (like the google-scrapy-scraper project) handle pagination and output CSV/JSON, but lean on a commercial proxy service — the free tier of the commonly bundled one caps you at 1,000 requests and concurrency of 1.

Open source wins if your volume is genuinely small and you enjoy tinkering. The moment CAPTCHAs and proxy rotation become your problem, you're building infrastructure.

**Route 3 — a managed SERP API.** You send a query, the provider's infrastructure (rotating proxies, CAPTCHA solving, rendering) handles Google's defenses, and you get structured JSON back. ScraperAPI is one of the established options here — running since 2018 — and it's the one this guide goes deep on, because it's the route where the pricing details actually matter for your decision.

## ScraperAPI's Google Search API: what a request looks like

ScraperAPI exposes a dedicated structured endpoint for Google search that returns parsed JSON or CSV, so you don't parse HTML at all. A minimal Python call:

python
import requests

params = {
    "api_key": "YOUR_API_KEY",
    "query": "python web scraping",
    "tld": "com",
    "num": 20,
}

r = requests.get(
    "https://api.scraperapi.com/structured/google/search",
    params=params,
)
data = r.json()

for result in data["organic_results"]:
    print(result["position"], result["title"], result["link"])

print("Total results:", data["search_information"]["total_results"])


The response's `organic_results` include `position`, `title`, `snippet`, `link`, `displayed_link`, and inline sitelinks; `search_information` gives you the displayed result count and timing; there's also a `related_searches` block. Add `"output_format": "csv"` if you'd rather get a spreadsheet-friendly file.

Parameters worth knowing, all documented officially:

- **`tld`** picks the Google domain — around 20 are supported, including `com`, `co.uk`, `de`, `fr`, `co.jp`, `com.au`, `in`, `com.br`, and more. Cross-border rule: to scrape google.com *from* another country (say, from Canada) and get US results, you must set both `tld` and `country_code`.
- **Google-native parameters** pass straight through: `num` (result count), `start` (offset — with `num=10`, `start=10` starts at page 2, which is how you paginate), `hl` (interface language), `gl` (country weighting), `uule` (precise geographic targeting), and `ie`/`oe` for character encodings.
- **`tbs`** filters by time range on the async endpoint: `tbs=d` for the past day, with `h/d/w/m/y` for hour, day, week, month, year. Useful for news monitoring.

For anything beyond a handful of queries, use the **async endpoint** instead: you POST to `https://async.scraperapi.com/structured/google/search` with your key and query (or a `queries` array to batch many keywords in one request), get back a task ID and status URL immediately, and either poll the status URL or set a webhook `callback` to receive results when done. The completed response structure matches the sync endpoint. Batching a nightly keyword list this way is the normal production pattern — you submit once, results trickle back via webhook, no timeout juggling on your side.

## ScraperAPI plans: the full table, then the math that matters

Every plan currently listed on the official pricing page:

| Plan | Credits/month | Concurrent threads | Monthly price | Annual (per-month) | Link |
| --- | --- | --- | --- | --- | --- |
| Hobby | 100,000 | 20 | $49 | $44.10 | [ Try Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | $149 | $134.10 | [ Try Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 | 100 | $299 | $269.10 | [ Try Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | $475 | $427.50 | [ Try Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | $975 | $877.50 | [ Try Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | $1,975 | $1,777.50 | [ Try Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | custom quote | custom | [ Contact sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

All paid plans include JavaScript rendering, premium residential and mobile IPs, JSON auto-parsing, the structured data APIs, unlimited bandwidth, and a 99.9% uptime guarantee. Annual billing knocks roughly 10% off — Hobby drops to $44.10/month, which is $528 a year versus $588 monthly.

Now the part most comparison posts skip: **Google requests cost 25 credits each**, not 1. A standard web page costs 1 credit, Amazon 5, Google and Bing (including subdomains) 25, LinkedIn 30. Divide your plan's credits by 25 and you get your real Google budget:

| Plan | Google searches/month | Effective cost per search |
| --- | --- | --- |
| Hobby ($49) | 4,000 | ~$0.012 |
| Startup ($149) | 40,000 | ~$0.0037 |
| Business ($299) | 120,000 | ~$0.0025 |
| Scaling ($475) | 200,000 | ~$0.0024 |

Two things fall out of this table. Hobby's 100,000 credits sounds generous until you're doing Google work — 4,000 searches a month is fine for rank-tracking a few hundred keywords daily, and it disappears quickly if you're paginating deep (each `start` offset is its own request). Second, the per-search price keeps dropping as you go up-tier, and the biggest single jump is Hobby → Startup: 10× the Google volume for 3× the money. If you're burning through Hobby's allocation, Startup is almost always the right move.

Not sure what a specific URL will cost? The dashboard has a Domain Multiplier to look up any URL's credit cost, an API Playground for test runs, and a `max_cost` parameter to cap any single request. 👉 [Compare credit allowances and current prices on the pricing page](https://www.scraperapi.com/pricing/?fp_ref=coupons).

## Limits buried in the docs that affect real usage

These are confirmed from the pricing page and official documentation, and they're the details that surface after you've already paid:

- **Geo-targeting is limited on lower tiers.** Hobby and Startup only support US and EU locations. Country-level geographic targeting for the rest of the world starts at Business ($299/month). If your SEO work needs local results from, say, Japan or Brazil, Hobby won't cut it regardless of credits.
- **Over-concurrency requests get rejected, not queued.** Exceed your plan's thread limit and ScraperAPI returns a 429 immediately. No extra charge, but your script needs retry logic — this bites people who fire off async batches without throttling.
- **Billing covers 200 and 404 responses, plus requests you cancel within 70 seconds.** Failed requests (5xx, timeouts) don't cost credits. Every response includes a `sa-credit-cost` header showing exactly what that request consumed.
- **Analytics retention is 30 days on Hobby and Startup**, unlimited from Business up. Only relevant if you audit usage patterns months back, but worth knowing before you commit to the cheap tier for accounting purposes.
- **Running out of credits pauses scraping by default.** You can opt into auto-upgrade (available Hobby through Professional, bumps you to the next plan automatically) or pay-as-you-go continuation (Scaling and above, at a fixed per-credit rate with a monthly spending cap you set).

On the trust side: the 7-day trial needs no credit card, and there's a permanent free tier — 1,000 credits with 5 concurrent threads on signup. That's 40 Google searches at the 25-credit rate, enough to verify the endpoint and response structure fit your pipeline before spending anything. 👉 [Sign up for the free trial and test the Google endpoint](https://www.scraperapi.com/signup?fp_ref=coupons).

## When a free open-source scraper is genuinely the better choice

To be fair to the other side of this decision: if your need is a few hundred Google searches per month and you have a desktop that can run a browser, a local scraper costs $0. The omkarcloud project in particular is capable — full result pages, AI Overviews, 12 verticals, clean JSON. The hidden costs are just relocated rather than eliminated: a CAPTCHA-solving service for roughly 1 in 30 searches, residential proxies once volume grows, a machine that stays on, and maintenance when Google shifts its markup. Those API subscriptions for CAPTCHAs and proxies add up too, which is the part "free" tutorials tend to omit.

One factual note that applies to every route on this page: scraping Google search results sits in tension with Google's terms of service. That's true whether you do it with requests, a local browser, or a third-party API — the API route simply moves the risk and the infrastructure to the provider. For routine public SERP data collection (rank tracking, research) this is a widely accepted practice in the SEO industry, but it's your call to make with open eyes.

## FAQ

**How many Google searches can I do on ScraperAPI's free tier?**
The signup free tier gives 1,000 credits and 5 concurrent threads, which is 40 Google searches at 25 credits per request. The 7-day trial adds 5,000 credits (200 Google searches) and requires no credit card.

**Does ScraperAPI return structured JSON for Google, or just HTML?**
Both exist. The generic API returns raw HTML for any URL; the dedicated Google Search endpoint (`api.scraperapi.com/structured/google/search`, plus an async version) returns parsed JSON with `organic_results`, `search_information`, and `related_searches`, or CSV if you set `output_format=csv`.

**How do I get page 2 and beyond of Google results?**
Use the `start` parameter as an offset. With `num=10`, sending `start=10` returns results starting from position 10 — effectively page 2. Each paginated request is a separate Google request, so it consumes another 25 credits.

**Can I scrape google.co.uk or google.de results?**
Yes — set the `tld` parameter to the matching Google domain. Around 20 TLDs are supported. Remember the cross-border rule: fetching a foreign Google domain from your own country requires both `tld` and `country_code` to be set.

**What happens if I exceed my plan's thread limit?**
Requests are rejected with a 429 response rather than queued, and no credits are consumed. Add retry logic or reduce parallelism; upgrading the plan raises the concurrent thread ceiling (20 on Hobby up to 500 on Advanced).

## Picking, in one pass

Only need 40 or so Google searches to prototype something? Use the free 1,000-credit tier — pay nothing, decide later. Running a keyword list of a few hundred terms daily, US/EU results only? Hobby at $49 does 4,000 searches monthly. Passing Hobby's ceiling, or need country-level geo-targeting outside US/EU? Startup or Business, depending on where the credit math lands for your query volume. Doing genuine bulk work — tens of thousands of searches a month — the per-search cost keeps improving up to Scaling, and batch submissions through the async endpoint with a webhook callback will save you more engineering time than the plan upgrade itself.

Whatever tier fits, check two things at checkout: that annual billing is selected if you're keeping the plan past a couple of months (it's about 10% off), and that the credit allowance covers your Google volume at 25 credits per search rather than the 1-credit default people assume. 👉 [Open ScraperAPI's pricing page and start with the free trial](https://www.scraperapi.com/pricing/?fp_ref=coupons).
