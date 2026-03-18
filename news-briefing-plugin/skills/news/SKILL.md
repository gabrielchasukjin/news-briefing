---
name: news
description: Generate a curated news briefing as a visual HTML page
argument-hint: [topic]
---

# /news

Generate an editorially curated news briefing on a given topic (or general tech/AI news if no topic is provided) and output it as a self-contained HTML page that opens in the browser.

The output is a dark-themed editorial page inspired by Bloomberg and WSJ, with:
- A Bloomberg-style horizontal ticker bar with live market indices and story-relevant stock tickers
- A WSJ-style hero zone: dominant headline on the left, stacked secondary stories on the right
- Editorial labels on each story (NEW RELEASE, ANALYSIS, DEEP DIVE, GTC 2026, etc.)
- A lower section with two feature stories and a "Latest" sidebar feed
- Real links to source articles
- Real images pulled from article og:image tags

## Process

Follow these steps precisely. Do NOT skip steps or take shortcuts.

### Phase 0: Understand the User

Before searching for anything, build a profile of what this specific user cares about. This phase runs ONCE on first use and is cached for future runs. If a preferences file already exists, skip to Phase 0D.

#### Phase 0A: Read Claude Code Memory

Use the **Glob** tool to find all MEMORY.md files in the user's Claude Code memory:
```
~/.claude/projects/**/memory/MEMORY.md
```

Read each MEMORY.md file found. Extract:
- **Role** — Are they a student, founder, engineer, researcher, PM?
- **Domain** — What field are they in? (robotics, fintech, SaaS, ML research, etc.)
- **Tech stack** — What languages, frameworks, and tools do they use?
- **Active projects** — What are they building right now?
- **Interests** — What topics come up repeatedly across their projects?

Also glob for any individual memory files:
```
~/.claude/projects/**/memory/*.md
```
Scan these for additional context (project details, preferences, areas of expertise).

#### Phase 0B: Read GitHub Signals

Run these **Bash** commands in parallel:
```bash
gh api user --jq '{login, bio, company, blog}'
```
```bash
gh api user/starred --per-page=50 --jq '.[].full_name'
```

From the GitHub data, extract:
- **Bio/company** — professional identity
- **Starred repos** — what tools, frameworks, and projects they follow. Group stars by theme (e.g., "12 AI/ML repos, 8 Rust repos, 5 DevOps repos").

#### Phase 0C: Synthesize Interest Profile

Combine memory + GitHub signals into an **interest profile** — a ranked list of topic tags with weights. Example:

```
Interest Profile:
1. robotics, embodied-ai, physical-ai (HIGH — active startup in this space)
2. agent-architectures, langgraph, tool-use (HIGH — building coding agents)
3. ml-research, training-data, computer-vision (HIGH — ML coursework + startup)
4. startups, fundraising, developer-tools (MEDIUM — founder, building products)
5. infrastructure, data-pipelines, deployment (MEDIUM — building data platform)
6. typescript, python, next.js, fastapi (CONTEXT — tech stack signals)
```

The weights are:
- **HIGH** — actively working in this area (from projects and memory)
- **MEDIUM** — shows clear interest (from GitHub stars, coursework)
- **CONTEXT** — tech stack they use (useful for filtering developer tool stories)

#### Phase 0D: Save Preferences

Write the interest profile to `~/.news-briefing/profile.json` using the **Write** tool:

```json
{
  "generated_at": "ISO-8601-timestamp",
  "interests": [
    { "tags": ["robotics", "embodied-ai", "physical-ai"], "weight": "high", "reason": "active startup building robot training platform" },
    { "tags": ["agent-architectures", "langgraph", "tool-use"], "weight": "high", "reason": "building LangGraph-based coding agents" }
  ],
  "tech_stack": ["typescript", "python", "next.js", "react", "fastapi", "langgraph", "tailwind"],
  "role": "student-founder",
  "sources": ["claude-memory", "github-stars"],
  "domain_sources": []
}
```

The `domain_sources` field starts empty and is populated by Phase 0.5 after the first run.

On future runs, read this file first. If it exists and is less than 30 days old, skip Phase 0A-0C and use the cached profile. If `domain_sources` is already populated, also skip Phase 0.5.

#### How the profile shapes search

The interest profile directly influences Phase 0.5, Phase 1, and Phase 2:
- **Phase 0.5**: The profile's HIGH-weight interest tags are used to discover domain-specific high-signal sources dynamically.
- **Phase 1**: Fetch from both universal feeds AND the domain-specific sources discovered in Phase 0.5.
- **Phase 2**: Score stories higher if they match HIGH-weight interest tags.
- **Explicit topic override**: If the user types `/news crypto`, the explicit topic OVERRIDES the profile for that run. But if they just type `/news` with no topic, the profile IS the topic.

