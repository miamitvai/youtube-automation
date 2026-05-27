---
name: youtube-automation
description: >
  End-to-end YouTube video creation and publishing workflow. Use this skill whenever the user wants to:
  create a YouTube video, make a nature scenery video, generate images for a video, upload a video to YouTube,
  manage YouTube playlists, generate video titles/descriptions/tags, set up the YouTube API, or automate
  any part of the YouTube content creation pipeline. Trigger even if the user only mentions one step of
  the workflow (e.g. "upload my video" or "generate tags for my video" or "make a scenery video in CapCut").
---

# YouTube Automation Skill

This skill guides you through the full pipeline: generating beautiful scenery images with Grok → assembling a video in CapCut → generating SEO metadata → uploading to YouTube via the API → adding to a playlist.

Work through the stages in order, or jump to whichever stage the user needs.

---

## Stage 1 — Generate Nature Scenery Images with Grok

Generate **5 unique images** for the video using Grok's image generation at [grok.com](https://grok.com).

### Prompt the user to open grok.com
Ask them to go to grok.com and click the image/Aurora generation tab.

### Suggest 5 image prompts
Generate 5 varied prompts covering different types of beautiful nature scenery. Mix landscapes so the video feels dynamic. Examples:

1. `aerial view of a misty mountain range at sunrise, golden light, ultra-realistic, 4K`
2. `crystal clear turquoise waterfall in a lush tropical rainforest, sunbeams through canopy, cinematic`
3. `vast golden wheat field at sunset with dramatic clouds, warm tones, photorealistic`
4. `snow-capped alpine lake reflection, mirror-still water, blue hour, ultra-detailed`
5. `ancient redwood forest path, shafts of light through fog, serene and majestic`

Tailor prompts to any theme the user specifies (seasons, regions, mood). Generate one image per prompt.

### Download
Ask the user to download all 5 images to a single folder (e.g., `~/Videos/scenery-project/images/`).

---

## Stage 2 — Assemble the Video in CapCut

Open the existing CapCut project called **MUSIC**. Do NOT create a new project. The MUSIC project already has the logo and intro set up — only replace the video clips and the background music track. Everything else stays untouched.

### Steps
1. **Open the MUSIC project** in CapCut (desktop or mobile)
2. **Replace the video clips** → select each existing scenery clip on the timeline and swap it with the new Grok image:
   - Right-click the clip → "Replace" (desktop) or long-press → "Replace" (mobile)
   - Replace all 5 clips with the 5 new images in order
   - Keep the same durations already set in the project
3. **Replace the music** → find the audio/music track on the timeline:
   - Delete the existing music track only
   - Add the new audio file in its place
   - Trim or loop the audio to match the video length
4. **Leave untouched** → logo, intro, text overlays, transitions, effects, and any other elements already in the project
5. **Export** → Export at the same resolution already used in the project (1080p or 4K), MP4, to `~/Videos/scenery-project/`

### Checklist before export
- [ ] MUSIC project opened (not a new project)
- [ ] All 5 scenery clips replaced with new Grok images
- [ ] Music track replaced with new audio
- [ ] Logo and intro untouched
- [ ] Duration and transitions unchanged

---

## Stage 3 — Generate Video Metadata

Generate optimized metadata for the video before uploading.

Ask the user: **What is the theme or mood of this video?** (e.g., "relaxing nature", "morning motivation", "4K scenery")

Then generate:

### Title (3 options, pick one)
- Keep under 70 characters
- Include keywords people search for
- Example: `Breathtaking Nature Scenery 4K | Relaxing Landscapes [No Music]`

### Description
```
[2-3 sentence hook about the video]

🌿 Watch stunning nature scenes in beautiful 4K quality. Perfect for relaxation, study, or sleep.

Timestamps:
0:00 - [Scene 1 name]
0:06 - [Scene 2 name]
...

🔔 Subscribe for more nature videos: [channel link]
📌 Playlist: [playlist link]

#nature #scenery #relaxing #4K #landscape #naturevideo #beautifulnature #calmingvideos
```

### Tags (15–20)
Generate relevant tags: nature, scenery, 4K nature, relaxing video, landscape, beautiful scenery, nature sounds, calm, meditation, study music, sleep, ambient, etc.

---

## Stage 4 — Set Up YouTube Data API

First-time setup only. Skip to Stage 5 if already configured.

### Step-by-step setup

