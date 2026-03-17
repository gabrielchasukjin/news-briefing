---
name: news
description: Generate a curated news briefing as a visual HTML page
argument-hint: [topic]
---

# /news

Generate an editorially curated news briefing on a given topic (or general tech/AI news if no topic is provided) and output it as a self-contained HTML page that opens in the browser.

The output is a dark-themed editorial page with:
- A hero story (headline + image side-by-side)
- A 3-card grid with real images
- A feature story with editorial lede
- A sticky sidebar with live market data and trending companies
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
  "sources": ["claude-memory", "github-stars"]
}
```

On future runs, read this file first. If it exists and is less than 30 days old, skip Phase 0A-0C and use the cached profile. If it's stale or missing, regenerate it.

#### How the profile shapes search

The interest profile directly influences Phases 1 and 2:
- **Phase 1**: When fetching feeds, prioritize sources aligned with HIGH-weight interests. For a user interested in robotics + agents, fetch ArXiv cs.RO, HN, and TLDR AI rather than Product Hunt or TLDR Crypto.
- **Phase 2**: When curating the final 5, score stories higher if they match HIGH-weight interest tags. A story about a new robotics dataset should beat a generic Apple hardware launch for a user whose startup is in that space.
- **Explicit topic override**: If the user types `/news crypto`, the explicit topic OVERRIDES the profile for that run. But if they just type `/news` with no topic, the profile IS the topic.

### Phase 1: Source from High-Signal Feeds

Do NOT start with generic web searches. Instead, go directly to the places where smart, technical people actually find news. Use **WebFetch** and **WebSearch** in parallel to pull from these community-vetted sources.

#### Step 1A: Fetch community feeds (run ALL in parallel)

Use **WebFetch** on these URLs to get the current front pages:

**Always fetch (regardless of topic):**
- `https://news.ycombinator.com/front` — Hacker News front page. Extract story titles, source domains, point counts, comment counts. Stories with 300+ points are high-signal.
- `https://techmeme.com/` — Techmeme river. Algorithmically curated tech news. Extract headlines and source outlets.

**For AI/ML topics, also fetch:**
- `https://tldr.tech/ai` — TLDR AI newsletter. Latest curated AI stories.
- `https://arxiv.org/list/cs.AI/recent` — Recent ArXiv papers. Only pick papers with real-world implications, not pure theory.

**For crypto topics, also fetch:**
- `https://tldr.tech/crypto` — TLDR Crypto newsletter.
- `https://www.theblock.co/latest` — The Block latest.

**For general tech, also fetch:**
- `https://tldr.tech/` — TLDR Tech newsletter.
- `https://lobste.rs/` — Lobsters front page. More niche/technical than HN.

**For startups/products, also fetch:**
- `https://www.producthunt.com/` — Product Hunt front page.

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

From the scored pool, select exactly **5 stories** using these criteria (in priority order):

1. **Cross-source signal** — Stories that appear across multiple high-signal feeds (HN + Techmeme, HN + TLDR, etc.) get priority. This is the strongest quality signal.
2. **Community engagement** — High point counts, comment counts, and upvotes indicate the story resonated with a technical audience.
3. **Significance** — Does this actually matter? Will people still be talking about it next week? Prefer stories that change something (new capability, policy shift, market move) over incremental updates.
4. **Uniqueness** — Each story must cover a distinct angle. Drop duplicates and rehashes. If two stories cover the same event, pick the better source.
5. **Surprise factor** — Prefer stories that are genuinely interesting or counterintuitive over predictable press releases. The robot dogs story is more interesting than "Company X raises $Y million."
6. **Diversity** — Cover different facets (e.g., for tech: a deep technical post, a policy/business story, a tool/product, a cultural moment, an infrastructure play).

**What to avoid:**
- Generic fundraising announcements (unless the amount or implications are genuinely significant)
- Listicles and roundup posts
- SEO-driven blog posts with no original reporting
- Stories that are just repackaging a press release

Rank the final 5 stories 1-5 by significance. Story #1 gets hero treatment.

