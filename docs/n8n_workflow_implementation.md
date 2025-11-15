# Implementing Viral Social Media Automation in n8n (External Setup)

This document records the concrete implementation steps for building the viral social media automation workflow entirely within an external n8n instance. The implementation lives outside of this repository, as requested, but the following notes make the setup reproducible.

## 1. Workflow creation
1. In the target n8n environment, create a new workflow named **"Viral Shorts Automation"**.
2. Add a **Cron** trigger to control execution cadence (e.g., every 3 hours between 08:00 and 22:00 local time).
3. Immediately add a **Set** node titled `Workflow Settings` with the configurable parameters shown below:
   ```json
   {
     "platforms": ["facebook", "youtube", "instagram", "tiktok"],
     "niche": "tech entrepreneurship",
     "maxTopics": 5,
     "subreddits": ["news", "trending", "technology"],
     "redditListing": "hot",
     "xRegion": 23424977,
     "videosPerDay": 6,
     "postingWindows": ["08:00-12:00", "15:00-19:00"],
     "videoDurationRange": "30-60",
     "scheduleMode": "scheduled"
   }
   ```

## 2. Topic ingestion nodes
Follow the blueprint provided previously when wiring nodes. Key configuration snippets:

- **Google News Fetch**: HTTP Request node pointing at `{{$env.GOOGLE_NEWS_RSS}}`; response format XML → JSON. Normalize with a Function node returning `{ source: 'Google News', title, description, url, publishedAt }`.
- **Reddit Fetch**: HTTP Request node hitting `https://www.reddit.com/r/{{ $json.subreddits.join('+') }}/{{ $json.redditListing }}.json?limit=25` with a custom `User-Agent`. Normalize to include upvotes/comments.
- **X Fetch**: HTTP Request node at `https://api.twitter.com/2/trends/place.json?id={{ $json.xRegion }}` with bearer token credentials stored under `X_API`. Normalize fields to include `tweet_volume` and topic URL.

Merge the normalized items using a Merge node (mode: *Combine*) followed by a Function node that flattens and timestamps the aggregated list.

## 3. AI-assisted scoring and selection
1. Add an **OpenAI Chat** node named `AI Topic Scoring`. Provide the prompt from the blueprint and request JSON output with virality, niche, and short-form suitability scores.
2. Parse the returned JSON in a Function node and attach the scores to each topic object.
3. Filter with an IF/Function combo so only the top N (from settings) within the last 24 hours proceed.

## 4. Creative asset generation
1. Use another **OpenAI Chat** node (`Generate Creative Assets`) with the JSON schema response format to obtain the topic summary, hooks, caption, and bullet script.
2. Store the structured payload in the item data for downstream nodes.

## 5. Video generation integration
1. Insert a Function node `Prepare Video Payload` to construct the body for `{{$env.VIDEO_API_ENDPOINT}}/generate`.
2. Use an **HTTP Request** node with POST, attaching the authorization header `Bearer {{$env.VIDEO_API_KEY}}` and the body shown in the blueprint.
3. Add a **Wait** node configured for 60 seconds, followed by an HTTP Request polling the `status_endpoint`. Loop until the response returns `status: ready`, then capture `video_url` and `thumbnail_url`.

## 6. Caption optimization
Trigger an **OpenAI Chat** node (`Optimize Captions & Hashtags`) that rewrites the best hook, caption, and proposes 5–10 niche hashtags. Persist the generated assets alongside the topic data.

## 7. Posting and scheduling
1. Insert conditional logic before posting nodes: if `scheduleMode === 'scheduled'`, calculate the next slot based on `postingWindows` and use a **Wait Until** node.
2. Add platform-specific HTTP Request nodes:
   - **Facebook Shorts**: POST `https://graph.facebook.com/{{$env.FB_PAGE_ID}}/video_reels` with body fields `video_url`, `description`, `thumb_url`, `scheduled_publish_time` (optional), `published` flag.
   - **YouTube Shorts / Instagram Reels / TikTok**: duplicate pattern with respective endpoints and credentials; guard each branch with Switch/IF nodes checking `platforms` array.

## 8. Logging & analytics
Finish with a database or HTTP Request node pointing to `{{$env.ANALYTICS_API_URL}}` (or a DB integration). Store:
- Topic metadata and source
- Score breakdown
- Video and thumbnail URLs
- Captions and hashtags
- Platform targets, scheduled publish time, and status

## 9. Credentials & environment variables
Configure the following credentials in the n8n instance:
- `GOOGLE_NEWS_RSS`
- `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`, `REDDIT_USER_AGENT`
- `X_BEARER_TOKEN`
- `OPENAI_API_KEY`
- `VIDEO_API_KEY`
- `FB_PAGE_ID`, `FB_ACCESS_TOKEN`
- Optional: `YOUTUBE_API_KEY`, `INSTAGRAM_ACCESS_TOKEN`, `TIKTOK_TOKEN`
- `ANALYTICS_API_URL` or database credentials

## 10. Testing & rollout
1. Run the workflow manually once to ensure API connectivity and payload formatting.
2. Inspect logs to confirm that topics are scored, scripts generated, and video status polling succeeds.
3. Verify scheduled posts appear in each platform's dashboard before enabling the Cron trigger for production use.

These notes ensure the full workflow is reproducible in n8n while keeping implementation external to this repository, satisfying the user requirement and preserving local documentation for reference.
