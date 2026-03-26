# Opportunity Scout

> An evidence-first Claude Code skill that finds AI startup opportunities from live market signals, not brainstormed guesses.

[![License: MIT](https://img.shields.io/badge/license-MIT-black.svg)](./LICENSE)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-blue.svg)](https://github.com/SENNGAMI/Opportunity-scout)
[![Research Source](https://img.shields.io/badge/Live%20Research-Tavily-green.svg)](https://tavily.com)

Opportunity Scout is a market intelligence skill for Claude Code. It scans live sources, filters weak ideas, maps competitive gaps, and returns ranked AI opportunities with timing analysis, wedge statements, and concrete validation plans.

It is built for prompts like:

- "Find me a startup idea"
- "What should I build?"
- "What AI opportunities exist right now?"
- "Scan the market for gaps"
- "What's underdeveloped in AI?"

## Why This Skill Exists

Most idea-generation workflows produce polished fiction.

Opportunity Scout is opinionated in the opposite direction:

- It starts from real user pain, not vibes
- It treats competition as evidence of demand
- It looks for structural timing shifts, not "hot markets"
- It refuses to fabricate sources or hide weak evidence

## What You Get

For each surviving opportunity, the skill produces:

- A one-line opportunity hypothesis
- The specific buyer and wedge
- Competitive weaknesses in the current market
- Timing analysis based on structural triggers
- Weighted scoring across five dimensions
- Source-backed evidence with URLs
- A two-week validation plan
- A recommendation: `KILL`, `REVISIT`, or `PROCEED`

## How It Works

```text
Phase 1a
  signal-scanner   -> finds user pain across YC, Product Hunt, Reddit, HN, reviews, GitHub
  gap-detector     -> finds adjacent markets with missing or weak AI products

Phase 1b
  competitor-mapper -> maps incumbents, weaknesses, and wedge statements

Checkpoint
  you review candidates before deeper scoring

Phase 2
  timing-judge -> checks whether the market timing is structurally right

Phase 3
  ranked opportunity memo -> final scoring, recommendation, validation plan
```

## Scoring Model

| Dimension | Weight | Measures |
| --- | ---: | --- |
| Pain strength | 25% | How frequent and costly the problem is |
| Buyer clarity | 20% | How clearly the first customer can be named |
| Timing maturity | 20% | Whether a real trigger event makes the opportunity viable now |
| Wedge quality | 20% | Why incumbents cannot or will not respond well |
| Buildability | 15% | Whether an MVP can be tested in four weeks |

Maximum weighted score: `5.00`

## Requirements

### 1. Claude Code

This repo is a Claude Code skill, not a standalone app.

### 2. Tavily MCP

Opportunity Scout depends on Tavily for live web research.

Add Tavily to Claude Code:

```bash
claude mcp add --transport http tavily https://mcp.tavily.com/mcp/?tavilyApiKey=YOUR_TAVILY_API_KEY
```

Get an API key from [tavily.com](https://tavily.com).

### 3. Allow Tavily Tools For Sub-Agents

Sub-agents cannot stop for interactive permission prompts, so allow the Tavily tools in `~/.claude/settings.local.json`:

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

## Install

Clone the repo:

```bash
git clone https://github.com/SENNGAMI/Opportunity-scout.git
```

Copy the skill into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills
cp -R Opportunity-scout/plugins/opportunity-scout ~/.claude/skills/
```

Restart Claude Code. The skill will then be available as `/opportunity-scout`.

## Usage

Run:

```bash
/opportunity-scout
```

Or invoke it naturally:

- "Find me a startup idea in B2B AI"
- "What should I build in workflow automation?"
- "Scan the market for underdeveloped AI products"

## Output Style

The final memo is designed to answer five practical questions fast:

1. What is the opportunity?
2. Who feels the pain badly enough to buy?
3. Why now?
4. Why are current products weak?
5. What should I do in the next two weeks to validate it?

## Repo Structure

```text
plugins/opportunity-scout/
├── SKILL.md
├── agents/
│   ├── signal-scanner.md
│   ├── gap-detector.md
│   ├── competitor-mapper.md
│   └── timing-judge.md
├── knowledge/
├── memory/
└── templates/
```

## Design Principles

- Main thread handles orchestration and synthesis only
- Web research is delegated to sub-agents
- Weak evidence lowers confidence instead of getting hidden
- Candidates fail fast through kill filters before ranking
- The skill stops on sub-agent failure instead of pretending the data exists

## Example Use Cases

- Solo founders looking for high-signal AI opportunities
- Operators exploring vertical SaaS wedges
- Builders who want evidence before writing code
- Teams running recurring market scans

## License

[MIT](./LICENSE)
