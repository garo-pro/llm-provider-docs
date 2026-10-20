Title: Agents for financial services

URL Source: https://www.anthropic.com/news/finance-agents

Markdown Content:
We’re releasing ten ready-to-run agent templates for the most time-consuming work in financial services: building pitchbooks, screening KYC files, and closing the books at month-end. Each one ships as a [plugin](https://support.claude.com/en/articles/13837440-use-plugins-in-claude-cowork) in Claude Cowork and Claude Code, and as a cookbook for [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview), so a team can put Claude on real financial work in days rather than months.

Claude also now works across Microsoft Excel, PowerPoint, Word, and Outlook (coming soon) through the Claude add-ins for Microsoft 365. Once the add-ins are installed, context carries automatically between applications, so work that starts in a model can end in a deck without re-explaining anything in between.

Finally, we’re continuing to expand our partner ecosystem with new connectors and an MCP app, so the agents draw on the data financial professionals already use. Connectors give Claude governed, real-time access to a provider’s data, and MCP apps go a step further by embedding the provider’s own tools directly inside Claude.

These updates pair best with Claude Opus 4.7, which is state-of-the-art on financial tasks and leads the industry on [Vals AI's Finance Agent benchmark](https://www.vals.ai/benchmarks/finance_agent), at 64.37%.

## New agent templates for finance work

Each agent template is a reference architecture that packages three things: skills (instructions and domain knowledge for the task), connectors (governed access to the data the task runs on), and subagents (additional Claude models that are called upon by the main agent, for specific sub-tasks such as comparables selection or methodology checks). Firms can adapt any of them to their own modeling conventions, risk policies, and approval flows.

Enable these new agent templates either as plugins within Claude Cowork or Claude Code, or as cookbooks for Claude Managed Agents. Find all the plugins and cookbooks at the [financial services marketplace](https://github.com/anthropics/financial-services).

The full list of new agents is as follows:

**Research and client coverage**

- **Pitch builder** creates target lists, runs comparables, and drafts pitchbooks for client meetings;
- **Meeting preparer** assembles client and counterparty briefs ahead of calls;
- **Earnings reviewer** reads transcripts and filings, updates models, and flags thesis-relevant changes;
- **Model builder** creates and maintains financial models from filings, data feeds, and analyst inputs;
- **Market researcher** tracks sector and issuer developments, synthesizes news, filings, and broker research, and flags items for credit and risk review.

**Finance and operations**

- **Valuation reviewer** checks valuations against comparables, methodology, and the firm's review standards;
- **General ledger reconciler** reconciles general ledger accounts and runs net asset value calculations against the books of record;
- **Month-end closer** runs the close checklist, prepares journal entries, and produces close reports;
- **Statement auditor** reviews financial statements for consistency, completeness, and audit-readiness;
- **KYC screener** assembles entity files, reviews source documents, and packages escalations for compliance review.

![Video thumbnail](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fimg.youtube.com%2Fvi%2FfoxeK2AXfHQ%2Fmaxresdefault.jpg&w=3840&q=75)

There are two ways to put these to work.

As a plugin in Claude Cowork or Claude Code, the template runs alongside the analyst, using the software already on their desktop. Hand the Pitch agent a target list, and you can get back a comps model in Excel, a pitchbook drafted in PowerPoint, and a cover note ready in Outlook.

As a [Claude Managed Agent](https://platform.claude.com/docs/en/managed-agents/overview), the same template runs autonomously on the Claude Platform, for work that spans a whole book of deals or a nightly schedule. The cookbooks stand it up with the building blocks a firm would otherwise engineer themselves: long-running sessions that can work throughout a multi-hour deal close, per-tool permissions, managed credential vaults, and a full audit log in the Claude Console where compliance and engineering teams can inspect every tool call and decision.

In both scenarios, users stay firmly in the loop—reviewing, iterating on, and approving Claude’s work before it goes to a client, gets filed, or is acted on.

## Claude across Excel, PowerPoint, Word, and Outlook

Claude can work directly in Microsoft [Excel](https://claude.com/claude-for-excel), [PowerPoint](https://claude.com/claude-for-powerpoint), [Word](https://claude.com/claude-for-word), and Outlook via add-ins.

In Outlook, it can act as a chief of staff that triages your inbox, arranges meetings, and drafts responses in your voice. In Excel, it builds financial models from filings and data feeds, audits formulas across linked workbooks, and runs sensitivity analyses. In PowerPoint, it drafts decks that update automatically when the underlying numbers change. In Word, it edits credit memos against a firm’s own templates. Claude carries its knowledge and context across all four platforms: an analyst who’s started a model in Excel doesn’t need to re-explain it when that work moves to PowerPoint.

In Claude Cowork, users can also assign Claude work tasks from anywhere—by text or by voice—using [Dispatch](https://support.claude.com/en/articles/13947068-assign-tasks-from-anywhere-in-claude-cowork). Claude can keep working on analysts’ local files while they’re away from their desk, with finished work ready for review by the time they’re back.

## The broadest ecosystem for financial services

AI agents are only as good as the data and context they can access. Claude connects to dozens of market data, research platforms, and financial companies’ internal systems—including FactSet, S&P Capital IQ, MSCI, PitchBook, Morningstar, Chronograph, LSEG, and Daloopa—along with firms’ own data warehouses, research repositories, and CRMs, all under governed access controls.

We’re now adding connectors and an MCP app from new partners. The new connectors give direct, real-time access to market and research data, while the MCP app surfaces custom, interactive UI directly within Claude.

The new connectors are:

- **Dun & Bradstreet** , which provides the global standard for verified business identity and helps enterprises connect systems of record and scale AI-enabled workflows;
- **Fiscal AI** , which extends real-time fundamentals coverage across public equities for deeper research and benchmarking;
- **Financial Modeling Prep** , which provides real-time quotes, fundamentals, statements, filings, and transcripts across equities, ETFs, crypto, forex, and commodities;
- **Guidepoint** , which searches 100,000+ compliance-reviewed expert interview transcripts and provides verbatim excerpts linked to source;
- **IBISWorld** , which tracks industry-level revenue, financial ratios, risk scores, cost structures, and forecasts across thousands of sectors;
- **SS&C Intralinks** , which gives Claude access to DealCenter AI data rooms for document search, diligence Q&A, and deal-activity tracking;
- **Third Bridge** , which gives Claude access to primary-source expert interviews on companies, sectors, and value chains;
- **Verisk** , which provides property, casualty, and specialty insurance data for underwriting, claims, and risk analysis.

In addition, **Moody's** has launched an MCP app that brings proprietary credit ratings and data on more than 600 million public and private companies for use in compliance, credit analysis, and business development.

## Claude's impact in financial services

Many leading banks, asset managers, and insurers choose Claude. It supports the full range of these organizations' work: front office tasks like research and client experience, middle office work in underwriting, risk, and compliance, and back office work like code modernization and operations.

## Getting started

Our new Claude agents are available today at our [financial services marketplace](https://github.com/anthropics/financial-services). They can be used as plugins in Claude Cowork or Claude Code on all paid plans, or as Managed Agents in the Claude Platform (in public beta) for programmatic use. The new connectors and Moody’s MCP app are also available to joint customers on paid plans.

The Claude for Excel, PowerPoint, and Word add-ins are generally available, and Claude for Outlook is coming soon.

To see these capabilities in action, you can register for our [livestreamed keynote](https://www.anthropic.com/events/the-briefing-financial-services-virtual-event), and [hands-on webinar](https://www.anthropic.com/webinars/claude-for-financial-services-putting-agents-to-work) which will provide deeper practical adoption guidance. For additional support, [contact our sales team](https://claude.com/contact-sales/financial-services), and learn more about [our solutions for financial services](https://claude.com/solutions/financial-services).