### Phase 0.5: Discover Domain-Specific Sources

This is the key step that makes the briefing genuinely personalized. Different professionals read different sources — a bioinformatician reads Nature and bioRxiv, not Product Hunt. An algo trader reads Matt Levine and FT, not Lobsters.

#### Step 0.5A: Search for high-signal sources

Using the HIGH-weight interest tags from the user profile, run **WebSearch** queries in parallel to discover what sources experts in those domains actually read:

For each HIGH-weight interest cluster, search:
- `"best news sources for {domain} professionals high signal"`
- `"where do {domain} experts get their news reddit"`
- `"top {domain} newsletters blogs publications 2025 2026"`

Examples of what these searches should surface:

| Domain | Expected high-signal sources |
|---|---|
| Bioinformatics / Biotech | Nature, bioRxiv, STAT News, Endpoints News, FierceBiotech, PubMed Trending |
| Algo Trading / Quant | Bloomberg, Financial Times, Matt Levine (Money Stuff), QuantNews, r/algotrading, Risk.net |
| Open Source / Law | EFF, Ars Technica Policy, TechDirt, LawFare Blog, SCOTUSblog |
| Robotics | IEEE Spectrum, ArXiv cs.RO, The Robot Report, Hackaday, IO Rodeo |
| Climate / Energy | Carbon Brief, Canary Media, Utility Dive, BloombergNEF |
| Cybersecurity | Krebs on Security, The Record, Risky Business, Dark Reading |
| Game Dev | Gamasutra, IGN, RPS, itch.io devlogs |

#### Step 0.5B: Validate and select sources

From the search results, pick **3-5 domain-specific sources** that:
1. Have a fetchable front page or recent articles list (not paywalled RSS-only)
2. Are recognized by the professional community (appear in multiple search results)
3. Publish frequently enough to have current content

#### Step 0.5C: Cache the sources

Add the discovered sources to the user's `~/.news-briefing/profile.json` under a new `domain_sources` field:

```json
{
  "domain_sources": [
    { "url": "https://www.nature.com/news", "name": "Nature News", "domain": "bioinformatics" },
    { "url": "https://www.biorxiv.org/", "name": "bioRxiv", "domain": "bioinformatics" },
    { "url": "https://www.statnews.com/", "name": "STAT News", "domain": "biotech" }
  ]
}
```

On future runs, if `domain_sources` already exists in the cached profile, skip this phase and use the cached sources directly.

### Phase 1: Source from High-Signal Feeds

Fetch from TWO categories of sources in parallel: **universal feeds** (tech community at large) and **domain-specific feeds** (from Phase 0.5).

#### Step 1A: Fetch universal feeds (run ALL in parallel)

These are always fetched regardless of topic or user profile:

- `https://news.ycombinator.com/front` — Hacker News front page. Extract story titles, source domains, point counts, comment counts. Stories with 300+ points are high-signal.
- `https://techmeme.com/` — Techmeme river. Algorithmically curated tech news. Extract headlines and source outlets.

#### Step 1A.5: Fetch domain-specific feeds (run ALL in parallel with Step 1A)

Fetch the front pages of ALL sources from Phase 0.5's `domain_sources` list. For each, extract story titles, URLs, and any engagement signals.

Additionally, based on the user's interest profile, fetch relevant specialty feeds:

**For AI/ML interests:**
- `https://tldr.tech/ai` — TLDR AI newsletter
- `https://arxiv.org/list/cs.AI/recent` — Recent ArXiv AI papers

**For crypto interests:**
- `https://tldr.tech/crypto` — TLDR Crypto
- `https://www.theblock.co/latest` — The Block

**For general tech interests:**
- `https://tldr.tech/` — TLDR Tech
- `https://lobste.rs/` — Lobsters

**For startup/product interests:**
- `https://www.producthunt.com/` — Product Hunt

For each feed, extract story titles, URLs, and any engagement signals (points, comments, upvotes).

#### Step 1B: Cross-reference with web search (run in parallel with 1A)

Run 2 **WebSearch** queries to catch anything the feeds might miss:
- `"most important {topic} news {current month} {current year}"`
- `"{topic} breaking news today {current date}"`

If no topic was provided, default to tech and AI.

#### Step 1C: Merge and score

Combine all results from 1A and 1B into a single pool. You should have 30-50 raw stories.

