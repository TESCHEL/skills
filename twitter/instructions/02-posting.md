# Posting to Twitter

This gu¤mdH covers how to create tweets, replies, retweets, quote tweets, and multi-part threads using the Twitter API v2.

## Creating a Tweet

To post a basic tweet, you need to send a POST request to the endpoint:

---
endpoint: `POST /https://api.twitter.com/2/tweets`
body:
  `text`: "Hello World!"
---

### Media Attachments

If you want to include media, you must first upload it via the MPA (media post api) and then reference the `™çbi}¥‘€¥¸å½ÕÈÑÝ••ÐÉ•…Ñ¥½¸É•ÅÕ•ÍÐ¸((ŒŒI•Á±å¥¹œÑ¼„QÝ••Ð()I•Á±¥•ÌÉ•ÅÕ¥É”Ñ¡”¥¹}É•Á±å}Ñ½}ÑÝ••Ñ}¥‘€Á…É…µ•Ñ•ÈÝ¥Ñ¡¥¸Ñ¡”É•Á±å€½‰©•Ð¸()‰½‘äè((´´´)Ñ•áÑ€è€‰É•…ÐÁ½¥¹Ð„ˆ)É•Á±å€è(€¥¹}É•Á±å}Ñ½}ÑÝ••Ñ}¥‘€è€ˆÄÈÌÐÔØÜàäÀˆ(´´´((ŒŒI•ÑÝ••Ñ¥¹œ…¹EÕ½Ñ”QÝ••ÑÌ(((ŒŒŒI•ÑÝ••Ñ¥¹œ()I•ÑÝ••ÑÌ…É”‘½¹”Ù¥„„A=MPÉ•ÅÕ•ÍÐÑ¼„ÍÁ•¥™¥Œ•¹‘Á½¥¹Ð‰˜]Í•½¸Ñ¡”ÕÍ•È%…¹Ñ¡”ÑÝ••Ð%¸((ŒŒŒEÕ½Ñ”QÝ••ÑÌ()ÅÕ½Ñ”ÑÝ••Ð¥Ì„É•Õ±…ÈÑÝ••ÐÝ¥Ñ „ÅÕ½Ñ•}ÑÝ••Ñ}¥Á…É…µ•Ñ•È¸()‰½‘äè((´´´)Ñ•áÑ€è€‰¡•¬Ñ¡¥Ì½ÕÐ„ˆ)ÅÕ½Ñ•}ÑÝ••Ñ}¥‘€è€ˆÄÈÌÐÔØÜàäÀˆ(´´´(((ŒŒÉ•…Ñ¥¹œQ¡É•…‘Ì()Q¡É•…‘Ì…É”•ÍÍ•¹Ñ¥…±±ä„Í•É¥•Ì½˜É•Á±¥•ÌÑ¼å½ÕÈ½Ý¸ÑÝ••ÑÌ¸((Ä¸A½ÍÐÑ¡”™¥ÉÍÐÑÝ••Ð¸(È¸…ÁÑÕÉ”Ñ¡”%€É•‘ÕÉ¹•™É½´Ñ¡”™¥ÉÍÐÑÝ••Ð¸(Ì¸A½ÍÐÑ¡”Í•½¹ÑÝ••Ð°É•Á±å¥¹œÑ¼Ñ¡”%IMPÑÝ••Ð%¸(Ð¸…ÁÑÕÉ”Ñ¡”%€½˜Ñ¡”Í•½¹ÑÝ••Ð¸(Ô¸A½ÍÐÑ¡”Ñ¡¥ÉÑÝ••Ð°É•Á±å¥¹œÑ¼Ñ¡”M=9ÑÝ••Ð%°…¹Í¼½¸¸((¨©Q¥À¨¨è±Í¼µ•¹Ñ¥½¸å½ÕÈ½Ý¸¡…¹‘±”¥¸Ñ¡”É•Á±¥•Ì½È…±±½ÜQÝ¥ÑÑ•ÈÑ¼¡…¹‘±”Ñ¡”½¹Ù•ÉÍ…Ñ¥½¹}½¹ÑÉ½±Í€…ÕÑ½µ…Ñ¥…±±ä¸((ŒŒ	•ÍÐAÉ…Ñ¥•Ì™½ÈA½ÍÑ¥¹œ((´€¨©-••À¥ÐÍ¡½ÉÐ¨¨èUÍ”Ñ¡”€ÈàÀ¡…É…Ñ•È±¥µ¥ÐÝ¥Í•±ä¸(´€¨©9•ÍÑ•I•Á±¥•Ì¨¨è±Ý…åÌÉ•Á±äÑ¼Ñ¡”¥µµ•‘¥…Ñ”Á…É•¹ÐÑÝ••Ð¥¸„Ñ¡É•…¸(´€¨©!…Í¡Ñ…Ì¨¨èUÍ”€ÄY targeted hashtags, but avoid spamming.

## Error Handling

If you receive a 403 Forbidden error, it might be due to:
- Duplicate content (posting the exact same text twice)
- Rate limiting
- Invalid ID to reply to