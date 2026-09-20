# promptYou are a senior full-stack engineer and technical writer. Your task is to take the existing open-source project walterwhite-69/ReAnime.to-API and turn it into a fully deployed, publicly accessible anime metadata and playback-resolution service that powers the frontend at https://owais-anime-stream.onrender.com/. You will also produce complete documentation on the same repository so any developer can self-host and integrate it.

1. Project Context
Backend repository: https://github.com/walterwhite-69/ReAnime.to-API
Backend description: A self-hosted anime metadata and playback-resolution API. Pure Python (FastAPI) plus a Node.js WASM bridge that handles the CDN's rotating AES-256-CBC token layer for HLS delivery. No headless browsers.
Frontend URL: https://owais-anime-stream.onrender.com/
Frontend behavior: A catalog console with search, trending releases, browse-all, and a playback view. It currently has no backend wired to it.
Goal: Deploy the backend on Render, connect the frontend to it, make anime search and playback work end-to-end, open-source the integration, and document everything on the repository.

2. Repository Structure to Create
Fork or clone the backend repo. Then add the following structure so the project is self-documenting and deployable:

text
ReAnime.to-API/
├── reanime/
│   ├── __init__.py
│   ├── app.py              # FastAPI application, routes
│   ├── catalog.py          # reanime.to catalog data retrieval
│   ├── resolver.py         # Node WASM bridge for CDN token resolution
│   └── models.py           # Pydantic response models
├── node/
│   ├── resolve.js          # WASM loader + AES-256-CBC token processing
│   └── package.json        # Node dependencies for WASM runtime
├── requirements.txt        # Python dependencies
├── render.yaml             # Render deployment blueprint
├── Procfile                # Process definition for Render (optional)
├── .env.example            # Environment variable template
├── README.md               # Complete setup, API docs, deployment guide
├── CONTRIBUTING.md         # Contribution guidelines
├── LICENSE                 # MIT or Apache 2.0
└── frontend-integration/
    ├── api-client.js       # Drop-in JS client for the frontend
    └── README.md           # How to connect any frontend to this API
3. Backend Implementation Requirements
3.1 FastAPI Application (reanime/app.py)
Implement the following endpoints exactly as specified in the original repo:

Method	Path	Description
GET	/search?q=...&limit=20	Search anime by name
GET	/home?limit=20	Latest aired + top weekly
GET	/top?period=week&limit=20	Top anime (day / week / month)
GET	/schedule	Weekly airing schedule
GET	/info/{slug}	Anime metadata + full episode list
GET	/episodes/{slug}	Episode list only
GET	/servers/{slug}/{episode}	All playback servers for an episode
GET	/stream/{access_id}?v=2	Resolve stream → HLS URL + subtitles
GET	/stream/from-link?link={url}	Same, but pass the full CDN URL
GET	/thumbnails/{anilist_id}	Episode thumbnail data
GET	/recommendations/{slug}	Related anime
The slug is the URL-friendly anime ID from reanime.to (e.g., one-piece-xamk74).

3.2 CORS Configuration
Add CORSMiddleware to the FastAPI app. Allow the frontend origin https://owais-anime-stream.onrender.com and http://localhost:* for local development.

python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://owais-anime-stream.onrender.com",
        "http://localhost:3000",
        "http://localhost:5173",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
If the frontend is served from the same Render service, add the frontend’s origin or use allow_origins=["*"] only for development. In production, restrict to the exact frontend URL.

3.3 Node WASM Bridge
The /stream and /stream/from-link endpoints must call a Node.js child process or a lightweight Node service that loads the CDN’s WASM module and processes the AES-256-CBC token payload.

Use subprocess in Python to invoke node node/resolve.js --input <payload>.

Or, if performance permits, run a persistent Node microservice on a separate port and proxy from FastAPI.

The routine must match the CDN’s rotating WASM-based AES-256-CBC token format used by the delivery layer.

Return the resolved .m3u8 URL, subtitles (SRT/VTT, multiple languages), thumbnail VTT, intro/outro chapter timestamps, and video title.

3.4 Response Shape Compliance
Ensure /servers returns:

json
{
  "sub": [
    {
      "serverName": "HD-2",
      "dataLink": "https://flixcloud.cc/e/abc123?v=2",
      "dataType": "sub"
    }
  ],
  "dub": [],
  "anilist_id": 178005,
  "anime": {},
  "intro_start": 90,
  "intro_end": 180
}
Ensure /stream returns:

