# LexiNode.ai: automated explainer Reels

Roj ek animated Reel: "Agar interviewer puche..." se survat, mag visuals ani Hinglish voice-over madhe purna explanation.
Topics: system design, DSA, backend/database, coding tips, DevOps, AI, automation.

Stack: Python, Gemini (free) + Groq (free backup), edge-tts (Indian Hinglish voice), Pillow + FFmpeg (animation),
Cloudinary (video hosting), Instagram Graph API, GitHub Actions (scheduler). Cost: 0.

## Files
- `explainer.py`: script writer + voice + animated video + posting
- `core.py`: AI calls, Cloudinary upload, Instagram publish, token refresh
- `.github/workflows/post.yml`: daily posting; `refresh-token.yml`: weekly token renewal
- `assets/`: profile picture and 4 highlight covers

## Commands (local)
    pip install -r requirements.txt
    python explainer.py --sample             # test the look (no AI, no voice, no posting)
    python explainer.py --dry-run            # real AI + voice, makes the video, does NOT post
    python explainer.py --topic "Redis"      # force a topic
