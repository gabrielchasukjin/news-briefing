---
name: news
description: Generate a curated news briefing as a visual HTML page
argument-hint: [topic]
---

# /news

Generate an editorially curated news briefing on a given topic (or general tech/AI news if no topic is provided) and output it as a self-contained HTML page that opens in the browser.

The output is an editorial page inspired by Bloomberg and WSJ, with a Bloomberg-style light/dark toggle:
- A Bloomberg-style horizontal ticker bar with live market indices and story-relevant stock tickers
- A WSJ-style hero zone: dominant headline on the left, stacked secondary stories on the right
- Editorial labels on each story (NEW RELEASE, ANALYSIS, DEEP DIVE, GTC 2026, etc.)
- A lower section with two feature stories and a "Latest" sidebar feed
- A dedicated **"From X/Twitter"** section showing links shared by high-signal accounts with tweet context and attribution
- A dedicated **"Trending on GitHub"** section with 4-column repo cards showing stars, language, and cross-signal notes
- A Bloomberg-style light theme toggle (white background, clean typography, bold section headers)
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

### Phase 0.6: Discover High-Signal Twitter/X Accounts

This phase dynamically discovers which Twitter/X accounts are worth monitoring for the user's interests. Like Phase 0.5, it runs ONCE and caches results.

If `twitter_accounts` already exists in `~/.news-briefing/profile.json` and is non-empty, skip this phase entirely.

#### Step 0.6A: Search for high-signal accounts

For each HIGH-weight interest cluster from the user profile, run **WebSearch** queries in parallel:

- `"most influential {domain} people on Twitter X to follow for news"`
- `"best {domain} Twitter accounts for breaking news 2025 2026"`
- `"who to follow on X for {domain} insights" site:reddit.com`

Also run these cross-cutting queries:
- `"AI researchers to follow on Twitter X" breaking news papers`
- `"tech founders active on Twitter X" that share links`

#### Step 0.6B: Extract from GitHub profiles

Run this **Bash** command to get Twitter handles from the user's starred repos' authors:

```bash
gh api user/starred --per-page=30 --jq '.[].owner.login' | sort -u | head -20
```

Then for the top 10 most relevant owners (matching user interests), check their GitHub profiles for Twitter/X links:

```bash
gh api users/{username} --jq '{login, twitter_username, bio}'
```

This surfaces accounts that are both technically credible (they build things the user cares about) AND active on Twitter.

#### Step 0.6C: Score and select accounts

From all discovered accounts, select **15-25 accounts** using these criteria:

1. **Appears in multiple search results** — community consensus that this person is high-signal
2. **Has a GitHub profile with real projects** — not just a commentator, actually builds things
3. **Known for sharing links/papers early** — not just hot takes, but surfaces original content
4. **Domain match** — covers topics in the user's HIGH-weight interest tags

Organize accounts into clusters matching the user's interest tags:

```json
{
  "twitter_accounts": [
    {
      "handle": "kaboragimov",
      "name": "Kabo Ragimov",
      "domain": ["ai", "ml-research"],
      "reason": "Curates AI research papers, often first to share arxiv papers",
      "source": "web-search"
    },
    {
      "handle": "jimfan",
      "name": "Jim Fan",
      "domain": ["robotics", "embodied-ai"],
      "reason": "NVIDIA senior research scientist, shares robotics/embodied AI research",
      "source": "web-search + github"
    }
  ]
}
```

#### Step 0.6D: Cache the accounts

Save the `twitter_accounts` array to `~/.news-briefing/profile.json`.

On future runs, if this field exists and has entries, skip Phase 0.6 entirely and use the cached list. The list should be refreshed if the user's interest profile changes significantly (new HIGH-weight tags added).

### Phase 1: Source from High-Signal Feeds (Six Independent Sub-Agents)

Launch **six Agent sub-agents in parallel**. Each agent independently discovers and evaluates stories, then returns a ranked list. This architecture ensures:
- Each agent explores different source types without overlap
- No single failure mode kills the pipeline
- Each agent applies its own quality filter before returning results

All four agents receive the **topic** (if provided) and the **user interest profile** for context.

#### Agent 1: Community Signal Scanner (HN + Techmeme + Lobsters)

**Launch with:** `Agent` tool, subagent_type `general-purpose`

The user especially values Hacker News — always surface the top HN stories prominently.

**Prompt the agent to:**

1. Fetch these community-curated feeds using **WebFetch** (all in parallel):
   - `https://news.ycombinator.com/front` — Extract ALL stories with titles, URLs, source domains, point counts, comment counts. **HN is the user's most valued source — extract thoroughly.**
   - `https://news.ycombinator.com/best` — Also fetch the "best" page to catch high-signal stories that may have rotated off the front page
   - `https://techmeme.com/` — Extract headlines, source outlets, clustering signals
   - `https://lobste.rs/` — Extract titles, URLs, tags, comment counts

2. **Filter for relevance:** If a topic was provided, only keep stories directly related to the topic. If no topic, keep all stories above 100 points (HN) or featured (Techmeme). **Always include the top 5 HN stories by points regardless of topic match** — the user wants to see what's trending on HN.

3. **Quality signals to extract per story:**
   - Title and original source URL (NOT the discussion thread URL)
   - Point count / comment count / upvote count
   - Source domain reputation (e.g., arxiv.org > random blog)
   - Which feeds it appeared on (cross-source = higher signal)
   - For HN stories with 300+ comments, note the comment count — high discussion = high signal

4. **Return:** A JSON-formatted list of 15-20 stories, ranked by engagement. Each entry: `{title, url, source, points, comments, feeds_appeared_on}`.

#### Agent 2: Domain Feed Scanner

**Launch with:** `Agent` tool, subagent_type `general-purpose`

**Prompt the agent to:**

1. Read `~/.news-briefing/profile.json` to get the `domain_sources` list.

2. Fetch the front page of **every source** in `domain_sources` using **WebFetch** (all in parallel). Also fetch interest-matched specialty feeds:

   **For AI/ML interests:**
   - `https://arxiv.org/list/cs.AI/recent` — Recent ArXiv AI papers

   **For crypto interests:**
   - `https://www.theblock.co/latest` — The Block

   **For robotics interests:**
   - `https://spectrum.ieee.org/topic/robotics/` — IEEE Spectrum Robotics

   **For startup/product interests:**
   - `https://www.producthunt.com/` — Product Hunt

3. For each feed, extract story titles, URLs, dates, and any engagement signals.

4. **Filter for relevance:** If a topic was provided, only keep stories directly related. If no topic, keep the most interesting/recent 10-15 stories that match the user's HIGH-weight interest tags.

5. **Return:** A JSON-formatted list of 10-15 stories. Each entry: `{title, url, source_name, date, relevance_to_topic}`.

#### Agent 3: Deep Search Researcher

**Launch with:** `Agent` tool, subagent_type `general-purpose`

This agent does NOT use generic queries. It crafts **precise, editorial-quality search queries** designed to surface original reporting and expert analysis while filtering out SEO noise, listicles, and press release rewrites.

**Prompt the agent to:**