**Score each story higher if it appears in multiple sources.** A story on both HN (300+ pts) and Techmeme is much higher signal than one that only appeared in a generic Google search result.

### Phase 2: Editorial Curation

From the scored pool, select exactly **5 stories** for the main layout, plus **~6 additional stories** for the "Latest" sidebar feed.

**Main 5 stories** — use these criteria (in priority order):

1. **Cross-source signal** — Stories that appear across multiple high-signal feeds (HN + Techmeme, HN + TLDR, etc.) get priority. This is the strongest quality signal.
2. **Community engagement** — High point counts, comment counts, and upvotes indicate the story resonated with a technical audience.
3. **Significance** — Does this actually matter? Will people still be talking about it next week? Prefer stories that change something (new capability, policy shift, market move) over incremental updates.
4. **Uniqueness** — Each story must cover a distinct angle. Drop duplicates and rehashes. If two stories cover the same event, pick the better source.
5. **Surprise factor** — Prefer stories that are genuinely interesting or counterintuitive over predictable press releases.
6. **Diversity** — Cover different facets (e.g., for tech: a deep technical post, a policy/business story, a tool/product, a cultural moment, an infrastructure play).

**"Latest" sidebar stories** — pick 6 additional stories from the pool that didn't make the main 5 but are still interesting. These only need a headline, source, and approximate time. Include HN point counts or comment counts where available as social proof.

**What to avoid:**
- Generic fundraising announcements (unless the amount or implications are genuinely significant)
- Listicles and roundup posts
- SEO-driven blog posts with no original reporting
- Stories that are just repackaging a press release

Rank the main 5 stories 1-5 by significance. Story #1 gets hero treatment. Stories #2-3 go in the hero sidebar. Stories #4-5 get the lower feature treatment.

**Editorial labels** — assign each main story a label that tells readers what kind of content it is:
- `NEW RELEASE` — product or model launches
- `ANALYSIS` — deeper takes with implications
- `DEEP DIVE` — technical or investigative pieces
- `EXCLUSIVE` — scoops or first reports
- `POLICY` — regulatory, legal, or government stories
- Or a relevant event name like `GTC 2026`, `WWDC`, etc.
Do NOT use "BREAKING" unless something literally just happened in the last hour.

**For source linking:** When a story originated from a community feed (HN, Lobsters), link to the **original source article**, not the discussion thread. But note the community engagement (e.g., "654 points on HN") in the editorial lede if it adds context.

### Phase 3: Deep Content Fetch

For each of the 5 main stories, use **WebFetch** in parallel to:

1. Fetch the article page and extract:
   - The `og:image` meta tag URL (for the real article image)
   - A 3-4 sentence summary of the article's key points

Keep the actual article URL for linking.

If an og:image cannot be found for a story, try fetching a related article on the same topic that has one.

### Phase 4: Market Data Fetch

This phase has two parts: **index data** and **dynamic ticker extraction**.

#### Step 4A: Extract tickers from stories

After selecting the 5 main stories, identify **every publicly traded company** mentioned in or directly related to the stories. For each, determine its stock ticker symbol.

Examples:
- Story about OpenAI → no public ticker (skip), but Microsoft (MSFT) is a major investor → include MSFT
- Story about Google Personal Intelligence → GOOG
- Story about Nvidia GTC → NVDA
- Story about Kalshi → no public ticker, but could include related companies

Collect up to **8 tickers** from the stories. If fewer than 4 are found, pad with major tech names (AAPL, MSFT, GOOG, META, NVDA, AMZN, TSLA) that are contextually relevant.

#### Step 4B: Fetch exact index values

Use **WebFetch** in parallel on Google Finance pages to get **exact, current index values**:

- `https://www.google.com/finance/quote/.INX:INDEXSP` — S&P 500 (extract exact value like `6,716.09` and day change)
- `https://www.google.com/finance/quote/.IXIC:INDEXNASDAQ` — NASDAQ Composite (extract exact value and day change)

Also fetch **WebFetch** on these for Bitcoin and other relevant indicators:
- `https://www.google.com/finance/quote/BTC-USD` — Bitcoin price

If Google Finance pages fail or don't return data, fall back to **WebSearch** for `"S&P 500 price today"` etc.

#### Step 4C: Fetch individual stock prices

For each ticker from Step 4A, use **WebFetch** in parallel on:
- `https://www.google.com/finance/quote/{TICKER}:NASDAQ` or
- `https://www.google.com/finance/quote/{TICKER}:NYSE`

Extract: current price, day change amount, day change percentage.

If Google Finance fails for a ticker, fall back to `https://stockanalysis.com/stocks/{TICKER}/`.

