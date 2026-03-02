# World Monitor — Data Sources

World Monitor aggregates **40+ real-time data sources** across geopolitics, finance, climate,
cybersecurity, aviation, maritime, and more. Every API integration is designed with
**graceful degradation** — if a key is missing or an upstream is down, the feature simply
disables rather than breaking the dashboard.

This document catalogues every external data source, organised by authentication tier.

---

## Table of Contents

- [Tier 1 — Fully Public (No Key Required)](#tier-1--fully-public-no-key-required)
- [Tier 2 — Free Registration Required](#tier-2--free-registration-required)
- [Tier 3 — Paid / Commercial](#tier-3--paid--commercial)
- [Infrastructure Services](#infrastructure-services)
- [Quick Reference](#quick-reference)

---

## Tier 1 — Fully Public (No Key Required)

These APIs are completely open. No registration, no API key, no auth headers.
The dashboard works out of the box with these sources.

---

### 1. USGS Earthquake API

| Field | Value |
|---|---|
| **Category** | Seismology |
| **Endpoint** | `https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/4.5_day.geojson` |
| **Format** | GeoJSON |
| **Cache TTL** | 30 minutes |
| **Timeout** | 10 seconds |
| **Source file** | `server/worldmonitor/seismology/v1/list-earthquakes.ts` |

**What it provides:** All M4.5+ earthquakes in the last 24 hours. Returns GeoJSON features
with magnitude, coordinates (lon/lat/depth), place name, timestamp, and USGS detail URL.

**Rate limits:** None — USGS provides this as a public feed without restrictions.

**Fallback:** Returns empty array on network failure.

---

### 2. Open-Meteo Climate Archive API

| Field | Value |
|---|---|
| **Category** | Climate |
| **Endpoint** | `https://archive-api.open-meteo.com/v1/archive` |
| **Format** | JSON |
| **Cache TTL** | 3 hours |
| **Timeout** | 20 seconds |
| **Source file** | `server/worldmonitor/climate/v1/list-climate-anomalies.ts` |

**What it provides:** ERA5 reanalysis data (2–7 day lag) for 15 monitored geographic zones:
Ukraine, Middle East, Sahel, Horn of Africa, South Asia, California, Amazon, Australia,
Mediterranean, Taiwan Strait, Myanmar, Central Africa, Southern Africa, Central Asia,
and Caribbean. Returns daily mean temperature and precipitation sums.

**Parameters:** `latitude`, `longitude`, `start_date`, `end_date`,
`daily=temperature_2m_mean,precipitation_sum`, `timezone=UTC`

**Rate limits:** None.

**Fallback:** Returns empty anomalies on failure.

---

### 3. FAA Airport Delays

| Field | Value |
|---|---|
| **Category** | Aviation |
| **Endpoint** | FAA XML delay feed (parsed internally) |
| **Format** | XML → JSON |
| **Cache TTL** | 2 hours |
| **Timeout** | 15 seconds |
| **Source file** | `server/worldmonitor/aviation/v1/list-airport-delays.ts` |

**What it provides:** Real-time US airport delays and cancellations from the Federal Aviation
Administration. Returns average delay in minutes, delay reason (weather, volume, etc.),
and delay type classification.

**Rate limits:** None.

**Fallback:** Falls back to simulated delay data if the feed is unreachable.

---

### 4. CoinGecko Cryptocurrency API

| Field | Value |
|---|---|
| **Category** | Markets — Crypto |
| **Endpoint** | `https://api.coingecko.com/api/v3/coins/markets` |
| **Format** | JSON |
| **Cache TTL** | 10 minutes (Redis) · 8 minutes (in-memory fallback) |
| **Timeout** | 10 seconds |
| **Source file** | `server/worldmonitor/market/v1/list-crypto-quotes.ts` |

**What it provides:** Price, 24h change, market cap, volume, and 7-day sparkline for
tracked cryptocurrencies (bitcoin, ethereum, solana, ripple).

**Parameters:** `vs_currency=usd`, `ids=bitcoin,ethereum,...`, `sparkline=true`,
`price_change_percentage=24h`

**Rate limits:** CoinGecko enforces rate limiting on the free tier (~10–50 req/min).

**Fallback:** Returns in-memory cached result if Redis or upstream fails.

---

### 5. CoinGecko Stablecoin Markets

| Field | Value |
|---|---|
| **Category** | Markets — Stablecoins |
| **Endpoint** | `https://api.coingecko.com/api/v3/coins/markets` |
| **Format** | JSON |
| **Cache TTL** | 10 minutes (Redis) · 8 minutes (in-memory fallback) |
| **Timeout** | 10 seconds |
| **Source file** | `server/worldmonitor/market/v1/list-stablecoin-markets.ts` |

**What it provides:** Peg-deviation monitoring for major stablecoins: USDT, USDC, DAI,
FDUSD, USDe. Computes `deviation = |price - 1.0| × 100` and classifies peg status as
`ON PEG`, `SLIGHT DEPEG`, or `DEPEGGED`.

**Rate limits:** Same CoinGecko free-tier limits as above.

**Fallback:** Returns 8-minute in-memory cached result on failure.

---

### 6. Yahoo Finance

| Field | Value |
|---|---|
| **Category** | Markets — Stocks, Indices, Commodities |
| **Endpoint** | `https://query1.finance.yahoo.com/v8/finance/chart/{symbol}` |
| **Format** | JSON |
| **Cache TTL** | 10 minutes |
| **Timeout** | 10 seconds |
| **Source file** | `server/worldmonitor/market/v1/_shared.ts` |

**What it provides:** Real-time and historical prices for any Yahoo Finance symbol:
- **Indices:** ^GSPC (S&P 500), ^DJI (Dow), ^IXIC (Nasdaq), ^VIX
- **Futures:** GC=F (gold), CL=F (crude oil), NG=F (natural gas), SI=F (silver), HG=F (copper)
- **Stocks:** Any ticker (AAPL, MSFT, etc.)

Returns current price, previous close, and historical close prices for sparkline rendering.

**Rate limits:** Yahoo Finance enforces rate gating; the code uses staggered requests
to avoid 429 errors.

**Fallback:** Returns null on HTTP error. Acts as the fallback source when Finnhub key
is unavailable.

---

### 7. Alternative.me Fear & Greed Index

| Field | Value |
|---|---|
| **Category** | Markets — Sentiment |
| **Endpoint** | `https://api.alternative.me/fng/?limit=30&format=json` |
| **Format** | JSON |
| **Source file** | Market service |

**What it provides:** Bitcoin/crypto Fear & Greed Index — a composite sentiment score
(0 = Extreme Fear, 100 = Extreme Greed) with 30-day history.

**Rate limits:** None.

---

### 8. UNHCR Population API

| Field | Value |
|---|---|
| **Category** | Displacement |
| **Endpoint** | UNHCR population data endpoint |
| **Format** | JSON (paginated) |
| **Cache TTL** | 12 hours |
| **Source file** | `server/worldmonitor/displacement/v1/get-displacement-summary.ts` |

**What it provides:** Global displacement statistics by country — refugee, asylum seeker,
internally displaced person (IDP), and stateless population counts. Licensed under CC BY 4.0.

**Rate limits:** None.

**Fallback:** Returns empty results on failure.

---

### 9. GDELT (Global Database of Events, Language, and Tone)

| Field | Value |
|---|---|
| **Category** | Intelligence / Geopolitics |
| **Endpoint** | `https://api.gdeltproject.org/api/v2/doc/doc?query={query}&mode=artlist&...` |
| **Format** | JSON |
| **Cache TTL** | 10 minutes |
| **Source file** | `server/worldmonitor/intelligence/v1/search-gdelt-documents.ts` |

**What it provides:** Real-time news article search with tone/sentiment scoring.
Queries return articles matching keyword/country filters with metadata including
source domain, publication date, and GDELT tone values.

**Rate limits:** Public API, no authentication required.

**Fallback:** Returns empty articles array with error message.

---

### 10. BIS (Bank for International Settlements)

| Field | Value |
|---|---|
| **Category** | Economics |
| **Endpoints** | `https://stats.bis.org/api/v1/data/{dataset}/{key}?format=csv` |
| **Format** | CSV (SDMX API) |
| **Cache TTL** | 12 hours |
| **Source files** | `server/worldmonitor/economic/v1/_bis-shared.ts`, `get-bis-credit.ts`, `get-bis-exchange-rates.ts`, `get-bis-policy-rates.ts` |

**What it provides:** Three dataset families:
- **Credit-to-GDP ratios** — household and corporate credit as % of GDP
- **Real effective exchange rates (REER)** — broad currency baskets
- **Central bank policy rates** — key interest rates by country

Uses curated country lists for each dataset.

**Rate limits:** None (public SDMX API).

**Fallback:** Returns empty entries on failure.

---

### 11. World Bank Development Indicators

| Field | Value |
|---|---|
| **Category** | Economics |
| **Endpoint** | `https://api.worldbank.org/v2/country/{codes}/indicator/{code}?format=json&date={years}&per_page=1000` |
| **Format** | JSON |
| **Cache TTL** | 24 hours |
| **Source file** | `server/worldmonitor/economic/v1/list-world-bank-indicators.ts` |

**What it provides:** Annual development indicators by country — GDP, population,
literacy rates, and other World Bank metrics.

**Rate limits:** None (public API).

**Fallback:** Returns empty data array on failure.

---

### 12. Polymarket Prediction Markets

| Field | Value |
|---|---|
| **Category** | Prediction Markets |
| **Endpoints** | `https://gamma-api.polymarket.com/events`, `https://gamma-api.polymarket.com/markets` |
| **Format** | JSON |
| **Cache TTL** | 10 minutes |
| **Timeout** | 8 seconds |
| **Source files** | `api/polymarket.js`, `server/worldmonitor/prediction/v1/list-prediction-markets.ts` |

**What it provides:** Active prediction market events and outcomes — event titles,
trading volumes, and outcome probabilities (yes/no prices). Supports category filtering
via `tag_slug`.

**Rate limits:** Behind Cloudflare JA3 fingerprint detection — server-side TLS may be blocked.

**Fallback:** Gracefully returns empty results on Cloudflare block (expected behaviour).

---

### 13. RSS Feeds (98+ Domains)

| Field | Value |
|---|---|
| **Category** | News |
| **Endpoint** | Proxied via `/api/rss-proxy.js` |
| **Format** | XML (RSS 2.0 / Atom) |
| **Cache TTL** | 15 minutes (CDN) · 1 hour stale-while-revalidate |
| **Timeout** | 20 seconds (Google News) · 12 seconds (others) |
| **Source file** | `api/rss-proxy.js` |

**What it provides:** Aggregated news from 98+ allowlisted domains including:

| Domain | Source |
|---|---|
| `feeds.bbci.co.uk` | BBC |
| `www.theguardian.com` | The Guardian |
| `feeds.npr.org` | NPR |
| `news.google.com` | Google News |
| `www.aljazeera.com` | Al Jazeera (EN) |
| `www.aljazeera.net` | Al Jazeera (AR) |
| `rss.cnn.com` | CNN |
| `feeds.arstechnica.com` | Ars Technica |
| `www.theverge.com` | The Verge |
| `techcrunch.com` | TechCrunch |
| `www.technologyreview.com` | MIT Technology Review |
| `rss.arxiv.org` | arXiv |
| `hnrss.org` | Hacker News (RSS mirror) |

…and 85+ more. All feeds are proxied server-side via a domain allowlist to avoid
CORS issues and keep client-side fetching clean.

**Fallback:** Tries direct fetch first, then Railway relay (if configured), then retries relay.

---

### 14. arXiv API

| Field | Value |
|---|---|
| **Category** | Research |
| **Endpoint** | `https://export.arxiv.org/api/query` |
| **Format** | Atom XML (parsed via fast-xml-parser) |
| **Cache TTL** | 1 hour |
| **Timeout** | 15 seconds |
| **Source file** | `server/worldmonitor/research/v1/list-arxiv-papers.ts` |

**What it provides:** Latest research papers by category (default: `cs.AI`).
Returns paper ID, title, abstract, authors, categories, publication date, and URL.

**Parameters:** `search_query=cat:cs.AI`, `start=0`, `max_results=50`

**Rate limits:** None.

**Fallback:** Returns empty array on parse failure.

---

### 15. Hacker News API

| Field | Value |
|---|---|
| **Category** | Research / Tech |
| **Endpoints** | `https://hacker-news.firebaseio.com/v0/{feedType}stories.json` (IDs), `https://hacker-news.firebaseio.com/v0/item/{id}.json` (details) |
| **Format** | JSON |
| **Cache TTL** | 10 minutes |
| **Timeout** | 10 seconds (IDs) · 5 seconds (per item) |
| **Source file** | `server/worldmonitor/research/v1/list-hackernews-items.ts` |

**What it provides:** Top, new, best, ask, show, and job stories from Hacker News.
Two-step fetch: first retrieves story IDs, then fetches individual items concurrently
(max 10 concurrent requests per batch). Returns title, URL, score, comment count,
author, and timestamp.

**Rate limits:** None (Firebase public API).

**Fallback:** Returns empty array on failure.

---

### 16. GitHub Trending Repositories

| Field | Value |
|---|---|
| **Category** | Research / Tech |
| **Primary endpoint** | `https://api.gitterapp.com/repositories` |
| **Fallback endpoint** | `https://gh-trending-api.herokuapp.com/repositories/{language}` |
| **Format** | JSON |
| **Cache TTL** | 1 hour |
| **Timeout** | 10 seconds |
| **Source file** | `server/worldmonitor/research/v1/list-trending-repos.ts` |

**What it provides:** Trending repositories by language and time window (daily/weekly/monthly).
Returns author, name, description, language, star count, period stars, forks, and URL.

**Rate limits:** None documented.

**Fallback:** Uses fallback API endpoint on primary failure.

---

### 17. Abuse.ch — Feodo Tracker

| Field | Value |
|---|---|
| **Category** | Cybersecurity — Threat Intelligence |
| **Endpoint** | `https://feodotracker.abuse.ch/downloads/ipblocklist.json` |
| **Format** | JSON |
| **Cache TTL** | 2 hours |
| **Timeout** | 8 seconds |
| **Source file** | `server/worldmonitor/cyber/v1/_shared.ts` |

**What it provides:** Botnet command-and-control (C2) IP blocklist. Returns IP addresses
with malware family classification (Emotet, Qakbot, etc.), online/offline status,
first-seen and last-online timestamps, and country codes.

**Rate limits:** None.

**Fallback:** Returns empty threats array on HTTP error.

---

### 18. C2IntelFeeds (GitHub CSV)

| Field | Value |
|---|---|
| **Category** | Cybersecurity — Threat Intelligence |
| **Endpoint** | `https://raw.githubusercontent.com/drb-ra/C2IntelFeeds/master/feeds/IPC2s-30day.csv` |
| **Format** | CSV |
| **Cache TTL** | 2 hours |
| **Timeout** | 8 seconds |
| **Source file** | `server/worldmonitor/cyber/v1/_shared.ts` |

**What it provides:** 30-day rolling list of C2 server IP addresses with malware family
descriptions (e.g., "Possible Cobalt Strike C2 IP"). Parsed from a two-column CSV
(IP, Description).

**Rate limits:** GitHub raw content limits (~60 req/min unauthenticated).

**Fallback:** Returns empty threats array on HTTP error.

---

### 19. GeoIP Lookup — ipinfo.io (Primary)

| Field | Value |
|---|---|
| **Category** | Cybersecurity — IP Geolocation |
| **Endpoint** | `https://ipinfo.io/{ip}/json` |
| **Format** | JSON |
| **Cache TTL** | 24 hours (in-memory LRU, max 2048 entries) |
| **Timeout** | 1.5 seconds per IP |
| **Source file** | `server/worldmonitor/cyber/v1/_shared.ts` |

**What it provides:** IP-to-coordinate resolution for threat map plotting.
Returns `loc` (lat/lon string) and `country` code.

**Rate limits:** ~1,500 requests/day on the free tier.

**Hydration config:** 16 concurrent workers, 15-second overall timeout, max 250 IPs per request.

---

### 20. GeoIP Lookup — freeipapi.com (Fallback)

| Field | Value |
|---|---|
| **Category** | Cybersecurity — IP Geolocation |
| **Endpoint** | `https://freeipapi.com/api/json/{ip}` |
| **Format** | JSON |
| **Cache TTL** | 24 hours (shared LRU cache with ipinfo.io) |
| **Timeout** | 1.5 seconds per IP |
| **Source file** | `server/worldmonitor/cyber/v1/_shared.ts` |

**What it provides:** Fallback IP geolocation when ipinfo.io is unavailable.
Returns latitude, longitude, country code, and country name.

**Rate limits:** None documented.

---

### 21. Techmeme / Dev.events (Tech Events)

| Field | Value |
|---|---|
| **Category** | Research / Events |
| **Endpoints** | `https://www.techmeme.com/newsy_events.ics`, `https://dev.events/rss.xml` |
| **Format** | ICS (iCalendar) / RSS XML |
| **Cache TTL** | 6 hours |
| **Source file** | `server/worldmonitor/research/v1/list-tech-events.ts` |

**What it provides:** Upcoming tech conferences and events. Techmeme provides an ICS
calendar (SUMMARY, LOCATION, DTSTART/DTEND, URL). Dev.events provides an RSS feed.
Also includes a curated static list of major conferences.

**Rate limits:** None.

**Fallback:** Gracefully ignores failed fetch, continues with other sources.

---

### 22. FwdStart Newsletter Archive

| Field | Value |
|---|---|
| **Category** | Research / Startups |
| **Endpoint** | `https://www.fwdstart.me/archive` |
| **Format** | HTML → RSS 2.0 (generated) |
| **Cache TTL** | 30 minutes |
| **Timeout** | 15 seconds |
| **Max items** | 30 most recent posts |
| **Source file** | `api/fwdstart.js` |

**What it provides:** Startup newsletter archive scraped from HTML and converted into
an RSS feed on the fly. Returns post title, link, publication date, and subtitle.

**Rate limits:** None.

**Fallback:** Returns 502 error if scrape fails.

---

## Tier 2 — Free Registration Required

These APIs require an API key, but offer generous free tiers. Registration is free.
When a key is missing, the corresponding feature **gracefully disables** — the dashboard
continues to function with remaining sources.

---

### 1. Groq API (AI/LLM — Primary)

| Field | Value |
|---|---|
| **Category** | AI Summarization |
| **Endpoint** | `https://api.groq.com/openai/v1/chat/completions` |
| **Model** | `llama-3.1-8b-instant` |
| **Env var** | `GROQ_API_KEY` |
| **Free tier** | 14,400 requests/day |
| **Cache TTL** | 24 hours (summaries cached in Redis) |
| **Registration** | https://console.groq.com/ |
| **Source files** | `server/worldmonitor/news/v1/summarize-article.ts`, `server/worldmonitor/intelligence/v1/get-country-intel-brief.ts`, `server/worldmonitor/intelligence/v1/classify-event.ts` |

**What it provides:** Powers three features:
1. **Article summarization** — condenses news articles into brief summaries
2. **Country intelligence briefs** — AI-generated geopolitical analysis per country
3. **Event classification** — categorises conflict/disaster events by type and severity

**Degradation:** Returns `{ cached: false, skipped: true, reason: 'GROQ_API_KEY not configured' }`

---

### 2. OpenRouter API (AI/LLM — Fallback)

| Field | Value |
|---|---|
| **Category** | AI Summarization |
| **Endpoint** | `https://openrouter.ai/api/v1/chat/completions` |
| **Model** | `openrouter/free` |
| **Env var** | `OPENROUTER_API_KEY` |
| **Free tier** | 50 requests/day |
| **Cache TTL** | 24 hours |
| **Registration** | https://openrouter.ai/ |
| **Source file** | `server/worldmonitor/news/v1/_shared.ts` |

**What it provides:** Fallback LLM provider when Groq is unavailable or rate-limited.
Same summarization capabilities with lower daily limits.

**Degradation:** Falls back to other providers in the chain (Ollama → browser-based T5).

---

### 3. Finnhub API (Stock Market)

| Field | Value |
|---|---|
| **Category** | Markets — Stocks |
| **Endpoint** | `https://finnhub.io/api/v1/quote?symbol={symbol}` |
| **Env var** | `FINNHUB_API_KEY` |
| **Free tier** | 60 API calls/minute |
| **Cache TTL** | 8 minutes (Redis + in-memory) |
| **Registration** | https://finnhub.io/ |
| **Source file** | `server/worldmonitor/market/v1/list-market-quotes.ts` |

**What it provides:** Real-time stock quotes — current price, % change, high, low, open, close.

**Degradation:** Falls back to Yahoo Finance (Tier 1, no key needed).

---

### 4. FRED API (Federal Reserve Economic Data)

| Field | Value |
|---|---|
| **Category** | Economics |
| **Endpoints** | `https://api.stlouisfed.org/fred/series/observations?series_id={id}&api_key={key}&...`, `https://api.stlouisfed.org/fred/series?series_id={id}&api_key={key}&...` |
| **Env var** | `FRED_API_KEY` |
| **Free tier** | Unlimited (with key) |
| **Cache TTL** | 1 hour |
| **Registration** | https://fred.stlouisfed.org/docs/api/api_key.html |
| **Source file** | `server/worldmonitor/economic/v1/get-fred-series.ts` |

**What it provides:** Federal Reserve time-series data — interest rates, inflation,
unemployment, GDP, and 800,000+ other economic series. Returns observations with
timestamps and series metadata (title, units, frequency).

**Degradation:** Returns `{ series: undefined }` when key missing.

---

### 5. EIA API (U.S. Energy Information Administration)

| Field | Value |
|---|---|
| **Category** | Energy / Economics |
| **Endpoints** | `https://api.eia.gov/v2/petroleum/pri/spt/data/?api_key={key}&...`, `https://api.eia.gov/v2/seriesid/{seriesId}?api_key={key}&...` |
| **Env var** | `EIA_API_KEY` |
| **Free tier** | Unlimited (with key) |
| **Cache TTL** | 1 hour |
| **Registration** | https://www.eia.gov/opendata/ |
| **Source files** | `server/worldmonitor/economic/v1/get-energy-prices.ts`, `api/eia/[[...path]].js` |

**What it provides:** Weekly petroleum data — WTI and Brent crude spot prices,
US crude production, and strategic petroleum reserve inventory levels.

**Degradation:** Returns `{ prices: [] }` when key missing.

---

### 6. NASA FIRMS API (Satellite Fire Detection)

| Field | Value |
|---|---|
| **Category** | Wildfire / Disasters |
| **Endpoint** | `https://firms.modaps.eosdis.nasa.gov/api/area/csv/{apiKey}/{source}/{bbox}/1` |
| **Env vars** | `NASA_FIRMS_API_KEY` or `FIRMS_API_KEY` |
| **Free tier** | Unlimited (with key) |
| **Cache TTL** | 1 hour |
| **Registration** | https://firms.modaps.eosdis.nasa.gov/ |
| **Source file** | `server/worldmonitor/wildfire/v1/list-fire-detections.ts` |

**What it provides:** Near-real-time VIIRS satellite fire detections (updates every ~3 hours).
Returns latitude, longitude, brightness, confidence level, and satellite info in CSV format.

**Degradation:** Returns `{ fireDetections: [], pagination: undefined }` when key missing.

---

### 7. ACLED API (Armed Conflict Location & Event Data)

| Field | Value |
|---|---|
| **Category** | Conflict |
| **Endpoint** | `https://acleddata.com/api/acled/read?event_type={types}&event_date={range}&...` |
| **Env var** | `ACLED_ACCESS_TOKEN` (Bearer token) |
| **Free tier** | Free for researchers and journalists |
| **Cache TTL** | 15 minutes (matches ACLED rate-limit window) |
| **Registration** | https://acleddata.com/ |
| **Source files** | `server/_shared/acled.ts`, `server/worldmonitor/conflict/v1/list-acled-events.ts` |

**What it provides:** Real-time conflict events — battles, explosions/remote violence,
violence against civilians, protests, and riots. Returns event location, date, actors,
fatalities, and event sub-type.

**Degradation:** Returns `{ events: [], pagination: undefined }` with local fallback cache.

---

### 8. UCDP API (Uppsala Conflict Data Program)

| Field | Value |
|---|---|
| **Category** | Conflict |
| **Env var** | `UCDP_ACCESS_TOKEN` |
| **Free tier** | Token required since 2025 |
| **Cache TTL** | 25-hour max-age · 12-hour fallback cache |
| **Registration** | https://ucdp.uu.se/apidocs/ |
| **Source file** | `server/worldmonitor/conflict/v1/list-ucdp-events.ts` |

**What it provides:** Systematic warfare event data from Uppsala University —
country, date, violence type, and conflict classification.

**Degradation:** Returns cached data if less than 12 hours old, otherwise empty array.

---

### 9. Cloudflare Radar API (Internet Outages)

| Field | Value |
|---|---|
| **Category** | Infrastructure |
| **Endpoint** | `https://api.cloudflare.com/client/v4/radar/annotations/outages?dateRange=7d&limit=50` |
| **Env var** | `CLOUDFLARE_API_TOKEN` (Bearer token) |
| **Free tier** | Free with Cloudflare account + Radar access |
| **Cache TTL** | 30 minutes |
| **Registration** | https://dash.cloudflare.com/ (free account) |
| **Source file** | `server/worldmonitor/infrastructure/v1/list-internet-outages.ts` |

**What it provides:** Internet outage annotations over the last 7 days — affected
locations, ASNs, duration, severity, and probable cause.

**Degradation:** Returns `{ outages: [] }` with 12-hour fallback cache.

---

### 10. OpenSky Network (Aircraft Tracking)

| Field | Value |
|---|---|
| **Category** | Military / Aviation |
| **Endpoint** | Proxied through Railway relay at `/opensky` |
| **Env vars** | `OPENSKY_CLIENT_ID`, `OPENSKY_CLIENT_SECRET` (OAuth2) |
| **Free tier** | Available (higher limits for cloud IPs with OAuth2) |
| **Cache TTL** | 2 minutes |
| **Registration** | https://opensky-network.org/ |
| **Source files** | `api/opensky.js`, `src/services/military-flights.ts` |

**What it provides:** Live aircraft state vectors — position, altitude, velocity,
heading, callsign, and ICAO24 transponder code. Primarily used for military flight tracking.

**Degradation:** Circuit breaker returns empty flights when unavailable.

---

### 11. Telegram MTProto (OSINT Channels)

| Field | Value |
|---|---|
| **Category** | OSINT |
| **Endpoint** | Telegram client protocol (stateful MTProto) |
| **Env vars** | `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, `TELEGRAM_SESSION` |
| **Free tier** | Free API keys |
| **Cache TTL** | 30 seconds (client-side) |
| **Registration** | https://my.telegram.org/apps |
| **Source files** | `src/services/telegram-intel.ts`, `api/telegram-feed.js`, `scripts/telegram/session-auth.mjs` |

**What it provides:** Messages from curated OSINT Telegram channels — breaking news,
conflict reports, military alerts, and intelligence. Runs as a long-lived polling
session on the Railway relay server.

**Degradation:** Returns empty items when unconfigured or unavailable.

---

### 12. AlienVault OTX (Threat Intelligence)

| Field | Value |
|---|---|
| **Category** | Cybersecurity |
| **Endpoint** | `https://otx.alienvault.com/api/v1/indicators/export?type=IPv4&modified_since={date}` |
| **Env var** | `OTX_API_KEY` (X-OTX-API-KEY header) |
| **Free tier** | Free pulse access |
| **Cache TTL** | 2 hours |
| **Registration** | https://otx.alienvault.com/ |
| **Source file** | `server/worldmonitor/cyber/v1/_shared.ts` |

**What it provides:** IPv4 threat indicators — malicious IPs with tags, malware family,
and severity classification. Filtered by `modified_since` date.

**Degradation:** Returns empty threats array when key missing.

---

### 13. AbuseIPDB (IP Reputation)

| Field | Value |
|---|---|
| **Category** | Cybersecurity |
| **Endpoint** | `https://api.abuseipdb.com/api/v2/blacklist?confidenceMinimum=90&limit={limit}` |
| **Env var** | `ABUSEIPDB_API_KEY` |
| **Free tier** | Available |
| **Cache TTL** | 2 hours |
| **Registration** | https://www.abuseipdb.com/ |
| **Source file** | `server/worldmonitor/cyber/v1/_shared.ts` |

**What it provides:** Malicious IP blacklist with abuse confidence scores (minimum 90%),
country, and last reported date.

**Degradation:** Returns empty threats array when key missing.

---

### 14. URLhaus (Malicious URLs)

| Field | Value |
|---|---|
| **Category** | Cybersecurity |
| **Endpoint** | abuse.ch URLhaus API |
| **Env var** | `URLHAUS_AUTH_KEY` |
| **Free tier** | Free endpoint exists; auth key adds functionality |
| **Cache TTL** | 2 hours |
| **Registration** | https://urlhaus.abuse.ch/ |
| **Source file** | `server/worldmonitor/cyber/v1/_shared.ts` |

**What it provides:** Recently reported malicious URLs from the abuse.ch community.

**Degradation:** Returns empty threats array when key missing.

---

## Tier 3 — Paid / Commercial

These APIs have commercial pricing, though some offer limited free tiers.

---

### 1. AviationStack (International Flight Delays)

| Field | Value |
|---|---|
| **Category** | Aviation |
| **Endpoint** | `https://api.aviationstack.com/v1/flights?access_key={key}&dep_iata={iata}&limit=100` |
| **Env var** | `AVIATIONSTACK_API` |
| **Pricing** | Free tier available (limited); paid plans for higher volume |
| **Source file** | `server/worldmonitor/aviation/v1/_shared.ts` |

**What it provides:** International airport flight status data — delays, cancellations,
and departure/arrival information by IATA airport code.

**Degradation:** Falls back to simulated delay data when key missing.

---

### 2. Wingbits (Aircraft Enrichment)

| Field | Value |
|---|---|
| **Category** | Military / Aviation |
| **Env var** | `WINGBITS_API_KEY` |
| **Pricing** | Contact for access |
| **Registration** | https://wingbits.com/ |
| **Source files** | `server/worldmonitor/military/v1/get-wingbits-status.ts`, `src/services/wingbits.ts` |

**What it provides:** Aircraft enrichment data — owner, operator, and aircraft type
information for tracked transponder codes.

**Degradation:** Returns `{ configured: false }` when key missing.

---

### 3. AISStream (Live Vessel Tracking)

| Field | Value |
|---|---|
| **Category** | Maritime |
| **Endpoint** | WebSocket connection |
| **Env var** | `AISSTREAM_API_KEY` |
| **Pricing** | Free tier available |
| **Registration** | https://aisstream.io/ |
| **Source file** | `scripts/ais-relay.cjs` (Railway relay) |

**What it provides:** Real-time AIS vessel positions and navigation data via WebSocket
streaming. Runs on the Railway relay server as a long-lived connection.

**Degradation:** Relay skips AIS streaming entirely when key missing.

---

## Infrastructure Services

These are not data-source APIs but backing services that the application depends on.

---

### Upstash Redis (Cross-User Cache)

| Field | Value |
|---|---|
| **Env vars** | `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` |
| **Free tier** | 10,000 commands/day |
| **Registration** | https://upstash.com/ |
| **Source files** | `server/_shared/redis.ts`, `server/_shared/rate-limit.ts` |

**What it provides:** Server-side caching layer shared across all users. Used for:
- API response caching (per-endpoint TTLs)
- Request deduplication
- Rate limiting (sliding window: 300 requests / 60 seconds)

**Degradation:** Returns null; endpoints fall back to uncached direct fetches.

---

### Convex (Registration Database)

| Field | Value |
|---|---|
| **Env var** | `CONVEX_URL` |
| **Free tier** | Available |
| **Registration** | https://dashboard.convex.dev/ |
| **Source file** | `api/register-interest.js` |

**What it provides:** Serverless database for email registration storage.
Rate-limited to 5 registrations per IP per hour.

**Degradation:** Returns 503 when unconfigured.

---

### Sentry (Error Monitoring)

| Field | Value |
|---|---|
| **Env var** | `VITE_SENTRY_DSN` |
| **Free tier** | 5,000 events/month |
| **Registration** | https://sentry.io/ |

**What it provides:** Client-side error tracking and performance monitoring.

**Degradation:** Silent — no error reporting when unconfigured.

---

## Quick Reference

### By Authentication Tier

| Tier | Count | Description |
|---|---|---|
| **Tier 1 — Fully Public** | 22 | No key needed. Dashboard works immediately. |
| **Tier 2 — Free Registration** | 14 | Free API key required. Graceful degradation. |
| **Tier 3 — Paid / Commercial** | 3 | Commercial APIs with optional free tiers. |
| **Infrastructure** | 3 | Backing services (cache, DB, monitoring). |

### By Domain

| Domain | Sources |
|---|---|
| **Markets & Finance** | Yahoo Finance, CoinGecko (×2), Alternative.me F&G, Finnhub, FRED, EIA, BIS, World Bank |
| **Conflict & Geopolitics** | ACLED, UCDP, GDELT, Telegram OSINT |
| **Climate & Disasters** | USGS, Open-Meteo, NASA FIRMS |
| **Aviation & Maritime** | FAA, AviationStack, OpenSky, AISStream, Wingbits |
| **Cybersecurity** | Feodo Tracker, C2IntelFeeds, OTX, AbuseIPDB, URLhaus, ipinfo.io, freeipapi.com |
| **News & Research** | RSS (98+ feeds), arXiv, HackerNews, GitHub Trending, Techmeme, FwdStart |
| **AI / LLM** | Groq, OpenRouter |
| **Prediction** | Polymarket |
| **Displacement** | UNHCR |
| **Infrastructure** | Cloudflare Radar |

### Cache TTL Summary

| TTL | APIs |
|---|---|
| **2 min** | OpenSky |
| **10 min** | CoinGecko, Yahoo Finance, GDELT, HackerNews, Polymarket |
| **15 min** | ACLED, RSS feeds |
| **30 min** | USGS, Cloudflare Radar, FwdStart |
| **1 hr** | FRED, EIA, NASA FIRMS, arXiv, GitHub Trending |
| **2 hr** | FAA, Feodo Tracker, C2IntelFeeds, OTX, AbuseIPDB |
| **3 hr** | Open-Meteo |
| **6 hr** | Techmeme events |
| **12 hr** | BIS, UNHCR |
| **24 hr** | World Bank, GeoIP, Groq/OpenRouter summaries |

### Environment Variables — All Keys

```bash
# Tier 2 — Free registration
GROQ_API_KEY=               # AI summarization (primary)
OPENROUTER_API_KEY=         # AI summarization (fallback)
FINNHUB_API_KEY=            # Stock quotes
FRED_API_KEY=               # Federal Reserve economic data
EIA_API_KEY=                # U.S. energy data
NASA_FIRMS_API_KEY=         # Satellite fire detection
ACLED_ACCESS_TOKEN=         # Conflict event data
UCDP_ACCESS_TOKEN=         # Uppsala conflict data
CLOUDFLARE_API_TOKEN=       # Internet outage tracking
OPENSKY_CLIENT_ID=          # Aircraft tracking (OAuth2)
OPENSKY_CLIENT_SECRET=      # Aircraft tracking (OAuth2)
TELEGRAM_API_ID=            # Telegram OSINT
TELEGRAM_API_HASH=          # Telegram OSINT
TELEGRAM_SESSION=           # Telegram session string
OTX_API_KEY=                # AlienVault threat intel
ABUSEIPDB_API_KEY=          # IP reputation blacklist
URLHAUS_AUTH_KEY=           # Malicious URL tracking

# Tier 3 — Commercial
AVIATIONSTACK_API=          # International flight delays
WINGBITS_API_KEY=           # Aircraft enrichment
AISSTREAM_API_KEY=          # Live vessel tracking

# Infrastructure
UPSTASH_REDIS_REST_URL=     # Redis cache URL
UPSTASH_REDIS_REST_TOKEN=   # Redis cache token
CONVEX_URL=                 # Registration database
VITE_SENTRY_DSN=            # Error monitoring

# Relay server (Railway)
WS_RELAY_URL=               # WebSocket relay endpoint
RELAY_SHARED_SECRET=        # Relay authentication
```
