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

- Each scanned track gets two embeddings: **style** (genre tags + description) and **text** (title + start of the lyrics). Relevance = 0.7 × style similarity + 0.3 × text similarity (cosine), so the genre matters more than the lyrics; +0.1 if all query words also appear literally. For a text query the style part is the average of the whole tag string and the single closest tag (so “space ambient” finds a track tagged `mysterious, ambient, soft, film score, …`); tag vectors are cached since tags repeat a lot. A text query is compared with both parts; *Similar* compares style with style and text with text.
- The dataset can't be indexed up front, so the search samples it: a track is queued if it scores at least 0.3 **and** is in the top 20% of everything scanned for this query. If the queue is still empty after ~100 scans, the floor softens to 0.2 so something from a large sample can still surface. The first 120 tracks are always scanned hard, and the queue keeps the 15 best, sorted.
- **Adaptive pace**: up to 8 parallel workers with no delay while the queue is empty; as matches accumulate, active workers taper down and the pause between probes grows (~0.2 s → ~3 s). Soft matches below 0.3 are dropped once a solid hit appears.
- If the model fails to load, search falls back to exact word matching.

Vocals detection: Suno / Udio / Riffusion: empty or `[Instrumental]` lyrics = instrumental; Sonauto: `instrumental` tag. Mureka has no lyrics metadata, so it is skipped when a filter is on.

## Hugging Face rate limits

Anonymous requests to `/resolve/` are limited to 3,000 per 5 minutes per IP
([docs](https://huggingface.co/docs/hub/rate-limits)). The page keeps its own cap of 2,400 per 5 minutes
and pauses on HTTP 429.

## Notes

- Track titles from some Riffusion songs are mis-encoded in the dataset itself.
- The audio is AI-generated; check the license of the dataset (MIT) and of the source platforms before reusing it.
