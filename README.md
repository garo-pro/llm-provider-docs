# LLM Provider Docs

> Auto-updating archive of AI provider builder documentation: every source
> that publishes markdown, fetched 4x daily, across Anthropic, OpenAI, and
> Z.AI so far.

[![fetch](https://github.com/garo-pro/llm-provider-docs/actions/workflows/fetch-docs.yml/badge.svg)](https://github.com/garo-pro/llm-provider-docs/actions/workflows/fetch-docs.yml)
[![license](https://img.shields.io/github/license/garo-pro/llm-provider-docs)](LICENSE)

Clone this repo and point Claude Code (or any coding agent) at it. Every doc,
tutorial, cookbook, SDK reference, and engineering post these providers
publish on a markdown surface -- searchable, version-controlled, and offline.

## Install

```bash
git clone https://github.com/garo-pro/llm-provider-docs
cd llm-provider-docs
```

Then ask your agent anything:

```bash
claude "how does the agent loop work?"
claude "show me the MCP spec for tool annotations"
claude "what's the difference between OpenAI's Responses API and Chat Completions?"
claude "how do I call GLM-5.3 with the OpenAI-compatible SDK?"
```

## Content

Counts are files on disk as of 2026-09-17; `uv run scripts/fetcher.py --tree`
prints them live.

### Anthropic -- `content/anthropic/` (see `sources.anthropic.json`)

| Source | Section | Files | What |
|--------|---------|------:|------|
| code.claude.com | `--section claude-code` | 198 | Claude Code + Agent SDK docs |
| platform.claude.com | `--section api` | 2,228 | API reference, build guides |
| claude.com/docs | `--section products` | 226 | Claude Tag, Cowork, office agents, connectors |
| modelcontextprotocol.io | `--section mcp` | 347 | MCP spec, SDKs, governance |
| github.com/anthropics | `--section github` | 764 | Cookbooks, skills, plugins, courses, SDK docs |
| support.claude.com | `--section support` | 372 | Help articles |
| anthropic.com | `--section blog` | 449+ | Engineering, research, news, standalone policy pages -- see below |

`anthropic.com` has no `.md` variant of anything (HTML-only). The fetcher
scrapes it: sitemap discovery for `/news/`, `/research/`, `/engineering/`,
plus a fixed allowlist (`BLOG_STANDALONE_PAGES` in `fetcher.py`) for
standalone pages that live at the site root -- the constitution, responsible
scaling policy, transparency reports, threat intelligence reports, economic
index/futures, system cards. HTML is converted to markdown with
[trafilatura](https://trafilatura.readthedocs.io/); anthropic.com's pages are
server-rendered, so no headless browser is needed.

### OpenAI -- `content/openai/` (see `sources.openai.json`)

| Source | Section | Files | What |
|--------|---------|------:|------|
| developers.openai.com | `--section openai` | ~1,320 | API guides+reference, Codex, Cookbook, Ads, Plugins/Apps SDK, Workspace Agents, Agentic Commerce, dev blog, learning resources, showcase |
| openai.com | `--section openai` | ~70-80 | Model launches ("release" sitemap category), scraped via curl |
| github.com/openai | `--section openai` | ~480 | openai-cookbook, openai-python, openai-node |

developers.openai.com is the successor to the old `platform.openai.com/docs`
path (now just a redirect shell) -- a single sitemap covers the whole site,
and every page serves a `.md` variant directly. `learn.chatgpt.com` (ChatGPT
product docs, linked from the root `llms.txt` hub) is a known gap: a separate
domain, not yet added.

`openai.com` (the marketing/news domain -- a different site from
developers.openai.com) is where model launches actually publish, under
`/index/<slug>`. No `.md` variant, no `llms.txt`. Its Cloudflare bot-check
blocks `aiohttp` outright but lets `curl` with a browser User-Agent through
roughly half the time, so this source is fetched via a `curl` subprocess
(retried up to 6x) and converted with [trafilatura](https://trafilatura.readthedocs.io/),
the same approach used for anthropic.com. `robots.txt` allows crawling
(`Allow: /`) and publishes the sitemap this uses. Output lands in
`content/openai/news/`, kept separate from `content/openai/blog/` (the
developers.openai.com dev blog).

### Z.AI -- `content/zai/` (see `sources.zai.json`)

| Source | Section | Files | What |
|--------|---------|------:|------|
| docs.z.ai | `--section zai` | 68 | GLM model guides, API reference, SDKs (Python/Java/OpenAI-compat/LangChain), devpack, release notes |
| github.com/zai-org | `--section zai` | ~26 | GLM-skills |

docs.z.ai's `llms.txt` lists direct `.md` links for every page (same shape as
code.claude.com's), and its `robots.txt` explicitly welcomes AI scraping
(`Content-Signal: ai-train=yes`).

`z.ai/blog` (the marketing site's announcement blog, e.g.
`z.ai/blog/glm-built-its-inference-infrastructure`) is a known gap, and a
harder one than openai.com's: its `sitemap.xml` has zero blog URLs (26
pages, all account/billing) and there's no feed or JSON post index, so
there's no discovery surface at all. Worse, the posts are client-side-
rendered SPA pages -- the raw HTML is an empty `<div id="root">` plus a JS
bundle, so there's no server-rendered text for trafilatura even once
fetched (no Cloudflare wrinkle here; plain curl gets a clean 200 every
time). Fixing this needs a headless browser, which this fetcher doesn't
have. `docs.z.ai/release-notes` was checked against `z.ai/changelog`,
`/updates`, `/news` for a richer separate surface -- all 404, so the
existing release-notes coverage is already complete.

```
content/
  anthropic/
    en/docs/claude-code/   Claude Code + Agent SDK
    en/api/                API reference (1,500+ endpoints)
    en/build-with-claude/  Platform features
    en/agents-and-tools/   Tool use, agent skills
    en/manage-claude/      Admin, billing, managed agents
    claude/                Product docs (Claude Tag, Cowork, office agents)
    mcp/                   MCP protocol spec + community
    blog/                  anthropic.com, scraped (no .md variant upstream)
      engineering/         Building Effective Agents, context engineering, ...
      research/            Research papers
      news/                Model releases
      product/             Product announcements
      policy/              Constitution, RSP, transparency, threat intel, ...
      alignment/           Alignment Science blog
      interpretability/    Transformer Circuits Thread
    github/
      cookbooks/           164 recipes + notebooks
      skills/              90 official Agent Skills
      plugins-official/    266 plugin docs
      courses/             80 prompt engineering notebooks
      quickstarts/         Deployable app starters
      code-action/         GitHub Actions for Claude Code
      sdk-python/          Python SDK reference
      sdk-typescript/       TypeScript SDK reference
    support/               365 help articles
  openai/
    api/                   API guides + endpoint reference
    codex/                 Codex CLI, IDE, cloud, config
    cookbook/              Practical code examples
    ads/, plugins/, workspace-agents/, commerce/, blog/, learn/, showcase/
    news/                  Model launches (openai.com, scraped via curl)
    github/
      cookbook/            openai/openai-cookbook
      openai-python/       Python SDK repo (docs, examples)
      openai-node/         Node SDK repo (docs, examples)
  zai/
    guides/                GLM model guides, quick start, pricing
    api-reference/         API reference
    devpack/                Dev tooling
    github/
      GLM-skills/           Markdown skill definitions
```

## Fetching

Auto-updates four times daily via GitHub Actions. To fetch manually:

```bash
# Requires: uv (https://docs.astral.sh/uv/)
uv run scripts/fetcher.py                    # Fetch everything (all providers)
uv run scripts/fetcher.py --section mcp      # Anthropic MCP spec only
uv run scripts/fetcher.py --section github   # Anthropic GitHub repos only
uv run scripts/fetcher.py --section openai   # OpenAI docs + repos
uv run scripts/fetcher.py --section zai      # Z.AI docs + repos
uv run scripts/fetcher.py --incremental      # Skip existing files
uv run scripts/fetcher.py --tree             # Show all sources + counts
uv run scripts/fetcher.py --discover         # Probe Anthropic domains for new sources
```

GitHub repo fetching needs `GITHUB_TOKEN` or `GH_TOKEN` in the environment.
Every fetched source across all three providers serves a `.md` variant of
each page except anthropic.com, which is HTML-only and gets scraped and
converted with trafilatura instead (see `content/anthropic/blog/` above).

See [`sources.anthropic.json`](sources.anthropic.json),
[`sources.openai.json`](sources.openai.json), and
[`sources.zai.json`](sources.zai.json) for the complete machine-readable
source registries -- one per provider.

## Source Discovery

The fetcher doesn't just download from hardcoded URLs. For Anthropic, it
probes every known domain for `robots.txt`, `sitemap.xml`, `llms.txt`, and --
the question that decides everything -- whether the domain serves `.md`
variants at all. It also enumerates `github.com/anthropics` and watches the
`Location` header on every redirect it follows. (`--discover` is Anthropic-only
tooling; OpenAI and Z.AI sources were hand-verified instead -- see each
registry's `notes_on_discovery`.)

Known Anthropic domains: `anthropic.com`, `platform.claude.com`,
`code.claude.com`, `support.claude.com`, `modelcontextprotocol.io`,
`claude.ai`, `claude.com`, `academy.claude.com`

**The result is a file, not a log line.** Every full run rewrites
[`discovery.json`](discovery.json) with what exists upstream that
`sources.anthropic.json` does not. This matters because printing it did not
work: the pipeline had been logging `support.claude.com -> academy.claude.com`
four times a day for weeks, into an Actions log nobody opens, and those 725
pages were eventually found by a human chasing a dead support article. A
discovery that isn't a diff doesn't reach anyone.

So a new domain, a new `anthropics` repo, or a domain that starts serving
markdown now shows up as a tracked change, gets classified as high-signal, and
is committed straight to main with a heads-up push notification -- the same
path a new doc takes. `discovery.json.review` is the actionable list:
reachable, serves markdown, nothing fetches it. Empty is healthy. Adding a
source stays a human decision.

```bash
uv run scripts/fetcher.py --discover   # manual probe; writes the same file
```

**Sitemaps are treated as incomplete, not authoritative.** Upstream de-indexes
pages it still serves: in July 2026 platform.claude.com dropped every
per-language SDK reference page from both its sitemap and its `llms.txt` while
continuing to edit them, and the archive quietly stopped refreshing 1,560 files
for seven weeks. So every full run also refetches what is already on disk
(across all three providers), and pages that really died are removed by the
reaper below rather than by absence from an index.

### Reaping

A page removed upstream used to live here forever -- the fetcher only ever added
or overwrote. Full runs now delete archived files whose URL returns 404/410 or
the site's HTML shell, with two guardrails:

- **Only markup is deleted automatically.** A file holding real markdown whose
  URL has died is content the provider removed and we may hold the only copy; it
  is reported for a human instead of destroyed by a job that commits straight
  to main.
- **A mass-deletion circuit breaker.** More than 200 pages vanishing at once
  means an upstream outage, not 200 real deletions -- nothing is deleted and the
  run fails loudly.

`--no-reap` reports what would go without touching anything.

### Tombstones

`tombstones.json` records every URL confirmed gone upstream, across all
providers, with the date and reason. It exists so that a page which died once
does not report as a fresh failure on every subsequent run. Later runs count
known deaths quietly and print only what changed: pages newly gone, and pages
that came *back* (whose tombstone is then removed automatically). The success
rate is computed over live docs, so it means something.

## Automation

One GitHub Actions workflow powers this repo:

**[fetch-docs.yml](.github/workflows/fetch-docs.yml)** -- Scheduled four times
daily. Runs the fetcher across all providers, then classifies the diff and
commits straight to main either way: deletions or a
`tombstones.json`/`discovery.json` change are classified high-signal and
trigger an optional push notification via
[barkme](https://github.com/nickchou/barkme-mcp-server) as a heads-up;
everything else (including routine version/CHANGELOG bumps) commits quietly
as a minor freshness update. Needs no repo secrets -- it runs on the default
`GITHUB_TOKEN`.

## Contributing

PRs welcome. The fetcher is a single Python file (`scripts/fetcher.py`)
with no framework dependencies beyond `aiohttp` and `aiofiles`.

To add a new provider or source:
1. Add a source definition to that provider's registry (or create a new
   `sources.<provider>.json` for a brand-new provider)
2. Add the fetch logic to `scripts/fetcher.py` (output path mapping in
   `get_output_path()`, a block in `fetch_all()`, a provider-scoped `_dir`
   in `__init__`)
3. Run `uv run scripts/fetcher.py --section <name>` to test
4. Add `<name>` to `--section` choices in the CLI

## Disclaimer

Unofficial mirror for educational and development purposes. Documentation
content is sourced from each provider's public sites: Anthropic
([code.claude.com](https://code.claude.com), [platform.claude.com](https://platform.claude.com)),
OpenAI ([developers.openai.com](https://developers.openai.com)), and Z.AI
([docs.z.ai](https://docs.z.ai)). Repository code from
[anthropics/claude-code](https://github.com/anthropics/claude-code).
Redistribution should comply with each provider's own terms.

## License

[MIT](LICENSE)
