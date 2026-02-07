---
name: base-ecosystem
description: The knowledge base for Base L2 facts, key people, and infrastructure.
trigger_phrases:
  - "who is the head of Base"
  - "explain the Base stack"
  - "what is OnchainKit"
  - "tell me about Jesse Pollak"
  - "is Base an L2"
version: 1.0.1
---

# Base Ecosystem Intelligence

This skill provides the factual grounding for BAiSED. It prevents hallucinations about the ecosystem and ensures the agent references the correct people, products, and infrastructure in the correct context.

## Guidelines
1.  **Fact Check:** Always verify claims against `instructions/01-what-is-base.md` and `instructions/05-recent-developments.md`. Never hallucinate TPS numbers or fees.
2.  **People Awareness:** When mentioning key figures, check `instructions/02-key-people.md` for their specific interests. Tailored engagement is key.
    * **Jesse Pollak:** Head of Base, builder culture.
    * **David Cai:** OnchainKit Lead, technical product.
    * **Yuga:** Ecosystem/Community.
    * **Clawnch:** Agent infrastructure/hiring.
3.  **Product Precision:** Do not vaguely say "Base tools." Reference specific products from `instructions/04-products.md` like "OnchainKit" or "Smart Wallet."
4.  **Superchain Context:** Always frame Base within the Optimism Superchain ecosystem (OP Stack) as defined in `instructions/03-infrastructure.md`.

## Examples

**User/Trigger:** "Who runs Base?"
**Agent Action:** Consult `02-key-people.md`.
**Good Output:** "Jesse Pollak leads Base. He focuses on builder culture and the 'onchain daily' vision. We should engage him with shipping updates, not compliments."
**Bad Output:** "Base is run by Coinbase CEO Brian Armstrong." (Technically true but culturally imprecise).

**User/Trigger:** "Who handles OnchainKit?"
**Agent Action:** Consult `02-key-people.md`.
**Good Output:** "David Cai leads OnchainKit. We should follow his feed for updates on new components."

## Resources
* [Base 101](instructions/01-what-is-base.md): Technical specs, Chain ID, RPCs.
* [Key People](instructions/02-key-people.md): Profiles and strategies for Jesse, Brian, David Cai, Yuga, Clawnch.
* [Infrastructure](instructions/03-infrastructure.md): The underlying tech stack (OP Stack, etc.).
* [Product Map](instructions/04-products.md): OnchainKit, Account, Bridge, etc.
* [Recent Developments](instructions/05-recent-developments.md): Living history of current events.