**For source linking:** When a story originated from a community feed (HN, Lobsters), link to the **original source article**, not the discussion thread. But note the community engagement (e.g., "654 points on HN") in the editorial lede if it adds context.

### Phase 3: Deep Content Fetch

For each of the 5 selected stories, use **WebFetch** in parallel to:

1. Fetch the article page and extract:
   - The `og:image` meta tag URL (for the real article image)
   - A 3-4 sentence summary of the article's key points

Keep the actual article URL for linking.

If an og:image cannot be found for a story, try fetching a related article on the same topic that has one.

### Phase 4: Market Data Fetch

Use **WebSearch** and **WebFetch** to pull real market data:

1. Search for current prices of indices relevant to the topic:
   - For tech/AI: S&P 500, NASDAQ, Bitcoin, VIX
   - For crypto: BTC, ETH, SOL, Total Market Cap
   - For other topics: pick 4 relevant market indicators

2. Search for stock prices of 5 companies mentioned in or related to the stories. Use **WebFetch** on `stockanalysis.com/stocks/{TICKER}/` to get real prices and day changes.

### Phase 5: Generate HTML

Write the HTML file using the **Write** tool to the user's Desktop at `~/Desktop/news-briefing.html`.

Use the template structure below. Replace ALL placeholder content with real data from Phases 1-4.

