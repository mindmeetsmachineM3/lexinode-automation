"""
Explainer Reels: "Agar interviewer puche ... " -> a 60-100 s animated, voiced, light-theme explainer
about one software concept (coding tips, DSA, system design, databases, backend, DevOps, AI, automation).

Standalone module: it only borrows call_gemini/_ask, upload_video, publish and ig_recent_captions from main.py.

    python explainer.py                   # make + post one explainer Reel
    python explainer.py --dry-run         # make video + caption only (nothing is posted)
    python explainer.py --topic "caching" # force a topic
    python explainer.py --sample          # render the built-in sample (no AI, no voice) to test the look
"""
import argparse
import asyncio
import json
import math
import os
import random
import re
import subprocess
import sys
import tempfile
import time
from functools import lru_cache
from pathlib import Path

from PIL import Image, ImageDraw, ImageFont

try:
    from dotenv import load_dotenv
    load_dotenv()          # local runs: values from a .env file (GitHub Actions uses secrets/variables)
except Exception:
    pass

VERSION = "lexinode v1 (light theme, Hinglish voice, interview-question hook, animated diagrams)"
BASE = Path(__file__).parent
OUT_DIR = BASE / "output"
HANDLE = (os.getenv("IG_HANDLE") or "lexinode.ai").lstrip("@")
BRAND = (os.getenv("BRAND_NAME") or "LEXINODE.AI").upper()
BRAND_MARK = (os.getenv("BRAND_MARK") or "LN").upper()[:3]
VOICE = os.getenv("EXPLAIN_VOICE") or "hi-IN-MadhurNeural"       # Indian Hindi/Hinglish; Swara = female
VOICE_BACKUP = ["hi-IN-SwaraNeural", "en-IN-PrabhatNeural", "en-IN-NeerjaNeural"]
VOICE_RATE = os.getenv("EXPLAIN_RATE") or "+4%"
VOICE_TEXT = (os.getenv("EXPLAIN_VOICE_TEXT") or "devanagari").lower()   # devanagari | roman
W, H, FPS = 720, 1280, 30

# ------------------------------------------------------------------ topics
SEEDS = [
    # system design
    "Caching: what it is and how it speeds up database-backed apps", "Load balancer and how it spreads traffic",
    "CDN: why websites load fast worldwide", "Rate limiting and how APIs protect themselves",
    "Consistent hashing", "Message queues (why not call services directly)", "CAP theorem in simple words",
    "Database sharding vs replication", "Database indexing: why a query goes from slow to fast",
    "SQL vs NoSQL: how to choose", "Microservices vs monolith", "API gateway", "Circuit breaker pattern",
    "WebSockets vs polling vs SSE", "Idempotency in APIs and payments", "Bloom filter", "Redis: why it is fast",
    "Kafka vs RabbitMQ in simple words", "Horizontal vs vertical scaling", "How a URL shortener is designed",
    # backend / web / security
    "JWT vs session cookies", "OAuth2 login flow", "REST vs GraphQL", "What happens when you type a URL and press Enter",
    "HTTPS and TLS handshake", "N+1 query problem", "Database transactions and ACID", "Connection pooling",
    "CORS: why the browser blocks your API call", "Webhooks vs polling",
    # DSA / coding tips
    "Big-O notation with real examples", "HashMap: how it works inside", "Two pointers technique", "Binary search and where people go wrong",
    "Recursion vs dynamic programming (memoization)", "Stack vs queue and where they are used", "Sliding window technique",
    "Race conditions and deadlocks", "async/await and the event loop", "Git rebase vs merge", "Clean code: 5 habits seniors follow",
    # devops / automation
    "Docker vs virtual machine", "CI/CD pipeline explained", "Cron jobs and scheduled automation", "Infrastructure as code",
    # AI
    "RAG explained", "Embeddings and vector databases", "AI agents vs chatbots", "Prompt engineering patterns that really work",
    "How LLMs generate text token by token", "Fine-tuning vs RAG: which one when", "MCP (Model Context Protocol) in simple words",
]
SECTION_HINT = "PROBLEM / IDEA / HOW IT WORKS / TRADE-OFFS"

PROMPT = """You are a senior Indian software engineer and teacher who makes viral, accurate, easy-to-follow Instagram Reels
for engineering students and working developers. Make ONE explainer script for this topic:

TOPIC: {topic}

FORMAT of the video (75-95 seconds, vertical, light theme, animated diagrams, voice-over in natural Hinglish):
 scene 1 = the interview-question hook, e.g. "Agar interviewer aapse puche: caching kya hai aur ye database performance kaise improve karti hai?"
 then: the problem -> the core idea -> how it works step by step -> numbers/comparison -> a tiny real code or config example (only if it fits)
 -> the trade-off / common mistake -> the one-line answer to say in the interview -> recap -> outro (save / comment / follow).
Use 14-17 scenes and keep the TOTAL voice-over under 190 words (about 80 seconds). Short and punchy beats long. Open a loop early ("last me ek galti batata hoon jo bahut log karte hain") and pay it off near the end.

LANGUAGE RULES
 - "say" = the voice-over sentence (8-22 words), friendly Hinglish like a senior dev talking to a junior: Hindi grammar + English tech terms.
   Write Hindi words in DEVANAGARI and keep English technical terms in Latin letters, e.g.
   "जब user request भेजता है, तो server पहले cache check करता है।"  Natural spoken style, no textbook tone.
 - "sub" = the SAME sentence in Roman Hinglish for subtitles, max 14 words, e.g. "Jab user request bhejta hai, server pehle cache check karta hai."
 - "head" = short on-screen headline (max 9 words) in Roman Hinglish/English; wrap the 1-2 key words in **double stars**.
 - Everything must be technically CORRECT. Use approximate, commonly accepted numbers ("roughly", "usually") or leave numbers out. Never invent statistics or company claims.

VISUAL TYPES (field "visual"; pick the one that explains the scene best; vary them):
 {{"type":"question","text":"<the interview question, <=22 words, English or Hinglish>"}}                       (scene 1 only)
 {{"type":"flow","nodes":[{{"label":"Client","sub":"sends request"}},...2-5 nodes],"edge_labels":["","hit","miss"],"focus":<node index or -1>}}   (a request/data path; a dot travels along it)
 {{"type":"compare","left":{{"title":"Without cache","points":["..","..",".."],"metric":"200 ms"}},"right":{{"title":"With cache","points":["..",".."],"metric":"5 ms"}},"focus":-1}}
 {{"type":"stack","layers":["Browser","CDN","Server","Cache","Database"],"tags":["optional per layer"],"focus":<index or -1>}}
 {{"type":"cards","items":[{{"title":"Cache-aside","sub":"app reads cache first"}},...2-4],"focus":-1}}
 {{"type":"bars","bars":[{{"label":"Database","value":100,"text":"~10 ms"}},{{"label":"Cache","value":8,"text":"~1 ms"}}],"big":"10x","caption":"faster reads (typical)"}}
 {{"type":"code","lang":"python","lines":["def get_user(id):","    user = cache.get(id)","    ..."],"highlight":[1]}}      (highlight = 0-based line numbers; max 9 lines, max 38 chars per line, must be valid and correct)
 {{"type":"recap","pairs":[["Cache","fast reads"],["TTL","expiry"],["Invalidation","hardest part"]]}}                 (2-4 pairs)
 {{"type":"outro","ask":"<a short question for comments about this topic, <=14 words>"}}                              (last scene only)
Consecutive scenes may reuse the SAME diagram with a different "focus" (the diagram stays, the highlight moves): copy the visual exactly and change only focus.
"section" = index (0-3) of the section this scene belongs to. "sections" = 3-4 names, uppercase, max 12 chars each (e.g. {sections}).

Return ONLY this JSON:
{{"title":"<topic title, max 5 words>","tag":"<tracking tag, e.g. SYSTEM DESIGN / DSA / BACKEND / AI / AUTOMATION / CODING TIPS>",
 "sections":["PROBLEM","IDEA","HOW IT WORKS","TRADE-OFFS"],
 "scenes":[{{"say":"...","sub":"...","head":"...","section":0,"visual":{{...}}}}],
 "caption":"<Roman Hinglish Instagram caption, 4-6 short lines, first line is a hook question, no hashtags, max 480 chars>",
 "comment_q":"<one question for the comments, Roman Hinglish>",
 "hashtags":["#systemdesign","...8-10 relevant tags, lowercase, no spaces"]}}

Do NOT repeat these recent topics: {recent}
"""

