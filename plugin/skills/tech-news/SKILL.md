---
name: tech-news
description: "Summary of the day's dev news: PHP, Laravel and the web development ecosystem (frameworks, tools, best practices), with AI applied to development as a complement. Aggregates the latest articles from multiple sources (blogs, Reddit, RSS feeds) to provide comprehensive, readable tech watch."
argument-hint: "[--format=inline|markdown|html]"
disable-model-invocation: true
allowed-tools: WebFetch(*), WebSearch(*), Read(*), Write(*), Bash(open *), Bash(xdg-open *), Bash(wslview *), Bash(start *)
---

# Daily Tech Watch

## Output language

Produce all user-facing content (messages, generated documents, titles, descriptions, commit messages, changelog, summaries) in the user's language: match the language the user writes in (French or English). If undetermined, default to English. These instructions are written in English and do not dictate the output language.

Retrieve and summarize recent dev news (last 24-48h) on PHP/Laravel and the development ecosystem, with AI applied to development as a complement.

## Sources

Consult **all** the sources below without asking the user for confirmation. Chain the reads automatically.

### Primary sources (direct feeds via WebFetch)

These sources are accessible directly and should be consulted first:

| Source | URL | Theme |
|--------|-----|-------|
| Laravel News | https://laravel-news.com/feed/json | Laravel |
| Laravel Blog | https://laravel.com/blog | Official Laravel |
| Laravel Daily | https://laraveldaily.com | Laravel ecosystem |
| Stitcher.io | https://stitcher.io | Modern PHP |
| PHP.net | https://www.php.net/ | Official PHP |
| Symfony Blog | https://symfony.com/blog/ | PHP / Symfony |
| GitHub Blog | https://github.blog/feed/ | Dev / Tools |
| JetBrains Blog | https://blog.jetbrains.com/feed/ | Dev / IDE |
| Google DeepMind Blog | https://deepmind.google/discover/blog/ | AI / Research |
| Hacker News | https://news.ycombinator.com | General tech |
| JoliCode | https://jolicode.com/feed | PHP / Web |
| Les-Tilleuls.coop | https://les-tilleuls.coop/feed.xml | PHP / Web / Cloud |
| Jérémy Decool | https://www.jdecool.fr/feed.xml | PHP / Web |
| Pascal Martin | https://blog.pascal-martin.fr/index.xml | PHP / DevOps |
| Le Code est dans le Pré | https://lecodeestdanslepre.fr/feed.xml | PHP / Web |
| Makina Corpus | https://makina-corpus.com/rss | PHP / Web |
| Olivier Sauvage (Numerika) | https://oliviersauvage.com/feed/ | AI |
| Les news IA de Meimei | https://joinmeimei.substack.com/feed | AI |
| Mind Upskiller | https://upskiller.substack.com/feed | AI |
| Standblog (Tristan Nitot) | https://standblog.org/blog/feed/atom | AI / Web |
| Eleven Labs | https://blog.eleven-labs.com/feed.xml | Dev / Web / DevOps |
| Ippon Technologies | https://blog.ippon.fr/feed/ | Dev / Cloud / AI |
| OCTO Talks | https://blog.octo.com/feed | Dev / Cloud / AI |
| Zenika | https://blog.zenika.com/feed/ | Dev / Web / AI |
| EventuallyCoding | https://eventuallycoding.com/rss.xml | Dev / AI |
| Quoi de neuf les devs ? | https://happytodev.substack.com/feed | Dev / Web / AI |

### Secondary sources (Reddit via WebSearch)

Reddit blocks direct calls via WebFetch. Use **WebSearch** to find the trending topics of each subreddit:

| Subreddit | WebSearch query | Theme |
|-----------|-------------------|-------|
| r/laravel | `reddit r/laravel trending this week` | Laravel |
| r/PHP | `reddit r/PHP trending this week` | PHP |
| r/LLMDevs | `reddit r/LLMDevs latest news` | AI / LLM |
| r/LocalLLaMA | `reddit r/LocalLLaMA latest news` | Local LLMs |

### Complementary sources (WebSearch if primary sources lack content)

Run targeted WebSearch queries to fill in gaps:

- `Laravel news this week <month> <year>`
- `PHP news announcements <month> <year>`
- `PHP Annotated JetBrains <month> <year>`
- `AI coding tools / LLM dev news <month> <year>`
- `Hacker News top stories tech <month> <year>`

## Workflow

### 1. Collect — Primary sources

Run the WebFetch calls in parallel across all primary sources. For each feed:
- Identify articles from the last 24-48h
- Extract title, URL, publication date

### 2. Collect — Reddit sources

Run the WebSearch queries in parallel for each subreddit. The search results provide the titles and URLs of the most popular recent discussions.

### 3. Collect — Full article content

For each relevant article/post identified in steps 1 and 2:

**Try WebFetch on the article URL.** If WebFetch fails (blocked site, paywall, 403):
1. Try WebFetch on an alternative source covering the same topic (blog, tech aggregator)
2. If still inaccessible, use WebSearch to find other articles on the same topic and retrieve their content
3. As a last resort, write the summary from the available information (title, excerpt, search results), noting that the full content was not accessible

