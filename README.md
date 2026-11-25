# Reddit Scraper API: How ScraperAPI Bills Reddit Scrapes, Can It Return Full Comment Threads, and What Breaks at Scale? (Plan Table + Credit Math)

ScraperAPI has no Reddit-specific endpoint. You send it a reddit.com URL, it returns the page, and the bill is counted in credits: 1 for a standard page, plus 10 if the target sits behind bot protection, 10 more for JavaScript rendering, 10 more for premium proxies. The free plan gives you 1,000 credits a month at 5 concurrent threads, and a 7-day trial starts you at 5,000 credits without a card.

Those multipliers decide everything a Reddit pipeline cares about: cost per thread, whether deep comment trees survive the parse, and what happens the night your job hits the concurrency cap. What follows is the full plan table, the arithmetic for a realistic subreddit crawl, and the limits that apply whichever scraper you point at reddit.com.

> **Most Reddit jobs should start on the free plan, then move to Hobby.** 1,000 credits and 5 threads is enough to prove a pipeline on a handful of threads; Hobby's 100,000 credits and 20 threads is the first tier that suits a nightly job across a few dozen subreddits. Country-level geotargeting starts at Business, and Reddit rarely needs it.
> 👉 [Start on the free plan with 1,000 monthly credits](https://scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan, side by side

| Plan | API credits / month | Concurrent threads | Geotargeting | Purchase |
| --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | Not listed | [Claim the free tier](https://scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | [See the Hobby rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | [See the Startup rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | [See the Business rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | [See the Scaling rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | [See the Professional rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | [See the Advanced rate](https://scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | [See enterprise terms](https://scraperapi.com/pricing?fp_ref=coupons) |

Paid tiers are billed monthly and the per-credit rate improves as the allowance grows. The dollar figure attached to each tier moves with billing period and running promotions, so every row links to the page that shows the current number rather than a figure that ages badly.

Three billing rules matter more than the headline price:

- **The free plan is $0** with 1,000 credits and 5 threads. The trial is 7 days and 5,000 credits, no credit card.
- **Credits that run out pause your scraping** until they renew, unless you turn on auto-upgrade (Hobby, Startup, Business, Scaling, Professional) or pay-as-you-go, which bills at a fixed per-credit rate up to a monthly cap you set (Scaling, Professional, Advanced, Enterprise). A brand-monitoring job that dies quietly at 02:00 because the allowance ran dry is the most common way people lose a week of data.
- **Annual plans can renew early** for a fresh batch of credits, which is the cleanest way to absorb a launch-week spike without changing tier.

## The credit math for a Reddit crawl

ScraperAPI's published credit costs are the whole pricing model in one table: standard page 1 credit; e-commerce sites such as Amazon, Walmart and eBay 5; Google and Bing 25; LinkedIn 30. Bot-protection bypass (Cloudflare, Turnstile, DataDome, PerimeterX/HUMAN, Akamai) adds 10 credits per request, as does `render=true`, `premium=true` and `screenshot=true`. Premium proxies plus rendering together cost 25, ultra premium 30, and ultra premium with rendering 75. Geotargeting, sessions, device type, output format and auto parsing are free.

Reddit is a JavaScript-heavy target with anti-bot measures in front of it, so a thread fetch usually lands above the 1-credit floor. Run the multipliers against Hobby's 100,000 credits:

| What you send | Credits per page | Pages you get from 100,000 credits |
| --- | --- | --- |
| Plain request | 1 | 100,000 |
| With bot-protection bypass | 11 | ~9,090 |
| Bypass + `render=true` | 21 | ~4,760 |
| Bypass + rendering + premium proxy | 31 | ~3,225 |

The free plan tells the same story from the other end: 1,000 credits buys 1,000 plain page loads or roughly 32 fully loaded ones. The 5,000-credit trial is about 161 pages at the 31-credit rate.

Two habits keep that spread from turning into a surprise invoice. Every response carries its cost in the `sa-credit-cost` header, so log it and you will know your real per-thread number after twenty requests instead of after a month. And `max_cost` caps any single request, so one pathological thread cannot eat a day's allowance.

👉 [Check what Hobby's 100,000 credits cost per month](https://scraperapi.com/pricing?fp_ref=coupons)

## Can a Reddit scraper API return full comment threads?

Comments come from the post page, not from the feed, and that distinction is the whole answer.

Reddit caps any listing at roughly 1,000 items. Subreddit feeds, search results, sorted views: whatever the tool, a listing stops around a thousand entries, because the ceiling belongs to Reddit rather than to the scraper. Comments are not subject to it. A single post URL can carry thousands of comments, which is why comment collection is done per post and why paging a subreddit feed will never get you a complete discussion.

The practical split for a ScraperAPI pipeline: use listing pages and search pages for discovery (roughly 1,000 items per view), then hit each post URL individually for the thread. To go past the discovery ceiling, combine sorts and date windows, or run keyword searches, which surface older posts the main feed hides.

Two things no scraper fixes. Private subreddits and deleted posts cannot be fetched, and an empty response is more often a URL that 404s for you than a broken request. Age-gated communities are skipped by most Reddit tooling for the same reason.

If you want finished reply trees rather than HTML, budget for the parse. One published walkthrough of ScraperAPI on Reddit reports a shallow default parsed schema (9 fields per post, 4 per comment, comment IDs null, no parent or depth field), so thread hierarchy has to be rebuilt on your side. Validate that on one thread before you queue five thousand.

## What happens when your batch outgrows your plan

ScraperAPI answers excess concurrency with HTTP 429 and a message to bring your threads back inside the plan limit. There is no extra charge and no queue: the request is turned away rather than parked. For a Reddit crawl that means three requests per worker in flight, a backoff on 429, and a checkpoint file so a restart does not re-scrape threads you already have. Pair that with auto-upgrade and the allowance never becomes the failure point; the concurrency limit does, which is why thread count is the column to shop on once your volume is stable.

👉 [Compare concurrent-thread limits across plans](https://scraperapi.com/pricing?fp_ref=coupons)

## A request you can run in five minutes

python
import requests

target = "https://www.reddit.com/r/learnpython/comments/1w8chn8/"
resp = requests.get(
    "https://api.scraperapi.com",
    params={
        "api_key": "YOUR_API_KEY",
        "url": target,
        "render": "true",
        "max_cost": 25,
    },
    timeout=180,
)
print(resp.headers.get("sa-credit-cost"))  # credits this request consumed
print(resp.status_code, len(resp.text))


The response body is the rendered HTML, which you parse with BeautifulSoup or your extractor of choice; auto parsing, listed as a free option, returns structured output instead. Set `max_cost` before you loop and a single expensive page fails loudly instead of quietly draining the account. Then log `sa-credit-cost` for the first twenty threads and size your plan from that average, not from the 1-credit floor.

## How the dedicated Reddit options compare

| Option | How you pay | Verified limits worth knowing first |
| --- | --- | --- |
| ScraperAPI | Credits per request: 1 standard, +10 bot-protection bypass, +10 `render=true`, +10 `premium=true` | General-purpose rather than Reddit-shaped; free plan 1,000 credits / 5 threads; 429 instead of a queue at the concurrency cap |
| Bright Data Reddit Scraper API | 5,000 free credits per month on new accounts, described as up to 5,000 Reddit records | Posts and comments are separate datasets (comments need their own dataset ID); synchronous calls stop at 1 minute and return HTTP 202 with a `snapshot_id`; NDJSON, JSON, JSON lines, CSV and Parquet |
| Apify, hydrafetch actor | $2 per 1,000 posts ($0.002 each), billed only on returned results | Max 500 posts per input; 14 fields per post; JSON, CSV, Excel, XML |
| Apify, mikolabs actor | $5 monthly free credits, then $15 and $49 per month tiers | Upvote counts, vote ratio and comment counts appear only when `includeMediaLinks` is enabled |

Where the alternatives genuinely win: Bright Data if you want a Reddit-shaped schema and comment dataset without writing a parser, and you can live with asynchronous collection for anything beyond a single URL. Apify if per-record billing suits your budgeting better than credits and a 500-post-per-run ceiling fits your discovery step. ScraperAPI if Reddit is one target among many in the same pipeline and you would rather keep one API key and one invoice, or if you need the raw page because the structured field you want is not in anyone's schema.

## FAQ

**Do I need a Reddit API key or an account to scrape?** No. ScraperAPI fetches public reddit.com pages with its own proxy rotation and CAPTCHA handling; your only credential is the ScraperAPI key. The compliance burden still sits with you once personal data enters the dataset, which both vendors state plainly.

**Why did my request come back as an error instead of data?** A 429 means you exceeded your plan's concurrent threads. It is not a billing event and the request is not queued, so retry after a short backoff and lower your worker count.

**Why do some posts return nothing?** Private subreddits and deleted posts cannot be scraped by any tool. Check that the URL resolves for you in a browser first.

**How many credits will 1,000 threads cost?** It depends entirely on the parameters. At the 31-credit combination of bypass, rendering and premium proxies, 1,000 threads is 31,000 credits, which fits Hobby. At 21 credits it is 21,000. Measure with `sa-credit-cost` before you commit.

**Can I get every comment on a viral thread?** A post URL is not bound by the ~1,000 item listing ceiling, so the comments are reachable, but deep reply branches are where parsed output gets thin. Test one large thread end to end.

## Pick by workload

- **A few thousand posts a month, no comment trees** → Hobby. The 100,000 credits absorb the multipliers and 20 threads keep a nightly job moving.
- **Posts plus full comment threads, collected per post URL** → Startup. Comment collection multiplies request count by thread, not by post, and 1,000,000 credits gives you room to be wrong about the per-page cost for a week.
- **Brand monitoring across dozens of subreddits, or an AI data pipeline that runs hourly** → Business for country-level geotargeting and 100 threads, with pay-as-you-go once you know your steady-state burn.

Before you check out, confirm two things on the pricing page: the current per-tier rate for the billing period you want, and whether your tier includes pay-as-you-go or only auto-upgrade, because that determines what happens at 03:00 on the day you run out.

👉 [Start the 7-day trial with 5,000 credits](https://www.scraperapi.com/?fp_ref=coupons)
