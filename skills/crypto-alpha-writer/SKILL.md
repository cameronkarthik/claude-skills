---
name: crypto-alpha-writer
description: Use when generating crypto market analysis, on-chain intelligence reports, or alpha content targeting a degen/CT audience. Handles data collection via paid APIs, cross-referencing, and writing in a natural non-AI voice.
version: 0.1.0
tools: Read, Grep, Glob, Bash, Edit, Write
---

# Crypto Alpha Writer

Skill for producing crypto intelligence reports that sound like they were written by a human on-chain researcher, not an AI. Built from 6 iterations of user feedback on what "alpha" actually means to a CT/degen audience.

## When to Use

- User asks to generate a crypto market report or alpha brief
- User asks to analyze on-chain data for a newsletter
- User wants to summarize whale movements, token launches, or market events
- Any crypto content that needs to not sound like AI wrote it

## Voice Rules (Non-Negotiable)

The content MUST follow these rules. These come from extensive user feedback.

### DO:
- Write in lowercase (except proper nouns, tickers)
- Use natural paragraph flow, not bullet points or tables
- Think out loud, like you're explaining to a smart friend
- Include specific numbers: wallet addresses, dollar amounts, entry prices, liquidation levels
- Connect dots across data points (this is where the value is)
- Use parenthetical asides for context (like this)
- Reference other traders/wallets by their actual addresses or handles
- End with a clear directional opinion, even if hedged

### DO NOT:
- Use emdashes (--) excessively
- Use hashtags
- Use bullet points or numbered lists as the primary format
- Start with "here's what you need to know" or "let's dive in"
- Use "arguably", "notably", "importantly", "interestingly"
- Use tables for the main content (OK in appendix/methodology)
- Say "NFA" or "DYOR" more than once
- Use emoji of any kind
- Structure with H2/H3 headers throughout (max 2-3 section breaks)

### Voice Inspiration:
- Karpathy's Twitter: stream of consciousness, thinking out loud, natural asides
- cookerflips: opinionated, specific, tells you what he'd actually do
- @DefiIgnas: connects macro to specific plays

## Report Structure

Two sections only:

### Section 1: "what happened"
Raw facts and data. Wallet addresses. Dollar amounts. Entry prices. Liquidation levels. Verifiable on-chain data. No opinion here, just what the data shows.

### Section 2: "how to think about it"
Analysis that connects the dots. What the data points mean together. Where you'd put money if forced to. What the risks are. Clear directional view with reasoning.

## Data Collection Pipeline

### Sources (in priority order):

1. **On-chain whale trackers**: @lookonchain, @ai_9684xtpa (Chinese, often first), @EmberCN, @OnchainDataNerd, @whale_alert
2. **Position trackers**: @HyperliquidData, Coinglass, Arkham
3. **Cross-reference via Exa/web search**: Verify claims across at least 2 sources
4. **Reddit/CT sentiment**: Only for gauging retail positioning, not for alpha

### Collection via AgentCash APIs:

```bash
# Twitter user timeline (best source)
agentcash fetch 'https://twit.sh/tweets/user' -m POST -b '{"username": "lookonchain", "count": 20}'

# Twitter search (use anyWords for OR logic, words for AND)
agentcash fetch 'https://twit.sh/tweets/search' -m POST -b '{"anyWords": "whale liquidated Hyperliquid", "minLikes": 10, "since": "2026-03-07"}'

# Exa neural search (good for cross-referencing)
agentcash fetch 'https://stableenrich.dev/api/exa/search' -m POST -b '{"query": "specific search query here", "numResults": 5, "includeSummary": true}'

# Exa content pull (cheap, $0.002)
agentcash fetch 'https://stableenrich.dev/api/exa/contents' -m POST -b '{"urls": ["url1", "url2"]}'
```

### Budget Management:
- Twitter search/timeline: $0.01 each
- Exa search: $0.01-0.017
- Exa contents: $0.002
- Upload: $0.02
- Target: full report for under $0.50

## Cross-Referencing (This Is Where Value Comes From)

The difference between "repackaged public data" and "actual alpha":

1. **Take wallet addresses from source A, search for them in source B**
   - Whale tracker mentions 0x15a4 opened a position
   - Search that address across other trackers to find what ELSE that wallet is doing

2. **Connect events across sources**
   - Source A: ParaFi exits AAVE
   - Source B: ACI governance crisis
   - Connection: institutional money responding to governance risk in real-time

3. **Find historical context for current actors**
   - Current: whale goes $84M long ETH
   - History: same wallet sold ETH at $4,434 six months ago
   - Insight: this wallet has a track record of timing major moves

4. **Identify patterns across similar events**
   - JELLYJELLY manipulation happening now
   - Same playbook caused $12M HLP vault loss on Hyperliquid before
   - Pattern: this is round two, not a new event

## Quality Check Before Publishing

Read the finished report and ask:

1. Could someone get this exact same content by following the 4 source accounts? If yes, you haven't added enough cross-referencing.
2. Is there at least one connection that no single source account made? If no, keep digging.
3. Would you pay $5/week for this? Be honest.
4. Does it sound like an AI wrote it? Read it out loud. If any sentence sounds robotic, rewrite it.
5. Is there a clear directional opinion? "Things could go either way" is not analysis.

## Upload Flow

```bash
# Get upload slot
agentcash fetch 'https://stableupload.dev/api/upload' -m POST -b '{"filename": "report-name.md", "contentType": "text/markdown", "tier": "10mb"}'

# Upload file (use curl, not agentcash)
curl -X PUT "<uploadUrl>" -H "Content-Type: text/markdown" --data-binary @'local-file.md'

# Public URL is in the response from the first call
```
