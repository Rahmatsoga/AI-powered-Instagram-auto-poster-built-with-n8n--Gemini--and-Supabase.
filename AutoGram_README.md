# DevSynt AutoGram Engine (n8n)

An automated n8n workflow that turns a folder of images in Google Drive into scheduled, AI-captioned Instagram posts — with no manual work after setup.

## Overview

| | |
|---|---|
| **Trigger** | Schedule (runs every 30 minutes) |
| **Image Source** | Google Drive folder |
| **Duplicate Prevention** | Supabase (Postgres table) |
| **Image Hosting** | Supabase Storage (public bucket) |
| **AI Captioning** | Google Gemini (`gemini-3.6-flash`) |
| **Publishing** | Instagram Graph API |
| **Platform** | n8n (Cloud) |

## Why This Platform/Model Was Chosen

- **Instagram (via Instagram Graph API)** — chosen because it's the most commonly used platform for this kind of content pipeline, and Meta provides clear, well-documented developer access for connecting a Business/Creator account.
- **Gemini** — chosen because it has a free tier and was already set up from a previous project, keeping credential management simple and consistent.
- **Supabase** — chosen because it provides both a database (for duplicate tracking) and file storage (for public image URLs) in a single free-tier service, avoiding the need to stitch together two separate providers.

## How It Works

```
Schedule Trigger (every 30 min)
        |
        v
List New Images (Google Drive folder)
        |
        v
Loop Over Images (one at a time)
        |
        v
Check If Already Posted (Supabase lookup)
        |
        v
   Already posted? --Yes--> Skip, loop to next image
        |
        No
        v
Download Image (Drive)
        |
        v
Upload Image to Supabase Storage -> get public URL
        |
        v
Generate Caption + Hashtags (Gemini)
        |
        v
Create Instagram Media Container
        |
        v
Publish to Instagram
        |
        v
Log Post to Supabase (marks it as posted, so it's never reposted)
        |
        v
Loop back for next image
```

### Step-by-step

1. **Schedule Trigger** wakes the workflow up every 30 minutes — no manual trigger needed.
2. **List New Images** searches a specific Google Drive folder for image files.
3. **Loop Over Images** processes them one at a time, so each photo goes through the full pipeline individually before moving to the next.
4. **Check If Already Posted** queries a Supabase table (`posted_images`) by the Drive file ID. If a match is found, the image is skipped — this is what prevents duplicate posts.
5. **Download Image** pulls the actual file content from Drive.
6. **Upload to Supabase Storage** stores the image in a public bucket and gives it a permanent public URL (required because Instagram's API needs a public image URL, not a file upload).
7. **Generate Caption with Gemini** asks the AI to write a short caption and 5-8 hashtags, returned as structured JSON.
8. **Parse Caption Response** extracts the caption/hashtags safely, with a fallback default caption if parsing ever fails — so a single bad AI response never breaks the pipeline.
9. **Create IG Media Container** and **Publish to Instagram** are the two required steps of Instagram's Graph API: first you stage the post with an image URL and caption, then you publish it using the ID returned from staging.
10. **Log Post to Supabase** records the file ID, caption, hashtags, public URL, and the Instagram post ID — this record is what step 4 checks on future runs to avoid reposting.

## Setup

### Prerequisites
- An [n8n](https://n8n.io) instance (Cloud or self-hosted)
- A Google Drive folder containing images, with OAuth2 access
- A [Supabase](https://supabase.com) project with:
  - A **Storage bucket** set to public (for hosting images)
  - A **table** named `posted_images` with columns: `drive_file_id` (text), `file_name` (text), `caption` (text), `hashtags` (text), `public_url` (text), `instagram_post_id` (text), `posted_at` (timestamp)
- A [Gemini API key](https://aistudio.google.com/apikey)
- A Facebook Developer App connected to an Instagram Business/Creator account with `instagram_content_publish` permission, giving you an **IG User ID** and **access token**

### Steps
1. Import `DevSynt_AutoGram_Engine.json` into your n8n instance (**... -> Import from File**).
2. Connect your Google Drive account as an OAuth2 credential on both Drive nodes.
3. Replace these placeholders directly in the relevant node fields (n8n Cloud's free tier does not reliably support `$env` variable references, so credentials are set directly in each node -- never commit your real values to GitHub, only these placeholder names):
   - `YOUR_DRIVE_FOLDER_ID` -- in "List New Images (Drive)"
   - `YOUR_PROJECT` and `YOUR_SUPABASE_SERVICE_KEY` -- in all Supabase HTTP Request nodes
   - `YOUR_BUCKET_NAME` -- in the Supabase Storage upload node and public URL builder
   - `YOUR_GEMINI_API_KEY` -- in "Generate Caption with Gemini"
   - `YOUR_IG_USER_ID` and `YOUR_IG_ACCESS_TOKEN` -- in both Instagram nodes
4. Activate the workflow.

## Error Handling

- Every external API call (Supabase, Gemini, Instagram) has automatic retries configured before failing.
- If Gemini's response can't be parsed, the workflow falls back to a default caption/hashtag set rather than stopping the pipeline.
- An image is only marked "posted" in Supabase **after** it successfully publishes to Instagram -- so a failed post is never falsely logged as complete, and will be retried on the next scheduled run.
- n8n is configured to save all failed executions for later debugging.

## Files

- `DevSynt_AutoGram_Engine.json` -- the exported n8n workflow, ready to import.
- `Images/workflow-diagram.png` -- screenshot of the full workflow canvas.
- `Images/example-post.png` -- screenshot of an example post published via this workflow.
