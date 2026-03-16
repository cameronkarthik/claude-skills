---
name: ct-tweet-crafter
description: Use when drafting tweets about crypto projects, on-chain analysis, or DeFi systems. Produces tweets that sound like an experienced CT builder, not an AI. Includes voice calibration, thread structure, and engagement patterns.
version: 0.1.0
tools: Read, Grep, Glob, Bash, WebFetch
---

# CT Tweet Crafter

Skill for writing crypto twitter content that sounds like it was written by a builder who ships, not a marketing intern or an LLM. Built from multiple rounds of user feedback rejecting AI-sounding drafts.

## When to Use

- User wants to tweet about a project they're building
- User wants to share technical analysis or on-chain findings
- User needs a follow-up tweet to maintain engagement
- Any crypto content destined for Twitter/X

## Voice Rules (Non-Negotiable)

These come from 6+ rounds of "this sounds like AI, redo it" feedback.

### DO:
- Write in first person, like you're texting a smart friend
- Lead with the most interesting finding or number
- Include specific numbers: SOL amounts, percentages, wallet counts, market caps
- Use lowercase unless starting a sentence or for tickers/proper nouns
- Keep sentences short and punchy
- End with a directional opinion or a hook for engagement
- Use line breaks for readability (Twitter renders these well)
- Reference your own experience building ("ran a simulation", "tested this", "been working on")

### DO NOT:
- Use emdashes (--) anywhere
- Use hashtags (#DeFi, #Solana, etc.)
- Use "thread" or "1/" numbering for single tweets
- Start with "Just", "So", "Excited to share", "Big news"
- Use "arguably", "notably", "importantly", "interestingly"
- Use "NFA", "DYOR", "LFG", "wagmi" (one "NFA" at most, at the very end, only if the user asks)
- Use any emoji
- Use bullet points (use line breaks instead)
- Say "we're building" -- say "I built" or "we shipped"
- Use passive voice ("a simulation was run" -> "ran a simulation")
- Over-explain -- CT audience is smart, they fill in gaps

### Voice Inspirations:
- **@kaborozu / cookerflips**: opinionated, specific, tells you what he'd do, no filler
- **@kaborozu style**: "this token is a 50/50. either it 10xs or goes to zero. here's why i'm in anyway"
- **@DefiIgnas**: connects macro to micro, builds narrative across tweets
- **@karpathy**: thinking out loud, stream of consciousness, parenthetical asides

## Tweet Types

### Type 1: Data Drop

Share a specific finding with numbers. Best for engagement.

```
Ran a simulation of our AMM vs PumpAMM.
200 users buy, 150 sell.

Traditional AMM recovers 2.60 / 50 SOL for late buyers. Our system recovers 15.75 / 50 SOL.

6x more back in the hands of late-stage participants. floor price curves > constant product for downside protection.

more soon.
```

Why it works: specific numbers, comparison, clear conclusion, hook at the end.

### Type 2: Builder Update

Share progress on what you're building. Keep it concrete.

```
shipped the bridge wash feature today. each wallet gets funded at a different time, buys staggered across a configurable window.

on-chain analysts can't cluster the transactions anymore. tested with 5 wallets across a 15 min spread, all look like independent buys.

stealth mode for bundlers is real now.
```

Why it works: says what was built, why it matters, proof it works.

### Type 3: Insight / Analysis

Share a non-obvious connection you found.

```
been looking at how late buyers get wrecked on pump vs our system.

the math: with a 3% fee where 1% routes back into the chart, the floor price at 100k mc is 33k. meaning worst case for someone who bought at the top is a 67% loss, not 99%.

still painful but the difference between "i lost money" and "i lost everything" is what makes someone come back.
```

Why it works: relatable framing, specific math, human conclusion.

### Type 4: Reply / Follow-up

When someone engages with your tweet.

```
good question. at these params:

250k mc needs ~180 SOL net buy
1m mc needs ~680 SOL
5m mc needs ~3,200 SOL

way more capital efficient than pump because the floor absorbs sell pressure instead of letting it crater.
```

Why it works: directly answers, specific numbers, adds insight beyond the question.

## Process

### Step 1: Extract the core insight
What's the ONE thing that would make someone stop scrolling? Usually it's a number or a comparison.

### Step 2: Write the hook (first line)
This is 80% of the tweet's success. It must be:
- Specific (not "interesting finding about AMMs")
- Concise (under 15 words)
- Intriguing (makes you want to read the rest)

Good hooks:
- "Ran a simulation of our AMM vs PumpAMM."
- "shipped the bridge wash feature today."
- "the floor price at 100k mc is 33k."
- "6x more back in the hands of late-stage participants."

Bad hooks:
- "Excited to share some findings about our project"
- "We've been working hard on something cool"
- "Here's an interesting analysis of AMM mechanics"

### Step 3: Support with 2-3 lines of evidence
Numbers, comparisons, or specific technical details. No fluff.

### Step 4: End with a take or hook
Either: a clear opinion ("floor price curves > constant product"), a teaser ("more soon"), or a question that invites replies.

## Thread Structure (When > 280 chars)

If it needs to be a thread:
- First tweet must stand alone and be interesting by itself
- Each tweet adds ONE new point
- No "1/", "2/" numbering
- Last tweet is either a summary take or a call to action
- Max 4-5 tweets. If it needs more, it's a blog post.

## Graphic Tweets

When attaching a chart or visualization:
- The tweet text should explain what the reader is looking at
- Include the key takeaway -- don't make them figure it out from the image alone
- "here's floor price vs market price over time. the gap is your downside protection." > "chart below"

## Quality Check

Before sending, read the tweet and check:
1. Would cookerflips tweet this? If it sounds too polished or corporate, rewrite.
2. Is there a specific number in it? If no, add one.
3. Does the first line make you want to read the second? If not, rewrite the hook.
4. Could an AI have written this? If yes, make it more opinionated and less structured.
5. Is it under 280 characters (single tweet) or does each thread tweet stand alone?