**Critical rules:**
- Never fabricate financial data. If you can't fetch a real price, omit that ticker.
- Use exact values (e.g., `6,716.09` not `Flat`). Show the actual index number.
- Include both the value and the percentage change with colored up/down badges.

### Phase 5: Generate HTML

Write the HTML file using the **Write** tool to the user's Desktop at `~/Desktop/news-briefing.html`.

Use the template structure below. Replace ALL placeholder content with real data from Phases 1-4.

**Critical rules:**
- Every `<a href>` must point to the real article URL with `target="_blank"`
- Every `<img src>` must use the real og:image URL from Phase 3
- Every stock price, percentage, and index value must be real data from Phase 4
- Write editorial-quality summaries — not copy-pasted article text. Rewrite in your own voice with a point of view.
- Story #1 gets a full lede paragraph in the hero. Stories #2-3 get headline + short lede in the hero sidebar. Stories #4-5 get the lower feature treatment with images + ledes.

**Nav:**
- Only show a single **"For You"** tab (active). Do NOT generate additional tabs. Keep the nav clean and minimal.
- Show the current date on the right side of the nav.

**Ticker bar:**
- Horizontal bar above the nav with all market data.
- Show 3-4 index values (S&P 500, Nasdaq, Bitcoin, plus one more relevant to the topic) with exact values and colored change badges.
- Then show the dynamically extracted stock tickers from the stories.
- Each item shows: label, exact value, and colored percentage change badge.

**Sidebar:**
- The "Latest" sidebar shows ~6 additional headlines from the story pool with relative timestamps (e.g., "2h ago", "4h ago"), headline text, and source attribution with engagement signals.

### Phase 6: Open in Browser

Run this command with the **Bash** tool:
```bash
open ~/Desktop/news-briefing.html
```

Then tell the user the briefing is ready and what stories were selected.

### Phase 7: Archive and Index

After writing the main briefing file, archive it and update the index page.

#### Step 7A: Archive the briefing

Run these **Bash** commands:
```bash
mkdir -p ~/Desktop/news-briefings
```

Then copy the briefing to the archive with a dated filename:
```bash
cp ~/Desktop/news-briefing.html ~/Desktop/news-briefings/YYYY-MM-DD-{topic}.html
```

Use the current date and a slugified version of the topic (e.g., `2026-03-17-langchain.html`, `2026-03-17-top-universities.html`). If no topic was provided, use `for-you` as the slug.

#### Step 7B: Generate the archive index page

Use **Bash** to list all `.html` files in `~/Desktop/news-briefings/` (excluding `index.html`), sorted by date descending. Then use the **Write** tool to generate `~/Desktop/news-briefings/index.html` — a dark-themed index page that lists all past briefings with:
- The date
- The topic (extracted from the filename slug)
- A clickable link to open that briefing

Use this template for the index page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>News Briefing Archive</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=Playfair+Display:wght@700;800;900&display=swap');
    * { margin: 0; padding: 0; box-sizing: border-box; }
    :root {
      --bg: #0a0a0a; --surface: #141414; --text: #ede9e3;
      --text-sub: #a8a29e; --text-muted: #57534e; --border: #1f1f1f;
      --accent: #d4a574; --sans: 'Inter', sans-serif; --serif: 'Playfair Display', Georgia, serif;
    }
    body { font-family: var(--sans); background: var(--bg); color: var(--text); -webkit-font-smoothing: antialiased; }
    a { color: inherit; text-decoration: none; }
    .container { max-width: 700px; margin: 0 auto; padding: 3rem 2rem; }
    .brand { font-family: var(--serif); font-size: 1.5rem; font-weight: 800; margin-bottom: 0.5rem; }
    .brand span { color: var(--accent); }
    .subtitle { font-size: 0.75rem; color: var(--text-muted); margin-bottom: 2.5rem; }
    .briefing-item {
      display: flex; align-items: center; justify-content: space-between;
      padding: 1rem 0; border-bottom: 1px solid var(--border);
      transition: opacity 0.15s;
    }
    .briefing-item:hover { opacity: 0.8; }
    .briefing-date { font-size: 0.7rem; font-weight: 700; color: var(--accent); min-width: 100px; }
    .briefing-topic { font-size: 0.9rem; font-weight: 600; flex: 1; text-transform: capitalize; }
    .briefing-arrow { font-size: 0.75rem; color: var(--text-muted); }
  </style>
