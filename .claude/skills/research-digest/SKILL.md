---
name: research-digest
description: >-
  Run a time-sensitive, daily-fresh web research sweep on a configurable topic
  and publish a self-contained HTML digest to a GitHub repo. Each run does 75
  web searches + 25 Firecrawl crawls, synthesizes the top 5 findings (one
  citation each) into a clean HTML page with a visual JavaScript link-checker,
  and commits it to a topic folder as `<topic>-yyyy-MMM-dd.html`. Maintains a
  persistent topics knowledge base so the same finding is never reported twice.
  Use this skill whenever the user wants a daily/weekly research briefing, a
  scheduled "digest," a topic tracked over time and written to their repo or
  GitHub Pages, or any recurring unattended task that should scan the web, dedup
  against past runs, and publish HTML with citations. Trigger it even when the
  user just says "keep a daily digest of X in my repo," "research Y every
  morning and publish it," or "build me a research digest" — do not wait for the
  skill to be named.
---

# Research Digest

Produce a fresh, deduplicated, publish-ready HTML research digest for a single
configurable topic, every time it runs. Freshness and non-repetition are the
whole point: a digest that surfaces yesterday's news, or repeats a finding from
a prior run, has failed even if it looks polished.

## Configuration

Read these from the user's request (or a `config.yaml` at the repo root if one
exists). Ask only for what's missing; infer sensible defaults otherwise.

| Key | Meaning | Default |
| --- | --- | --- |
| `topic` | The research subject, e.g. `ai-safety` | (required) |
| `repo` | `owner/name` GitHub repo to publish into | (required) |
| `topic_folder` | Folder in the repo for this topic's pages | slugified `topic` |
| `kb_path` | Path to the topics knowledge base file | `<topic_folder>/knowledge-base.md` |
| `freshness_hours` | How recent a source must be to count | `24` |
| `search_count` | Web searches per run | `75` |
| `crawl_count` | Firecrawl crawls per run | `25` |

Slugify topics for paths (lowercase, hyphens): `AI Safety` → `ai-safety`.

## Workflow

Run these steps in order. Each run is self-contained and unattended — assume no
human is watching, so fail loudly in the digest itself rather than silently
dropping findings.

### Step 1 — Scan the web (time-sensitive)

Goal: gather a wide, *recent* pool of candidate findings. Enforce freshness at
the query level, not just by filtering afterward — most of the budget is wasted
if you search broadly and then throw away stale results.

1. **75 web searches** using `firecrawl_search` (preferred — it returns
   full-page content and supports `tbs`/recency and `sources: news`). Fall back
   to the native `WebSearch` tool if Firecrawl is unavailable.
   - Bias every query toward the last `freshness_hours`. Append date-scoping
     terms (`today`, `this week`, the current ISO date) and use recency filters
     where the tool supports them.
   - Vary angles so the 75 searches aren't near-duplicates: news, releases,
     papers, discussion (forums/social), primary sources, and contrarian takes.
     Rotate query templates rather than issuing 75 rephrasings of one question.
   - After a `firecrawl_search`, call `firecrawl_search_feedback` with the
     search ID (it improves quality and refunds a credit).
2. **25 Firecrawl crawls** using `firecrawl_crawl` (or `firecrawl_scrape` for a
   single page) against the most promising domains surfaced in step 1 — the
   primary sources behind the search snippets, changelogs, docs, and index
   pages worth expanding. Prefer crawling the *origin* of a claim over a
   secondary rewrite of it.
3. Collect every candidate as a record: `{title, url, published_at, source,
   one_line_summary}`. Drop anything older than `freshness_hours` or without a
   verifiable date.

Keep raw notes in the scratchpad; only the final 5 reach the digest.

### Step 2 — Deduplicate against the knowledge base

Load `kb_path` from the repo (see format below). Discard any candidate whose
story you've already covered — match on normalized URL first, then on semantic
overlap of the finding (same event/paper/release described differently). The
knowledge base is the memory that keeps daily runs from rehashing themselves; a
finding that survives here must be genuinely new relative to every prior run.

If deduplication leaves fewer than 5 new findings, that's fine — publish what's
genuinely new and say so in the digest rather than padding with old material.

### Step 3 — Synthesize the HTML digest

Pick the **top 5** new findings by significance and recency. For each, write a
tight synthesis (2–4 sentences) and attach **exactly one** best citation — the
most authoritative primary URL, not a search results page.

**Validate each citation at generation time** — this is the source of truth for
link status. Probe each URL with `firecrawl_scrape` (or `WebFetch`); mark it
`valid` on a 2xx/served response, `broken` otherwise. Bake that status into the
HTML as a `data-status` attribute. The in-page JavaScript then re-checks links
live in the reader's browser, but because browsers can't read cross-origin HTTP
status (CORS returns opaque responses), the JS only *upgrades confidence or
flags a timeout* — it never overrides a `broken` you already proved. When the
browser can't determine status, it shows `unknown`, not a false `valid`.