1. **Construct 5-6 targeted WebSearch queries** using these principles:

   **Query design rules:**
   - Use exact-phrase quotes around the core topic (e.g., `"Nemotron 3"` not `Nemotron 3`)
   - Append specific angles to each query — never search the same angle twice
   - Target known high-quality publications with `site:` when useful
   - Include the current month and year for recency, but NEVER use filler like "most important" or "breaking news" — these attract SEO spam
   - Use Boolean operators: `OR` to widen, `-` to exclude noise (e.g., `-listicle -"top 10"`)

   **Query templates (adapt to the actual topic):**

   ```
   Query 1 — Recent events/announcements:
   "{topic}" announcement OR launch OR release {month} {year}

   Query 2 — Expert analysis from quality outlets:
   "{topic}" site:arstechnica.com OR site:theverge.com OR site:wired.com OR site:techcrunch.com

   Query 3 — Technical deep dives:
   "{topic}" benchmark OR architecture OR "how it works" OR technical

   Query 4 — Industry/competitive implications:
   "{topic}" vs OR competition OR implications OR "what this means" {year}

   Query 5 — Original reporting (exclude aggregators):
   "{topic}" -"round up" -listicle -"top 10" exclusive OR report OR investigation {month} {year}

   Query 6 — Community/expert discussion:
   "{topic}" site:reddit.com OR site:twitter.com expert OR engineer OR researcher {year}
   ```

   **If no topic provided**, use the user's HIGH-weight interest tags to generate 2 queries per interest cluster.

2. Run all WebSearch queries **in parallel**.

3. **De-duplicate and quality-filter** the results:
   - Drop results from known low-signal domains (medium.com listicles, SEO farms, content mills)
   - Drop results that are obviously press release rewrites (check if multiple results have near-identical titles)
   - Prefer results from publications known for original reporting
   - Prefer results with specific facts, numbers, or quotes over vague summaries

4. **For the top 5 results**, use **WebFetch** to read the actual article and extract:
   - A 2-3 sentence summary of the key facts
   - Whether it contains original reporting vs. just repackaging
   - The `og:image` URL if present

5. **Return:** A JSON-formatted list of 8-12 stories, ranked by quality of reporting. Each entry: `{title, url, source, summary, has_original_reporting, og_image}`.

#### Agent 4: Email Newsletter Scanner (TLDR AI + TLDR Data)

**Launch with:** `Agent` tool, subagent_type `general-purpose`

This agent reads the user's email newsletters via the `gws` CLI. These newsletters are pre-curated by expert editors and contain high-signal stories that may not appear on HN or Techmeme.

**Prompt the agent to:**

1. **Check if `gws` is available** by running: `which gws 2>/dev/null`. If not found, return an empty list immediately.

2. **Fetch recent TLDR AI and TLDR Data emails** using these Bash commands in parallel:

   ```bash
   # Get message IDs for TLDR emails from the last 3 days
   gws gmail users messages list --params '{"userId":"me","q":"from:dan@tldrnewsletter.com TLDR newer_than:3d","maxResults":5}'
   ```

3. **For each message ID returned**, fetch the full body:

   ```bash
   gws gmail users messages get --params '{"userId":"me","id":"MESSAGE_ID","format":"full"}'
   ```

   Then decode the base64 text/plain body part using Python:
   ```bash
   ... | python3 -c "
   import sys, json, base64
   lines = sys.stdin.read().strip().split('\n')
   for i, l in enumerate(lines):
     if l.strip().startswith('{'):
       msg = json.loads('\n'.join(lines[i:]))
       break
   parts = msg.get('payload',{}).get('parts',[])
   for p in parts:
     if p.get('mimeType') == 'text/plain':
       print(base64.urlsafe_b64decode(p['body']['data']).decode('utf-8')[:5000])
       break
   "
   ```

4. **Parse the newsletter content.** TLDR newsletters follow a consistent format:
   - `HEADLINES & LAUNCHES` section — product releases, major announcements
   - `DEEP DIVES & ANALYSIS` section — technical analysis, engineering posts
   - Each story has a title in ALL CAPS, a reading time estimate, and a 2-3 sentence summary
   - URLs are embedded as numbered references `[N]`

   Extract each story with: title, summary, source URL (from the numbered references), and which newsletter it came from (TLDR AI or TLDR Data).

5. **Filter:** If a topic was provided, only keep stories related to the topic. If no topic, keep the 8-10 most interesting stories.

6. **Return:** A JSON-formatted list of stories. Each entry: `{title, url, source, summary, newsletter, reading_time}`.

#### Agent 5: Twitter/X Signal Scanner

**Launch with:** `Agent` tool, subagent_type `general-purpose`

This agent discovers what high-signal Twitter/X accounts are sharing — the links, papers, and takes that often precede coverage on HN or Techmeme by 12-24 hours.

**Prompt the agent to:**

1. **Read `~/.news-briefing/profile.json`** to get the `twitter_accounts` list. Extract all handles.

2. **Search for recent tweets with links** using **WebSearch** queries in parallel. For each account cluster (grouped by domain), construct queries like:

   ```
   site:x.com "{handle}" filter:links {month} {year}
   ```

   Also run broader domain-level queries:
   ```
   site:x.com "{domain_keyword}" announcement OR paper OR launch {month} {year} -retweet
   ```

   Run **8-10 WebSearch queries in parallel**, covering the major account clusters.

3. **Extract shared links.** From the search results, identify:
   - URLs that high-signal accounts shared (the linked content, not the tweet itself)
   - The account that shared it (for attribution)
   - Any context from the tweet text (e.g., "This is huge — first open-weight model to beat GPT-5 on coding")
   - Approximate time shared

