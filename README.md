# 🎧 AI FM

Every track made by AI. Press next. Keep what you love.

A single-page web radio that plays random AI-generated music from the
[ai-music/ai-music-deduplicated](https://huggingface.co/datasets/ai-music/ai-music-deduplicated)
dataset (~870K songs from Suno, Udio, Riffusion, Mureka and Sonauto). No backend, no build step.

## Features

- **Random radio**: press *Next* (or let a track end) to get another random song.
- **Vocals filter**: All / 🎤 With vocals / 🎹 Instrumental (based on the lyrics metadata).
- **Semantic search**: describe a vibe (“rainy night lo-fi”, “епічна музика для бою”) and press Enter. Tracks are matched by meaning, not by exact words, across languages; each match shows a % score and the best ones play first. Example chips under the search box.
- **≈ Similar**: find more tracks like the one playing.
- **Live queue**: matches are collected in the background; the page shows how many were found and you can play any of them right away.
- **Like** ♥ to keep a track in the *Saved* list, **Save** ⬇ to download the audio file.
- **Back** ⏮ goes to the previous track (restarts the current one if it has played for more than 3 s).
- **EN / UA** language switch (picked from the browser language: Ukrainian → UA, anything else → EN; manual choice is remembered).
- Volume slider; volume, likes and history are stored in `localStorage`.

## Run

It is a static page, serve it with any static server (opening it via `file://` may be blocked by CORS):

```bash
python3 -m http.server 8765
# open http://localhost:8765
```

## How it works

The dataset is ~2.5 TB of `.tar` shards (audio + JSON metadata sidecar per song).
The Hugging Face datasets-server only exposes the first 1,200 rows, so the page reads the shards directly:

1. Picks a random shard (weighted by size) and a random 512-byte-aligned offset, using HTTP `Range` requests.
2. Scans forward to the next tar header of an audio file, then walks a few neighbouring tracks, reading each JSON sidecar (title, tags, lyrics).
3. Downloads only the chosen track's byte range and plays it from a Blob.

### Semantic search

A multilingual sentence-embedding model ([paraphrase-multilingual-MiniLM-L12-v2](https://huggingface.co/Xenova/paraphrase-multilingual-MiniLM-L12-v2), int8, ~118 MB, downloaded once and cached by the browser)
runs in the page via [transformers.js](https://huggingface.co/docs/transformers.js). It starts loading when the search box gets focus.

- Each scanned track is embedded from its title, tags, description and the start of its lyrics; the query (or the current track for *Similar*) is embedded too, and relevance is cosine similarity (+0.1 if all query words also appear literally).
- The dataset can't be indexed up front, so the search samples it: a track is queued if it scores at least 0.4 **and** is in the top 20% of everything scanned for this query. The first 120 tracks are always scanned, and the queue keeps the 15 best, sorted.
- If the model fails to load, search falls back to exact word matching.

Vocals detection: Suno / Udio / Riffusion: empty or `[Instrumental]` lyrics = instrumental; Sonauto: `instrumental` tag. Mureka has no lyrics metadata, so it is skipped when a filter is on.

## Hugging Face rate limits

Anonymous requests to `/resolve/` are limited to 3,000 per 5 minutes per IP
([docs](https://huggingface.co/docs/hub/rate-limits)). The page keeps its own cap of 2,400 per 5 minutes
and pauses on HTTP 429.

## Notes

- Track titles from some Riffusion songs are mis-encoded in the dataset itself.
- The audio is AI-generated; check the license of the dataset (MIT) and of the source platforms before reusing it.