Generate one self-contained `.html` file from this template (inline all CSS/JS;
no external assets so it renders anywhere, including GitHub Pages):

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{{TOPIC}} — Research Digest {{DATE_HUMAN}}</title>
<style>
  :root { color-scheme: light dark; }
  body { font: 16px/1.6 system-ui, sans-serif; max-width: 760px; margin: 2rem auto; padding: 0 1rem; }
  h1 { margin-bottom: .25rem; }
  .meta { color: #6b7280; font-size: .9rem; margin-bottom: 2rem; }
  .finding { border-top: 1px solid #e5e7eb; padding: 1.25rem 0; }
  .finding h2 { font-size: 1.15rem; margin: 0 0 .5rem; }
  .cite { font-size: .9rem; word-break: break-all; }
  .status { display: inline-block; min-width: 5.5rem; text-align: center;
            font-size: .72rem; font-weight: 600; padding: .12rem .5rem;
            border-radius: 999px; margin-left: .4rem; vertical-align: middle; }
  .status[data-state="valid"]   { background:#dcfce7; color:#166534; }
  .status[data-state="broken"]  { background:#fee2e2; color:#991b1b; }
  .status[data-state="unknown"] { background:#f3f4f6; color:#4b5563; }
  .status[data-state="checking"]{ background:#fef9c3; color:#854d0e; }
</style>
</head>
<body>
  <h1>{{TOPIC_HUMAN}} — Daily Research Digest</h1>
  <p class="meta">{{DATE_HUMAN}} · {{N_FINDINGS}} new findings · freshness window {{FRESHNESS_HOURS}}h</p>

  <!-- Repeat this block for each of the top-5 findings -->
  <article class="finding">
    <h2>{{FINDING_TITLE}}</h2>
    <p>{{FINDING_SYNTHESIS}}</p>
    <p class="cite">
      <a href="{{CITATION_URL}}" data-status="{{VALIDATED_STATUS}}">{{CITATION_URL}}</a>
      <span class="status" data-state="{{VALIDATED_STATUS}}">{{VALIDATED_STATUS}}</span>
    </p>
  </article>

<script>
// Live re-check. Generation-time status (data-status) is authoritative:
// a proven "broken" is never overridden. Cross-origin CORS means the browser
// often can't read real status, so undetermined links show "unknown".
(function () {
  document.querySelectorAll('a[data-status]').forEach(function (a) {
    var badge = a.parentElement.querySelector('.status');
    if (a.dataset.status === 'broken') return;      // trust build-time proof
    badge.dataset.state = 'checking'; badge.textContent = 'checking';
    var done = false;
    var t = setTimeout(function () {
      if (done) return; done = true;
      badge.dataset.state = 'unknown'; badge.textContent = 'unknown';
    }, 6000);
    fetch(a.href, { method: 'HEAD', mode: 'no-cors' })
      .then(function () {                            // opaque = reachable, not readable
        if (done) return; done = true; clearTimeout(t);
        badge.dataset.state = 'valid'; badge.textContent = 'valid';
      })
      .catch(function () {
        if (done) return; done = true; clearTimeout(t);
        badge.dataset.state = 'broken'; badge.textContent = 'broken';
      });
  });
})();
</script>
</body>
</html>
```

### Step 4 — Publish to GitHub

Commit the file into the repo under the topic folder, named with the run date:

```
<topic_folder>/<topic>-yyyy-MMM-dd.html
```

`yyyy-MMM-dd` uses a three-letter month, e.g. `ai-safety-2026-Jul-23.html`.
One file per run — never overwrite a prior day's digest; the folder is an
archive of the topic over time.

Use the GitHub MCP tools: `mcp__github__create_or_update_file` for the single
HTML page, or `mcp__github__push_files` to commit the page and the updated
knowledge base together in one commit (preferred — keeps them in sync). Commit
message: `digest(<topic>): <yyyy-MMM-dd>`.

### Step 5 — Update the knowledge base

Append every finding you published to `kb_path` so the next run dedups against
it. Commit it in the same push as the HTML.

## Knowledge base format

Markdown with a YAML block per run — human-readable, greppable, and easy to
diff. Match new candidates against `url` and `finding` fields here in Step 2.

```markdown
# Knowledge Base — {{TOPIC_HUMAN}}

Covered findings. Each run appends its top-5 so they are never reported twice.

## 2026-Jul-23
```yaml
- finding: "OpenReview mandates reproducibility statements for 2026 submissions"
  url: "https://example.org/announcement"
  published_at: "2026-07-23"
  status: valid
- finding: "New interpretability benchmark released covering 40 model families"
  url: "https://example.org/benchmark"
  published_at: "2026-07-22"
  status: valid
```
```

## Running unattended / on a schedule

This skill is built to run in a scheduled cloud task or remote routine with no
human present. When set up on a schedule, each firing should: re-read config,
run Steps 1–5 end to end, and commit. If a step yields nothing new, still
publish a short digest noting "no new findings in the last {{FRESHNESS_HOURS}}h"
so the archive shows the run happened — silence is indistinguishable from
breakage.