1. **Go to** [console.cloud.google.com](https://console.cloud.google.com)
2. **Create a new project** → click "New Project" → name it `youtube-uploader`
3. **Enable the API**:
   - Go to "APIs & Services" → "Library"
   - Search "YouTube Data API v3" → click Enable
4. **Create credentials**:
   - Go to "APIs & Services" → "Credentials"
   - Click "Create Credentials" → "OAuth client ID"
   - Application type: **Desktop app**
   - Download the JSON file → rename it `client_secrets.json`
   - Save to `~/Videos/scenery-project/client_secrets.json`
5. **Configure OAuth consent screen**:
   - Go to "OAuth consent screen"
   - Set to **External**, fill in app name
   - Add your Gmail as a test user

### Install the upload script dependencies
```bash
pip install google-auth google-auth-oauthlib google-api-python-client
```

---

## Stage 5 — Upload to YouTube

### Upload script
Create `upload_video.py` in `~/Videos/scenery-project/`:

```python
import os
import pickle
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request

SCOPES = ["https://www.googleapis.com/auth/youtube.upload",
          "https://www.googleapis.com/auth/youtube"]

def authenticate():
    creds = None
    if os.path.exists("token.pickle"):
        with open("token.pickle", "rb") as f:
            creds = pickle.load(f)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file("client_secrets.json", SCOPES)
            creds = flow.run_local_server(port=0)
        with open("token.pickle", "wb") as f:
            pickle.dump(creds, f)
    return build("youtube", "v3", credentials=creds)

def upload_video(youtube, video_file, title, description, tags, playlist_id=None):
    body = {
        "snippet": {
            "title": title,
            "description": description,
            "tags": tags,
            "categoryId": "22"  # People & Blogs; use 19 for Travel & Events
        },
        "status": {"privacyStatus": "public"}  # or "private" / "unlisted"
    }
    media = MediaFileUpload(video_file, chunksize=-1, resumable=True)
    request = youtube.videos().insert(part="snippet,status", body=body, media_body=media)
    response = request.execute()
    video_id = response["id"]
    print(f"✅ Uploaded: https://youtube.com/watch?v={video_id}")

    if playlist_id:
        youtube.playlistItems().insert(
            part="snippet",
            body={"snippet": {"playlistId": playlist_id, "resourceId": {"kind": "youtube#video", "videoId": video_id}}}
        ).execute()
        print(f"✅ Added to playlist")

    return video_id

if __name__ == "__main__":
    youtube = authenticate()

    # --- FILL THESE IN ---
    VIDEO_FILE = "your-video.mp4"
    TITLE = "Breathtaking Nature Scenery 4K | Relaxing Landscapes"
    DESCRIPTION = """Watch stunning nature scenes...

#nature #scenery #relaxing"""
    TAGS = ["nature", "scenery", "4K", "relaxing", "landscape", "beautiful"]
    PLAYLIST_ID = None  # Set to your playlist ID, or None to skip
    # ---------------------

    upload_video(youtube, VIDEO_FILE, TITLE, DESCRIPTION, TAGS, PLAYLIST_ID)
```

### Run it
```bash
cd ~/Videos/scenery-project
python upload_video.py
```
A browser window will open for Google sign-in on the first run. After that, the token is saved and future uploads are fully automatic.

---

## Stage 6 — Manage Playlists

### Create a new playlist
```python
playlist = youtube.playlists().insert(
    part="snippet,status",
    body={
        "snippet": {"title": "Beautiful Nature Scenery", "description": "Relaxing 4K nature videos"},
        "status": {"privacyStatus": "public"}
    }
).execute()
playlist_id = playlist["id"]
print(f"Playlist ID: {playlist_id}")
```

Pass this `playlist_id` into `upload_video()` to automatically add each video to the playlist.

### List existing playlists
```python
playlists = youtube.playlists().list(part="snippet", mine=True, maxResults=20).execute()
for p in playlists["items"]:
    print(p["id"], p["snippet"]["title"])
```

---

## Quick-reference checklist

| Step | Tool | Done? |
|------|------|-------|
| Generate 5 images | grok.com | [ ] |
| Assemble video | CapCut | [ ] |
| Generate metadata | Claude | [ ] |
| Set up YouTube API | Google Cloud | [ ] |
| Upload video | upload_video.py | [ ] |
| Add to playlist | upload_video.py | [ ] |
