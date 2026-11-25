# Bing Scraper: How Many Results You Can Really Pull, Why Fields Come Back Empty, and What 1,000 Queries Cost

There are three ways to get Bing results into a database: a hosted API that returns structured JSON, a ready-made scraper you configure, or your own parser running through a proxy pool. All three work. They differ in cost per thousand queries, how complete the fields are, and how much of your week goes into maintenance.

ScraperAPI is the third option with the fetch layer already handled. It bills Bing result pages at **25 credits per request**, which turns its $49 Hobby plan into roughly 4,000 Bing pages a month, or about 130 a day. Below are the full plans, the real per-page cost at each tier, the three different result caps that most Bing scraping tutorials mash into one number, and the fields that will come back empty no matter what you pay.

> **Quick verdict:** most people searching for a Bing scraper need 4,000 to 40,000 result pages a month, not millions. **Hobby at $49/month** covers 4,000 Bing pages; **Startup at $149/month** covers 40,000. Step up to **Business at $299** the moment you need a market outside the US and EU, because country-level geotargeting starts there and nowhere below it.
>
> 👉 [Compare ScraperAPI plans and monthly credit allowances](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Every ScraperAPI plan, priced against Bing

Bing requests are charged at 25 credits each, including subdomains. The "Bing pages/mo" column is that credit pool divided by 25, and the cost column divides the monthly price by those pages.

| Plan | Credits/month | Bing pages/mo | Cost per 1,000 pages | Price | Start |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 40 | $0 | $0 | [Open a free account](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | 4,000 | $12.25 | $49/mo, or $44.10/mo billed annually | [Get Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 1,000,000 | 40,000 | $3.73 | $149/mo, or $134.10/mo billed annually | [Get Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 | 120,000 | $2.49 | $299/mo, or $269.10/mo billed annually | [Get Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200,000 | $2.38 | $475/mo, or $427.50/mo billed annually | [Get Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | 10,500,000 | 420,000 | $2.32 | $975/mo, or $877.50/mo billed annually | [Get Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 | 860,000 | $2.30 | $1,975/mo, or $1,777.50/mo billed annually | [Get Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 880,000+ | Custom | Custom quote | [Talk to sales about volume pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Two things in that table change how you should read it. First, the unit price flattens fast: Scaling saves you about 11 cents per thousand pages over Business, so if you move up from Business it should be for concurrency or pay-as-you-go billing, not for a cheaper page. Second, 25 credits is the base rate. When ScraperAPI's bypass logic fires on a request, a 10-credit surcharge is applied automatically, taking a Bing page to 35 credits. Budget on 25 to 35 and check the `sa-credit-cost` response header on your first batch to see which side you land on. At 35 credits, Hobby's pool covers about 2,850 pages instead of 4,000.

The rest of the differences sit outside the credit math:

| Plan | Concurrent threads | Country-level geotargeting | Pay-as-you-go overage | Support |
| --- | --- | --- | --- | --- |
| Hobby | 20 | US and EU only | No | Standard |
| Startup | 50 | US and EU only | No | Standard |
| Business | 100 | Global | No | Standard |
| Scaling | 200 | Global | Yes | Standard |
| Professional | 300 | Global | Yes | Priority |
| Advanced | 500 | Global | Yes | Priority routing |
| Enterprise | 500+ | Global | Yes | Dedicated team, Slack |

Below Scaling, running out of credits returns a 429 instead of a charge. That is a hard stop rather than a surprise invoice, and it's worth knowing before you schedule a nightly job that quietly dies on the 28th.

All plans include JS rendering, rotating proxy pools, premium residential and mobile IPs, CAPTCHA handling, automatic retries, custom headers and JSON auto-parsing. Geotargeting, sessions, device type and output format add no credits.

## Three different Bing result limits, and only one of them is Bing's

Ask how many results a Bing scraper can return and you'll get "about 2,000", "50 per query", or "as many as you want" depending on who answers. They're answering different questions.

**Bing's own ceiling.** Bing presents a page of about 10 organic results and stops serving new ones past page 200. That works out to roughly 2,000 organic results for a single query, however many millions the result count claims. Bing's own documentation puts it as never displaying more than a few thousand results for one search. If you need broader coverage, the answer is more queries with different phrasing, markets and locations, not deeper pagination.

**Pagination is an offset, not a page number.** The `first` parameter sets which organic result moves to the top of the page. `first=10` puts the tenth organic result in position one. A full crawl runs `first=0, 10, 20` and so on, and Bing may append its own parameters after `first`, so parse the query string rather than assuming a fixed URL shape. If you switch between tools, check the starting index first: more than one API documents `first` defaulting to 1 rather than 0, and comparing result sets across those two conventions will quietly shift everything by one position.

**Product caps.** A ready-made actor may cap a query at 20 pages and 50 unique organic results, deduplicating URLs across pages. That's a design decision by the tool builder, not a Bing limit, and it's often the number you actually hit.

Your plan adds a fourth limit that nobody advertises. Exhausting one query to Bing's 2,000-result ceiling means 200 pages at 25 credits, or 5,000 credits. Hobby's 100,000 credits buy you 20 fully exhausted queries a month. That number should shape how you write the loop before you write it.

## Empty fields are normal, not a broken parser

Bing results don't arrive as a fixed schema. A result block sits in `li.b_algo`, the title and link inside its `h2`, and that's the reliable part. Everything else is conditional.

The date is the usual complaint. It appears in `span.news_dt`, often in a format that needs a `\d{2}-\d{2}-\d{4}` regex pass to extract, and when Bing doesn't print one the correct output is `null`. Sitelinks, related searches, People Also Ask, answer cards and knowledge panel summaries appear or don't, by query and by market. Any pipeline that treats a missing field as a failed scrape will report failure rates that don't exist.

One case worth flagging because it looks like a bug: Bing serves a lighter page to Linux browsers that omits the related searches box. A headless Linux setup can fetch clean organic results all day and never once see the related-search module. Presenting as Windows fixes it. If complete SERP modules are why you're paying for a hosted tool, test that specific module against your own keyword list in week one, not month three.

## What writing your own Bing scraper actually costs

The parser itself is an afternoon. `li.b_algo` for the result block, `h2 a` for the link, a full user-agent string like a real Chrome on Windows rather than the generic `Mozilla/5.0`, and a status check that logs non-200s instead of parsing an error page.

The part that doesn't end is everything around it. Bing soft-blocks: it can ignore a `count` or `resultsPerPage` setting above 10 and return a page of ten anyway, return an empty page for a query that has results, or return results unrelated to the search. A common workaround is firing several concurrent requests for page one with different extra URL parameters and cancelling the queued ones once one succeeds. Then there's the proxy pool, because residential IPs are what keeps a few thousand requests a day from turning into a wall of empty pages.

An empty field usually means Bing didn't print it. An empty page usually means you've been soft-blocked. Only one of those is worth retrying, and telling them apart is the maintenance work.

👉 [Put a managed fetch and proxy layer under your existing Bing parser](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Freshness, caching, and why your rankings won't reproduce

Bing results vary by time, by country, by SafeSearch setting and by Bing's own experiments. Two runs with identical inputs a week apart can produce different rankings. Two runs with different inputs produce numbers that mean nothing.

Fix the inputs you can. `mkt` takes a language-country pair like `en-US` and cannot be combined with a two-letter `cc` country code; pick one. `location` at city level is more precise than a country, and if you omit it your results may inherit the location of the proxy instead. Set `safeSearch` to Off, Moderate or Strict and keep it constant: Moderate filters adult images and video, Strict also filters adult text. A `safeSearch` level that drifts between runs will move your brand's positions.

Caching is the other variable. Some hosted SERP APIs return a cached copy for up to an hour unless you explicitly request fresh results, and some offer asynchronous retrieval for large batches, which usually can't be combined with a no-cache flag. Whatever you use, confirm the cache window before you build a rank tracker on top of it. At 25 credits per request, re-pulling an identical page is a real line item, so cache on your side as well.

## If you'd rather buy finished rows than fetch HTML

Two other products come up constantly in this search, and both are reasonable if you want structured output without writing a parser.

SerpApi's Bing endpoint returns JSON with `organic_results` fields (position, title, link, Bing's tracking link, displayed link, snippet) plus related searches, and can return Markdown instead of JSON for LLM pipelines. Note that the Markdown output is text, not JSON, and its field names don't imply a fixed structure. It documents a one-hour cache and an `async` mode that requires fetching results afterward from an archive.

Apify's Bing actor bills one event per delivered organic result row, so run starts, retries, empty queries and failed queries aren't charged. It decodes Bing's redirect links so the URL points at the destination rather than Bing's tracker, deduplicates across pages, and caps at 50 unique organic results per query. It states plainly that it doesn't solve CAPTCHAs or bypass access controls.

The trade-off is straightforward. If you want finished rows and no parsing code, per-result or per-search pricing is what you're buying. If you already have a parser you trust and only need the fetch to survive Bing's blocking, a credit-based fetch layer prices per page and leaves the schema to you.

## The legal part, briefly

Scraping publicly available search results is broadly treated as permissible, but Bing's terms of use and local rules on personal data and intellectual property still apply to what you collect and how you store it. Tools that handle proxies and CAPTCHAs for you are not a legal opinion, and none of them grant you access to anything Bing has put behind a control. For anything involving personal data or redistribution, get your own advice.

## Which plan to buy

Match the plan to the number of Bing pages you actually fetch in a month, not to the size of your ambition.

- **Under 4,000 pages, one US or EU market:** Hobby at $49. Daily rank checks for a few hundred keywords fit comfortably.
- **Tens of thousands of pages, still US or EU:** Startup at $149 covers 40,000 Bing pages with 50 threads.
- **Any market outside the US and EU:** Business at $299 is the floor. Country-level geotargeting doesn't exist below it, so `mkt=ja-JP` or a Brazilian location isn't available on Hobby or Startup at any volume.
- **Volume past 120,000 pages with a need for burst billing:** Scaling adds pay-as-you-go overage and 200 threads, which matters more than the unit-price difference.
- **Continuous pipelines with SLAs:** Professional or Advanced, for thread counts in the hundreds and priority routing.

Before you enter a card, confirm two things. Whether the specific Bing URL pattern you're hitting costs 25 credits or 35, which the `sa-credit-cost` header on a test response will tell you and `max_cost` will cap per request. And whether credits roll over, because they don't: unused credits are lost at each renewal date.

The 7-day trial includes 5,000 credits with no card, which at the base Bing rate is about 200 result pages. Enough to run your real keyword list, see whether the fields you need actually come back, and check the cost header on every response.

👉 [Start the 7-day ScraperAPI trial and run your own Bing queries](https://www.scraperapi.com/pricing/?fp_ref=coupons)