</head>
<body>
<div class="container">
  <div class="brand">Briefing<span> Archive</span></div>
  <div class="subtitle">All past news briefings, newest first.</div>
  <!-- One .briefing-item per archived file -->
  {{ARCHIVE_ITEMS}}
</div>
</body>
</html>
```

Each archive item should be:
```html
<a class="briefing-item" href="./YYYY-MM-DD-topic.html">
  <span class="briefing-date">Mar 17, 2026</span>
  <span class="briefing-topic">Topic Name</span>
  <span class="briefing-arrow">&rarr;</span>
</a>
```

Scan the archive directory for all `.html` files (excluding `index.html`) to build the list. Sort by filename descending (newest first).

---

## HTML Template

Use this exact structure. Replace all `{{placeholders}}` with real content.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{BRIEFING_TITLE}}</title>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=Playfair+Display:ital,wght@0,400;0,700;0,800;0,900;1,400&family=Source+Serif+4:ital,wght@0,400;0,500;1,400&display=swap');

    * { margin: 0; padding: 0; box-sizing: border-box; }

    :root {
      --bg: #0a0a0a;
      --surface: #141414;
      --surface-2: #1c1c1c;
      --text: #ede9e3;
      --text-sub: #a8a29e;
      --text-muted: #57534e;
      --border: #1f1f1f;
      --border-light: #292524;
      --accent: #d4a574;
      --accent-dim: rgba(212,165,116,0.12);
      --green: #4ade80;
      --red: #f87171;
      --sans: 'Inter', -apple-system, sans-serif;
      --serif: 'Playfair Display', Georgia, serif;
      --body-serif: 'Source Serif 4', Georgia, serif;
    }

    body {
      font-family: var(--sans);
      background: var(--bg);
      color: var(--text);
      -webkit-font-smoothing: antialiased;
    }
    a { color: inherit; text-decoration: none; }

    /* ── Ticker Bar (Bloomberg-style) ── */
    .ticker-bar {
      background: var(--surface);
      border-bottom: 1px solid var(--border);
      overflow: hidden;
      white-space: nowrap;
    }
    .ticker-inner {
      display: flex;
      align-items: center;
      height: 36px;
      max-width: 1300px;
      margin: 0 auto;
      padding: 0 2rem;
      gap: 0;
      overflow-x: auto;
      scrollbar-width: none;
    }
    .ticker-inner::-webkit-scrollbar { display: none; }
    .ticker-item {
      display: inline-flex;
      align-items: center;
      gap: 0.5rem;
      padding: 0 1.25rem;
      border-right: 1px solid var(--border);
      flex-shrink: 0;
    }
    .ticker-item:last-child { border-right: none; }
    .ticker-label {
      font-size: 0.65rem;
      font-weight: 700;
      color: var(--text-muted);
      letter-spacing: 0.03em;
    }
    .ticker-value {
      font-size: 0.72rem;
      font-weight: 700;
      color: var(--text);
      font-variant-numeric: tabular-nums;
    }
    .ticker-change {
      font-size: 0.65rem;
      font-weight: 700;
      padding: 1px 6px;
      border-radius: 3px;
    }
    .ticker-change.up { color: var(--green); background: rgba(74,222,128,0.1); }
    .ticker-change.down { color: var(--red); background: rgba(248,113,113,0.1); }

    /* ── Nav ── */
    .nav {
      position: sticky; top: 0; z-index: 100;
      background: var(--bg);
      border-bottom: 1px solid var(--border);
      padding: 0 2rem;
    }
    .nav-inner {
      max-width: 1300px; margin: 0 auto;
      display: flex; align-items: center; justify-content: space-between;
      height: 56px;
    }
    .nav-brand {
      font-family: var(--serif);
      font-size: 1.35rem; font-weight: 800;
      letter-spacing: -0.01em;
    }
    .nav-brand span { color: var(--accent); }
    .nav-right {
      display: flex; align-items: center; gap: 1.5rem;
    }
    .nav-tab {
      font-size: 0.72rem; font-weight: 600;
      color: var(--text);
      padding: 0.5rem 0;
      border-bottom: 2px solid var(--accent);
    }
    .nav-date {
      font-size: 0.68rem;
      color: var(--text-muted);
      font-weight: 500;
    }

    /* ── Layout ── */
    .layout {
      max-width: 1300px; margin: 0 auto;
      padding: 0 2rem;
    }

    /* ── Hero Zone (WSJ-style: big headline + side stories) ── */
    .hero-zone {
      display: grid;
      grid-template-columns: 1fr 380px;
      gap: 0;
      border-bottom: 1px solid var(--border-light);
      padding: 2rem 0;
    }
    .hero-main {
      padding-right: 2.5rem;
      border-right: 1px solid var(--border-light);
    }

    /* Editorial labels */
    .label {
      display: inline-block;
      font-size: 0.6rem;
      font-weight: 800;
      letter-spacing: 0.08em;
      text-transform: uppercase;
      padding: 3px 8px;
      border-radius: 3px;
      margin-bottom: 1rem;
    }
    .label-breaking { color: var(--red); border: 1px solid var(--red); }
    .label-analysis { color: var(--accent); border: 1px solid var(--accent); }
    .label-deepdive { color: #818cf8; border: 1px solid #818cf8; }
    .label-exclusive { color: var(--green); border: 1px solid var(--green); }
    .label-policy { color: #f59e0b; border: 1px solid #f59e0b; }

    .hero-main h1 {
      font-family: var(--serif);
      font-size: 2.6rem;
      font-weight: 900;
      line-height: 1.12;
      color: var(--text);
      margin-bottom: 1rem;
      letter-spacing: -0.02em;
      transition: color 0.2s;
    }
    .hero-main a:hover h1 { color: var(--accent); }

    .hero-main .hero-img {
      width: 100%;
      aspect-ratio: 16/9;
      object-fit: cover;
      border-radius: 6px;
      margin-bottom: 1rem;
    }

    .hero-main .lede {
      font-family: var(--body-serif);
      font-size: 1rem;
      color: var(--text-sub);
      line-height: 1.7;
      margin-bottom: 0.75rem;
      max-width: 600px;
    }
    .hero-main .meta {
      font-size: 0.6rem;
      color: var(--text-muted);
    }

    /* ── Hero Sidebar (secondary stories stacked) ── */
    .hero-sidebar {
      padding-left: 2rem;
      display: flex;
      flex-direction: column;
    }
    .hero-side-story {
      display: block;
      padding: 1.25rem 0;
      border-bottom: 1px solid var(--border-light);
      transition: background 0.15s;
    }
    .hero-side-story:first-child { padding-top: 0; }
    .hero-side-story:last-child { border-bottom: none; }
    .hero-side-story h3 {
      font-family: var(--serif);
      font-size: 1.05rem;
      font-weight: 700;
      line-height: 1.3;
      color: var(--text);
      margin-bottom: 0.5rem;
      transition: color 0.2s;
    }
    .hero-side-story:hover h3 { color: var(--accent); }
    .hero-side-story .side-lede {
      font-family: var(--body-serif);
      font-size: 0.82rem;
      color: var(--text-sub);
      line-height: 1.6;
      margin-bottom: 0.4rem;
    }
    .hero-side-story .meta {
      font-size: 0.58rem;
      color: var(--text-muted);
    }

    /* ── Lower Grid (3 col: feature + story + latest sidebar) ── */
    .lower-section {
      display: grid;
      grid-template-columns: 1fr 1fr 300px;
      gap: 0;
      padding: 1.75rem 0;
      border-bottom: 1px solid var(--border-light);
    }

    .lower-feature {
      padding-right: 2rem;
      border-right: 1px solid var(--border-light);
    }
    .lower-feature .feature-img {
      width: 100%;
      aspect-ratio: 16/10;
      object-fit: cover;
      border-radius: 5px;
      margin-bottom: 1rem;
    }
    .lower-feature h2 {
      font-family: var(--serif);
      font-size: 1.35rem;
      font-weight: 700;
      line-height: 1.25;
      color: var(--text);
      margin-bottom: 0.6rem;
      transition: color 0.2s;
    }
    .lower-feature a:hover h2 { color: var(--accent); }
    .lower-feature .lede {
      font-family: var(--body-serif);
      font-size: 0.85rem;
      color: var(--text-sub);
      line-height: 1.65;
      margin-bottom: 0.5rem;
    }
    .lower-feature .meta {
      font-size: 0.58rem;
      color: var(--text-muted);
    }

    .lower-mid {
      padding: 0 2rem;
      border-right: 1px solid var(--border-light);
    }
    .lower-mid .mid-img {
      width: 100%;
      aspect-ratio: 16/10;
      object-fit: cover;
      border-radius: 5px;
      margin-bottom: 1rem;
    }
    .lower-mid h2 {
      font-family: var(--serif);
      font-size: 1.35rem;
      font-weight: 700;
      line-height: 1.25;
      color: var(--text);
      margin-bottom: 0.6rem;
      transition: color 0.2s;
    }
    .lower-mid a:hover h2 { color: var(--accent); }
    .lower-mid .lede {
      font-family: var(--body-serif);
      font-size: 0.85rem;
      color: var(--text-sub);
      line-height: 1.65;
      margin-bottom: 0.5rem;
    }
    .lower-mid .meta {
      font-size: 0.58rem;
      color: var(--text-muted);
    }

    /* ── Sidebar: Latest ── */
    .lower-sidebar {
      padding-left: 1.5rem;
    }
    .sidebar-title {
      font-size: 0.72rem;
      font-weight: 800;
      letter-spacing: 0.04em;
      text-transform: uppercase;
      color: var(--accent);
      margin-bottom: 1rem;
    }

    .latest-item {
      display: block;
      padding: 0.7rem 0;
      border-bottom: 1px solid var(--border);
      transition: opacity 0.15s;
    }
    .latest-item:last-child { border-bottom: none; }
    .latest-item:hover { opacity: 0.8; }
    .latest-time {
      font-size: 0.58rem;
      font-weight: 700;
      color: var(--accent);
      margin-bottom: 0.3rem;
    }
    .latest-headline {
      font-family: var(--sans);
      font-size: 0.78rem;
      font-weight: 600;
      line-height: 1.4;
      color: var(--text);
    }
    .latest-source {
      font-size: 0.55rem;
      color: var(--text-muted);
      margin-top: 0.2rem;
    }

    /* ── Colophon ── */
    .colophon {
      padding: 2rem 0 2.5rem;
      font-size: 0.58rem;
      color: var(--text-muted);
      text-align: center;
      line-height: 1.8;
    }
    .colophon a { color: var(--accent); }

    /* ── Responsive ── */
    @media (max-width: 1060px) {
      .hero-zone { grid-template-columns: 1fr; }
      .hero-main { padding-right: 0; border-right: none; border-bottom: 1px solid var(--border-light); padding-bottom: 1.5rem; }
      .hero-sidebar { padding-left: 0; padding-top: 1.5rem; }
      .lower-section { grid-template-columns: 1fr; }
      .lower-feature { padding-right: 0; border-right: none; border-bottom: 1px solid var(--border-light); padding-bottom: 1.5rem; }
      .lower-mid { padding: 1.5rem 0; border-right: none; border-bottom: 1px solid var(--border-light); }
      .lower-sidebar { padding-left: 0; padding-top: 1.5rem; }
    }
    @media (max-width: 600px) {
      .layout { padding: 0 1rem; }
      .nav { padding: 0 1rem; }
      .ticker-inner { padding: 0 1rem; }
      .hero-main h1 { font-size: 1.8rem; }
      .nav-date { display: none; }
    }
  </style>
</head>
<body>

<!-- ── Ticker Bar ── -->
<div class="ticker-bar">
  <div class="ticker-inner">
    <!-- Index values first (3-4 indices with EXACT values) -->
    {{INDEX_TICKER_ITEMS}}
    <!-- Then story-relevant stock tickers (dynamically extracted) -->
    {{STOCK_TICKER_ITEMS}}
  </div>
</div>

<!-- ── Nav ── -->
<div class="nav">
  <div class="nav-inner">
    <div class="nav-brand">The AI<span> Briefing</span></div>
    <div class="nav-right">
      <span class="nav-tab">For You</span>
      <span class="nav-date">{{FULL_DATE}}</span>
    </div>
  </div>
</div>

<div class="layout">

  <!-- ── Hero Zone ── -->
  <div class="hero-zone">
    <div class="hero-main">
      <div class="label {{STORY_1_LABEL_CLASS}}">{{STORY_1_LABEL}}</div>
      <a href="{{STORY_1_URL}}" target="_blank">
        <h1>{{STORY_1_HEADLINE}}</h1>
      </a>
      <a href="{{STORY_1_URL}}" target="_blank">
        <img class="hero-img" src="{{STORY_1_IMAGE}}" alt="{{STORY_1_ALT}}">
      </a>
      <p class="lede">{{STORY_1_LEDE}}</p>
      <div class="meta">{{STORY_1_SOURCES}}</div>
    </div>

    <div class="hero-sidebar">
      <!-- Story #2 -->
      <a class="hero-side-story" href="{{STORY_2_URL}}" target="_blank">
        <div class="label {{STORY_2_LABEL_CLASS}}">{{STORY_2_LABEL}}</div>
        <h3>{{STORY_2_HEADLINE}}</h3>
        <p class="side-lede">{{STORY_2_SHORT_LEDE}}</p>
        <div class="meta">{{STORY_2_SOURCES}}</div>
      </a>
      <!-- Story #3 -->
      <a class="hero-side-story" href="{{STORY_3_URL}}" target="_blank">
        <h3>{{STORY_3_HEADLINE}}</h3>
        <p class="side-lede">{{STORY_3_SHORT_LEDE}}</p>
        <div class="meta">{{STORY_3_SOURCES}}</div>
      </a>
      <!-- Additional sidebar headlines from remaining pool (optional) -->
      {{HERO_SIDEBAR_EXTRAS}}
    </div>
  </div>

  <!-- ── Lower Section ── -->
  <div class="lower-section">
    <!-- Story #4 -->
    <div class="lower-feature">
      <div class="label {{STORY_4_LABEL_CLASS}}">{{STORY_4_LABEL}}</div>
      <a href="{{STORY_4_URL}}" target="_blank">
        <img class="feature-img" src="{{STORY_4_IMAGE}}" alt="{{STORY_4_ALT}}">
      </a>
      <a href="{{STORY_4_URL}}" target="_blank">
        <h2>{{STORY_4_HEADLINE}}</h2>
      </a>
      <p class="lede">{{STORY_4_LEDE}}</p>
      <div class="meta">{{STORY_4_SOURCES}}</div>
    </div>

    <!-- Story #5 -->
    <div class="lower-mid">
      <div class="label {{STORY_5_LABEL_CLASS}}">{{STORY_5_LABEL}}</div>
      <a href="{{STORY_5_URL}}" target="_blank">
        <img class="mid-img" src="{{STORY_5_IMAGE}}" alt="{{STORY_5_ALT}}">
      </a>
      <a href="{{STORY_5_URL}}" target="_blank">
        <h2>{{STORY_5_HEADLINE}}</h2>
      </a>
      <p class="lede">{{STORY_5_LEDE}}</p>
      <div class="meta">{{STORY_5_SOURCES}}</div>
    </div>

    <!-- Latest sidebar -->
    <div class="lower-sidebar">
      <div class="sidebar-title">Latest</div>
      <!-- ~6 additional stories from the pool -->
      {{LATEST_ITEMS}}
    </div>
  </div>

  <div class="colophon">
    Sources: {{ALL_SOURCES_WITH_LINKS}}<br>
    Editorially curated &middot; Built with Claude Code &middot; {{CURRENT_DATE}}
  </div>

</div>

</body>
</html>
```

