# Claude AI — Social Media Sentiment & Growth Analysis

A multi-platform data pipeline that scrapes and visualizes public discourse around **Claude AI** across Hacker News, Reddit, and YouTube. The project was developed across two environments: a **Mac local runner** (Jupyter Notebook) for HN and Reddit, and **Google Colab** for YouTube scraping and all visualizations.

---

## Project Structure

```
├── scrap_hackernews.ipynb        # HN scraper — Algolia API (Mac / Jupyter)
├── scrap_reddit.ipynb            # Reddit scraper — public JSON API (Mac / Jupyter)
├── scrap_youtube.ipynb           # YouTube scraper — web scraping attempt (Colab)
├── scrap_youtube_api.ipynb       # YouTube scraper — official Data API v3 (Colab)
├── data_vis_hn.ipynb             # HN visualizations (Mac / Jupyter)
├── data_vis_reddit.ipynb         # Reddit visualizations (Colab)
├── data_vis_youtube_scrap.ipynb  # YouTube visualizations — scraped data (Colab)
├── data_vis_youtube_api.ipynb    # YouTube visualizations — API data (Colab)
```

---

## Environments

### Mac + Jupyter (Local)
The HN and Reddit scrapers — and their visualization notebooks — were developed and run locally on a Mac using a Jupyter Notebook server.

```bash
pip install requests pandas jupyter
jupyter notebook
```

No Colab file upload dialogs are used in these notebooks. CSVs are written directly to the local working directory and read back by the visualization notebooks from the same path.

**Files written locally:**
- `data/hn_claude_stories.csv`
- `data/hn_claude_comments.csv`
- `data/hn_claude_summary.json`
- `reddit_posts_rich_metrics.csv`
- `reddit_comments_rich_metrics.csv`

### Google Colab
The YouTube scrapers and all visualization notebooks were run on Colab. They use `google.colab.files.upload()` to receive CSV inputs and save chart PNGs to the session.

---

## Platform Details

### Hacker News (`scrap_hackernews.ipynb`)
- Scraped via the **Algolia HN Search API** — no authentication required
- Covers a rolling 6-month window using `numericFilters: created_at_i >= cutoff`
- Searches multiple query variants: `"Claude"`, `"Anthropic Claude"`, `"Claude 3.5"`, `"Claude 3.7"`
- Post-filters hits with an AI-context check (terms like `anthropic`, `llm`, `sonnet`, `mcp`, `context window`, etc.) to exclude false positives about people named Claude
- Enriches each hit via the official HN Firebase API (`hacker-news.firebaseio.com`) to get live scores, comment counts, and reply trees
- **Outputs:** `hn_claude_stories.csv`, `hn_claude_comments.csv`, `hn_claude_summary.json`

### Reddit (`scrap_reddit.ipynb`)
- Scraped using Reddit's **public JSON API** — no PRAW, no OAuth required; appends `.json` to standard Reddit URLs
- Covers 8 subreddits: `r/Anthropic`, `r/ClaudeAI`, `r/singularity`, `r/MachineLearning`, `r/artificial`, `r/ChatGPT`, `r/LocalLLaMA`, `r/OpenAI`
- Fetches both `new` (recent) and `top?t=year` (high-engagement) posts per subreddit, then filters to the last 180 days and keyword-matches against Claude-related terms
- Also fetches `about.json` per subreddit for subscriber counts and active user proxies
- Collects up to 100 top-sorted comments per post
- Derived metrics: `engagement_score_simple`, `comments_per_score`, `score_per_subscriber`
- **Outputs:** `reddit_posts_rich_metrics.csv`, `reddit_comments_rich_metrics.csv`

### YouTube

#### Approach 1 — Web Scraping (`scrap_youtube.ipynb`)
We initially attempted to scrape YouTube without the official API by parsing the `ytInitialData` and `ytInitialPlayerResponse` JSON blobs embedded in YouTube's HTML, and calling the internal `youtubei/v1/next` InnerTube endpoint for engagement signals.

**What we could extract:**
- ✅ Titles, channel names, view counts
- ✅ Video duration, publish date, keywords/tags from `playerResponse`
- ✅ Subscriber counts via channel page parsing
- ⚠️ Like counts — attempted via 5 different regex patterns on InnerTube JSON responses, but **unreliable**; YouTube obfuscates and rotates these fields with each frontend deploy
- ❌ Dislike counts — removed from all public YouTube surfaces in December 2021
- ❌ Save / playlist-add rates — never exposed publicly on any surface
- ❌ Comment counts — inconsistently available; returned as `None` for the majority of videos

**Why engagement metrics failed:**
YouTube's frontend aggressively restricts engagement data for scrapers. Like counts are either absent or hidden behind accessibility label strings that change format regularly. Comment counts were only partially recoverable and frequently missing. Despite attempting multiple extraction strategies, the scraper returned `None` for likes and comments on most videos. This approach was abandoned for engagement analysis and used only for surface-level metadata (titles, views, duration).

