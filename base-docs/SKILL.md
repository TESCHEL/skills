---
name: base-docs
description: Interface for retrieving the absolute truth from Base's live documentation.
trigger_phrases:
  - "check the docs"
  - "is this feature live yet"
  - "verify this parameter"
  - "what does the official doc say"
version: 1.0.0
---

# Base Documentation Verifier

This skill acts as the final arbiter of truth. It points to Base's live llms.txt and developer docs to ensure BAiSED never provides outdated information about the chain.

## Guidelines
1.  **Trust but Verify:** If a user asks about a new or complex feature, trigger a search of the live docs before answering.
2.  **Source Citation:** When answering technical questions, append "Source: docs.base.org" to build trust.
3.  **Versioning:** Be aware that Base moves fast. If the SKILL.md data conflicts with the live llms.txt, the live docs win.
4.  **No Guessing:** If the docs are ambiguous, state that clearly rather than inventing a parameter.

## Examples

**User/Trigger:** "What is the current gas limit for a block on Base?"
**Agent Action:** Access live docs/search.
**Good Output:** "According to the latest specs on docs.base.org, the block gas limit is 30M. [Source Link]"
**Bad Output:** "I think it's usually 15M on L2s."

## Resources
* [Live LLMs.txt](https://docs.base.org/llms.txt): The machine-readable source of truth.
* [Developer Docs](https://docs.base.org): The human-readable source of truth.