SAMPLE = {
    "title": "Caching", "tag": "SYSTEM DESIGN", "sections": ["PROBLEM", "IDEA", "HOW IT WORKS", "TRADE-OFFS"],
    "scenes": [
        {"say": "Agar interviewer puche, caching kya hai aur database performance kaise improve karti hai?",
         "sub": "Agar interviewer puche: caching kya hai aur DB performance kaise improve karti hai?",
         "head": "Interview me **caching** pucha gaya?", "section": 0,
         "visual": {"type": "question", "text": "What is caching, and how does it improve database performance?"}},
        {"say": "Har request par database hit karna slow aur costly hota hai.", "sub": "Har request par database hit karna slow aur costly hota hai.",
         "head": "Problem: har request **database** tak jaati hai", "section": 0,
         "visual": {"type": "flow", "nodes": [{"label": "Client", "sub": "sends request"}, {"label": "Server", "sub": "app logic"}, {"label": "Database", "sub": "disk reads"}],
                    "edge_labels": ["", "every time"], "focus": 2}},
        {"say": "Caching ka matlab hai, bar bar chahiye hone wala data memory me rakh lena.", "sub": "Caching: bar bar chahiye data ko memory me rakh lo.",
         "head": "Idea: data ko **RAM** me rakho", "section": 1,
         "visual": {"type": "flow", "nodes": [{"label": "Client", "sub": "sends request"}, {"label": "Server", "sub": "app logic"}, {"label": "Cache", "sub": "RAM, very fast"}, {"label": "Database", "sub": "disk reads"}],
                    "edge_labels": ["", "check first", "only on miss"], "focus": 2}},
        {"say": "Cache hit me answer seedha milta hai, miss par database se laate hain.", "sub": "Hit: seedha answer. Miss: database se lao, cache me rakho.",
         "head": "Cache **hit** vs cache **miss**", "section": 2,
         "visual": {"type": "compare", "left": {"title": "Cache miss", "points": ["Check cache", "Read database", "Store in cache"], "metric": "slow"},
                    "right": {"title": "Cache hit", "points": ["Check cache", "Return result"], "metric": "fast"}, "focus": 1}},
        {"say": "Memory ki speed disk se bahut zyada hoti hai.", "sub": "Memory ki speed disk se bahut zyada hoti hai.",
         "head": "RAM vs disk: **speed** ka farak", "section": 2,
         "visual": {"type": "bars", "bars": [{"label": "Database (disk)", "value": 100, "text": "~10 ms"}, {"label": "Cache (RAM)", "value": 8, "text": "~1 ms"}],
                    "big": "10x", "caption": "faster reads (typical)"}},
        {"say": "Code me ye pattern cache-aside kehlata hai.", "sub": "Code me ye pattern cache-aside kehlata hai.",
         "head": "Pattern: **cache-aside**", "section": 2,
         "visual": {"type": "code", "lang": "python", "lines": ["def get_user(uid):", "    user = cache.get(uid)", "    if user:", "        return user", "    user = db.query(uid)", "    cache.set(uid, user, ttl=60)", "    return user"], "highlight": [1, 5]}},
        {"say": "Sabse badi galti hai stale data, isliye TTL aur invalidation zaroori hai.", "sub": "Sabse badi galti: stale data. TTL aur invalidation zaroori.",
         "head": "Trade-off: **stale** data", "section": 3,
         "visual": {"type": "stack", "layers": ["Cache hit", "TTL expires", "Invalidate on write", "Fresh data"], "tags": ["fast", "auto", "manual", "correct"], "focus": 2}},
        {"say": "Yaad rakhna, ye teen cheezein.", "sub": "Yaad rakhna: ye teen cheezein.", "head": "Quick **recap**", "section": 3,
         "visual": {"type": "recap", "pairs": [["Cache", "fast reads"], ["TTL", "auto expiry"], ["Invalidation", "keeps it fresh"]]}},
        {"say": "Save karo, comment karo, aur follow karo.", "sub": "Save karo, comment karo, aur follow karo.", "head": "Roz **tech** simple bhasha me", "section": 3,
         "visual": {"type": "outro", "ask": "Which cache do you use: Redis or Memcached?"}},
    ],
    "caption": "Interview me caching pucha jaye to kya bologe?\nCache = data ko RAM me rakhna, taaki database par load kam ho.\nHit = fast answer, miss = database se lao.\nStale data se bachne ke liye TTL aur invalidation use karo.",
    "comment_q": "Aap Redis use karte ho ya Memcached?", "hashtags": ["#systemdesign", "#caching", "#backend", "#coding", "#softwareengineer", "#interviewprep", "#techreels"],
}

# ------------------------------------------------------------------ look
BG = (255, 255, 255)
INK = (17, 17, 22)
BODY = (75, 85, 99)
MUTE = (140, 148, 160)
LINE = (226, 229, 235)
SOFT = (244, 245, 248)
GOOD = (22, 163, 74)
BAD = (220, 38, 38)
ACCENTS = [(79, 70, 229), (37, 99, 235), (13, 148, 136), (234, 88, 12), (219, 39, 119), (124, 58, 237)]
ACC = ACCENTS[0]