### Ticker item format

Each ticker item in the bar should follow this HTML structure:

```html
<div class="ticker-item">
  <span class="ticker-label">S&P 500</span>
  <span class="ticker-value">6,716.09</span>
  <span class="ticker-change up">+0.25%</span>
</div>
```

Use class `up` for positive changes (green), `down` for negative (red).

### Latest item format

Each latest sidebar item should follow this structure:

```html
<a class="latest-item" href="{{URL}}" target="_blank">
  <div class="latest-time">{{RELATIVE_TIME}}</div>
  <div class="latest-headline">{{HEADLINE}}</div>
  <div class="latest-source">{{SOURCE}} · {{ENGAGEMENT}}</div>
</a>
```

### Label class mapping

| Label text | CSS class |
|---|---|
| BREAKING | `label-breaking` |
| NEW RELEASE | `label-exclusive` |
| ANALYSIS | `label-analysis` |
| DEEP DIVE | `label-deepdive` |
| EXCLUSIVE | `label-exclusive` |
| POLICY | `label-policy` |
| Event names (GTC 2026, etc.) | `label-exclusive` |

## Important Notes

- **Always use the current date** when searching. Never hardcode dates.
- **Parallel tool calls**: Run WebSearch calls in parallel. Run WebFetch calls in parallel. This dramatically speeds up the process.
- **Editorial voice**: Rewrite summaries in your own voice. Never copy-paste article text verbatim. Have a point of view.
- **Market data**: If you can't fetch a real stock price, omit that ticker rather than making up data. Never fabricate financial data. Always show exact index values (e.g., `6,716.09`) not vague descriptions like "Flat".
- **Dynamic tickers**: Extract tickers from the stories themselves. The ticker bar should feel contextually relevant to the briefing content, not generic.
- **Google Finance first**: Always try Google Finance URLs first for market data. Fall back to stockanalysis.com or WebSearch only if Google Finance fails.
- **Image fallbacks**: If an og:image URL returns a 404 or can't be found, try searching for an alternative article on the same story that has one.
- **File location**: Always write to `~/Desktop/news-briefing.html` unless the user specifies otherwise.
- **Nav brand**: Use "The AI Briefing" for tech/AI topics. For other topics, adapt the name (e.g., "The Crypto Briefing", "The Climate Briefing").
