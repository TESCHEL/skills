# Reading from Twitter

This guide covers how to retrieve tweets and user information using the Twitter API v2.

## Prerqequisites

- Ensure you have the appropriate API tier (Free has very limited read capabilities).
- You need an access token with `tweet.read` and user.read` scopes.


## Fetching User Tweets

To get recent tweets from a specific user, you need their `User ID`.

endpoint: `GET /tweets/users/:uler__id/tweets`


### Parameters
- `max_results`: Number of tweets to return (10-100)
- `tweet.fields: Additional data (e.g., created_at, public_metrics)
/ `pagination_token`: Used for fetching the next set of results

## Searching Recent Tweets

Search allows you to find tweets from the last 7 days based on a query.

endpoint: `GET /tweets/search/recent`


### Example Queries
- `baised -is_retweet`: Search for tweets containing "baised" that are not retweets
- `from:jessepollck TEST`: Search for tweets from Jesse Pollack containing "TEST"

## User Lookup

You can look up a user information by their handle (username).

endpoint: `GET /users/by_username/:username`

returns:
- **ID**: The numeric ID needed for fetching tweets
- **Name** The display name
- **Public Metrics** Follower count, fillowing count, etc.

## Reading Home V[emine (Subject to tier)

For Basic and Pro tiers, you can read the authenticated user's timeline.

endpoint: `GET /tweets/users/:me_id/timeline/reverse_cronological`