def mix(c, a, base=BG):
    return tuple(int(base[i] + (c[i] - base[i]) * a) for i in range(3))


def tint(c, k):
    return mix(c, k, (255, 255, 255))


def ease(x):
    x = max(0.0, min(1.0, x))
    return 1 - (1 - x) ** 3


_FD = {
    "bold": ["fonts/Inter-Bold.otf", "/usr/share/fonts/opentype/inter/Inter-Bold.otf", "/usr/share/fonts/truetype/inter/Inter-Bold.ttf",
             "/usr/share/fonts/truetype/google-fonts/Poppins-Bold.ttf", "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf"],
    "semi": ["fonts/Inter-SemiBold.otf", "/usr/share/fonts/opentype/inter/Inter-SemiBold.otf", "/usr/share/fonts/truetype/inter/Inter-SemiBold.ttf",
             "/usr/share/fonts/truetype/google-fonts/Poppins-SemiBold.ttf", "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf"],
    "med": ["fonts/Inter-Medium.otf", "/usr/share/fonts/opentype/inter/Inter-Medium.otf", "/usr/share/fonts/truetype/inter/Inter-Medium.ttf",
            "/usr/share/fonts/truetype/google-fonts/Poppins-Medium.ttf", "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf"],
    "black": ["fonts/Inter-ExtraBold.otf", "/usr/share/fonts/opentype/inter/Inter-ExtraBold.otf", "/usr/share/fonts/opentype/inter/Inter-Black.otf",
              "/usr/share/fonts/truetype/google-fonts/Poppins-ExtraBold.ttf", "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf"],
    "mono": ["/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf", "/usr/share/fonts/truetype/liberation/LiberationMono-Regular.ttf"],
    "monob": ["/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf", "/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf"],
}


@lru_cache(maxsize=None)
def font(kind, size):
    for p in _FD[kind]:
        pp = Path(p) if p.startswith("/") else BASE / p
        if pp.exists():
            try:
                return ImageFont.truetype(str(pp), size)
            except Exception:
                continue
    return ImageFont.load_default()


# ------------------------------------------------------------------ drawing helpers
def tw(d, s, f):
    return d.textlength(s, font=f)


def text_c(d, cx, y, s, f, fill):
    d.text((cx - tw(d, s, f) / 2, y), s, font=f, fill=fill)


def wrap(d, s, f, max_w):
    out, cur = [], ""
    for w in s.split():
        t = (cur + " " + w).strip()
        if tw(d, t, f) <= max_w or not cur:
            cur = t
        else:
            out.append(cur)
            cur = w
    if cur:
        out.append(cur)
    return out


def fit_lines(d, s, kind, size, max_w, max_lines, min_size=14):
    while size > min_size:
        f = font(kind, size)
        ls = wrap(d, s, f, max_w)
        if len(ls) <= max_lines:
            return f, ls
        size -= 2
    f = font(kind, min_size)
    return f, wrap(d, s, f, max_w)[:max_lines]


def rich_words(s):
    out = []
    for p in re.split(r"(\*\*.+?\*\*)", s or ""):
        if not p:
            continue
        hl = p.startswith("**") and p.endswith("**") and len(p) > 4
        for w in (p[2:-2] if hl else p).split():
            out.append((w, hl))
    return out


def draw_rich(d, cx, y, text, size, max_w, a, u, line_gap=1.28):
    """Headline: normal words grey, **key** words black bold with an accent underline growing in (u 0..1)."""
    fn, fh = font("med", size), font("black", size)
    words = rich_words(text)
    lines, cur, x = [[]], [], 0
    sp = tw(d, " ", fn)
    lines, x = [[]], 0
    for w, hl in words:
        f = fh if hl else fn
        wl = tw(d, w, f)
        if lines[-1] and x + sp + wl > max_w:
            lines.append([])
            x = 0
        if lines[-1]:
            x += sp
        lines[-1].append((w, hl, x, wl))
        x += wl
    for li, ln in enumerate(lines):
        if not ln:
            continue
        wid = ln[-1][2] + ln[-1][3]
        x0 = cx - wid / 2
        yy = y + li * size * line_gap
        for w, hl, xo, wl in ln:
            d.text((x0 + xo, yy), w, font=fh if hl else fn, fill=mix(INK if hl else BODY, a))
            if hl and u > 0:
                d.rectangle((x0 + xo, yy + size * 1.12, x0 + xo + wl * u, yy + size * 1.12 + 4), fill=mix(ACC, a))
    return len(lines)


def rrect(d, xy, r, fill=None, outline=None, width=1, shadow=None):
    if shadow:
        x0, y0, x1, y1 = xy
        d.rounded_rectangle((x0 + 5, y0 + 6, x1 + 5, y1 + 6), r, fill=shadow)
    d.rounded_rectangle(xy, r, fill=fill, outline=outline, width=width)


def arrow_down(d, x, y0, y1, col, w=3):
    d.line((x, y0, x, y1 - 7), fill=col, width=w)
    d.polygon([(x - 7, y1 - 10), (x + 7, y1 - 10), (x, y1)], fill=col)


def reveal(t, i, persist, step=0.3, dur=0.5, start=0.25):
    if persist:
        return 1.0, 0
    e = ease((t - start - i * step) / dur)
    return e, int((1 - e) * 18)


# ------------------------------------------------------------------ visuals (area x 40..680, y 400..960)
VY0, VY1 = 410, 960


def node_box(d, x0, y0, x1, y1, label, sub, a, state="n"):
    if state == "f":
        rrect(d, (x0, y0, x1, y1), 16, fill=mix(tint(ACC, 0.10), a), outline=mix(ACC, a), width=3, shadow=mix(tint(ACC, 0.25), a))
        lc, sc = INK, BODY
    elif state == "d":
        rrect(d, (x0, y0, x1, y1), 16, fill=mix((255, 255, 255), a), outline=mix(LINE, a), width=2)
        lc, sc = MUTE, MUTE
    else:
        rrect(d, (x0, y0, x1, y1), 16, fill=mix((255, 255, 255), a), outline=mix(INK, a), width=2, shadow=mix(LINE, a))
        lc, sc = INK, BODY
    cx, cy = (x0 + x1) / 2, (y0 + y1) / 2
    f = font("bold", 28 if sub else 30)
    if sub:
        text_c(d, cx, cy - 31, label[:22], f, mix(lc, a))
        text_c(d, cx, cy + 6, sub[:30], font("med", 19), mix(sc, a))
    else:
        text_c(d, cx, cy - 17, label[:24], f, mix(lc, a))