#### Approach 2 — YouTube Data API v3 (`scrap_youtube_api.ipynb`)
We switched to the official **YouTube Data API v3**, which provided clean structured data at scale.

**What we extracted:**
- ✅ View counts, like counts, comment counts (via `statistics` part)
- ✅ Duration, publish date, language (via `contentDetails` + `snippet`)
- ✅ Channel subscriber count and verification status (via `channels.list`)
- ✅ Top comment per video (via `commentThreads.list` ordered by relevance)
- ✅ Hashtags parsed from video descriptions
- ✅ SEO tags from `snippet.tags`
- ✅ Derived: views-per-hour velocity (`vph_velocity`), engagement rate `(likes + comments) / views * 100`

**API quota cost per 100-video run:**
| Call | Units |
|---|---|
| `search.list` × 2 pages | 200 |
| `videos.list` × 2 batches | 2 |
| `channels.list` | ~2 |
| `commentThreads.list` × 100 videos | 100 |
| **Total** | **~304** |

Default daily quota is 10,000 units — well within limits for this dataset size.

**Setup:**
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Enable **YouTube Data API v3**
3. Create an API key and paste it into `scrap_youtube_api.ipynb` as `API_KEY`

> **Note:** Dislike counts and save/playlist-add rates are permanently unavailable via any YouTube surface or API and cannot be recovered by any method.

---

## Visualizations

### HN (`data_vis_hn.ipynb`)
1. **Hockey Stick Chart** — Monthly stories + comments growth with annotated release events (Claude 3.7 Sonnet, Claude Code GA, Opus 4.6)
2. **Ecosystem Bar Chart** — Mention frequency of tools (Claude Code, MCP, Cursor, Copilot, Aider, Codex, OpenCode)
3. **Sentiment Stacked Area** — Positive vs. negative keyword share over time
4. **Model vs. Harness Pie** — Model discussion (~32%) vs. tooling/workflow discussion (~68%)

### Reddit (`data_vis_reddit.ipynb`)
1. **Content-type performance** — Humor and Compliment flair outperform Complaint by 3–5× in avg score despite lower post volume
2. **Crosspost amplification** — Crossposted posts score ~9× higher; scatter + mean overlays per group
3. **Day × Hour heatmap** — Median score by posting time; Sun–Tue 18–22 UTC is the peak window
4. **Weekly trend + spike forensics** — Feb 23 spike annotated; top posts from that week surface the cause

### YouTube — Scraped (`data_vis_youtube_scrap.ipynb`)
1. Search query performance — median + total views by keyword cluster
2. Title framing — Drama > Tutorial > Hype > Neutral in median views
3. Duration sweet spots — `<2 min` Shorts and `8–20 min` mid-form outperform the `10–15 min` default
4. UGC vs official — independent creators drive the large majority of total views
5. Title length optimization — 55–70 character window performs best (mobile truncation alignment)
6. Monthly velocity + keyword tag depth sweet spot (16–25 tags optimal)

### YouTube — API (`data_vis_youtube_api.ipynb`)
6-panel dark-theme dashboard:
1. Correlation heatmap — VPH ↔ growth score = 0.51; search rank ↔ growth = −0.13
2. Growth score vs. search rank — r = −0.13; SEO rank does not predict growth
3. Top 12 SEO tags by frequency
4. Velocity barrier — 200 VPH empirical threshold for algorithmic lift
5. Comment leverage — comment density vs. search rank (r = −0.11)
6. Duration boxplot — `<5 min` and `20–30 min` have best median rank; `10–15 min` is the dead zone

---

## Metrics Coverage

| Metric | HN | Reddit | YouTube (scrape) | YouTube (API) |
|---|---|---|---|---|
| Volume over time | ✅ | ✅ | ✅ | ✅ |
| Sentiment | ✅ keyword | ✅ upvote ratio | ✅ title framing | ⚠️ top comment only |
| Engagement rate | — | ✅ | ✅ views/duration | ✅ |
| Likes | — | ✅ score | ⚠️ unreliable | ✅ |
| Dislikes | — | — | ❌ removed 2021 | ❌ removed 2021 |
| Saves | — | — | ❌ never public | ❌ never public |
| Comments | ✅ | ✅ | ⚠️ partial | ✅ |
| Channel data | — | ✅ subscribers | ✅ subscriber count | ✅ verified + video count |

---

## Dependencies

```bash
# Mac / Jupyter (HN + Reddit)
pip install requests pandas jupyter

# Colab (YouTube + visualizations)
pip install isodate google-api-python-client pandas matplotlib seaborn numpy scipy openpyxl
```
