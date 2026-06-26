# How to Scrape Google Maps Reviews: 3 Methods Ranked from Easy to Painful — Which One Actually Works at Scale, How to Pick the Right Tool, and Why Most DIY Scripts Break by Monday

Google Maps reviews are one of the most valuable datasets on the public web. If you run a local business, manage brand reputation across dozens of locations, or just want to track what people are saying about a competitor in real time, you already know the pain of trying to get that data out.

The problem? Google doesn't make it easy.

The official Places API gives you **a maximum of 5 reviews per business listing**. That's it. Five. For a restaurant with 3,000 reviews, that's somewhere between useless and insulting. So most people end up trying to scrape Google Maps directly — and that's where things get interesting (and by "interesting" I mean "frustrating").

This guide covers three ways to scrape Google Maps reviews, what actually breaks in each one, and how to set up a working solution that doesn't fall apart every time Google tweaks its front-end.

---

## Why Scraping Google Maps Reviews Is Trickier Than It Looks

Before we get into the methods, it helps to understand what you're dealing with.

Google Maps is not a static website. Reviews don't live in neat HTML tags waiting to be scraped with `requests` and `BeautifulSoup`. The page loads dynamically, reviews are revealed as you scroll, and every batch of additional reviews triggers separate network requests that a static scraper simply can't see.

On top of that:

- CSS class names rotate frequently. Selectors that worked last Friday can silently break by Monday.
- The first page load typically shows only 10–20 reviews. To get more, your scraper has to actually scroll — like a real user would.
- Aggressive request patterns from a single IP draw rate limits and reCAPTCHAs fast.
- There's a cookie consent wall in many regions that blocks the results panel until it's dismissed.

So a plain `requests.get()` returns basically an empty shell. You need a browser that executes JavaScript, scrolls the page, and handles all the anti-bot stuff.

That's the starting point. Now let's look at the three main ways people actually do this.

---

## Method 1: DIY Selenium or Playwright Scraper (The Hard Way)

This is the classic approach — spin up a headless Chrome browser, navigate to a Google Maps listing, scroll through the reviews panel, extract the data, save to CSV.

Here's what a basic Playwright setup looks like:

python
from playwright.sync_api import sync_playwright

def scrape_reviews(business_url, max_reviews=100):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        page.goto(business_url)
        
        # Click the reviews tab
        page.wait_for_selector('[data-tab-index="1"]')
        page.click('[data-tab-index="1"]')
        
        reviews = []
        while len(reviews) < max_reviews:
            # Scroll within the reviews panel
            page.evaluate('document.querySelector(".m6QErb").scrollTop += 1000')
            page.wait_for_timeout(2000)
            
            # Extract current reviews
            items = page.query_selector_all('.jftiEf')
            for item in items:
                reviewer = item.query_selector('.d4r55')
                rating = item.query_selector('.kvMYJc')
                text = item.query_selector('.wiI7pd')
                
                reviews.append({
                    'reviewer': reviewer.inner_text() if reviewer else '',
                    'rating': rating.get_attribute('aria-label') if rating else '',
                    'text': text.inner_text() if text else ''
                })
            
            if len(items) >= max_reviews:
                break
        
        browser.close()
        return reviews


This works. For a handful of businesses, tested manually, it'll pull reviews just fine.

The problem shows up when you try to scale. Run this against 50 businesses in quick succession and Google blocks the IP. Run it the next day and the selectors `.jftiEf`, `.d4r55`, `.wiI7pd` might not exist anymore — Google rotates class names as a basic anti-scraping measure. Now your scraper is silently returning empty results and you won't know until you check the output.

To keep a DIY scraper working, you need:

1. A rotating proxy pool (40M+ IPs minimum if you want to avoid blocks)
2. Randomized delays between requests and scroll actions
3. CAPTCHA solving logic
4. Ongoing selector maintenance — plan for this taking a few hours every month

For a one-off research project? Fine. For anything production-grade or ongoing? The maintenance overhead adds up fast.

---

## Method 2: Scraping Google Maps with ScraperAPI (The Practical Way)

This is where most developers end up once they've burned a few weekends on the DIY approach.