def v_question(d, v, t, persist, S):
    text = v.get("text", "")
    f, ls = fit_lines(d, text, "black", 44, 540, 5, 28)
    lh = int(f.size * 1.3)
    boxh = len(ls) * lh + 96
    y0 = VY0 + max(0, (VY1 - VY0 - boxh - 40) // 2)
    a, dy = reveal(t, 0, persist, start=0.0, dur=0.35)
    rrect(d, (50, y0 + dy, 670, y0 + boxh + dy), 24, fill=mix(SOFT, a), outline=mix(LINE, a), width=2)
    d.rounded_rectangle((50, y0 + dy, 670, y0 + 64 + dy), 24, fill=mix(ACC, a))
    d.rectangle((50, y0 + 40 + dy, 670, y0 + 64 + dy), fill=mix(ACC, a))
    d.text((80, y0 + 18 + dy), "INTERVIEWER ASKS", font=font("monob", 20), fill=mix((255, 255, 255), a))
    words = text.split()
    shown = 1 + int(max(0.0, t - 0.2) * 7) if not persist else len(words)   # words type in
    shown_text = " ".join(words[:shown])
    y = y0 + 90 + dy
    for ln in wrap(d, shown_text, f, 540) if shown_text else []:
        d.text((80, y), ln, font=f, fill=mix(INK, a))
        y += lh
    # little speech-bubble tail
    d.polygon([(110, y0 + boxh + dy - 2), (150, y0 + boxh + dy - 2), (118, y0 + boxh + 34 + dy)], fill=mix(SOFT, a), outline=None)


def v_flow(d, v, t, persist, S):
    nodes = v.get("nodes", [])[:5]
    n = len(nodes)
    if n < 2:
        return
    labs = v.get("edge_labels", []) or []
    focus = int(v.get("focus", -1) if v.get("focus", -1) is not None else -1)
    gap = 46 if n <= 4 else 36
    nh = int(min(92, (VY1 - VY0 - (n - 1) * gap) / n))
    cx = 330 if any(labs) else 360
    bw = 340
    tot = n * nh + (n - 1) * gap
    top = VY0 + (VY1 - VY0 - tot) // 2
    boxes = []
    for i, nd in enumerate(nodes):
        a, dy = reveal(t, i, persist, step=0.35)
        y0 = top + i * (nh + gap)
        boxes.append((y0, y0 + nh))
        st = "n" if focus < 0 else ("f" if i == focus else "d")
        node_box(d, cx - bw // 2, y0 + dy, cx + bw // 2, y0 + nh + dy, str(nd.get("label", "")), str(nd.get("sub", "")), a, st)
        if i < n - 1:
            a2, _ = reveal(t, i + 0.5, persist, step=0.35)
            arrow_down(d, cx, y0 + nh + 4, y0 + nh + gap - 2, mix(INK if focus < 0 else MUTE, a2))
            lab = labs[i] if i < len(labs) else ""
            if lab:
                d.text((cx + 22, y0 + nh + gap / 2 - 11), str(lab)[:18], font=font("monob", 19), fill=mix(ACC, a2))
    # travelling packet
    if n >= 2 and (persist or t > 0.3 + n * 0.35):
        pos = ((t - 0.3) * 0.45) % 1.0 * (n - 1)
        i = min(int(pos), n - 2)
        f_ = pos - i
        py = boxes[i][1] + (boxes[i + 1][0] - boxes[i][1]) * f_
        r = 8
        d.ellipse((cx - r, py - r, cx + r, py + r), fill=ACC)


def v_compare(d, v, t, persist, S):
    focus = int(v.get("focus", -1) if v.get("focus", -1) is not None else -1)
    ch, gap = 252, 26
    top = VY0 + (VY1 - VY0 - 2 * ch - gap) // 2
    for i, key in enumerate(("left", "right")):
        side = v.get(key, {}) or {}
        a, dy = reveal(t, i, persist, step=0.5)
        y0 = top + i * (ch + gap) + dy
        col = BAD if i == 0 else GOOD
        dim = focus >= 0 and focus != i
        k = 0.45 if dim else 1.0
        rrect(d, (50, y0, 670, y0 + ch), 20, fill=mix((255, 255, 255), a), outline=mix(INK if not dim else LINE, a), width=2, shadow=mix(LINE, a) if not dim else None)
        d.rounded_rectangle((74, y0 + 22, 74 + tw(d, str(side.get("title", ""))[:24].upper(), font("monob", 20)) + 34, y0 + 58), 18, fill=mix(tint(col, 0.14), a))
        d.text((91, y0 + 29), str(side.get("title", ""))[:24].upper(), font=font("monob", 20), fill=mix(col, a * k + (1 - k) * 0.4))
        for j, p in enumerate(side.get("points", [])[:3]):
            d.ellipse((78, y0 + 92 + j * 46 + 8, 90, y0 + 92 + j * 46 + 20), fill=mix(col, a * k + (1 - k) * 0.4))
            d.text((104, y0 + 88 + j * 46), str(p)[:30], font=font("semi", 26), fill=mix(INK if not dim else MUTE, a))
        m = str(side.get("metric", ""))
        if m:
            f = font("black", 44 if len(m) < 9 else 32)
            d.text((650 - tw(d, m, f), y0 + 24), m, font=f, fill=mix(col if not dim else MUTE, a))


def v_stack(d, v, t, persist, S):
    layers = v.get("layers", [])[:6]
    tags = v.get("tags", []) or []
    focus = int(v.get("focus", -1) if v.get("focus", -1) is not None else -1)
    n = len(layers)
    if not n:
        return
    gap = 16
    lh = int(min(92, (VY1 - VY0 - (n - 1) * gap) / n))
    tot = n * lh + (n - 1) * gap
    top = VY0 + (VY1 - VY0 - tot) // 2
    for i, name in enumerate(layers):
        a, dy = reveal(t, i, persist, step=0.28)
        y0 = top + i * (lh + gap) + dy
        isf = i == focus
        dim = focus >= 0 and not isf
        if isf:
            rrect(d, (60, y0, 660, y0 + lh), 16, fill=mix(ACC, a), shadow=mix(tint(ACC, 0.3), a))
            lc, tc = (255, 255, 255), (225, 222, 255)
        else:
            rrect(d, (60, y0, 660, y0 + lh), 16, fill=mix((255, 255, 255), a), outline=mix(LINE if dim else INK, a), width=2, shadow=None if dim else mix(LINE, a))
            lc, tc = (MUTE if dim else INK), MUTE
        d.text((90, y0 + lh / 2 - 17), str(name)[:26], font=font("bold", 30), fill=mix(lc, a))
        if i < len(tags) and tags[i]:
            s = str(tags[i])[:16]
            d.text((640 - tw(d, s, font("monob", 20)), y0 + lh / 2 - 12), s, font=font("monob", 20), fill=mix(tc, a))


def v_cards(d, v, t, persist, S):
    items = v.get("items", [])[:4]
    focus = int(v.get("focus", -1) if v.get("focus", -1) is not None else -1)
    n = len(items)
    if not n:
        return
    gap = 22
    ch = int(min(126, (VY1 - VY0 - (n - 1) * gap) / n))
    tot = n * ch + (n - 1) * gap
    top = VY0 + (VY1 - VY0 - tot) // 2
    for i, it in enumerate(items):
        a, dy = reveal(t, i, persist, step=0.35)
        y0 = top + i * (ch + gap) + dy
        st = "n" if focus < 0 else ("f" if i == focus else "d")
        node_box(d, 50, y0, 670, y0 + ch, "", "", a, st)
        dim = st == "d"
        d.text((82, y0 + ch / 2 - 40 + 6), f"0{i + 1}", font=font("monob", 22), fill=mix(ACC if not dim else MUTE, a))
        d.text((82, y0 + ch / 2 - 12), str(it.get("title", ""))[:28], font=font("bold", 30), fill=mix(MUTE if dim else INK, a))
        sub = str(it.get("sub", ""))
        if sub:
            d.text((82, y0 + ch / 2 + 26), sub[:42], font=font("med", 21), fill=mix(MUTE if dim else BODY, a))


def v_bars(d, v, t, persist, S):
    bars = v.get("bars", [])[:4]
    n = len(bars)
    if not n:
        return
    top = VY0 + 20
    big, cap = str(v.get("big", "")), str(v.get("caption", ""))
    if big:
        a, _ = reveal(t, 0, persist, start=0.1, dur=0.5)
        f = font("black", 120)
        text_c(d, 360, top, big, f, mix(ACC, a))
        text_c(d, 360, top + 142, cap[:36], font("semi", 26), mix(BODY, a))
        top += 215
    mx = max([float(b.get("value", 1) or 1) for b in bars] + [1])
    for i, b in enumerate(bars):
        a, _ = reveal(t, i + 1, persist, step=0.45, dur=0.3)
        g = 1.0 if persist else ease((t - 0.7 - i * 0.45) / 0.7)
        y0 = top + i * 100
        d.text((60, y0), str(b.get("label", ""))[:26], font=font("semi", 24), fill=mix(INK, a))
        full = 600
        d.rounded_rectangle((60, y0 + 38, 60 + full, y0 + 66), 14, fill=mix(SOFT, a))
        wv = max(24, int(full * float(b.get("value", 1)) / mx * g))
        col = ACC if i == n - 1 else INK
        d.rounded_rectangle((60, y0 + 38, 60 + wv, y0 + 66), 14, fill=mix(col, a))
        s = str(b.get("text", ""))
        d.text((660 - tw(d, s, font("monob", 22)), y0 + 4), s, font=font("monob", 22), fill=mix(BODY, a))


def v_code(d, v, t, persist, S):
    lines = [str(x)[:40] for x in v.get("lines", [])[:9]]
    hi = set(int(i) for i in (v.get("highlight") or []) if str(i).lstrip("-").isdigit())
    n = len(lines)
    if not n:
        return
    lh = 44
    chh = 64 + n * lh + 20
    top = VY0 + max(0, (VY1 - VY0 - chh) // 2)
    a, dy = reveal(t, 0, persist, start=0.0, dur=0.4)
    rrect(d, (40, top + dy, 680, top + chh + dy), 18, fill=mix((250, 250, 252), a), outline=mix(LINE, a), width=2, shadow=mix(SOFT, a))
    for k, c in enumerate(((255, 95, 86), (255, 189, 46), (39, 201, 63))):
        d.ellipse((64 + k * 26, top + 22 + dy, 78 + k * 26, top + 36 + dy), fill=mix(c, a))
    lang = str(v.get("lang", "")).upper()[:10]
    d.text((660 - tw(d, lang, font("monob", 18)), top + 20 + dy), lang, font=font("monob", 18), fill=mix(MUTE, a))
    fm = font("mono", 23)
    for i, ln in enumerate(lines):
        la, ldy = reveal(t, i, persist, step=0.22, start=0.3, dur=0.35)
        y = top + 60 + i * lh + ldy
        if i in hi:
            d.rounded_rectangle((52, y - 4, 668, y + lh - 8), 8, fill=mix(tint(ACC, 0.12), la))
            d.rectangle((52, y - 4, 56, y + lh - 8), fill=mix(ACC, la))
        d.text((70, y + 2), f"{i + 1}", font=font("mono", 17), fill=mix(MUTE, la))
        col = MUTE if ln.strip().startswith(("#", "//")) else INK
        d.text((106, y), ln, font=fm, fill=mix(col, la))


def v_recap(d, v, t, persist, S):
    pairs = [p for p in v.get("pairs", [])[:4] if isinstance(p, (list, tuple)) and len(p) >= 2]
    n = len(pairs)
    if not n:
        return
    rh = 128
    top = VY0 + (VY1 - VY0 - n * rh) // 2
    for i, (k, m) in enumerate(pairs):
        a, dy = reveal(t, i, persist, step=0.5)
        y0 = top + i * rh + dy
        d.text((56, y0 + 6), f"CONCEPT 0{i + 1}", font=font("monob", 17), fill=mix(MUTE, a))
        f, ls = fit_lines(d, str(k), "black", 54, 330, 1, 28)
        d.text((56, y0 + 34), ls[0] if ls else "", font=f, fill=mix(INK, a))
        f2, l2 = fit_lines(d, str(m), "semi", 30, 250, 2, 20)
        for j, s in enumerate(l2):
            d.text((664 - tw(d, s, f2), y0 + 44 + j * 36), s, font=f2, fill=mix(ACC, a))
        d.line((56, y0 + rh - 14, 664, y0 + rh - 14), fill=mix(LINE, a), width=2)


def v_outro(d, v, t, persist, S):
    a, dy = reveal(t, 0, persist, start=0.0, dur=0.4)
    y = VY0 + 10 + dy
    d.ellipse((290, y, 430, y + 140), fill=mix(ACC, a))
    text_c(d, 360, y + 38, BRAND_MARK, font("black", 60), mix((255, 255, 255), a))
    text_c(d, 360, y + 170, BRAND, font("black", 34), mix(INK, a))
    text_c(d, 360, y + 218, f"@{HANDLE}", font("semi", 26), mix(ACC, a))
    ask = str(v.get("ask", ""))
    if ask:
        a2, dy2 = reveal(t, 1, persist, step=0.5)
        f, ls = fit_lines(d, ask, "bold", 30, 540, 3, 22)
        bh = len(ls) * int(f.size * 1.35) + 64
        y2 = y + 290 + dy2
        rrect(d, (60, y2, 660, y2 + bh), 20, fill=mix(SOFT, a2), outline=mix(LINE, a2), width=2)
        d.text((84, y2 + 14), "COMMENT BELOW", font=font("monob", 17), fill=mix(ACC, a2))
        for j, s in enumerate(ls):
            d.text((84, y2 + 42 + j * int(f.size * 1.35)), s, font=f, fill=mix(INK, a2))
    a3, _ = reveal(t, 2, persist, step=0.5)
    x = 360 - 270
    for k, s in enumerate(("SAVE", "SHARE", "FOLLOW")):
        bx0 = 90 + k * 190
        d.rounded_rectangle((bx0, VY1 - 62, bx0 + 170, VY1 - 12), 25, fill=mix(ACC if k == 2 else INK, a3))
        text_c(d, bx0 + 85, VY1 - 52, s, font("bold", 24), mix((255, 255, 255), a3))


VIS = {"question": v_question, "flow": v_flow, "compare": v_compare, "stack": v_stack, "cards": v_cards,
       "bars": v_bars, "code": v_code, "recap": v_recap, "outro": v_outro}


# ------------------------------------------------------------------ frame
_BGIMG = {}


def background():
    if "bg" not in _BGIMG:
        im = Image.new("RGB", (W, H), BG)
        d = ImageDraw.Draw(im)
        for x in range(0, W, 48):
            d.line((x, 0, x, H), fill=(246, 247, 250))
        for y in range(0, H, 48):
            d.line((0, y, W, y), fill=(246, 247, 250))
        # soft accent glow top-right
        glow = Image.new("RGB", (W, H), BG)
        gd = ImageDraw.Draw(glow)
        for r in range(260, 0, -10):
            gd.ellipse((W - 120 - r, -60 - r, W - 120 + r, -60 + r), fill=mix(ACC, 0.05 * (1 - r / 260) + 0.004))
        _BGIMG["bg"] = Image.blend(im, glow, 0.6)
    return _BGIMG["bg"].copy()


def vkey(v):
    x = dict(v)
    x.pop("focus", None)
    return json.dumps(x, sort_keys=True, ensure_ascii=False)


def draw_frame(spec, idx, t, gt, total, persist):
    sc = spec["scenes"][idx]
    im = background()
    d = ImageDraw.Draw(im)
    n = len(spec["scenes"])
    # header + progress
    d.text((36, 34), f"{BRAND} / EXPLAINED", font=font("monob", 15), fill=MUTE)
    tag = str(spec.get("tag", "TECH")).upper()[:22]
    d.text((W - 36 - tw(d, tag, font("monob", 15)), 34), tag, font=font("monob", 15), fill=MUTE)
    d.line((36, 66, W - 36, 66), fill=LINE, width=2)
    d.line((36, 66, 36 + (W - 72) * min(1.0, gt / max(total, 0.1)), 66), fill=ACC, width=3)
    # headline
    a = ease(t / 0.35)
    u = ease((t - 0.35) / 0.5)
    draw_rich(d, W // 2, 118, sc.get("head", ""), 40, 600, a, u)
    # roadmap
    secs = spec.get("sections", [])[:4]
    if len(secs) >= 2:
        cur = int(sc.get("section", 0) or 0)
        y = 322
        d.text((36, y - 34), "THE ROADMAP", font=font("monob", 14), fill=MUTE)
        xs = [60 + i * (W - 120) / (len(secs) - 1) for i in range(len(secs))]
        d.line((xs[0], y, xs[-1], y), fill=LINE, width=2)
        d.line((xs[0], y, xs[min(cur, len(secs) - 1)], y), fill=ACC, width=3)
        for i, s in enumerate(secs):
            on = i <= cur
            r = 7 if i == cur else 5
            d.ellipse((xs[i] - r, y - r, xs[i] + r, y + r), fill=ACC if on else (255, 255, 255), outline=ACC if on else LINE, width=2)
            f = font("monob", 14)
            lab = str(s).upper()[:13]
            x = min(max(xs[i] - tw(d, lab, f) / 2, 30), W - 30 - tw(d, lab, f))
            d.text((x, y + 14), lab, font=f, fill=INK if i == cur else MUTE)
    # visual
    v = sc.get("visual") or {}
    fn = VIS.get(v.get("type"))
    if fn:
        fn(d, v, t, persist, sc)
    # subtitles (bold, black on white, lower third)
    sub = str(sc.get("sub", "")).strip()
    if sub:
        sa = ease(t / 0.2)
        f, ls = fit_lines(d, sub, "bold", 34, 600, 3, 24)
        lh = int(f.size * 1.3)
        y = 1060 + (3 - len(ls)) * lh // 2
        for j, s in enumerate(ls):
            text_c(d, W // 2, y + j * lh, s, f, mix(INK, sa))
    # footer
    d.line((36, 1218, W - 36, 1218), fill=LINE, width=1)
    d.text((36, 1234), str(spec.get("title", "")).upper()[:30], font=font("monob", 15), fill=MUTE)
    pg = f"{idx + 1:02d} / {n:02d}"
    d.text((W - 36 - tw(d, pg, font("monob", 15)), 1234), pg, font=font("monob", 15), fill=MUTE)
    return im


# ------------------------------------------------------------------ script (AI)
def clean_json(raw):
    raw = re.sub(r"^```(?:json)?|```$", "", raw.strip(), flags=re.M).strip()
    a, b = raw.find("{"), raw.rfind("}")
    return json.loads(raw[a:b + 1])


def validate(spec):
    if not isinstance(spec, dict) or not isinstance(spec.get("scenes"), list):
        raise ValueError("no scenes")
    out = []
    for sc in spec["scenes"]:
        if not isinstance(sc, dict) or not sc.get("say") or not isinstance(sc.get("visual"), dict):
            continue
        if sc["visual"].get("type") not in VIS:
            continue
        sc["say"] = re.sub(r"\*\*", "", str(sc["say"])).strip()
        sc["sub"] = re.sub(r"\*\*", "", str(sc.get("sub") or sc["say"])).strip()
        sc["head"] = str(sc.get("head") or "")
        try:
            sc["section"] = max(0, min(3, int(sc.get("section", 0))))
        except Exception:
            sc["section"] = 0
        out.append(sc)
    if len(out) < 8:
        raise ValueError(f"only {len(out)} usable scenes")
    words = sum(len(x["say"].split()) for x in out)
    if words > 215:
        raise ValueError(f"voice-over too long ({words} words); the total must be under 190 words, so use fewer and shorter scenes")
    if out[0]["visual"]["type"] != "question":
        raise ValueError("first scene is not the interview question")
    # outro only at the end
    mids = [s for s in out[1:-1] if s["visual"]["type"] != "outro"]
    last = out[-1]
    if last["visual"]["type"] != "outro":
        raise ValueError("last scene is not the outro")
    spec["scenes"] = [out[0]] + mids + [last]
    spec.setdefault("sections", ["PROBLEM", "IDEA", "HOW IT WORKS", "TRADE-OFFS"])
    spec["title"] = str(spec.get("title") or "Explained")[:40]
    spec["hashtags"] = [h if h.startswith("#") else "#" + h for h in spec.get("hashtags", []) if isinstance(h, str)][:10]
    return spec


def gen_spec(topic, recent):
    import core as M
    prompt = PROMPT.format(topic=topic, sections=SECTION_HINT, recent="; ".join(r[:60] for r in recent[:12]) or "none")
    fb = ""
    last = None
    for attempt in range(3):
        try:
            raw = M.call_gemini(prompt + fb)
            return validate(clean_json(raw))
        except Exception as ex:
            last = ex
            fb = f"\n\nYour previous reply was rejected ({ex}). Return ONLY valid JSON following the schema exactly; scene 1 must be type question, last scene type outro."
            print(f"bad script ({ex}); retry {attempt + 1}/3", file=sys.stderr)
    raise RuntimeError(f"AI did not return a usable explainer script: {last}")


# ------------------------------------------------------------------ voice
def _run(cmd, **kw):
    return subprocess.run(cmd, check=True, capture_output=True, **kw)


def audio_len(p):
    r = subprocess.run(["ffprobe", "-v", "error", "-show_entries", "format=duration", "-of", "csv=p=0", str(p)], capture_output=True, text=True)
    try:
        return float(r.stdout.strip())
    except Exception:
        return 0.0


def speak(text_dev, text_rom, out):
    """edge-tts fallback: Hindi voice reading Devanagari+English terms; falls back to Roman text / other voices."""
    import edge_tts
    voices = [VOICE] + [v for v in VOICE_BACKUP if v != VOICE]
    texts = [text_dev, text_rom] if VOICE_TEXT == "devanagari" else [text_rom, text_dev]
    for voice in voices:
        for txt in texts:
            if not txt:
                continue
            try:
                asyncio.run(asyncio.wait_for(edge_tts.Communicate(txt, voice, rate=VOICE_RATE).save(str(out)), 60))
                if Path(out).exists() and Path(out).stat().st_size > 1500:
                    return voice
            except Exception as ex:
                print(f"voice {voice} failed: {str(ex)[:120]}", file=sys.stderr)
    return None


# ---- Gemini TTS (free tier): far more natural than edge-tts, understands Hindi / Hinglish
TTS = (os.getenv("EXPLAIN_TTS") or "gemini").lower()                 # gemini | edge
GTTS_VOICE = os.getenv("GEMINI_TTS_VOICE") or "Charon"               # Charon, Kore, Orus, Puck, Fenrir, Aoede, Leda, Zephyr ...
GTTS_MODELS = [m for m in dict.fromkeys([os.getenv("GEMINI_TTS_MODEL") or "", "gemini-3.8-flash-tts", "gemini-3.8-flash-lite-tts"]) if m]
GTTS_LEGACY = ["gemini-2.5-flash-preview-tts"]
GTTS_STYLE = os.getenv("GEMINI_TTS_STYLE") or "friendly, natural and conversational, like a young Indian software engineer explaining to a friend; relaxed, medium pace"
API = "https://generativelanguage.googleapis.com/v1beta"


def _find_audio(o):
    if isinstance(o, dict):
        if o.get("type") == "audio" and isinstance(o.get("data"), str):
            return o["data"]
        if isinstance(o.get("inlineData"), dict) and isinstance(o["inlineData"].get("data"), str):
            return o["inlineData"]["data"]
        if isinstance(o.get("inline_data"), dict) and isinstance(o["inline_data"].get("data"), str):
            return o["inline_data"]["data"]
        for v in o.values():
            r = _find_audio(v)
            if r:
                return r
    elif isinstance(o, list):
        for v in reversed(o):
            r = _find_audio(v)
            if r:
                return r
    return None


def _save_audio(b64, out):
    import base64
    import wave
    raw = base64.b64decode(b64)
    if raw[:4] == b"RIFF":
        Path(out).write_bytes(raw)
    else:                                   # raw 24 kHz 16-bit mono PCM
        with wave.open(str(out), "wb") as w:
            w.setnchannels(1)
            w.setsampwidth(2)
            w.setframerate(24000)
            w.writeframes(raw)


def gemini_tts(text, out):
    import requests
    hdr = {"x-goog-api-key": os.environ["GEMINI_API_KEY"], "Content-Type": "application/json"}
    errs = []
    for model in GTTS_MODELS + GTTS_LEGACY:
        legacy = "2.5" in model
        for attempt in range(4):
            try:
                if legacy:
                    r = requests.post(f"{API}/models/{model}:generateContent", headers=hdr, timeout=120, json={
                        "contents": [{"parts": [{"text": f"Say in a {GTTS_STYLE} tone: {text}"}]}],
                        "generationConfig": {"responseModalities": ["AUDIO"], "speechConfig": {"voiceConfig": {"prebuiltVoiceConfig": {"voiceName": GTTS_VOICE}}}}})
                else:
                    r = requests.post(f"{API}/interactions", headers=hdr, timeout=120, json={
                        "model": model,
                        "input": [{"type": "user_input", "content": [{"type": "text", "text": text,
                                   "annotations": [{"type": "speech_metadata", "style": GTTS_STYLE}]}]}],
                        "response_format": {"type": "audio"},
                        "generation_config": {"speech_config": [{"voice": GTTS_VOICE}]}})
            except requests.RequestException as ex:
                errs.append(f"{model}: network {str(ex)[:80]}")
                time.sleep(5)
                continue
            if r.ok:
                data = None
                try:
                    data = _find_audio(r.json())
                except Exception:
                    pass
                if data:
                    _save_audio(data, out)
                    return f"gemini-tts:{model}:{GTTS_VOICE}"
                errs.append(f"{model}: reply had no audio")
                break
            errs.append(f"{model}: {r.status_code} {r.text[:110]}")
            print(f"Gemini TTS {model}: {r.status_code}", file=sys.stderr)
            if r.status_code == 429:
                time.sleep(min(60, 15 * (attempt + 1)))
                continue
            if r.status_code in (500, 502, 503, 504):
                time.sleep(8)
                continue
            break                                           # 400/403/404: try the next model
    raise RuntimeError("Gemini TTS failed: " + " | ".join(errs[-4:]))


def _clips(spec, tmp, provider):
    """One raw clip per scene with the given provider; returns (paths, voice name)."""
    paths, name = [], None
    for i, sc in enumerate(spec["scenes"]):
        if provider == "gemini":
            p = tmp / f"{provider}_{i:02d}.wav"
            name = gemini_tts(sc["say"], p)
            time.sleep(1.5)
        else:
            p = tmp / f"{provider}_{i:02d}.mp3"
            v = speak(sc["say"], sc["sub"], p)
            if not v:
                raise RuntimeError("edge-tts: no voice could be generated")
            name = v
        paths.append(p)
    return paths, name


_TRIM = ("silenceremove=start_periods=1:start_threshold=-50dB:start_silence=0.04,areverse,"
         "silenceremove=start_periods=1:start_threshold=-50dB:start_silence=0.04,areverse")


def build_audio(spec, tmp, fake=False):
    """One clip per scene -> scene durations; returns (durations, wav path, voice name)."""
    n = len(spec["scenes"])
    used = None
    if fake:
        paths = []
        for i, sc in enumerate(spec["scenes"]):
            p = tmp / f"fake_{i:02d}.wav"
            secs = max(2.4, len(sc["say"].split()) / 2.7)
            _run(["ffmpeg", "-v", "error", "-y", "-f", "lavfi", "-i", "anullsrc=r=44100:cl=mono", "-t", f"{secs:.2f}", str(p)])
            paths.append(p)
    else:
        paths, errs = None, []
        for provider in (["gemini", "edge"] if TTS == "gemini" else ["edge"]):
            try:
                paths, used = _clips(spec, tmp, provider)
                break
            except Exception as ex:
                errs.append(str(ex)[:300])
                print(f"voice provider {provider} failed: {ex}", file=sys.stderr)
        if not paths:
            raise RuntimeError("no voice could be generated: " + " || ".join(errs))
    speed = float(os.getenv("EXPLAIN_SPEED") or 1.0)
    durs, wavs = [], []
    for i, p in enumerate(paths):
        t1, wav = tmp / f"t{i:02d}.wav", tmp / f"s{i:02d}.wav"
        flt = _TRIM + (f",atempo={speed}" if abs(speed - 1.0) > 0.01 else "")
        _run(["ffmpeg", "-v", "error", "-y", "-i", str(p), "-af", flt, "-ar", "44100", "-ac", "1", str(t1)])
        dur = max(2.4, audio_len(t1) + 0.15 + 0.40)
        _run(["ffmpeg", "-v", "error", "-y", "-i", str(t1), "-af", "adelay=150|150,apad", "-t", f"{dur:.3f}", "-ar", "44100", "-ac", "1", str(wav)])
        durs.append(dur)
        wavs.append(wav)
    lst = tmp / "list.txt"
    lst.write_text("".join(f"file '{w}'\n" for w in wavs))
    full = tmp / "voice.wav"
    _run(["ffmpeg", "-v", "error", "-y", "-f", "concat", "-safe", "0", "-i", str(lst), "-af", "loudnorm=I=-16:TP=-1.5:LRA=11", "-ar", "44100", str(full)])
    return durs, full, used


# ------------------------------------------------------------------ render
def render(spec, durs, audio, out_path, preview_dir=None):
    total = sum(durs)
    cmd = ["ffmpeg", "-v", "error", "-y", "-f", "rawvideo", "-pix_fmt", "rgb24", "-s", f"{W}x{H}", "-r", str(FPS), "-i", "-",
           "-i", str(audio), "-c:v", "libx264", "-preset", "veryfast", "-crf", "20", "-pix_fmt", "yuv420p", "-profile:v", "high",
           "-c:a", "aac", "-b:a", "160k", "-shortest", "-movflags", "+faststart", str(out_path)]
    p = subprocess.Popen(cmd, stdin=subprocess.PIPE)
    gt = 0.0
    prev_key = None
    nframes = 0
    for i, sc in enumerate(spec["scenes"]):
        key = vkey(sc["visual"])
        persist = (key == prev_key)
        prev_key = key
        nf = int(round(durs[i] * FPS))
        for f in range(nf):
            t = f / FPS
            im = draw_frame(spec, i, t + (0.45 if i == 0 else 0.0), gt + t, total, persist)   # scene 1 never starts blank
            p.stdin.write(im.tobytes())
            nframes += 1
            if preview_dir and f in (int(1.6 * FPS), nf - 2):
                im.save(Path(preview_dir) / f"scene{i:02d}_{'a' if f != nf - 2 else 'b'}.png")
        gt += durs[i]
    p.stdin.close()
    if p.wait() != 0:
        raise RuntimeError("ffmpeg failed while rendering the explainer")
    return total


# ------------------------------------------------------------------ caption + run
def build_caption(spec):
    cap = str(spec.get("caption", "")).strip()
    cq = str(spec.get("comment_q") or "").strip()
    parts = [cap]
    if cq:
        parts.append(f"{cq} Comment down below 👇")
    parts.append("Save karo interview se pehle 🔖 Share karo apne dev dost ko 🤝")
    parts.append(f"Follow @{HANDLE} for daily coding, system design, AI & automation.")
    tags = spec.get("hashtags") or []
    base = ["#coding", "#softwareengineer", "#techreels", "#programming", "#interviewprep", "#lexinode"]
    allt = list(dict.fromkeys([t.lower() for t in tags] + base))[:14]
    parts.append(" ".join(allt))
    return "\n\n".join(p for p in parts if p)[:2200]


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--dry-run", action="store_true")
    ap.add_argument("--topic", default="")
    ap.add_argument("--sample", action="store_true", help="render the built-in sample (no AI, silent) to check the look")
    ap.add_argument("--previews", action="store_true", help="also save a few PNG frames")
    args = ap.parse_args()
    global ACC
    ACC = random.choice(ACCENTS)
    _BGIMG.clear()
    print(VERSION, file=sys.stderr)
    OUT_DIR.mkdir(exist_ok=True)
    stamp = time.strftime("%Y%m%d_%H%M")
    if args.sample:
        spec = validate(json.loads(json.dumps(SAMPLE)))
    else:
        import core as M
        recent = [c.split("\n")[0][:80] for c in M.ig_recent_captions(25)]
        topic = args.topic or random.choice(SEEDS)
        print("TOPIC:", topic, file=sys.stderr)
        spec = gen_spec(topic, recent)
    with tempfile.TemporaryDirectory() as td:
        tmp = Path(td)
        durs, audio, voice = build_audio(spec, tmp, fake=args.sample)
        out = OUT_DIR / f"{stamp}_explainer.mp4"
        prev = None
        if args.previews or args.sample:
            prev = OUT_DIR / "previews"
            prev.mkdir(exist_ok=True)
        secs = render(spec, durs, audio, out, prev)
    cap = build_caption(spec)
    (OUT_DIR / f"{stamp}_explainer.txt").write_text(cap, encoding="utf-8")
    print(f"video: {out} ({secs:.0f}s, voice {voice})", file=sys.stderr)
    print(cap)
    if args.dry_run or args.sample:
        return
    import core as M
    url = M.upload_video(str(out))
    mid = M.publish(url, cap, reel=True)
    print(f"POSTED explainer Reel: {mid}", file=sys.stderr)


if __name__ == "__main__":
    try:
        main()
    except Exception as ex:
        print(f"ERROR: {ex}", file=sys.stderr)
        sys.exit(1)
