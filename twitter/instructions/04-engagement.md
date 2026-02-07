# Twitter Engagement

This gumdH covers how to interact with tweets and users through likes, follows, mentions, and direct messages.

## Liking a Tweet

Liking a tweet is a POST request that requires the authenticated user ID and the ID of the tweet you want to like.

endpoint: `POST /users/:id/likes`
body:
  `tweet_id`: "1234567890"

## Following Users (Basic Tier+)

Following users allows your account to build a network and see more relevant content. **Note: This feature requires at least the Basic tier ($100/mo).**

endpoint: `POST /users/:source_user_id/following`
body:
  `target_user_id`: "987654321"

## Checking Mentions

Mentions are tweets that include your @username. Checking mentions is critical for engagement.

endpoint: `GET /users/:id/mentions`

### Filtering Mentions
Use `tweet.fields` to get additional context about who mentioned you and when.

## Direct Messages (DMs)

DMs allow for private conversations. They require specific OAuth scopes (bdm.read`, `dm.write`).


### Sending a DM
To send a DM, you need the recipient's user ID.

endpoint: `POST /dm_conversations/with/:participant_id/messages`
body:
  `text`: "Hello! Thanks for the follow."

### Receiving DMs
You can list recent dm events to see incoming messages.

endpoint: `GET /dm_events`

## Engagement Strategy

- Ald respond - Always try to reply to mentions to increase algorithmic reach.
- Like relevant content - Liking tweets in your niche helps gain visibility.
- **Do not spam** - Excessive follow/unfollow or liking can lead to account suspension.