👉 [Try ScraperAPI free — 5,000 API credits, no credit card required](https://www.scraperapi.com/?fp_ref=coupons)

ScraperAPI sits between your code and Google Maps. You send it a URL and your API key, it handles the proxy rotation, browser rendering, CAPTCHA solving, and geographic targeting on its end, and sends back the HTML you actually need. The whole infrastructure — 40M+ proxies across 50+ countries, headless browser fleet, anti-bot fingerprinting — is just there. You don't build it, you don't maintain it.

Here's what using ScraperAPI with Selenium looks like for Google Maps:

python
from seleniumwire import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.chrome.service import Service
from bs4 import BeautifulSoup
import time

API_KEY = "YOUR_SCRAPERAPI_KEY"

proxy_options = {
    "proxy": {
        "http": f"http://scraperapi.render=true.country_code=us:{API_KEY}@proxy-server.scraperapi.com:8001",
        "https": f"http://scraperapi.render=true.country_code=us:{API_KEY}@proxy-server.scraperapi.com:8001",
    }
}

options = webdriver.ChromeOptions()
options.add_argument("--headless")

driver = webdriver.Chrome(
    service=Service(ChromeDriverManager().install()),
    options=options,
    seleniumwire_options=proxy_options
)

driver.get("https://www.google.com/maps/search/pizza+in+New+York")
time.sleep(5)

# Scroll and extract
reviews_panel = driver.find_element("css selector", ".m6QErb")
for _ in range(10):
    driver.execute_script("arguments[0].scrollTop += 1000", reviews_panel)
    time.sleep(2)

soup = BeautifulSoup(driver.page_source, "lxml")
# Parse reviews from soup...

driver.quit()


The `render=true` parameter tells ScraperAPI to use a full headless browser for JavaScript rendering. The `country_code=us` routes the request through a US-based IP so you're getting localized results.

What you're getting out of this:

- **Proxy rotation handled automatically** — no more IP blocks
- **CAPTCHA resolution** — ScraperAPI solves them behind the scenes
- **JavaScript rendering** — review content actually loads
- **Geographic targeting** — scrape from any of 50+ countries

You still write your own parsing logic, which means you control the output format. But the infrastructure problem is solved.

---

## Method 3: Structured Data APIs (The Fast Way, If You're Scraping Specific Domains)

If you need Google Maps data specifically at scale, it's worth knowing that ScraperAPI offers Structured Data Endpoints — pre-built scrapers for high-demand domains that return clean JSON instead of raw HTML.

Rather than getting a blob of HTML back and writing your own parser, you get structured data out of the box: business name, rating, review count, address, hours, and more — already formatted, already clean.

This is particularly useful for competitive research across multiple businesses, or if you're feeding data into a dashboard or database where you need consistent field names every time.

---

## What Data Can You Actually Pull from Google Maps Reviews?

It's worth being specific here because a lot of guides treat "scraping Google Maps" as a monolithic task, but there are actually two distinct data layers:

**Listing-level data** (the business card):
- Business name
- Overall star rating
- Total review count
- Address and hours
- Category (restaurant, hotel, etc.)
- Service options (dine-in, delivery, etc.)
- Phone number and website

**Per-review data** (the actual reviews):
- Review text / snippet
- Reviewer name and profile link
- Star rating (1–5)
- Review date
- Owner response (when present)
- Photos attached to the review
- Topic tags (like "Service" or "Atmosphere")

Most use cases care about that second layer. Reputation monitoring, sentiment analysis, competitor benchmarking — all of that requires the actual text and ratings, not just the aggregate score.

---

## Common Use Cases for Scraping Google Maps Reviews

People come at this problem from a lot of different directions:

**Reputation monitoring** — A chain with 50 locations can't manually check Google Maps every day. A scraper pulls fresh reviews automatically and alerts the team when a new 1-star review hits.

**Competitive analysis** — What are customers complaining about at the competitor across the street? Review text is basically free market research if you can get it at scale.

**Sentiment tracking over time** — Aggregate reviews across a time range and you can measure whether a rebrand, menu change, or service improvement actually moved the needle.

**Lead generation** — Some agencies use Google Maps data (business name, category, rating, review count) to identify high-potential local business leads for outreach.

**Training data for ML models** — Review text with attached star ratings is useful labeled data for sentiment classifiers and recommendation systems.

---

## Anti-Detection Best Practices (Whether You Use an API or Not)

If you're running your own scraper alongside or instead of a managed API, here are the practices that actually make a difference:

**Randomize your delays.** A scraper that waits exactly 2.0 seconds between every scroll looks like a robot. A scraper that waits between 1.5 and 4.2 seconds looks more like a person. Use `time.sleep(random.uniform(1.5, 4.2))`.

**Rotate user agents.** Don't send every request with the same browser fingerprint. Maintain a list of realistic user agent strings and rotate through them.

**Use residential proxies, not datacenter proxies.** Google is good at identifying datacenter IP ranges. Residential proxies that route through actual consumer connections are harder to filter.

**Don't hammer a single business listing.** Scraping 500 reviews from one business in 5 minutes is a pattern. Spread it out, or scrape multiple businesses in parallel with separate sessions.

**Handle the cookie consent wall.** In EU regions especially, Google Maps shows a consent dialog before the map even loads. Your scraper needs logic to detect and dismiss this before navigating to a listing.

---

## ScraperAPI Plans: Which One Fits Your Use Case?

ScraperAPI runs on a credit-based model. A basic HTTP request costs 1 credit. JavaScript-rendered pages cost more (typically 5 credits), and premium domains like Google cost 25 credits per request. That variable pricing makes sense — you're paying proportionally for the actual compute resources.

Here's the full plan breakdown:

| Plan | Monthly Credits | Concurrent Threads | Geotargeting | Price/Month | Annual Price/Month |
|---|---|---|---|---|---|
| **Hobby** | 100,000 | 20 | US & EU only | $49 | $44.10 |
| **Startup** | 1,000,000 | 50 | US & EU only | $149 | $134.10 |
| **Business** | 3,000,000 | 100 | Global | $299 | $269.10 |
| **Scaling** ⭐ | 5,000,000 | 200 | Global | $475 | $427.50 |
| **Professional** | 10,500,000 | 300 | Global | $975 | $877.50 |
| **Advanced** | 21,500,000 | 500 | Global | $1,975 | $1,777.50 |
| **Enterprise** | 22M+ | 500+ | Global | Custom | Custom |

All plans include JS rendering, premium proxies, CAPTCHA/anti-bot handling, automatic retries, custom header support, rotating proxy pools, JSON auto-parsing, and unlimited bandwidth.

The **Hobby** plan is fine for small projects — testing a scraper, pulling reviews for a handful of businesses, light research. But if you're scraping Google Maps specifically, remember that Google requests cost 25 credits each. At 100,000 credits/month, that's 4,000 Google requests — roughly enough to pull reviews from a few hundred businesses depending on how deep you go.

For anything production-grade or at moderate scale, the **Business** plan at $299/month adds global geotargeting and 3 million credits — much more headroom.

The **Scaling** plan is flagged as the most popular for good reason: 5 million credits, 200 concurrent threads, global geotargeting, and pay-as-you-go overflow so you're never hard-blocked if you spike past your limit.

👉 [Start your 7-day free trial — no credit card needed](https://www.scraperapi.com/?fp_ref=coupons)

---

## Discount Codes That Actually Work

The coupon code landscape for ScraperAPI is... noisy. Aggregator sites are full of codes with impressive-sounding percentages that don't activate. Here's what's actually verified:

- **START10** — 10% off your first month. This is the officially verified code, confirmed across multiple independent sources.
- **Annual billing** — Choosing yearly billing gives you a flat 10% off (reflected in the monthly prices in the table above).
- **Free trial** — 5,000 API credits with 5 concurrent connections, no credit card required. Good enough to actually test your Google Maps scraping setup before committing.

The "50% off" and "28% off" codes you'll see on coupon aggregator sites are generally unverified or auto-generated. Save yourself the hassle.

---

## How to Get Started in Under 10 Minutes

1. **Sign up** for a free ScraperAPI account — you get 5,000 API credits to start.
2. **Grab your API key** from the dashboard.
3. **Install the Python library** (or use the proxy mode with Selenium/Playwright as shown above):
   bash
   pip install scraperapi-sdk
   
4. **Make your first request**:
   python
   from scraperapi_sdk import ScraperAPIClient
   
   client = ScraperAPIClient('YOUR_API_KEY')
   result = client.get('https://www.google.com/maps/search/coffee+in+Seattle')
   print(result.text)
   
5. **Parse the HTML** with BeautifulSoup or lxml.
6. **Add scrolling logic** if you need to paginate through reviews.

The free trial is generous enough to actually test your specific use case — not just ping the API once and call it done.

👉 [Get started free with ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons)

---

## The Bottom Line

Scraping Google Maps reviews isn't complicated in concept. The execution is what trips people up — the JavaScript rendering, the scroll-driven pagination, the rotating CSS selectors, the IP blocks.

The DIY route with Selenium or Playwright works for small one-off projects. For anything ongoing, the maintenance overhead is real. Selectors break, proxies get flagged, CAPTCHAs appear. Every one of those requires you to stop what you're doing and fix the scraper.

ScraperAPI removes that layer entirely. You write the parsing logic. They handle everything else. For most teams working with Google Maps data at any meaningful scale, that trade-off makes a lot of sense.

Start with the free trial, see how far 5,000 credits gets you on your specific target, then choose a plan based on what you actually need.