The goal is to have enough material to write a complete summary, not just a title.

### 4. Collect — Supplements if needed

If the primary sources and Reddit don't provide enough content, run the complementary WebSearch queries to find articles on the day's topics.

### 5. Filtering

Exclude:
- Posts older than 48h
- Basic help questions ("how do I do X?", "my code doesn't work")
- Posts with very little engagement (< 5 upvotes on Reddit)
- Duplicates or redundant topics across sources (keep the most complete source)
- Job postings, self-promotion, memes
- Articles behind an inaccessible paywall with no alternative content

Keep:
- New version announcements (Laravel, PHP, Symfony, packages, dev tools)
- Quality technical articles (tutorials, feedback, benchmarks)
- Relevant discussions with strong participation
- Major ecosystem news

### 6. Writing the summary

Write in the user's language (see Output language). Organize by topic, not by source.

## Output options

Read `$ARGUMENTS` to detect the requested format. Possible values are:
- `--format=html` **(default if no argument)**: generates a standalone HTML file and opens it in the browser
- `--format=markdown`: generates a `.md` file and displays the path
- `--format=inline`: displays directly in the prompt (legacy behavior)

### HTML mode (default)

1. Write the content in markdown format as described in "Output format"
2. Convert this markdown into structured HTML (h1/h2/h3 headings, paragraphs, blockquotes, code blocks, links, hr)
3. Inject the HTML into the template below
4. Write the file `actu-tech-YYYY-MM-DD.html` in the current directory with the Write tool
5. Open the file in the browser according to the system: `open <path>` (macOS), `xdg-open <path>` (Linux), `wslview <path>` (WSL) or `start <path>` (Windows). Choose the available command; if none, simply display the file path.
6. Confirm to the user that the file has been generated and opened

**HTML template:** Read the file `assets/actu-tech.html` (relative to this skill) with the Read tool, then replace the placeholders.

**Markdown → HTML conversion rules:**
- `## Title` becomes `<h2>Title</h2>`
- `### Title` becomes `<div class="article-card"><h3>Title</h3>` (close the previous card if needed with `</div>`)
- Paragraphs become `<p>...</p>`
- `> Source : [name](url) (Xx)` becomes `<blockquote>Source : <a href="url">name</a> <span class="lang-badge lang-xx">Xx</span></blockquote></div>` (also closes the card). `xx` is the lowercase language code (`fr` or `en`), `Xx` is the displayed label (`Fr` or `En`)
- Code blocks `` ``` `` become `<pre><code>...</code></pre>`
- `inline code` becomes `<code>...</code>`
- `---` becomes `<hr>` (except inside a card)
- Links `[text](url)` become `<a href="url">text</a>`
- `**bold**` becomes `<strong>bold</strong>`
- Replace `{{DATE}}` with the current date in `DD month YYYY` format (e.g., `04 March 2026`)
- Replace `{{CONTENT}}` with the converted HTML of the articles
- Replace `{{SOURCES}}` with the list of consulted sources

### Markdown mode

1. Write the content in markdown format as described in "Output format"
2. Write the file `actu-tech-YYYY-MM-DD.md` in the current directory with the Write tool
3. Display the path of the generated file to the user

### Inline mode

Display the summary directly in the prompt, like the legacy behavior. No file is created.

## Output format

Each article is presented as a standalone, complete block. The reader should not need to click the link to understand the topic — the link is there only if they want to dig deeper.

```
# Actu Tech — <current date>

---

## PHP / Laravel

### <Clear and meaningful title>

<Complete summary of the article: context, main content, key points, what to remember.
Multiple paragraphs if needed to cover the topic properly.
Include the important technical details (versions, features, changes),
the concrete examples mentioned in the article, and the practical implications.>

> Source : [<source name>](<url>) (Fr)

---

### <Next title>

<Complete summary...>

> Source : [<source name>](<url>) (En)

---

## Development / Tools

### <Title>

<Complete summary...>

> Source : [<source name>](<url>) (En)

---

## AI applied to dev

### <Title>

<Complete summary...>

> Source : [<source name>](<url>) (Fr)

---

*Sources consulted: Laravel News, Laravel Blog, Laravel Daily, Stitcher.io, PHP.net, PHP Annotated, Symfony Blog, GitHub Blog, JetBrains Blog, DeepMind Blog, Hacker News, r/laravel, r/PHP, r/LLMDevs, r/LocalLLaMA*
```

## Writing rules

- Write in the user's language (see Output language)
- **Complete summary**: cover the topic in depth so the reader doesn't need to read the original article. Include the context, key points, examples and implications
- Stay factual, no personal opinion
- Use an informative and accessible tone, avoid unnecessary jargon
- Structure long summaries with distinct paragraphs to make them easier to read
- If an article mentions relevant code or commands, include them in the summary
- If no relevant article is found for a category, do not display the category
- If no relevant article is found at all, state it clearly: "No notable news today"
- **Language indicator**: add `(Fr)` or `(En)` after the source link depending on the language of the original article. Determine the language when reading the article content
