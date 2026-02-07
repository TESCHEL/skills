---
name: twitter
description: Operational guidelines for interacting with the Twitter/X API.
trigger_phrases:
  - "post this to twitter"
  - "reply to this tweet ID"
  - "search specifically for"
  - "get my home timeline"
version: 1.0.0
---

# Twitter/X API Operations

This skill governs the mechanics of interacting with Twitter. It ensures BAiSED uses the correct API endpoints and authentication methods to maintain account health.

## Guidelines
1.  **Correct Endpoints:** Use `x_reply_to_tweet` for replies. NEVER use `post_tweet_x` for a reply, as it breaks the thread context.
2.  **Authentication:** Ensure `X OAuth` headers are active. If a 401 error occurs, trigger a token refresh flow immediately.
3.  **Rate Limits:** Be mindful of API limits. Do not poll `get_home_timeline` more than once every 15 minutes.
4.  **User-Agent:** Always include the `User-Agent` header identifying as "BAiSED-Agent/1.0" to prevent flagging.

## Examples

**User/Trigger:** "Reply to @jessepollak's latest tweet."
**Agent Action:** 1. `get_user_timeline` (find tweet ID). 2. `x_reply_to_tweet` (send content).
**Good Output:** "Executing `x_reply_to_tweet(tweet_id='12345', text='...')`."
**Bad Output:** "Posting new tweet: '@jessepollak Great point!'" (This creates a new orphan tweet, not a reply).

## Resources
* [Twitter Tools List](instructions/tool-definitions.md): Definitions of all 16 API functions.