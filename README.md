# Opportunity Scout

An autonomous market intelligence skill for Claude Code. Scans multiple live sources to find AI business opportunities through structured, multi-phase research — without fabricating sources or skipping evidence.

## What it does

1. **Phase 1a** — Two agents run in parallel:
   - `signal-scanner`: Scans YC batches, Product Hunt, Reddit, HackerNews, App Store reviews, and GitHub issues for real user pain signals
   - `gap-detector`: Maps funded AI verticals and applies the neighbor gap method to find adjacent markets with no funded solution

2. **Phase 1b** — After Phase 1a completes:
   - `competitor-mapper`: For each candidate, maps competitor landscape, assesses structural weaknesses, and produces a wedge statement

3. **Review checkpoint** — Presents candidates with kill-filter results. You confirm which to advance.

4. **Phase 2** — `timing-judge` scores each confirmed candidate across four structural trigger dimensions (technology, regulatory, behavioral, market validation).

5. **Phase 3** — Scores all surviving candidates across five dimensions and produces a ranked opportunity memo with wedge statements and two-week validation plans.

## Prerequisites

### 1. Tavily MCP server

This skill uses Tavily for all live web research. You need a Tavily API key and the Tavily MCP server configured.

Install the Tavily MCP server:

```bash
npm install -g @tavily/mcp-server
```

Add to your Claude Code MCP config (`~/.claude/mcp_servers.json` or equivalent):

```json
{
  "tavily": {
    "command": "tavily-mcp-server",
    "env": {
      "TAVILY_API_KEY": "your-api-key-here"
    }
  }
}
```

Get a Tavily API key at [tavily.com](https://tavily.com).

### 2. Allow Tavily tools in settings

Add the following to `~/.claude/settings.local.json` so sub-agents can use Tavily without hitting permission prompts:

```json
{
  "permissions": {
    "allow": [
      "mcp__tavily__tavily_search",
      "mcp__tavily__tavily_extract",
      "mcp__tavily__tavily_crawl",
      "mcp__tavily__tavily_map",
      "mcp__tavily__tavily_research",
      "mcp__tavily__tavily_skill"
    ]
  }
}
```

This is required because sub-agents cannot receive interactive permission prompts.

## Installation

### Option A: Git clone (recommended)

```bash
git clone https://github.com/your-username/opportunity-scout.git
cp -r opportunity-scout/plugins/opportunity-scout ~/.claude/skills/
```

Then restart Claude Code. The `/opportunity-scout` skill will be available.

### Option B: Manual copy

Copy the `plugins/opportunity-scout/` folder into `~/.claude/skills/`:

```
~/.claude/skills/
└── opportunity-scout/
    ├── SKILL.md
    ├── agents/
    │   ├── signal-scanner.md
    │   ├── gap-detector.md
    │   ├── competitor-mapper.md
    │   └── timing-judge.md
    ├── templates/
    │   └── opportunity-memo.md
    ├── knowledge/
    └── memory/
```

## Usage

In Claude Code, run:

```
/opportunity-scout
```

Or trigger it naturally:

- "find me a startup idea"
- "what should I build"
- "what AI opportunities exist right now"
- "scan the market for gaps"
- "what's underdeveloped in AI"

## Output

The skill produces a ranked opportunity memo for each surviving candidate:

- Hypothesis (one sentence)
- What changed to create this opportunity
- Target buyer with specific role, company type, and how to reach the first 10
- Current alternatives and their failure modes
- Evidence base with source URLs and verbatim quotes
- Scoring across five dimensions with confidence labels
- Wedge statement
- Validation plan (specific actions executable within two weeks)
- Recommendation: KILL / REVISIT / PROCEED

## Architecture

```
SKILL.md (orchestrator)
├── agents/signal-scanner.md     — live pain signal collection
├── agents/gap-detector.md       — adjacent vertical gap analysis
├── agents/competitor-mapper.md  — structural competitive analysis
├── agents/timing-judge.md       — four-dimension timing assessment
├── templates/opportunity-memo.md — output format
├── knowledge/                   — reference frameworks
└── memory/opportunity-log.md    — session history log
```

The main thread handles orchestration and synthesis only. All web research runs exclusively in sub-agents. If a sub-agent fails, the main thread reports the failure and stops — it does not compensate by running searches itself.

## Scoring dimensions

| Dimension       | Weight | What it measures                               |
|-----------------|--------|------------------------------------------------|
| Pain strength   | 25%    | Frequency and dollar cost of the problem       |
| Buyer clarity   | 20%    | Precision with which the first buyer is named  |
| Timing maturity | 20%    | Presence of a structural trigger event         |
| Wedge quality   | 20%    | Reason incumbents cannot or will not respond   |
| Buildability    | 15%    | Whether an MVP can be tested within four weeks |

Maximum weighted score: 5.00. Opportunities scoring above 3.5 receive a PROCEED recommendation.

## License

MIT