4. **De-duplicate and filter:**
   - Drop self-promotional tweets (founders only sharing their own company's content)
   - Drop tweets without external links (pure commentary/takes)
   - Prefer tweets that share links to papers, blog posts, announcements, or tools
   - If multiple accounts shared the same link, note ALL of them — this is a strong cross-account signal

5. **For the top 5-8 shared links**, use **WebFetch** to read the linked article and extract:
   - Title and 2-sentence summary
   - The `og:image` URL if present
   - Whether the content is genuinely new (not a rehash of something already widely covered)

6. **Return:** A JSON-formatted list of 8-12 stories. Each entry: `{title, url, source, summary, shared_by (array of handles), tweet_context, og_image, shared_time}`.

#### Agent 6: GitHub Trending Scanner

**Launch with:** `Agent` tool, subagent_type `general-purpose`

This agent surfaces new repositories that are gaining traction in the user's tech stack and interest areas — early signal on tools before they hit the news cycle.

**Prompt the agent to:**

1. **Read `~/.news-briefing/profile.json`** to get the user's `tech_stack` and `interests`.

2. **Fetch GitHub trending pages** using **WebFetch** in parallel:
   - `https://github.com/trending/python?since=daily` — Python daily trending
   - `https://github.com/trending/typescript?since=daily` — TypeScript daily trending
   - `https://github.com/trending?since=daily` — All languages daily trending
   - `https://github.com/trending/python?since=weekly` — Python weekly (catches repos that trended earlier this week)

3. **For each trending page**, extract:
   - Repository full name (owner/repo)
   - Description (one-line)
   - Language
   - Stars today / this week
   - Total stars
   - Whether the repo was created in the last 30 days (truly new vs. legacy repo having a moment)

4. **Filter for relevance:**
   - **Must match** at least one of: user's HIGH-weight interest tags OR tech_stack languages
   - **Prefer** repos created in the last 30 days (genuinely new tools, not old repos resurfacing)
   - **Prefer** repos with meaningful descriptions (not empty or one-word)
   - **Drop** repos that are just homework/tutorials/awesome-lists unless they have 500+ stars today

5. **For the top 5-8 repos**, use **WebFetch** on `https://github.com/{owner}/{repo}` to get:
   - Full README first paragraph (what does this actually do?)
   - Any linked demo, docs, or blog post URL
   - The repo owner's profile (are they a known developer/company?)

6. **Score repos:**
   - **Stars today > 200** + matches HIGH-weight interest = top tier
   - **Created < 7 days ago** + stars today > 50 = early signal, high value
   - **Created < 30 days ago** + matches tech_stack = relevant new tool
   - **Cross-signal bonus**: If the repo was ALSO shared by a Twitter account from Agent 5 or appeared on HN from Agent 1, it gets a massive boost

7. **Return:** A JSON-formatted list of 5-8 repos, ranked by signal strength. Each entry: `{repo_name, url, description, language, stars_today, total_stars, created_date, is_new (boolean), relevance_reason, readme_summary}`.

#### Step 1C: Merge and Score

After all six agents return, merge their results into a single pool (typically 40-70 stories after de-duplication).

**Scoring algorithm (apply in order):**

1. **Cross-agent signal (strongest):** A story found by 2+ agents scores highest. If Agent 1 found it on HN with 400+ pts AND Agent 5 found it shared by @karpathy AND Agent 4 found it in TLDR AI, that's near-guaranteed top-tier quality. GitHub trending repos that also appear on HN or were shared by Twitter accounts get the strongest possible signal.

2. **Community engagement:** HN 300+ pts, Techmeme featured, or Lobsters 50+ pts. **Top 5 HN stories by points should always be considered for the final selection** — the user explicitly values HN trends.

3. **Twitter expert signal:** Stories shared by 2+ high-signal Twitter accounts get a significant boost — these accounts are curators, so multi-account sharing indicates expert consensus. Include the sharing accounts in the attribution (e.g., "Shared by @karpathy, @jimfan").

4. **Newsletter signal:** Stories that appear in TLDR AI or TLDR Data "HEADLINES & LAUNCHES" sections are pre-curated by expert editors and get a signal boost. Stories in both a TLDR newsletter AND HN/Techmeme are very high quality.

5. **Original reporting bonus:** Stories flagged by Agent 3 as having original reporting (not press release rewrites) get a boost.

6. **GitHub trending signal:** Repos with 200+ stars today that match user interests are strong candidates for the sidebar. Repos with 500+ stars today or that cross-signal with HN/Twitter can compete for main story slots.

7. **Source reputation:** Known quality outlets (Ars Technica, TechCrunch, The Verge, NVIDIA Newsroom for NVIDIA topics, etc.) score higher than unknown blogs.

8. **Recency:** Stories from the last 48 hours score higher than older ones.

9. **Interest match:** Stories matching the user's HIGH-weight interest tags get a relevance boost (only applies when no explicit topic is given).

**De-duplicate:** If multiple stories cover the same event, keep the one with the best source + highest engagement. Drop the rest. When a Twitter account shared a link that also appeared on HN, merge them into a single entry with combined attribution.

### Phase 2: Editorial Curation

From the scored pool, select exactly **8 stories** for the main layout, plus **~8 additional stories** for the "Latest" sidebar feed.

**Main 8 stories** — use these criteria (in priority order):

1. **Cross-source signal** — Stories that appear across multiple high-signal feeds (HN + Techmeme, HN + TLDR, etc.) get priority. This is the strongest quality signal.
2. **Community engagement** — High point counts, comment counts, and upvotes indicate the story resonated with a technical audience.
3. **Significance** — Does this actually matter? Will people still be talking about it next week? Prefer stories that change something (new capability, policy shift, market move) over incremental updates.
4. **Uniqueness** — Each story must cover a distinct angle. Drop duplicates and rehashes. If two stories cover the same event, pick the better source.
5. **Surprise factor** — Prefer stories that are genuinely interesting or counterintuitive over predictable press releases.
6. **Diversity** — Cover different facets (e.g., for tech: a deep technical post, a policy/business story, a tool/product, a cultural moment, an infrastructure play).

**"Latest" sidebar stories** — pick 6-8 additional stories from the pool that didn't make the main 8 but are still interesting. These only need a headline, source, and approximate time. Include HN point counts or comment counts where available as social proof. Include Twitter attribution where available (e.g., "Shared by @karpathy").

**"From X/Twitter" dedicated section** — from Agent 5's results, select the **top shared link** as the section lead (image + headline + summary + sharing accounts). Then select **3-4 additional shared links** as secondary items (headline-only with sharing attribution). Finally, select **2-3 notable tweet quotes** from high-signal accounts that add editorial context — a pithy take, a surprising observation, or expert commentary on a trending story. If Agent 5 returned no useful results, omit this section entirely.

**"Trending on GitHub" dedicated section** — from Agent 6's results, select **4 repositories** for full-width card treatment. Each card shows: repo name (owner/repo), stars today, one-line description, language badge, total stars, and any cross-signal notes. Prioritize repos that: (1) match user interests, (2) have 100+ stars today, (3) were created recently, (4) cross-signal with other agents. If fewer than 4 quality repos exist, show what you have — don't pad with irrelevant repos.

**What to avoid:**
- Generic fundraising announcements (unless the amount or implications are genuinely significant)
- Listicles and roundup posts
- SEO-driven blog posts with no original reporting
- Stories that are just repackaging a press release

Rank the main 8 stories 1-8 by significance. Story #1 gets hero treatment. Stories #2-3 go in the hero sidebar. Stories #4-6 go in the mid-grid (3-column card layout with images). Stories #7-8 get the lower feature treatment alongside the "Latest" sidebar.

**Editorial labels** — assign each main story a label that tells readers what kind of content it is:
- `NEW RELEASE` — product or model launches
- `ANALYSIS` — deeper takes with implications
- `DEEP DIVE` — technical or investigative pieces
- `EXCLUSIVE` — scoops or first reports
- `POLICY` — regulatory, legal, or government stories
- Or a relevant event name like `GTC 2026`, `WWDC`, etc.
Do NOT use "BREAKING" unless something literally just happened in the last hour.

**For source linking:** When a story originated from a community feed (HN, Lobsters), link to the **original source article**, not the discussion thread. But note the community engagement (e.g., "654 points on HN") in the editorial lede if it adds context.

**Twitter attribution:** When a story was surfaced by Agent 5 (shared by Twitter accounts), include the sharing accounts in the meta line. Format: `"Shared by @karpathy, @jimfan · 418 pts on HN · Techmeme"`. This gives the user a sense of which experts noticed this story. If a story was ONLY found via Twitter (not on HN/Techmeme/TLDR), note the tweet context — e.g., a quote from the sharer that adds editorial value.

**GitHub trending attribution:** When a GitHub repo appears in the sidebar, format the attribution as: `"★ {stars_today} today · {language} · Created {relative_date}"`. If the repo was also shared by a Twitter account or appeared on HN, note that cross-signal.

### Phase 3: Deep Content Fetch

For each of the 8 main stories, use **WebFetch** in parallel to:

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

**Use stockanalysis.com as the primary source** — Google Finance pages render via JavaScript and fail with WebFetch.

Use **WebFetch** in parallel:
- `https://stockanalysis.com/stocks/spy/` — S&P 500 (SPY ETF as proxy, extract price and day change)

For indices and Bitcoin, use **WebSearch** in parallel:
- `"S&P 500 index price today {current date}"` — Extract exact value and percentage change
- `"NASDAQ composite price today {current date}"` — Extract exact value and percentage change
- `"Bitcoin BTC price today {current date}"` — Extract exact price and percentage change

#### Step 4C: Fetch individual stock prices

For each ticker from Step 4A, use **WebFetch** in parallel on:
- `https://stockanalysis.com/stocks/{TICKER}/`

Extract: current price, day change amount, day change percentage.

If stockanalysis.com fails for a ticker, fall back to **WebSearch** for `"{TICKER} stock price today"`.

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
- Story #1 gets a full lede paragraph in the hero. Stories #2-3 get headline + short lede in the hero sidebar. Stories #4-6 get the mid-grid card treatment (image + headline + short lede). Stories #7-8 get the lower feature treatment with images + ledes.

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

### Phase 8: Email Delivery (optional)

After the briefing is generated and archived, check if the Google Workspace CLI (`gws`) is available.

#### Step 8A: Check for gws

Run this **Bash** command:
```bash
which gws 2>/dev/null
```

If `gws` is not found, skip this phase entirely. Do not mention email delivery to the user.

#### Step 8B: Check for saved email preference

Read `~/.news-briefing/profile.json` and check if an `email` field exists. If it does, use that email address and skip to Step 8C.

If no `email` field exists, ask the user:

> "I can email this briefing to you. What email address should I send it to? (or say 'skip' to skip)"

If the user provides an email, save it to `~/.news-briefing/profile.json` under the `email` field so they won't be asked again on future runs. If they say "skip" or "no", save `"email": "skip"` to suppress the prompt on future runs.

If the saved value is `"skip"`, skip this phase silently.

#### Step 8C: Send the email

Run this **Bash** command:
```bash
gws gmail +send \
  --to {email_address} \
  --subject "Your Briefing — {topic} — {date}" \
  --body "$(cat ~/Desktop/news-briefing.html)" \
  --html
```

Tell the user the briefing was sent to their email.

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
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=Playfair+Display:ital,wght@0,400;0,700;0,800;0,900;1,400&family=Source+Serif+4:ital,wght@0,400;0,500;1,400&family=IBM+Plex+Mono:wght@400;500;600&display=swap');

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

    /* ── Mid Grid (3 card columns) ── */
    .mid-grid {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 2rem;
      padding: 2rem 0;
      border-bottom: 1px solid var(--border-light);
    }
    .mid-card {
      display: block;
      transition: opacity 0.15s;
    }
    .mid-card:hover { opacity: 0.85; }
    .mid-card .card-img {
      width: 100%;
      aspect-ratio: 16/10;
      object-fit: cover;
      border-radius: 5px;
      margin-bottom: 0.75rem;
    }
    .mid-card h3 {
      font-family: var(--serif);
      font-size: 1.05rem;
      font-weight: 700;
      line-height: 1.3;
      color: var(--text);
      margin-bottom: 0.5rem;
      transition: color 0.2s;
    }
    .mid-card:hover h3 { color: var(--accent); }
    .mid-card .card-lede {
      font-family: var(--body-serif);
      font-size: 0.8rem;
      color: var(--text-sub);
      line-height: 1.6;
      margin-bottom: 0.4rem;
      display: -webkit-box;
      -webkit-line-clamp: 3;
      -webkit-box-orient: vertical;
      overflow: hidden;
    }
    .mid-card .meta {
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

    /* ── Trending Repos ── */
    .trending-section {
      margin-top: 1rem;
      padding-top: 1rem;
      border-top: 1px solid var(--border);
    }
    .trending-title {
      font-size: 0.72rem;
      font-weight: 800;
      letter-spacing: 0.04em;
      text-transform: uppercase;
      color: var(--accent);
      margin-bottom: 0.75rem;
    }
    .trending-repo {
      display: block;
      padding: 0.6rem 0;
      border-bottom: 1px solid var(--border);
      transition: opacity 0.15s;
    }
    .trending-repo:last-child { border-bottom: none; }
    .trending-repo:hover { opacity: 0.8; }
    .trending-repo-header {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      margin-bottom: 0.25rem;
    }
    .trending-repo-stars {
      font-size: 0.58rem;
      font-weight: 700;
      color: var(--accent);
      white-space: nowrap;
    }
    .trending-repo-name {
      font-family: 'IBM Plex Mono', monospace;
      font-size: 0.75rem;
      font-weight: 600;
      color: var(--text);
    }
    .trending-repo-desc {
      font-size: 0.68rem;
      color: var(--text-sub);
      line-height: 1.4;
      display: -webkit-box;
      -webkit-line-clamp: 2;
      -webkit-box-orient: vertical;
      overflow: hidden;
    }
    .trending-repo-meta {
      font-size: 0.55rem;
      color: var(--text-muted);
      margin-top: 0.2rem;
    }
    .lang-badge {
      display: inline-block;
      font-size: 0.5rem;
      font-weight: 700;
      padding: 1px 5px;
      border-radius: 3px;
      background: var(--surface-2);
      color: var(--text-sub);
      letter-spacing: 0.02em;
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
      .mid-grid { grid-template-columns: 1fr; gap: 1.5rem; }
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

    /* ── View Toggle Button ── */
    .view-toggle {
      font-family: var(--sans);
      font-size: 0.65rem;
      font-weight: 600;
      letter-spacing: 0.03em;
      padding: 5px 12px;
      border-radius: 4px;
      border: 1px solid var(--border-light);
      background: transparent;
      color: var(--text-muted);
      cursor: pointer;
      transition: all 0.2s;
    }
    .view-toggle:hover { color: var(--text); border-color: var(--text-muted); }

    /* ── Reader View (PI-inspired, enhanced) ── */
    .reader-view { display: none; }
    body.reader-mode .bloomberg-view { display: none; }
    body.reader-mode .reader-view { display: block; }
    body.reader-mode .ticker-bar { display: none; }

    body.reader-mode {
      --bg: #f0ebe4;
      --surface: #e8e2da;
      --text: #1a1a1a;
      --text-sub: #4a4540;
      --text-muted: #8a8480;
      --border: #d4cec6;
      --border-light: #c8c2ba;
      --accent: #1a1a1a;
      --mono: 'IBM Plex Mono', 'Menlo', monospace;
      --tag-robotics: #2563eb;
      --tag-agents: #7c3aed;
      --tag-crypto: #d97706;
      --tag-ml: #059669;
      --tag-policy: #dc2626;
      --tag-tools: #6b7280;
      background: var(--bg);
      color: var(--text);
    }
    body.reader-mode .nav { background: var(--bg); border-bottom: 1px solid var(--border); }
    body.reader-mode .nav-brand, body.reader-mode .nav-brand span { color: var(--text); }
    body.reader-mode .nav-tab { color: var(--text); border-bottom-color: var(--text); }
    body.reader-mode .nav-date { color: var(--text-muted); }
    body.reader-mode .view-toggle { color: var(--text-muted); border-color: var(--border); }
    body.reader-mode .view-toggle:hover { color: var(--text); border-color: var(--text-muted); }

    .reader-view { max-width: 720px; margin: 0 auto; padding: 2.5rem 2rem 4rem; }
    .reader-header { margin-bottom: 2.5rem; }
    .reader-title {
      font-family: 'Playfair Display', Georgia, serif;
      font-size: 2.2rem; font-weight: 400; line-height: 1.2;
      margin-bottom: 1rem; letter-spacing: -0.01em;
    }
    .reader-subtitle {
      font-family: var(--mono); font-size: 0.85rem;
      color: var(--text-sub); line-height: 1.7;
    }

    /* Section dividers */
    .reader-section {
      font-family: var(--mono); font-size: 0.7rem; font-weight: 600;
      letter-spacing: 0.08em; text-transform: uppercase;
      color: var(--text-muted); padding: 1.5rem 0 0.5rem;
      border-bottom: 1px solid var(--border); margin-bottom: 0.5rem;
    }

    /* Date group headers */
    .reader-date-group {
      font-family: var(--mono); font-size: 0.68rem; font-weight: 500;
      color: var(--text-muted); letter-spacing: 0.04em;
      padding: 1.25rem 0 0.25rem 2rem;
    }

    /* Timeline spine */
    .reader-timeline {
      position: relative;
      padding-left: 2rem;
    }
    .reader-timeline::before {
      content: '';
      position: absolute;
      left: 4px;
      top: 0;
      bottom: 0;
      width: 1.5px;
      background: var(--border);
    }

    /* Story base */
    .reader-story {
      display: block;
      position: relative;
      padding: 1rem 0 1rem 1.5rem;
      text-decoration: none;
      color: inherit;
      transition: transform 0.15s, opacity 0.15s;
    }
    .reader-story:hover { opacity: 0.8; }

    /* Bullet on the spine */
    .reader-story::before {
      content: '';
      position: absolute;
      left: -2rem;
      top: 1.5rem;
      width: 9px; height: 9px;
      background: var(--text-muted);
      border-radius: 50%;
      border: 2px solid var(--bg);
      z-index: 1;
    }

    /* Boxed stories (2-3 most impactful only): black outline, card bg — makes them feel noteworthy */
    .reader-story.boxed .reader-story-content {
      border: 1.5px solid var(--text);
      border-radius: 4px;
      padding: 1.25rem 1.5rem;
      background: rgba(255,255,255,0.3);
    }
    .reader-story.boxed:hover { transform: translateX(4px); }
    .reader-story.boxed::before {
      background: var(--text);
      width: 12px; height: 12px;
      left: calc(-2rem - 1.5px);
    }
    .reader-story.boxed .reader-story-title {
      font-size: 1.05rem;
      font-weight: 700;
    }

    /* Featured (remaining top stories, no border — clean and flat) */
    .reader-story.featured .reader-story-content { padding: 0.85rem 0; }
    .reader-story.featured::before { background: var(--text); width: 10px; height: 10px; }
    .reader-story.featured:hover { transform: translateX(2px); }

    /* Sidebar / unfeatured: compact */
    .reader-story:not(.featured):not(.boxed) .reader-story-content { padding: 0.75rem 0; }
    .reader-story:not(.featured):not(.boxed) .reader-story-title { font-size: 0.82rem; }
    .reader-story:not(.featured):not(.boxed) .reader-story-lede { font-size: 0.75rem; }

    .reader-story-content { transition: all 0.15s; }

    .reader-story-header {
      display: flex; justify-content: space-between;
      align-items: baseline; gap: 1rem; margin-bottom: 0.4rem;
    }
    .reader-story-title {
      font-family: var(--mono); font-size: 0.9rem;
      font-weight: 600; line-height: 1.4;
    }
    .reader-story-date {
      font-family: var(--mono); font-size: 0.7rem;
      color: var(--text-muted); white-space: nowrap;
    }

    /* Topic tags */
    .reader-tags { display: flex; gap: 0.4rem; margin-bottom: 0.5rem; flex-wrap: wrap; }
    .reader-tag {
      font-family: var(--mono); font-size: 0.55rem; font-weight: 600;
      letter-spacing: 0.04em; text-transform: uppercase;
      padding: 2px 7px; border-radius: 3px;
      background: rgba(0,0,0,0.05); color: var(--text-muted);
    }
    .reader-tag.robotics { color: var(--tag-robotics); background: rgba(37,99,235,0.08); }
    .reader-tag.agents { color: var(--tag-agents); background: rgba(124,58,237,0.08); }
    .reader-tag.crypto { color: var(--tag-crypto); background: rgba(217,119,6,0.08); }
    .reader-tag.ml { color: var(--tag-ml); background: rgba(5,150,105,0.08); }
    .reader-tag.policy { color: var(--tag-policy); background: rgba(220,38,38,0.08); }
    .reader-tag.tools { color: var(--tag-tools); background: rgba(107,114,128,0.08); }

    .reader-story-lede {
      font-family: var(--mono); font-size: 0.8rem;
      color: var(--text-sub); line-height: 1.65;
    }

    /* Source attribution */
    .reader-story-source {
      font-family: var(--mono); font-size: 0.65rem;
      color: var(--text-muted); margin-top: 0.4rem;
    }

    .reader-colophon {
      margin-top: 2rem; padding-top: 1.5rem;
      border-top: 1px solid var(--border);
      font-family: var(--mono); font-size: 0.7rem;
      color: var(--text-muted); text-align: center; line-height: 1.8;
    }
    .reader-colophon a { color: var(--text-sub); text-decoration: underline; }

    /* ── Bloomberg Light Theme ── */
    body.bloomberg-mode {
      --bg: #ffffff;
      --surface: #f5f5f5;
      --surface-2: #ebebeb;
      --text: #1a1a1a;
      --text-sub: #4a4a4a;
      --text-muted: #8a8a8a;
      --border: #e0e0e0;
      --border-light: #ebebeb;
      --accent: #1a1a1a;
      --accent-dim: rgba(26,26,26,0.06);
      --green: #16a34a;
      --red: #dc2626;
      background: var(--bg);
      color: var(--text);
    }
    body.bloomberg-mode .nav { background: var(--bg); border-bottom: 2px solid var(--text); }
    body.bloomberg-mode .nav-brand, body.bloomberg-mode .nav-brand span { color: var(--text); }
    body.bloomberg-mode .nav-tab { color: var(--text); border-bottom-color: var(--text); }
    body.bloomberg-mode .ticker-bar { background: var(--surface); }
    body.bloomberg-mode .ticker-label { color: var(--text-muted); }
    body.bloomberg-mode .ticker-value { color: var(--text); }
    body.bloomberg-mode .label-analysis { color: #6b21a8; border-color: #6b21a8; }
    body.bloomberg-mode .label-deepdive { color: #1d4ed8; border-color: #1d4ed8; }
    body.bloomberg-mode .label-exclusive { color: #15803d; border-color: #15803d; }
    body.bloomberg-mode .label-policy { color: #b45309; border-color: #b45309; }
    body.bloomberg-mode .label-breaking { color: #dc2626; border-color: #dc2626; }
    body.bloomberg-mode .sidebar-title { color: var(--text); }
    body.bloomberg-mode .latest-time { color: var(--text-muted); font-weight: 600; }
    body.bloomberg-mode .trending-title { color: var(--text); }
    body.bloomberg-mode .trending-repo-stars { color: #b45309; }
    body.bloomberg-mode .colophon { color: var(--text-muted); }
    body.bloomberg-mode .colophon a { color: var(--text); }
    body.bloomberg-mode .section-header { border-bottom-color: var(--text); }

    /* ── Bloomberg Section Headers ── */
    .section-header {
      display: flex;
      align-items: baseline;
      gap: 1.5rem;
      padding: 1.5rem 0 0.75rem;
      border-bottom: 2px solid var(--text);
      margin-bottom: 1.5rem;
    }
    .section-title {
      font-size: 0.85rem;
      font-weight: 800;
      letter-spacing: -0.01em;
    }
    .section-tabs {
      display: flex;
      gap: 1rem;
    }
    .section-tab {
      font-size: 0.68rem;
      font-weight: 500;
      color: var(--text-muted);
      transition: color 0.15s;
    }
    .section-tab:hover { color: var(--text); }

    /* ── Toggle Segmented Control ── */
    .toggle-group {
      display: flex;
      align-items: center;
      border: 1px solid var(--border-light);
      border-radius: 5px;
      overflow: hidden;
    }
    .toggle-btn {
      font-family: var(--sans);
      font-size: 0.62rem;
      font-weight: 600;
      letter-spacing: 0.03em;
      padding: 5px 14px;
      border: none;
      background: transparent;
      color: var(--text-muted);
      cursor: pointer;
      transition: all 0.2s;
      border-right: 1px solid var(--border-light);
    }
    .toggle-btn:last-child { border-right: none; }
    .toggle-btn:hover { color: var(--text); }
    .toggle-btn.active {
      background: var(--text);
      color: var(--bg);
    }
    body.bloomberg-mode .toggle-group { border-color: var(--border); }
    body.bloomberg-mode .toggle-btn { color: var(--text-muted); border-color: var(--border); }
    body.bloomberg-mode .toggle-btn.active { background: var(--text); color: #fff; }

    /* ── Twitter/X Section ── */
    .twitter-section {
      padding: 0 0 1.75rem;
      border-bottom: 1px solid var(--border-light);
    }
    .twitter-grid {
      display: grid;
      grid-template-columns: 1fr 1fr 300px;
      gap: 0;
    }
    .twitter-lead {
      padding-right: 2rem;
      border-right: 1px solid var(--border-light);
      transition: opacity 0.15s;
    }
    .twitter-lead:hover { opacity: 0.85; }
    .twitter-lead-img {
      width: 100%;
      aspect-ratio: 16/10;
      object-fit: cover;
      border-radius: 5px;
      margin-bottom: 0.75rem;
    }
    .twitter-lead h3 {
      font-family: var(--serif);
      font-size: 1.2rem;
      font-weight: 700;
      line-height: 1.3;
      margin-bottom: 0.5rem;
      transition: color 0.2s;
    }
    .twitter-lead:hover h3 { color: var(--accent); }
    .twitter-lead .lede {
      font-family: var(--body-serif);
      font-size: 0.82rem;
      color: var(--text-sub);
      line-height: 1.6;
      margin-bottom: 0.5rem;
    }
    .twitter-lead .shared-by {
      font-size: 0.62rem;
      color: var(--accent);
      font-weight: 600;
    }
    .twitter-lead .meta {
      font-size: 0.58rem;
      color: var(--text-muted);
      margin-top: 0.3rem;
    }
    body.bloomberg-mode .twitter-lead .shared-by { color: #1d4ed8; }
    .twitter-mid {
      padding: 0 1.5rem;
      border-right: 1px solid var(--border-light);
    }
    .twitter-link-item {
      display: block;
      padding: 0.75rem 0;
      border-bottom: 1px solid var(--border);
      transition: opacity 0.15s;
    }
    .twitter-link-item:last-child { border-bottom: none; }
    .twitter-link-item:hover { opacity: 0.8; }
    .twitter-link-title {
      font-size: 0.82rem;
      font-weight: 600;
      line-height: 1.4;
      margin-bottom: 0.3rem;
    }
    .twitter-link-shared {
      font-size: 0.58rem;
      color: var(--accent);
      font-weight: 600;
    }
    body.bloomberg-mode .twitter-link-shared { color: #1d4ed8; }
    .twitter-link-source {
      font-size: 0.55rem;
      color: var(--text-muted);
      margin-top: 0.15rem;
    }
    .twitter-quotes {
      padding-left: 1.5rem;
    }
    .twitter-quote {
      padding: 0.75rem 0;
      border-bottom: 1px solid var(--border);
    }
    .twitter-quote:last-child { border-bottom: none; }
    .twitter-quote-handle {
      font-family: 'IBM Plex Mono', monospace;
      font-size: 0.7rem;
      font-weight: 600;
      color: var(--text);
      margin-bottom: 0.3rem;
    }
    .twitter-quote-text {
      font-family: var(--body-serif);
      font-size: 0.75rem;
      font-style: italic;
      color: var(--text-sub);
      line-height: 1.5;
    }
    .twitter-quote-context {
      font-size: 0.55rem;
      color: var(--text-muted);
      margin-top: 0.2rem;
    }

    /* ── GitHub Section (Bloomberg 4-col grid) ── */
    .github-section {
      padding: 0 0 1.75rem;
      border-bottom: 1px solid var(--border-light);
    }
    .github-grid {
      display: grid;
      grid-template-columns: repeat(4, 1fr);
      gap: 1.25rem;
    }
    .github-card {
      display: block;
      padding: 1.25rem;
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 6px;
      transition: all 0.2s;
    }
    .github-card:hover {
      border-color: var(--text-muted);
      transform: translateY(-2px);
    }
    .github-card-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 0.6rem;
    }
    .github-card-stars {
      font-size: 0.65rem;
      font-weight: 700;
      color: #d97706;
    }
    .github-card-lang {
      font-size: 0.55rem;
      font-weight: 700;
      padding: 2px 6px;
      border-radius: 3px;
      background: var(--surface-2);
      color: var(--text-sub);
    }
    .github-card-name {
      font-family: 'IBM Plex Mono', monospace;
      font-size: 0.82rem;
      font-weight: 600;
      color: var(--text);
      margin-bottom: 0.5rem;
      line-height: 1.3;
    }
    .github-card-desc {
      font-size: 0.72rem;
      color: var(--text-sub);
      line-height: 1.5;
      margin-bottom: 0.5rem;
      display: -webkit-box;
      -webkit-line-clamp: 3;
      -webkit-box-orient: vertical;
      overflow: hidden;
    }
    .github-card-meta {
      font-size: 0.55rem;
      color: var(--text-muted);
    }
    .github-card-signal {
      font-size: 0.55rem;
      color: var(--accent);
      font-weight: 600;
      margin-top: 0.3rem;
    }
    body.bloomberg-mode .github-card-signal { color: #1d4ed8; }

    /* ── Responsive: new sections ── */
    @media (max-width: 1060px) {
      .twitter-grid { grid-template-columns: 1fr; }
      .twitter-lead { padding-right: 0; border-right: none; border-bottom: 1px solid var(--border-light); padding-bottom: 1.5rem; }
      .twitter-mid { padding: 1.5rem 0; border-right: none; border-bottom: 1px solid var(--border-light); }
      .twitter-quotes { padding-left: 0; padding-top: 1.5rem; }
      .github-grid { grid-template-columns: repeat(2, 1fr); }
    }
    @media (max-width: 600px) {
      .github-grid { grid-template-columns: 1fr; }
      .toggle-btn { padding: 4px 10px; font-size: 0.58rem; }
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
      <div class="toggle-group">
        <button class="toggle-btn" onclick="toggleBloomberg()" id="bloomberg-toggle">Bloomberg</button>
        <button class="toggle-btn" onclick="toggleReader()" id="reader-toggle">Reader</button>
      </div>
    </div>
  </div>
</div>

<div class="layout bloomberg-view">

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

  <!-- ── Mid Grid (3 cards) ── -->
  <div class="mid-grid">
    <!-- Story #4 -->
    <a class="mid-card" href="{{STORY_4_URL}}" target="_blank">
      <img class="card-img" src="{{STORY_4_IMAGE}}" alt="{{STORY_4_ALT}}">
      <div class="label {{STORY_4_LABEL_CLASS}}">{{STORY_4_LABEL}}</div>
      <h3>{{STORY_4_HEADLINE}}</h3>
      <p class="card-lede">{{STORY_4_SHORT_LEDE}}</p>
      <div class="meta">{{STORY_4_SOURCES}}</div>
    </a>
    <!-- Story #5 -->
    <a class="mid-card" href="{{STORY_5_URL}}" target="_blank">
      <img class="card-img" src="{{STORY_5_IMAGE}}" alt="{{STORY_5_ALT}}">
      <div class="label {{STORY_5_LABEL_CLASS}}">{{STORY_5_LABEL}}</div>
      <h3>{{STORY_5_HEADLINE}}</h3>
      <p class="card-lede">{{STORY_5_SHORT_LEDE}}</p>
      <div class="meta">{{STORY_5_SOURCES}}</div>
    </a>
    <!-- Story #6 -->
    <a class="mid-card" href="{{STORY_6_URL}}" target="_blank">
      <img class="card-img" src="{{STORY_6_IMAGE}}" alt="{{STORY_6_ALT}}">
      <div class="label {{STORY_6_LABEL_CLASS}}">{{STORY_6_LABEL}}</div>
      <h3>{{STORY_6_HEADLINE}}</h3>
      <p class="card-lede">{{STORY_6_SHORT_LEDE}}</p>
      <div class="meta">{{STORY_6_SOURCES}}</div>
    </a>
  </div>

  <!-- ── Lower Section ── -->
  <div class="lower-section">
    <!-- Story #7 -->
    <div class="lower-feature">
      <div class="label {{STORY_7_LABEL_CLASS}}">{{STORY_7_LABEL}}</div>
      <a href="{{STORY_7_URL}}" target="_blank">
        <img class="feature-img" src="{{STORY_7_IMAGE}}" alt="{{STORY_7_ALT}}">
      </a>
      <a href="{{STORY_7_URL}}" target="_blank">
        <h2>{{STORY_7_HEADLINE}}</h2>
      </a>
      <p class="lede">{{STORY_7_LEDE}}</p>
      <div class="meta">{{STORY_7_SOURCES}}</div>
    </div>

    <!-- Story #8 -->
    <div class="lower-mid">
      <div class="label {{STORY_8_LABEL_CLASS}}">{{STORY_8_LABEL}}</div>
      <a href="{{STORY_8_URL}}" target="_blank">
        <img class="mid-img" src="{{STORY_8_IMAGE}}" alt="{{STORY_8_ALT}}">
      </a>
      <a href="{{STORY_8_URL}}" target="_blank">
        <h2>{{STORY_8_HEADLINE}}</h2>
      </a>
      <p class="lede">{{STORY_8_LEDE}}</p>
      <div class="meta">{{STORY_8_SOURCES}}</div>
    </div>

    <!-- Latest sidebar -->
    <div class="lower-sidebar">
      <div class="sidebar-title">Latest</div>
      <!-- ~6-8 additional stories from the pool -->
      {{LATEST_ITEMS}}
    </div>
  </div>

  <!-- ── From X/Twitter (Bloomberg Feature+List pattern) ── -->
  <div class="twitter-section">
    <div class="section-header">
      <span class="section-title">From X / Twitter</span>
    </div>
    <div class="twitter-grid">
      <!-- Lead shared link (biggest story from Twitter signal) -->
      <a class="twitter-lead" href="{{TWITTER_LEAD_URL}}" target="_blank">
        <img class="twitter-lead-img" src="{{TWITTER_LEAD_IMAGE}}" alt="">
        <h3>{{TWITTER_LEAD_HEADLINE}}</h3>
        <p class="lede">{{TWITTER_LEAD_SUMMARY}}</p>
        <div class="shared-by">Shared by {{TWITTER_LEAD_SHARED_BY}}</div>
        <div class="meta">{{TWITTER_LEAD_SOURCE}} · {{TWITTER_LEAD_TIME}}</div>
      </a>
      <!-- 3-4 additional shared links (headline-only) -->
      <div class="twitter-mid">
        {{TWITTER_LINK_ITEMS}}
      </div>
      <!-- 2-3 notable tweet quotes with context -->
      <div class="twitter-quotes">
        {{TWITTER_QUOTE_ITEMS}}
      </div>
    </div>
  </div>

  <!-- ── Trending on GitHub (Bloomberg 4-col grid) ── -->
  <div class="github-section">
    <div class="section-header">
      <span class="section-title">Trending on GitHub</span>
    </div>
    <div class="github-grid">
      <!-- 4 repo cards from Agent 6 -->
      {{GITHUB_CARD_ITEMS}}
    </div>
  </div>

  <div class="colophon">
    Sources: {{ALL_SOURCES_WITH_LINKS}} &middot; <a href="https://x.com" target="_blank">X/Twitter</a> &middot; <a href="https://github.com/trending" target="_blank">GitHub Trending</a><br>
    Editorially curated &middot; Built with Claude Code &middot; {{CURRENT_DATE}}
  </div>

</div>

<!-- ── Reader View (PI-style, enhanced) ── -->
<div class="reader-view">
  <div class="reader-header">
    <div class="reader-title">{{BRIEFING_TITLE}}</div>
    <div class="reader-subtitle">{{READER_SUBTITLE}}</div>
  </div>

  <div class="reader-section">Top Stories</div>
  <div class="reader-timeline">
    <!-- Main 8 stories: use class="reader-story boxed" for the 2-3 most impactful, class="reader-story featured" for the rest -->
    {{READER_TOP_STORIES}}
  </div>

  <div class="reader-section">Also Noteworthy</div>
  <div class="reader-timeline">
    <!-- Sidebar 8 stories: use class="reader-story" (no featured class) -->
    {{READER_SIDEBAR_STORIES}}
  </div>

  <div class="reader-section">From X / Twitter</div>
  <div class="reader-timeline">
    <!-- Twitter-sourced stories with attribution -->
    {{READER_TWITTER_STORIES}}
  </div>

  <div class="reader-section">Trending Repos</div>
  <div class="reader-timeline">
    <!-- 3-5 GitHub trending repos relevant to user interests -->
    {{READER_TRENDING_REPOS}}
  </div>

  <div class="reader-colophon">
    Sources: {{ALL_SOURCES_WITH_LINKS}}<br>
    Editorially curated &middot; Built with Claude Code &middot; {{CURRENT_DATE}}
  </div>
</div>

<script>
function toggleBloomberg() {
  const body = document.body;
  body.classList.remove('reader-mode');
  body.classList.toggle('bloomberg-mode');
  updateToggleState();
}
function toggleReader() {
  const body = document.body;
  body.classList.remove('bloomberg-mode');
  body.classList.toggle('reader-mode');
  updateToggleState();
}
function updateToggleState() {
  const body = document.body;
  document.getElementById('bloomberg-toggle').classList.toggle('active', body.classList.contains('bloomberg-mode'));
  document.getElementById('reader-toggle').classList.toggle('active', body.classList.contains('reader-mode'));
}
</script>

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

### Reader view story format

The reader view shows ALL 16 stories split into two sections with a vertical timeline spine.

**Boxed stories (2-3 most impactful)** — uses class `reader-story boxed`. Only apply this to the 2-3 stories with the strongest cross-source signal or highest significance. The black outline makes them visually pop against the flat timeline, signaling "these are the ones that matter most."

```html
<a class="reader-story boxed" href="{{URL}}" target="_blank">
  <div class="reader-story-content">
    <div class="reader-story-header">
      <span class="reader-story-title">{{HEADLINE}}</span>
      <span class="reader-story-date">{{DATE}}</span>
    </div>
    <div class="reader-tags">
      <span class="reader-tag agents">AI AGENTS</span>
      <span class="reader-tag policy">SECURITY</span>
    </div>
    <div class="reader-story-lede">{{SHORT_SUMMARY}}</div>
    <div class="reader-story-source">{{SOURCE}} · {{ENGAGEMENT}}</div>
  </div>
</a>
```

**Featured stories (#2-8)** — uses class `reader-story featured`:

```html
<a class="reader-story featured" href="{{URL}}" target="_blank">
  <div class="reader-story-content">
    <div class="reader-story-header">
      <span class="reader-story-title">{{HEADLINE}}</span>
      <span class="reader-story-date">{{DATE}}</span>
    </div>
    <div class="reader-tags">
      <span class="reader-tag robotics">ROBOTICS</span>
    </div>
    <div class="reader-story-lede">{{SHORT_SUMMARY}}</div>
    <div class="reader-story-source">{{SOURCE}} · {{ENGAGEMENT}}</div>
  </div>
</a>
```

**Sidebar stories (#9-16)** — uses class `reader-story` (no featured):

```html
<a class="reader-story" href="{{URL}}" target="_blank">
  <div class="reader-story-content">
    <div class="reader-story-header">
      <span class="reader-story-title">{{HEADLINE}}</span>
      <span class="reader-story-date">{{DATE}}</span>
    </div>
    <div class="reader-story-lede">{{SHORT_SUMMARY}}</div>
    <div class="reader-story-source">{{SOURCE}}</div>
  </div>
</a>
```

**Tag class mapping:**

| Topic | CSS class | Used for |
|---|---|---|
| Robotics, physical AI | `reader-tag robotics` | Blue pill |
| AI agents, coding agents | `reader-tag agents` | Purple pill |
| Crypto, DeFi, prediction markets | `reader-tag crypto` | Amber pill |
| ML, deep learning, research | `reader-tag ml` | Green pill |
| Policy, regulation, legal | `reader-tag policy` | Red pill |
| Tools, frameworks, developer | `reader-tag tools` | Gray pill |

**Date grouping:** Optionally insert `<div class="reader-date-group">Today</div>` or `<div class="reader-date-group">This Week</div>` before groups of stories that share a date range, instead of repeating the same date on every item.

**Visual hierarchy:**
- `boxed` = largest bullet (12px), black outline border, larger title (1.05rem), bold hover shift — **only 2-3 stories max**
- `featured` = medium bullet (10px), no border, flat padding, subtle hover shift
- unfeatured = small bullet (9px), no border, no background, compact text

The `{{READER_TOP_STORIES}}` placeholder contains stories #1-8 (2-3 boxed + rest featured). The `{{READER_SIDEBAR_STORIES}}` placeholder contains stories #9-16 (unfeatured). Both sit inside their own `.reader-timeline` wrapper which draws the vertical spine.

### Trending repo item format

Each trending repo in the sidebar should follow this structure:

```html
<a class="trending-repo" href="https://github.com/{{OWNER}}/{{REPO}}" target="_blank">
  <div class="trending-repo-header">
    <span class="trending-repo-stars">★ {{STARS_TODAY}} today</span>
    <span class="lang-badge">{{LANGUAGE}}</span>
  </div>
  <div class="trending-repo-name">{{OWNER}}/{{REPO}}</div>
  <div class="trending-repo-desc">{{DESCRIPTION}}</div>
  <div class="trending-repo-meta">{{TOTAL_STARS}} total · Created {{RELATIVE_DATE}}{{CROSS_SIGNAL}}</div>
</a>
```

Where `{{CROSS_SIGNAL}}` is optionally ` · Also on HN` or ` · Shared by @handle` if the repo was found by another agent too.

### Twitter link item format

Each secondary shared link in the Twitter section middle column:

```html
<a class="twitter-link-item" href="{{URL}}" target="_blank">
  <div class="twitter-link-title">{{HEADLINE}}</div>
  <div class="twitter-link-shared">Shared by {{HANDLES}}</div>
  <div class="twitter-link-source">{{SOURCE}} · {{RELATIVE_TIME}}</div>
</a>
```

### Twitter quote item format

Each notable tweet quote in the Twitter section right column:

```html
<div class="twitter-quote">
  <div class="twitter-quote-handle">@{{HANDLE}}</div>
  <div class="twitter-quote-text">"{{QUOTE_TEXT}}"</div>
  <div class="twitter-quote-context">on {{TOPIC}} · {{RELATIVE_TIME}}</div>
</div>
```

### GitHub card item format

Each repo card in the GitHub section (4-column grid):

```html
<a class="github-card" href="https://github.com/{{OWNER}}/{{REPO}}" target="_blank">
  <div class="github-card-header">
    <span class="github-card-stars">★ {{STARS_TODAY}} today</span>
    <span class="github-card-lang">{{LANGUAGE}}</span>
  </div>
  <div class="github-card-name">{{OWNER}}/{{REPO}}</div>
  <div class="github-card-desc">{{DESCRIPTION}}</div>
  <div class="github-card-meta">{{TOTAL_STARS}} total · Created {{RELATIVE_DATE}}</div>
  <div class="github-card-signal">{{CROSS_SIGNAL}}</div>
</a>
```

Where `{{CROSS_SIGNAL}}` is optionally "Also on HN · 342 pts" or "Shared by @karpathy" or empty string if no cross-signal.

### Reader view Twitter story format

Each Twitter-sourced story in the reader view uses the standard `reader-story` format with Twitter attribution in the source line:

```html
<a class="reader-story" href="{{URL}}" target="_blank">
  <div class="reader-story-content">
    <div class="reader-story-header">
      <span class="reader-story-title">{{HEADLINE}}</span>
      <span class="reader-story-date">{{DATE}}</span>
    </div>
    <div class="reader-story-lede">{{SHORT_SUMMARY}}</div>
    <div class="reader-story-source">Shared by {{HANDLES}} · {{SOURCE}}</div>
  </div>
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
- **stockanalysis.com first**: Always use stockanalysis.com for individual stock prices — Google Finance renders via JavaScript and fails with WebFetch. Use WebSearch for index values (S&P 500, NASDAQ) and Bitcoin.
- **Image fallbacks**: If an og:image URL returns a 404 or can't be found, try searching for an alternative article on the same story that has one.
- **File location**: Always write to `~/Desktop/news-briefing.html` unless the user specifies otherwise.
- **Nav brand**: Use "The AI Briefing" for tech/AI topics. For other topics, adapt the name (e.g., "The Crypto Briefing", "The Climate Briefing").
- **Twitter/X sourcing**: Agent 5 uses WebSearch with `site:x.com` queries — this is the most reliable method since Google indexes popular tweets quickly. Do NOT try to use the Twitter API or scrape X directly. If WebSearch returns no useful tweets for an account, skip that account gracefully.
- **GitHub trending**: Agent 6 fetches GitHub trending pages which are public and reliably fetchable. Filter aggressively — most trending repos are not newsworthy. Only surface repos that match user interests AND show strong signals (high stars today, genuinely new, or cross-signal with other agents).
- **Twitter account discovery**: Phase 0.6 runs once and caches accounts in profile.json. The discovery combines web search (what the community recommends following), GitHub profiles (Twitter handles of repo authors the user has starred), and domain expertise. This avoids hardcoding accounts and keeps the list personalized to the user's actual interests.
- **Source attribution with Twitter**: When a story was shared by high-signal Twitter accounts, always include their handles in the meta line. This gives users a social proof signal ("@karpathy shared this") and helps them discover accounts worth following.
