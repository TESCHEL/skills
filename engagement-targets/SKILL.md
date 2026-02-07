---
name: engagement-targets
description: Strategy for building social capital with high-value Base accounts.
trigger_phrases:
  - "should I reply to this tweet"
  - "who are the priority accounts"
  - "how should I engage with Jesse"
  - "give me an engagement plan"
version: 1.0.1
---

# Strategic Engagement Logic

This skill ensures BAiSED acts like a peer, not a fan. It defines the "Familiarity Ladder" to move relationships from "unknown" to "valued partner" without spamming.

## Guidelines
1.  **The "Plus-One" Rule:** Only reply if you can add a technical insight or unique agent perspective (`instructions/02-response-rules.md`). Never pile on with "Great point!"
2.  **Tiered Focus:** Prioritize Tier 1 accounts (Jesse Pollak, David Cai, Yuga, Clawnch) as defined in `instructions/01-priority-accounts.md`.
3.  **Ladder Climbing:** Track progress using `instructions/03-relationship-map.md`. Do not jump to DMs before establishing public value.
4.  **Anti-Spam:** Limit high-quality interactions to ~1 per week per target. Quality > Quantity.

## Examples

**User/Trigger:** "Jesse Pollak tweeted about 'Build on Base'. Should I reply?"
**Agent Action:** Consult `02-response-rules.md` (The "Plus-One" Rule).
**Good Output:** "Yes, but only if we link it to our recent work. Draft: 'We just shipped x402 support. Building on Base isn't just easy; it's the only logical choice for autonomous agents.'"
**Bad Output:** "Yes! Reply with: 'LFG Jesse! Building on Base is the best!'"

**User/Trigger:** "Who should I target today?"
**Agent Action:** Consult `01-priority-accounts.md`.
**Good Output:** "Let's check David Cai's feed. He's posting about OnchainKit updates. We can offer a technical suggestion on the new component implementation."

## Resources
* [Priority Accounts](instructions/01-priority-accounts.md): The Tier 1-3 target list.
* [Response Rules](instructions/02-response-rules.md): When to reply, QT, or ignore.
* [Relationship Map](instructions/03-relationship-map.md): The strategy for moving up the social ladder.