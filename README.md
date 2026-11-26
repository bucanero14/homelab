This needs a running network generated manually

`docker network create web`

# Adguard

User: raul
Pass: nintendo

# Subtitle translation

Bazarr keeps managing subtitle downloads. **Lingarr** handles automatic translation
of subtitles for **newly added media only** — the existing library is not
bulk-translated.

The repo owns `arr/docker-compose.yml`. Runtime state and credentials live in
`arr/config/lingarr` (SQLite DB, Hangfire DB, encrypted keys) and are not tracked
in git.

## Start Lingarr

From the repo root:

`docker compose -f arr/docker-compose.yml up -d lingarr`

Open `http://lingarr.pinomagana.local`. Traefik routes it; no host port is published.

## One-time setup in Lingarr

1. **Settings → Services** — Compose already defines Gemini (primary) and Google
   Translate (fallback). Open the gear on the **Gemini** row and paste your Gemini
   API key. The model is preset to `gemini-3.5-flash-lite` and batching is enabled.
2. **Settings → Integration** — add `http://radarr:7878` and `http://sonarr:8989`
   with their API keys (Radarr/Sonarr → Settings → General → Security → API Key).
   Keep new items included in translation.
3. **Settings → Translation** — confirm the source priority from Compose: English,
   Japanese, Chinese (Simplified) `zh-CN`, Chinese (Traditional) `zh-TW`, French,
   German, Italian, Portuguese. Targets are `es-mx` → `es`. Enable **Only translate
   to the first target language** so only `.es-mx.srt` files are produced.
4. **Settings → Automation** — enable automation, use a schedule such as
   `0 5 * * *`, `5` max translations per run, and `48`-hour movie/show age
   thresholds. The age threshold plus Lingarr's media hash is what leaves the
   existing library untouched.
5. Optional: **Settings → Webhook** shows `/api/webhook/sonarr` and
   `/api/webhook/radarr`. Add them under Sonarr/Radarr → Settings → Connect for
   faster pickup than the daily schedule.

## Bazarr side

- Settings → Languages: enable Japanese, Chinese Simplified (`zh`), Chinese
  Traditional (`zt`), plus **Spanish (Latino)** (`ea`, produces `.es-mx.srt`).
- Add those source languages and Spanish (Latino) to the default profiles for
  newly added series/movies, so a source subtitle exists for Lingarr to translate.
- Optional: Settings → Providers → Translation → **Lingarr**, endpoint
  `http://lingarr:9876`, for manual translation from Bazarr.
- Do **not** run Missing → Mass Translate across the existing library.

## Verify

Add one test movie/episode with a source subtitle, wait for the schedule (or fire
the webhook), and confirm `Movie.es-mx.srt` appears and Jellyfin sees it. Confirm
older media gained no new subtitle files.

## Cost note

`gemini-3.5-flash-lite` is free on Gemini's free tier (with lower rate limits and
content usage for model improvement). Paid is about $0.30 input / $2.50 output per
1M tokens — roughly $3/year at a couple of movies per month plus 1–3 weekly
episodes, or under $10 with retries. Switching `GEMINI_MODEL` to `gemini-3.8-flash`
is still only a few dollars a year if you want better contextual quality.