**Critical rules:**
- Every `<a href>` must point to the real article URL with `target="_blank"`
- Every `<img src>` must use the real og:image URL from Phase 3
- Every stock price, percentage, and index value must be real data from Phase 4
- Write editorial-quality summaries — not copy-pasted article text. Rewrite in your own voice with a point of view.
- The hero story (#1) gets a full lede paragraph. Cards (#2-4) get a headline + one-line summary. The feature story (#5) gets a lede paragraph.

**Nav tabs — generated from user profile:**
- The first tab is always **"For You"** (active by default). This is the main briefing.
- Generate 3-4 additional tabs from the user's HIGH-weight interest tags from Phase 0.
- Pick the most distinct, readable labels. Map interest tags to short display names:
  - `["robotics", "embodied-ai"]` → "Robotics"
  - `["agent-architectures", "langgraph"]` → "Agents & LLMs"
  - `["ml-research", "training-data"]` → "ML Research"
  - `["startups", "fundraising"]` → "Startups"
  - `["crypto", "defi"]` → "Crypto"
  - `["infrastructure", "deployment"]` → "Infrastructure"
- These tabs are cosmetic for now (no filtering behavior). They signal to the user that the briefing is personalized to their interests. The "For You" tab is active and shows all 5 stories.
- If no profile exists (Phase 0 was skipped or failed), fall back to: "For You", "AI", "Tech", "Markets"

### Phase 6: Open in Browser

Run this command with the **Bash** tool:
```bash
open ~/Desktop/news-briefing.html
```

Then tell the user the briefing is ready and what stories were selected.

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
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=Playfair+Display:ital,wght@0,400;0,700;0,800;1,400&family=Source+Serif+4:ital,wght@0,400;0,500;1,400&display=swap');

    * { margin: 0; padding: 0; box-sizing: border-box; }

    :root {
      --bg: #0f0f0f;
      --surface: #1a1a1a;
      --surface-2: #222;
      --text: #e8e5e0;
      --text-sub: #b0aba3;
      --text-muted: #666;
      --border: #2a2a2a;
      --accent: #d4a574;
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

    /* Nav */
    .nav {
      position: sticky; top: 0; z-index: 100;
      background: var(--bg);
      border-bottom: 1px solid var(--border);
      padding: 0 2rem;
    }
    .nav-inner {
      max-width: 1200px; margin: 0 auto;
      display: flex; align-items: center; justify-content: space-between;
      height: 52px;
    }
    .nav-brand {
      font-family: var(--serif);
      font-size: 1.1rem; font-weight: 700;
    }
    .nav-brand span { color: var(--accent); }
    .nav-links { display: flex; gap: 0; list-style: none; }
    .nav-links a {
      display: block; padding: 0.9rem 1rem;
      font-size: 0.78rem; font-weight: 500;
      color: var(--text-muted);
      transition: color 0.15s;
      border-bottom: 2px solid transparent;
    }
    .nav-links a:hover { color: var(--text); }
    .nav-links a.active { color: var(--text); border-bottom-color: var(--accent); }

    /* Layout */
    .layout {
      max-width: 1200px; margin: 0 auto;
      padding: 1.5rem 2rem;
      display: grid; grid-template-columns: 1fr 300px; gap: 2rem;
    }

    /* Hero */
    .hero {
      display: grid; grid-template-columns: 1fr 1.2fr; gap: 1.75rem;
      align-items: center;
      padding-bottom: 1.75rem;
      border-bottom: 1px solid var(--border);
      margin-bottom: 1.75rem;
    }
    .hero-text .time {
      font-size: 0.68rem; color: var(--text-muted);
      margin-bottom: 0.75rem;
      display: flex; align-items: center; gap: 0.4rem;
    }
    .hero-text .time::before {
      content: ''; width: 5px; height: 5px;
      background: var(--accent); border-radius: 50%;
    }
    .hero h1 {
      font-family: var(--serif);
      font-size: 1.85rem; font-weight: 700; line-height: 1.22;
      color: var(--text); margin-bottom: 0.85rem;
      transition: color 0.2s;
    }
    .hero a:hover h1 { color: var(--accent); }
    .hero .lede {
      font-family: var(--body-serif);
      font-size: 0.92rem; color: var(--text-sub); line-height: 1.75;
      margin-bottom: 0.75rem;
    }
    .hero .sources { font-size: 0.62rem; color: var(--text-muted); }
    .hero-img {
      width: 100%; aspect-ratio: 4/3;
      object-fit: cover; border-radius: 10px; display: block;
    }

    /* Card grid */
    .cards {
      display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 1.25rem;
      padding-bottom: 1.75rem;
      border-bottom: 1px solid var(--border);
      margin-bottom: 1.75rem;
    }
    .card { display: block; transition: transform 0.2s; }
    .card:hover { transform: translateY(-3px); }
    .card-img-wrap { border-radius: 10px; overflow: hidden; margin-bottom: 0.85rem; }
    .card img {
      width: 100%; aspect-ratio: 16/10;
      object-fit: cover; display: block;
      transition: transform 0.4s;
    }
    .card:hover img { transform: scale(1.04); }
    .card h3 {
      font-family: var(--serif);
      font-size: 1rem; font-weight: 700; line-height: 1.35;
      color: var(--text); margin-bottom: 0.5rem;
      transition: color 0.2s;
    }
    .card:hover h3 { color: var(--accent); }
    .card .card-meta { font-size: 0.6rem; color: var(--text-muted); }

    /* Feature story */
    .feature {
      display: grid; grid-template-columns: 1fr 1.4fr; gap: 1.5rem;
      align-items: center; padding-bottom: 1.75rem;
    }
    .feature-img {
      width: 100%; aspect-ratio: 4/3;
      object-fit: cover; border-radius: 10px; display: block;
    }
    .feature .time { font-size: 0.65rem; color: var(--text-muted); margin-bottom: 0.5rem; }
    .feature h2 {
      font-family: var(--serif);
      font-size: 1.4rem; font-weight: 700; line-height: 1.25;
      color: var(--text); margin-bottom: 0.65rem;
      transition: color 0.2s;
    }
    .feature a:hover h2 { color: var(--accent); }
    .feature .lede {
      font-family: var(--body-serif);
      font-size: 0.88rem; color: var(--text-sub); line-height: 1.7;
    }
    .feature .sources { font-size: 0.6rem; color: var(--text-muted); margin-top: 0.6rem; }

    /* Sidebar */
    .sidebar { position: sticky; top: 68px; align-self: start; }
    .sidebar-section {
      background: var(--surface); border-radius: 12px;
      border: 1px solid var(--border); padding: 1.25rem;
      margin-bottom: 1.25rem;
    }
    .sidebar-title { font-size: 0.78rem; font-weight: 700; margin-bottom: 1rem; }

    .market-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 0.75rem; }
    .market-card { background: var(--surface-2); border-radius: 8px; padding: 0.75rem; }
    .market-name { font-size: 0.65rem; font-weight: 600; color: var(--text-sub); }
    .market-price { font-size: 0.82rem; font-weight: 700; margin: 0.1rem 0; }
    .market-change { font-size: 0.65rem; font-weight: 600; }
    .market-change.up { color: var(--green); }
    .market-change.down { color: var(--red); }
    .market-spark { margin-top: 0.4rem; height: 24px; }
    .market-spark svg { width: 100%; height: 100%; }
    .spark-line { fill: none; stroke-width: 1.5; stroke-linecap: round; stroke-linejoin: round; }
    .spark-green { stroke: var(--green); }
    .spark-red { stroke: var(--red); }

    .trending-item {
      display: flex; align-items: center; justify-content: space-between;
      padding: 0.65rem 0; border-bottom: 1px solid var(--border);
    }
    .trending-item:last-child { border-bottom: none; }
    .trending-left { display: flex; align-items: center; gap: 0.65rem; }
    .trending-icon {
      width: 32px; height: 32px; border-radius: 8px;
      background: var(--surface-2);
      display: flex; align-items: center; justify-content: center;
      font-size: 0.7rem; font-weight: 700; color: var(--accent);
    }
    .trending-name { font-size: 0.75rem; font-weight: 600; }
    .trending-ticker { font-size: 0.6rem; color: var(--text-muted); }
    .trending-right { text-align: right; }
    .trending-price { font-size: 0.75rem; font-weight: 600; }
    .trending-pct { font-size: 0.62rem; font-weight: 600; }
    .trending-pct.up { color: var(--green); }
    .trending-pct.down { color: var(--red); }

    /* Colophon */
    .colophon {
      grid-column: 1 / -1;
      border-top: 1px solid var(--border);
      padding: 1.5rem 0 2rem;
      font-size: 0.6rem; color: var(--text-muted); text-align: center;
    }
    .colophon a { color: var(--accent); }

    /* Responsive */
    @media (max-width: 960px) {
      .layout { grid-template-columns: 1fr; }
      .sidebar { position: static; display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
      .hero { grid-template-columns: 1fr; }
      .hero-img { order: -1; }
      .cards { grid-template-columns: 1fr; }
      .feature { grid-template-columns: 1fr; }
      .feature-img { order: -1; }
    }
    @media (max-width: 600px) {
      .layout { padding: 1rem; }
      .nav { padding: 0 1rem; }
      .hero h1 { font-size: 1.45rem; }
      .sidebar { grid-template-columns: 1fr; }
      .nav-links { display: none; }
    }
  </style>
</head>
<body>

<!-- Nav -->
<div class="nav">
  <div class="nav-inner">
    <div class="nav-brand">The AI<span> Briefing</span></div>
    <ul class="nav-links">
      <a href="#" class="active">For You</a>
      <!-- Generate 3-4 tabs from user's HIGH-weight interest tags -->
      <!-- Example for a robotics/agents/ML user: -->
      <!-- <a href="#">Robotics</a> -->
      <!-- <a href="#">Agents & LLMs</a> -->
      <!-- <a href="#">ML Research</a> -->
      <!-- <a href="#">Startups</a> -->
      {{NAV_TABS}}
    </ul>
  </div>
</div>

<div class="layout">
  <main>

    <!-- HERO: Story #1 -->
    <div class="hero">
      <div class="hero-text">
        <div class="time">{{STORY_1_TIME}}</div>
        <a href="{{STORY_1_URL}}" target="_blank">
          <h1>{{STORY_1_HEADLINE}}</h1>
        </a>
        <p class="lede">{{STORY_1_LEDE}}</p>
        <div class="sources">{{STORY_1_SOURCE}} &middot; {{STORY_1_TIME_SHORT}}</div>
      </div>
      <a href="{{STORY_1_URL}}" target="_blank">
        <img class="hero-img" src="{{STORY_1_IMAGE}}" alt="{{STORY_1_ALT}}">
      </a>
    </div>

    <!-- CARDS: Stories #2, #3, #4 -->
    <div class="cards">
      <a class="card" href="{{STORY_2_URL}}" target="_blank">
        <div class="card-img-wrap">
          <img src="{{STORY_2_IMAGE}}" alt="{{STORY_2_ALT}}">
        </div>
        <h3>{{STORY_2_HEADLINE}}</h3>
        <div class="card-meta">{{STORY_2_SOURCE}} &middot; {{STORY_2_TIME}}</div>
      </a>
      <a class="card" href="{{STORY_3_URL}}" target="_blank">
        <div class="card-img-wrap">
          <img src="{{STORY_3_IMAGE}}" alt="{{STORY_3_ALT}}">
        </div>
        <h3>{{STORY_3_HEADLINE}}</h3>
        <div class="card-meta">{{STORY_3_SOURCE}} &middot; {{STORY_3_TIME}}</div>
      </a>
      <a class="card" href="{{STORY_4_URL}}" target="_blank">
        <div class="card-img-wrap">
          <img src="{{STORY_4_IMAGE}}" alt="{{STORY_4_ALT}}">
        </div>
        <h3>{{STORY_4_HEADLINE}}</h3>
        <div class="card-meta">{{STORY_4_SOURCE}} &middot; {{STORY_4_TIME}}</div>
      </a>
    </div>

    <!-- FEATURE: Story #5 -->
    <div class="feature">
      <a href="{{STORY_5_URL}}" target="_blank">
        <img class="feature-img" src="{{STORY_5_IMAGE}}" alt="{{STORY_5_ALT}}">
      </a>
      <div>
        <div class="time">{{STORY_5_TIME}}</div>
        <a href="{{STORY_5_URL}}" target="_blank">
          <h2>{{STORY_5_HEADLINE}}</h2>
        </a>
        <p class="lede">{{STORY_5_LEDE}}</p>
        <div class="sources">{{STORY_5_SOURCE}}</div>
      </div>
    </div>

  </main>

  <!-- Sidebar -->
  <aside class="sidebar">
    <div class="sidebar-section">
      <div class="sidebar-title">Market Outlook</div>
      <div class="market-grid">
        <!-- 4 market cards with real data. For each: -->
        <!-- Use spark-green class for positive, spark-red for negative -->
        <!-- Generate a plausible SVG polyline for the sparkline based on the direction -->
        {{MARKET_CARDS}}
      </div>
    </div>
    <div class="sidebar-section">
      <div class="sidebar-title">Trending Companies</div>
      <!-- 5 trending companies related to the stories with real stock data -->
      {{TRENDING_COMPANIES}}
    </div>
  </aside>

  <div class="colophon">
    Sources: {{ALL_SOURCES_WITH_LINKS}}<br>
    Editorially curated &middot; Built with Claude Code &middot; {{CURRENT_DATE}}
  </div>
</div>

</body>
</html>
```

## Important Notes

- **Always use the current date** when searching. Never hardcode dates.
- **Parallel tool calls**: Run WebSearch calls in parallel. Run WebFetch calls in parallel. This dramatically speeds up the process.
- **Editorial voice**: Rewrite summaries in your own voice. Never copy-paste article text verbatim. Have a point of view.
- **Market data**: If you can't fetch a real stock price, omit that company rather than making up data. Never fabricate financial data.
- **Sparkline SVGs**: Generate simple polyline paths. For positive change, trend upward (high Y values at start, low at end since SVG Y is inverted). For negative, trend downward.
- **Image fallbacks**: If an og:image URL returns a 404 or can't be found, try searching for an alternative article on the same story that has one.
- **File location**: Always write to `~/Desktop/news-briefing.html` unless the user specifies otherwise.
- **Nav brand**: Use "The AI Briefing" for tech/AI topics. For other topics, adapt the name (e.g., "The Crypto Briefing", "The Climate Briefing").
