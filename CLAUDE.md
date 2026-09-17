# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Comprehensive archive of builder documentation across multiple AI providers,
all auto-updated four times daily (see Fetcher):

- **Anthropic** (`content/anthropic/`) -- 4,000+ docs from 14 sources.
  Originally the whole repo (see below for the full source list); still the
  largest and most mature part of the archive.
- **OpenAI** (`content/openai/`) -- ~1,800 docs + repos from
  developers.openai.com and github.com/openai. Added 2026-09-17.
- **Z.AI** (`content/zai/`) -- ~94 docs + repos from docs.z.ai (GLM models)
  and github.com/zai-org. Added 2026-09-17.

Each provider has its own source registry (`sources.<provider>.json`) and its
own subtree under `content/`. `scripts/fetcher.py` is a single file that
fetches all of them; adding a provider means adding fetch logic there, not
switching frameworks -- see "When adding new sections" below.

## Fetcher

`scripts/fetcher.py` -- single-file multi-source, multi-provider fetcher.

### Anthropic (see `sources.anthropic.json`)

Sources: code.claude.com, platform.claude.com, claude.com/docs,
modelcontextprotocol.io, support.claude.com (sitemap + .md),
github.com/anthropics/* (10 repos). anthropic.com blog
(engineering/research/news, plus a fixed allowlist of standalone policy/
report pages) was a FROZEN archive from 2026-07 to 2026-09: the site is
HTML-only with no `.md` variant, and the jina.ai proxy path had been
removed. Unfrozen once trafilatura-based HTML scraping replaced it --
anthropic.com's pages turn out to be server-rendered (no JS execution
needed to get real body text out of them). The same trafilatura path also
covers two more Anthropic-run research blogs added 2026-09-13:
alignment.anthropic.com and transformer-circuits.pub -- neither has a
`.md` variant either, and neither has a sitemap, so each needed its own
discovery method (a homepage that lists every post, and an Atom feed,
respectively).

Five rules keep the archive honest, all learned the hard way:

- **Discovery surfaces are incomplete.** Sitemaps and llms.txt undercount what
  a site serves, so every full run also refetches what is already on disk.
  Without this, 1,560 de-indexed-but-live API pages sat stale for seven weeks.
- **The archive must be able to shrink.** Full runs reap files whose URL is
  gone (404/410 or HTML shell). Only markup is deleted automatically; real
  markdown that died upstream is reported for a human. A >200-file reap is
  refused outright as an upstream outage.
- **A redirect means the content belongs elsewhere.** If a fetch lands on a
  different path, the body in hand is the *target's*; writing it back to the
  requested path misattributes it. That is how the 471KB
  `release-notes/system-prompts.md` briefly became the 3.7KB overview it now
  redirects to, after upstream split it into per-model pages.
- **A known failure must not look like a new one.** `tombstones.json` records
  every URL confirmed gone, so a page that died once is counted quietly on
  later runs instead of re-reported. Only *new* deaths and *resurrections*
  are printed loudly, and the success rate is computed over live docs — 123
  standing failures pinning it at 96.9% would hide the next real breakage.
- **A discovery only counts once it is a file.** `discovery.json` records what
  exists upstream that `sources.anthropic.json` does not: unknown redirect targets, new
  anthropics repos, and whether each domain serves `.md` at all. Printing it
  was not enough — the pipeline had logged
  `support.claude.com -> academy.claude.com` for weeks into an Actions log
  with no reader, and a human found those 725 pages by chasing a dead article.
  The decision agent reads `git status`, so a new source now arrives as a diff.

`--discover` and `discovery.json` are Anthropic-only tooling (probes
`DISCOVER_DOMAINS` in `fetcher.py`, which is a hardcoded Anthropic domain
list). OpenAI and Z.AI sources were verified by hand instead -- see each
registry's `notes_on_discovery`.

### OpenAI (see `sources.openai.json`)

developers.openai.com -- single sitemap (`sitemap-0.xml`, 1,321 URLs) covers
the whole site: API guides+reference, Codex, Cookbook, Ads, Plugins/Apps SDK,
Workspace Agents, Agentic Commerce, dev blog, learning resources, showcase.
Every page serves a `.md` variant by appending `.md` to the URL (after
stripping the trailing slash sitemap entries carry). Migrated off
`platform.openai.com/docs` in 2026 -- that path now just redirects here.
Plus `github.com/openai/*`: openai-cookbook, openai-python, openai-node.
`learn.chatgpt.com` (ChatGPT product docs) is a known gap, not yet added.

`openai.com` (the marketing/news domain, **not** developers.openai.com) is
where model launches actually publish -- under `/index/<slug>`, in a
sitemap category literally called "release" (81 URLs). No `.md` variant, no
`llms.txt` (both confirmed 404/403). Its Cloudflare bot-check blocks aiohttp
outright (0/15 requests got through in testing) but lets a plain curl
request with a browser User-Agent through roughly half the time -- a
client-signature check, not a real block; `robots.txt` says `Allow: /` and
publishes the sitemap. Fetched via `fetch_html_via_curl()` (a curl
subprocess, retried up to 6x) + `_extract_blog_page()`, the same
trafilatura path as anthropic.com below, into `content/openai/news/`
(kept separate from `openai/blog/`, which is developers.openai.com's dev
blog). One real run: 68/81 succeeded; the rest self-heal across later
scheduled runs since nothing is tombstoned or reaped on a failed scrape.

### Z.AI (see `sources.zai.json`)

docs.z.ai -- `llms.txt` lists direct `.md` links for every page (same shape
as code.claude.com's), covering the same ~68 pages as its `sitemap.xml`.
robots.txt explicitly welcomes AI scraping (`Content-Signal: ai-train=yes`).
Plus `github.com/zai-org/GLM-skills` (the one repo in that org that's mostly
markdown, not model weights or inference code). Investigated 2026-09-17
whether `z.ai`'s own release-notes are richer than `docs.z.ai/release-notes`
(they aren't -- `/changelog`, `/updates`, `/news` all 404) and whether
`z.ai/blog` (e.g. `z.ai/blog/glm-built-its-inference-infrastructure`) could
be added the same way as openai.com's release posts: **not with the current
architecture**. `z.ai/sitemap.xml` has zero blog URLs (26 pages, all
account/billing), there's no feed or JSON post index, and the posts
themselves are client-side-rendered SPA pages -- the raw HTML is an empty
`<div id="root">` plus a JS bundle, so there's no server-rendered text for
trafilatura to extract even once fetched. No Cloudflare wrinkle here (plain
curl, no UA, gets a clean 200 every time) -- the blocker is needing a
headless browser to execute JS, which this fetcher doesn't have. Documented
as a known gap, not wired up.

```bash
uv run scripts/fetcher.py                    # Fetch everything, all providers
uv run scripts/fetcher.py --section mcp      # Single Anthropic section
uv run scripts/fetcher.py --section openai   # OpenAI docs + repos
uv run scripts/fetcher.py --section zai      # Z.AI docs + repos
uv run scripts/fetcher.py --tree             # Show sources
uv run scripts/fetcher.py --discover         # Probe Anthropic domains for new sources
uv run scripts/fetcher.py --no-reap          # Report dead pages, delete none
```

A full run refreshes `discovery.json` on its own; `--discover` is the manual
probe and writes the same file. Neither adds a source — that stays a human
decision. `discovery.json.review` is the actionable list: domains reachable
and serving markdown that nothing fetches yet. Empty is the healthy state.

Sections: `claude-code`, `api`, `platform`, `mcp`, `github`, `support`,
`products`, `blog` (Anthropic) -- `openai`, `zai` (other providers) -- `all`

Source registries: `sources.anthropic.json`, `sources.openai.json`,
`sources.zai.json` -- one per provider
Confirmed-dead URLs: `tombstones.json` (self-maintaining, all providers; an
entry disappears if the URL answers again)
Sources we do NOT hold: `discovery.json` (self-maintaining; regenerated by
every full run; Anthropic-only)
Architecture: `REFACTOR.md`

When adding a new source to an existing provider:
1. Add source to that provider's `sources.<provider>.json`
2. Add fetch logic + output path mapping to `fetcher.py`
3. Test: `uv run scripts/fetcher.py --section <name>`
4. Update this file's doc references below

When adding a whole new provider: same steps, plus create
`sources.<provider>.json`, a `content/<provider>/` subtree, a
`self.<provider>_dir` in `Fetcher.__init__`, and add `<provider>` to
`--section` choices.

Reference documentation files in this repository when providing guidance.


### Documentation Resources

Use these paths to reference documentation when helping users. This section
covers Anthropic (`content/anthropic/`); see "OpenAI Documentation" and
"Z.AI Documentation" further below for the other two providers.

#### Claude Code Documentation (from code.claude.com)
- `@./content/anthropic/en/docs/claude-code/overview.md` - Claude Code overview and capabilities
- `@./content/anthropic/en/docs/claude-code/quickstart.md` - Getting started guide
- `@./content/anthropic/en/docs/claude-code/setup.md` - Installation and setup
- `@./content/anthropic/en/docs/claude-code/settings.md` - Configuration and permissions setup
- `@./content/anthropic/en/docs/claude-code/common-workflows.md` - Common usage patterns
- `@./content/anthropic/en/docs/claude-code/memory.md` - Memory management and CLAUDE.md
- `@./content/anthropic/en/docs/claude-code/interactive-mode.md` - Keyboard shortcuts and interactive features
- `@./content/anthropic/en/docs/claude-code/slash-commands.md` - Available slash commands
- `@./content/anthropic/en/docs/claude-code/hooks.md` - Hooks reference
- `@./content/anthropic/en/docs/claude-code/hooks-guide.md` - Hooks guide
- `@./content/anthropic/en/docs/claude-code/troubleshooting.md` - Problem solving
- `@./content/anthropic/en/docs/claude-code/cli-reference.md` - Command line interface reference
- `@./content/anthropic/en/docs/claude-code/jetbrains.md` - JetBrains IDE integration
- `@./content/anthropic/en/docs/claude-code/vs-code.md` - VS Code integration
- `@./content/anthropic/en/docs/claude-code/desktop.md` - Claude Code desktop app
- `@./content/anthropic/en/docs/claude-code/claude-code-on-the-web.md` - Claude Code on the web
- `@./content/anthropic/en/docs/claude-code/slack.md` - Claude Code in Slack
- `@./content/anthropic/en/docs/claude-code/mcp.md` - Model Context Protocol
- `@./content/anthropic/en/docs/claude-code/github-actions.md` - GitHub Actions integration
- `@./content/anthropic/en/docs/claude-code/gitlab-ci-cd.md` - GitLab CI/CD integration
- `@./content/anthropic/en/docs/claude-code/sdk/migration-guide.md` - SDK migration guide
- `@./content/anthropic/en/docs/claude-code/third-party-integrations.md` - Third-party integrations
- `@./content/anthropic/en/docs/claude-code/devcontainer.md` - Development containers
- `@./content/anthropic/en/docs/claude-code/security.md` - Security considerations
- `@./content/anthropic/en/docs/claude-code/sandboxing.md` - Sandboxed bash tool
- `@./content/anthropic/en/docs/claude-code/iam.md` - Authentication and permissions
- `@./content/anthropic/en/docs/claude-code/monitoring-usage.md` - OpenTelemetry monitoring
- `@./content/anthropic/en/docs/claude-code/analytics.md` - Analytics and usage tracking
- `@./content/anthropic/en/docs/claude-code/costs.md` - Cost management
- `@./content/anthropic/en/docs/claude-code/data-usage.md` - Data usage policies
- `@./content/anthropic/en/docs/claude-code/legal-and-compliance.md` - Legal and compliance
- `@./content/anthropic/en/docs/claude-code/amazon-bedrock.md` - Amazon Bedrock integration
- `@./content/anthropic/en/docs/claude-code/google-vertex-ai.md` - Google Vertex AI integration
- `@./content/anthropic/en/docs/claude-code/microsoft-foundry.md` - Microsoft Foundry integration
- `@./content/anthropic/en/docs/claude-code/llm-gateway.md` - LLM gateway configuration
- `@./content/anthropic/en/docs/claude-code/model-config.md` - Model configuration
- `@./content/anthropic/en/docs/claude-code/network-config.md` - Network configuration
- `@./content/anthropic/en/docs/claude-code/terminal-config.md` - Terminal configuration
- `@./content/anthropic/en/docs/claude-code/output-styles.md` - Output styling and formatting
- `@./content/anthropic/en/docs/claude-code/statusline.md` - Status line configuration
- `@./content/anthropic/en/docs/claude-code/checkpointing.md` - Session checkpointing
- `@./content/anthropic/en/docs/claude-code/headless.md` - Headless mode
- `@./content/anthropic/en/docs/claude-code/plugins.md` - Plugin system
- `@./content/anthropic/en/docs/claude-code/plugins-reference.md` - Plugin reference
- `@./content/anthropic/en/docs/claude-code/plugin-marketplaces.md` - Plugin marketplaces
- `@./content/anthropic/en/docs/claude-code/skills.md` - Claude Skills
- `@./content/anthropic/en/docs/claude-code/sub-agents.md` - Sub-agents
- `@./content/anthropic/CHANGELOG.md` - Claude Code GitHub CHANGELOG

#### Platform Docs (from platform.claude.com)
- `content/anthropic/en/api/` - API reference (1,500+ docs)
- `content/anthropic/en/build-with-claude/` - Platform features, streaming, batch
- `content/anthropic/en/agents-and-tools/` - Tool use, agent skills, MCP tunnels
- `content/anthropic/en/manage-claude/` - Admin, billing, organizations
- `content/anthropic/en/managed-agents/` - Managed agents API
- `content/anthropic/en/test-and-evaluate/` - Testing and evaluation

#### Product Docs (from claude.com/docs)
- `content/anthropic/claude/claude-tag/` - Claude Tag / Claude in Slack (65)
- `content/anthropic/claude/government/` - Government offerings (38)
- `content/anthropic/claude/connectors/` - Connectors, building + publishing (33)
- `content/anthropic/claude/claude-science/` - Claude for Science (29)
- `content/anthropic/claude/third-party/` - Bedrock, Vertex, Foundry desktop setups (28)
- `content/anthropic/claude/office-agents/` - Claude for Excel, Word, PowerPoint, Outlook (12)
- `content/anthropic/claude/cowork/` - Claude Cowork (6)

#### Release Notes (from platform.claude.com)
- `content/anthropic/en/release-notes/system-prompts/` - Claude.ai system prompts, one
  page per model (split from a single file upstream on 2026-08-24)
- `content/anthropic/en/models/` - Per-model overviews and "what's new" pages

#### MCP Protocol (from modelcontextprotocol.io)
- `content/anthropic/mcp/docs/` - Getting started, build client/server
- `content/anthropic/mcp/specification/` - Protocol spec versions
- `content/anthropic/mcp/seps/` - Specification Enhancement Proposals
- `content/anthropic/mcp/community/` - Governance, working groups

#### Engineering & Research (from anthropic.com)
- `content/anthropic/blog/engineering/` - "Building Effective Agents", tool use, harness design
- `content/anthropic/blog/research/` - Research papers
- `content/anthropic/blog/news/` - Model releases, announcements
- `content/anthropic/blog/policy/` - Standalone pages: constitution, responsible scaling
  policy, transparency, threat intelligence reports, economic index/futures,
  system cards (fixed allowlist, `BLOG_STANDALONE_PAGES` in `fetcher.py` --
  these sit at the site root, outside the sitemap-crawled prefixes)
- `content/anthropic/blog/alignment/` - Alignment Science blog (alignment.anthropic.com,
  60 posts). No sitemap/feed; the homepage itself lists every post and is the
  index. Added 2026-09-13.
- `content/anthropic/blog/interpretability/` - Transformer Circuits Thread
  (transformer-circuits.pub, 56 posts back to 2021), discovered via its Atom
  feed at `/feed.xml`. Added 2026-09-13. red.anthropic.com (Frontier Red Team
  blog) was checked the same day and NOT added: every article but one
  redirects to www.anthropic.com/research/*, already covered above.

#### GitHub Repos (from github.com/anthropics)
- `content/anthropic/github/cookbooks/` - 164 recipes + notebooks
- `content/anthropic/github/skills/` - 90 official Agent Skills
- `content/anthropic/github/plugins-official/` - 266 plugin docs
- `content/anthropic/github/courses/` - 80 prompt engineering notebooks
- `content/anthropic/github/code-action/` - GitHub Actions docs
- `content/anthropic/github/sdk-python/` - Python SDK reference
- `content/anthropic/github/sdk-typescript/` - TypeScript SDK reference

### OpenAI Documentation (developers.openai.com + openai.com, see `sources.openai.json`)
- `content/openai/api/` - API guides + endpoint reference
- `content/openai/codex/` - Codex CLI, IDE, cloud, config.toml
- `content/openai/cookbook/` - Practical code examples
- `content/openai/ads/`, `plugins/`, `workspace-agents/`, `commerce/` - Ads API, Apps SDK/plugins, Workspace Agents API, Agentic Commerce
- `content/openai/blog/` - Developer blog (developers.openai.com)
- `content/openai/news/` - Model launches (openai.com, scraped via curl -- see Fetcher above)
- `content/openai/learn/`, `showcase/` - Learning resources, project showcase
- `content/openai/github/cookbook/` - openai/openai-cookbook
- `content/openai/github/openai-python/`, `openai-node/` - SDK repos (docs, examples)

### Z.AI Documentation (from docs.z.ai, GLM models, see `sources.zai.json`)
- `content/zai/guides/` - GLM model guides, quick start, pricing, SDKs
- `content/zai/api-reference/` - API reference
- `content/zai/devpack/` - Dev tooling
- `content/zai/release-notes/` - Release notes
- `content/zai/github/GLM-skills/` - Markdown skill definitions

## Repository Structure

```
content/
  anthropic/                   3,900+ files
    en/docs/claude-code/       Claude Code + Agent SDK (198)
    en/api/                    API reference (1,900+)
    en/build-with-claude/      Platform features
    en/agents-and-tools/       Tool use, agent skills
    claude/                    Product docs (215)
    mcp/                       MCP protocol spec (373)
    blog/                      Engineering, research, news, policy, alignment, interpretability
    github/                    10 repos (718 files)
    support/                   Help articles (365)
  openai/                      ~1,400 docs (developers.openai.com + openai.com)
    api/, codex/, cookbook/, ads/, plugins/, workspace-agents/, commerce/, blog/, learn/, showcase/
    news/                      Model launches (openai.com/index, scraped via curl)
    github/                    3 repos (cookbook, openai-python, openai-node)
  zai/                         68 docs
    guides/, api-reference/, devpack/, release-notes/
    github/                    1 repo (GLM-skills)
scripts/
  fetcher.py                   Multi-source, multi-provider fetcher
sources.anthropic.json         Anthropic source registry
sources.openai.json            OpenAI source registry
sources.zai.json               Z.AI source registry
```

### External Resources

- https://github.com/anthropics/claude-code/issues
- https://code.claude.com/docs/en/overview
- https://platform.claude.com/docs/en/home
- https://modelcontextprotocol.io
- https://claude.com/docs/llms.txt
- https://developers.openai.com/llms.txt
- https://docs.z.ai/llms.txt
