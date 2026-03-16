---
name: deep-research-burn
description: Use when the user wants to burn unused Claude Code weekly/daily limits by running parallelized deep research agents. Produces context-dense research files on specified topics, repos, or problems that can be reused across sessions.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch, Task
---

# Deep Research Burn

Fan out parallel research agents to productively burn unused Claude Code usage limits. Each agent produces a standalone, context-dense research file that persists and can be referenced in future sessions.

Inspired by [@elvissun's approach](https://x.com/elvissun/status/2033316575798001771): "if you have unused weekly limits the best way to burn them is just spamming fan-out deep research in cc/codex."

## When to Use

- User is approaching weekly/daily limit reset and wants to use remaining capacity
- User says "burn X% of my limits on research about Y"
- User wants deep context gathered on a topic, repo, or problem before a future session
- User wants to pre-research multiple topics in parallel

## Why This Works

- 0 review cycles needed (research, not code changes)
- Context-dense files you reuse across sessions
- No slop generated (source material, not final output)
- Feeds into product, content, marketing, or competitor intel later
- Parallel agents maximize throughput per unit time

## Research Modes

### Mode 1: Topic Research
Deep dive into a concept, technology, or trend.

```
/deep-research-burn "Solana Token-2022 extensions" "deBridge v2 API changes" "Hyperliquid vault strategies"
```

### Mode 2: Repo Analysis
Analyze a public GitHub repo's architecture, patterns, and decisions.

```
/deep-research-burn repo:karpathy/autoresearch repo:jup-ag/jupiter-quote-api
```

### Mode 3: Problem Research
Gather context on a specific problem you're facing.

```
/deep-research-burn "why do deBridge Solana transactions fail with custom program error 0x1" "Token-2022 ATA creation costs"
```

### Mode 4: Competitive Intel
Research competing projects, protocols, or tools.

```
/deep-research-burn "pump.fun AMM mechanics" "moonshot bonding curve" "raydium concentrated liquidity"
```

## Agent Dispatch Pattern

Each research topic gets its own subagent via the Task tool. All agents run in parallel.

```typescript
// Pseudocode for the dispatch pattern
for (const topic of topics) {
  Task({
    subagent_type: "general-purpose",
    description: `Research: ${topic}`,
    prompt: buildResearchPrompt(topic),
    run_in_background: true,
  });
}
```

### Research Agent Prompt Template

Each agent receives this prompt structure:

```
You are a deep research agent. Your job is to produce a comprehensive, context-dense
research file on the following topic:

TOPIC: {topic}

OUTPUT FORMAT:
Write a markdown file with the following sections:

## Overview
One paragraph summary of what this is and why it matters.

## Key Findings
The most important things discovered, with specific details (URLs, code snippets,
numbers, dates). No vague statements.

## Technical Details
Architecture, API endpoints, code patterns, configuration, gotchas.
Include actual code snippets where relevant.

## Comparison / Context
How this relates to alternatives or competing approaches.

## Open Questions
Things that couldn't be fully answered and would need further investigation.

## Sources
URLs and references used.

RULES:
- Be specific. Include URLs, code, numbers. No "generally speaking" or "it depends."
- If you can't find authoritative info on something, say so explicitly.
- Prefer primary sources (official docs, source code) over blog posts.
- Include code snippets that can be copy-pasted.
- This file will be read months from now -- include enough context to be useful standalone.
```

## Output Location

All research files go to a dedicated directory:

```
research/
  2026-03-16-solana-token-2022-extensions.md
  2026-03-16-debridge-v2-api-changes.md
  2026-03-16-hyperliquid-vault-strategies.md
```

Naming: `{date}-{slugified-topic}.md`

## Burn Budgeting

Rough estimates for planning burn percentage:

| Agent Count | Estimated Usage | Good For |
|---|---|---|
| 3-5 | ~3-5% weekly | Quick research pass |
| 8-12 | ~8-12% weekly | Medium deep dive |
| 15-22 | ~15-20% weekly | Heavy burn session |

These are rough -- actual usage depends on how much content each agent finds and processes.

### Targeting a Specific Burn Percentage

```
User: "burn about 10% of my weekly limit on crypto research"

-> Dispatch 8-10 agents on distinct subtopics
-> Each agent runs 3-5 minutes
-> Total wall clock: ~5 minutes (parallel)
-> Total token usage: ~10% of weekly limit
```

## Execution Flow

1. User provides topics (or asks for topic suggestions)
2. Create research output directory if it doesn't exist
3. Dispatch one Task agent per topic, all in background
4. Each agent writes its research file to the output directory
5. Once all agents complete, summarize what was produced
6. Files persist for future session reference

## Topic Suggestion Engine

If the user says "burn limits" without specifying topics, suggest based on:

1. **Current projects**: What repos are in ~/Desktop/claudecode/? Research adjacent tools and competitors.
2. **Recent sessions**: What problems came up? Research solutions in depth.
3. **Ecosystem trends**: What's new in the user's stack (Solana, Base, Next.js, etc.)?
4. **Skill gaps**: What skills exist in claude-skills/? Research topics that would improve them.

## Quality Standards

Each research file must:
- Be useful standalone (someone with no context can read it)
- Include at least 3 specific URLs or code references
- Have no "placeholder" or "TODO" sections
- Be written in present tense ("the API accepts..." not "the API would accept...")
- Include a date in the filename so staleness is obvious

## Integration with Other Skills

Research files can feed into:
- **crypto-alpha-writer**: research on market mechanics, whale strategies
- **debridge-integration**: research on API changes, new chain support
- **ct-tweet-crafter**: research on competitor positioning for tweet content
- **CLAUDE.md updates**: research that reveals new gotchas or patterns
