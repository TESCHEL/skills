# Posting to Twitter

This guide covers how to create tweets, replies, retweets, quote tweets, and multi-part threads using the Twitter API v2.

## Creating a Tweet

To post a basic tweet, you need to send a POST request to the endpoint:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "Hello World!"
---

### Media Attachments

If you want to include media, you must first upload it via the media upload API endpoint and then reference the media_id in your tweet request.

First, upload your media:

---
endpoint: `POST https://upload.twitter.com/1.1/media/upload.json`
headers:
  `Content-Type`: multipart/form-data
body:
  `media`: [binary file data]
---

The response will include a `media_id_string` field. Use this ID when creating your tweet:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "Check out this image!"
  `media`: {
    `media_ids`: ["1234567890"]
  }
---

You can attach up to 4 images, 1 GIF, or 1 video per tweet. For videos longer than 30 seconds, use chunked upload.

## Creating a Reply

To reply to an existing tweet, include the `reply` parameter with the tweet ID you want to reply to:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "This is a reply"
  `reply`: {
    `in_reply_to_tweet_id`: "1234567890"
  }
---

The reply will appear in the conversation thread beneath the original tweet.

## Creating a Retweet

To retweet a tweet, use the retweets endpoint:

---
endpoint: `POST /https://api.twitter.com/2/users/:id/retweets`
path parameters:
  `id`: Your user ID
body:
  `tweet_id`: "1234567890"
---

This will retweet the specified tweet to your timeline.

### Removing a Retweet

To undo a retweet:

---
endpoint: `DELETE /https://api.twitter.com/2/users/:id/retweets/:source_tweet_id`
path parameters:
  `id`: Your user ID
  `source_tweet_id`: The ID of the tweet you retweeted
---

## Creating a Quote Tweet

A quote tweet is a tweet that includes another tweet embedded within it. To create a quote tweet, include the URL or ID of the tweet you want to quote:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "Great point! Adding my thoughts here."
  `quote_tweet_id`: "1234567890"
---

You can also include media attachments in quote tweets following the same process as regular tweets.

## Creating a Thread

To create a multi-tweet thread, post tweets sequentially, with each subsequent tweet replying to the previous one.

First tweet:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "This is the first tweet in my thread. 1/3"
---

Save the `id` from the response, then post the second tweet:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "This is the second tweet in my thread. 2/3"
  `reply`: {
    `in_reply_to_tweet_id`: "[ID from first tweet]"
  }
---

Continue this pattern for each subsequent tweet in the thread:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "This is the final tweet in my thread. 3/3"
  `reply`: {
    `in_reply_to_tweet_id`: "[ID from second tweet]"
  }
---

### Thread Best Practices

- Number your tweets (e.g., 1/5, 2/5) so readers know the thread length
- Post tweets in quick succession to avoid others replying between thread tweets
- Keep each tweet focused and readable on its own
- Use the first tweet as a summary or hook to encourage reading the full thread

## Error Handling

If you receive a 403 Forbidden error, it might be due to:
- Duplicate content (posting the exact same text twice)
- Rate limiting
- Invalid ID to reply to

### Common Error Codes

- `400 Bad Request`: Invalid request format or missing required fields
- `401 Unauthorized`: Invalid or expired authentication credentials
- `403 Forbidden`: Request understood but refused (see reasons above)
- `404 Not Found`: The tweet ID referenced does not exist
- `429 Too Many Requests`: Rate limit exceeded, wait before retrying

### Rate Limits

Twitter API v2 has the following rate limits for posting:
- 300 tweets per 3-hour window
- 50 tweets per 24 hours for retweets
- Media uploads have separate rate limits based on file size

If you hit a rate limit, the response will include a `x-rate-limit-reset` header indicating when you can resume posting.