json
{
  "url": "https://fetch1.flixcloud.cc/_v7/{video_id}/master.m3u8?token=...",
  "subtitles": [
    {
      "url": "https://...",
      "language": "English (Track 2 (ENG))",
      "format": "srt",
      "default": true
    }
  ],
  "thumbnails_vtt": "https://fetch1.flixcloud.cc/thumbnails_vtt/{video_id}",
  "video_title": "Episode.Title.1080p.mkv",
  "intro_chapter": null,
  "outro_chapter": {
    "start": 1340,
    "end": 1420,
    "title": "Credits"
  },
  "video_id": "0f477519-..."
}
4. Render Deployment
4.1 render.yaml
Create a Render Blueprint file at the repository root:

yaml
services:
  - type: web
    name: reanime-api
    runtime: python
    plan: free
    buildCommand: |
      pip install -r requirements.txt
      cd node && npm install
    startCommand: uvicorn reanime:app --host 0.0.0.0 --port $PORT
    envVars:
      - key: PYTHON_VERSION
        value: 3.11.0
      - key: NODE_VERSION
        value: 20.0.0
    healthCheckPath: /home?limit=1
4.2 Environment Variables
Create .env.example:

text
PORT=8000
HOST=0.0.0.0
FRONTEND_ORIGIN=https://owais-anime-stream.onrender.com
NODE_ENV=production
4.3 Build and Start Commands
Build: pip install -r requirements.txt && cd node && npm install

Start: uvicorn reanime:app --host 0.0.0.0 --port $PORT

If the Node WASM layer needs to run as a separate process, use a Procfile:

text
web: uvicorn reanime:app --host 0.0.0.0 --port $PORT
node: node node/resolve-server.js
Render supports multiple process types via Procfile. If using a single service, embed the Node call as a subprocess.

4.4 Free Tier Considerations
Render free tier spins down after 15 minutes of inactivity.

Add a /health endpoint that returns {"status":"ok"}.

Document how to set up an external cron job (e.g., cron-job.org) to ping /health every 14 minutes to keep the service warm.

Note the cold-start delay (10–30 seconds) in the README.

5. Frontend Integration
5.1 API Client for Frontend
Create frontend-integration/api-client.js:

javascript
const API_BASE = "https://reanime-api.onrender.com";

export async function searchAnime(query, limit = 20) {
  const res = await fetch(`${API_BASE}/search?q=${encodeURIComponent(query)}&limit=${limit}`);
  return res.json();
}

export async function getHome(limit = 20) {
  const res = await fetch(`${API_BASE}/home?limit=${limit}`);
  return res.json();
}

export async function getTop(period = "week", limit = 20) {
  const res = await fetch(`${API_BASE}/top?period=${period}&limit=${limit}`);
  return res.json();
}

export async function getInfo(slug) {
  const res = await fetch(`${API_BASE}/info/${slug}`);
  return res.json();
}

export async function getServers(slug, episode) {
  const res = await fetch(`${API_BASE}/servers/${slug}/${episode}`);
  return res.json();
}

export async function getStream(accessId, v = 2) {
  const res = await fetch(`${API_BASE}/stream/${accessId}?v=${v}`);
  return res.json();
}

export async function getStreamFromLink(link) {
  const res = await fetch(`${API_BASE}/stream/from-link?link=${encodeURIComponent(link)}`);
  return res.json();
}
5.2 Frontend Wiring
The frontend at https://owais-anime-stream.onrender.com/ currently has:

A catalog console with “QUERY / Trending releases / MODE / Highest rated / Browse all”.

A playback view with synced subtitles.

Replace any mock data fetching with calls to the API client:

Search bar: call searchAnime(query).

Trending releases: call getHome().

Top charts: call getTop(period).

Browse all: call getHome(limit=50) or /schedule.

Playback view: on episode selection, call getServers(slug, episode), then for the chosen server call getStreamFromLink(dataLink) to get the .m3u8 URL and subtitles. Feed the .m3u8 URL into the player (e.g., HLS.js or native <video> with HLS support). Load subtitles from the subtitles array.

Thumbnails: call getThumbnails(anilist_id) for episode preview sprites.

5.3 CORS and Mixed Content
Ensure the frontend is served over HTTPS (Render does this automatically).

Ensure the API is also HTTPS. If the frontend and API are on different Render services, CORS must allow the frontend origin.

If the frontend is static and served from Render Static Site, the API must allow that origin.

6. Open-Source Documentation (README.md)
Write a complete README.md that covers:

6.1 Project Title and Badges
text
# ReAnime.to API
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Node.js 20+](https://img.shields.io/badge/node-20+-green.svg)](https://nodejs.org/)
[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy)
6.2 Description
Explain what the API does: reads anime metadata and episode listings from the reanime.to catalog, and resolves playable HLS URLs by handling the CDN’s rotating token layer. Returns .m3u8 URLs, subtitles, thumbnails, and chapter data. No headless browsers.

6.3 Features
Search anime, browse home/top charts, get airing schedules

Full anime info with episode lists

All available playback servers (HD-1 sub, HD-1 dub, HD-2 sub, HD-2 dub)

Resolve the actual .m3u8 stream URL

Returns subtitles (SRT/VTT, multiple languages), thumbnail VTT sprites, intro/outro chapter timestamps

6.4 Requirements
Python 3.11+

Node.js 20+

pip install fastapi uvicorn httpx[http2] pycryptodome

cd node && npm install

6.5 Local Setup
bash
git clone https://github.com/walterwhite-69/ReAnime.to-API.git
cd ReAnime.to-API
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r requirements.txt
cd node && npm install && cd ..
uvicorn reanime:app --host 0.0.0.0 --port 8000
6.6 API Endpoints Reference
Reproduce the endpoint table from Section 3.1 with request/response examples for at least /search, /servers, and /stream.

6.7 Typical Flow
GET /search?q=demon+slayer → pick a slug.

GET /servers/{slug}/{episode} → get sub[] and dub[] arrays with serverName + dataLink.

GET /stream/from-link?link={dataLink} → get resolved .m3u8 URL, subtitles, thumbnail VTT, chapters.

6.8 Deployment on Render
Step-by-step:

Push this repo to GitHub.

Go to Render Dashboard → New → Web Service.

Connect the GitHub repo.

Render reads render.yaml automatically. If not, set:

Runtime: Python

Build Command: pip install -r requirements.txt && cd node && npm install

Start Command: uvicorn reanime:app --host 0.0.0.0 --port $PORT

Add environment variables from .env.example.

Click Create Web Service.

Wait for build and deploy. Your API is live at https://<service-name>.onrender.com.

6.9 Frontend Integration Guide
Include the api-client.js code and a short example of wiring it into a React/Vue/Svelte or vanilla JS frontend. Show how to fetch .m3u8 and feed it to HLS.js.

6.10 Keeping the Service Warm (Free Tier)
Render free services spin down after 15 minutes of inactivity.

Use cron-job.org or UptimeRobot to ping https://<service>.onrender.com/health every 14 minutes.

Document the cold-start delay.

6.11 License
MIT License. Include full text.

6.12 Contributing
Link to CONTRIBUTING.md. Basic guidelines: fork, branch, PR, run tests (if any), update docs.

7. Additional Files
7.1 CONTRIBUTING.md
text
# Contributing to ReAnime.to API

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/your-feature`.
3. Commit your changes: `git commit -m "Add your feature"`.
4. Push to the branch: `git push origin feature/your-feature`.
5. Open a Pull Request.

Please include tests for new endpoints and update the README if behavior changes.
7.2 LICENSE
MIT License, full text, with copyright “2026 ReAnime.to API contributors”.

7.3 .github/workflows/deploy.yml
Optional: GitHub Actions workflow to deploy to Render on push to main.

yaml
name: Deploy to Render

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Render
        uses: render-examples/deploy-to-render@v1
        with:
          service-id: ${{ secrets.RENDER_SERVICE_ID }}
          api-key: ${{ secrets.RENDER_API_KEY }}
8. Definition of Done
The task is complete when:

The backend is deployed on Render and publicly accessible over HTTPS.

The frontend at https://owais-anime-stream.onrender.com/ can search anime, list trending, browse all, and play episodes using the deployed API.

CORS is configured so the frontend origin can call the API without errors.

The .m3u8 stream URL is resolved and playable in the browser.

Subtitles load and sync.

The repository contains a complete README.md with local setup, API reference, Render deployment steps, and frontend integration guide.

CONTRIBUTING.md and LICENSE are present.

render.yaml and .env.example are present.

The frontend-integration/api-client.js is committed and documented.

The repository is public and open-source.

9. Delivery Standards
Use the exact folder names, filenames, and endpoints given.

The code must be complete and runnable from a fresh clone.

Comments only where non-obvious. Clean, production-ready code.

Ship the full working implementation and thorough documentation in one pass.


---

### Support

If you enjoy this project and want to support more builds, you can optionally [support me on Ko-fi](https://ko-fi.com/yorusayano).
