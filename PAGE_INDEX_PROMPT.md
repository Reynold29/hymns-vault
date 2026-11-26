# Prompt: build the page index for the lyrics-correction automation

Paste everything below the line into Antigravity (or tell it to read this file).

---

You are helping me prepare the page-photo index used by my lyrics-correction automation. Photos of printed book
pages are in a local git repo; a Node tool reads each photo with Google Cloud Vision OCR, matches the text to the
stored songs, and writes `page-index.json` listing which page photo(s) each song is on. The n8n workflow reads that
file from GitHub to know which photo to compare a reported song against.

## Locations (my Mac)
- TOOLS (Node scripts, read the README there first): `/Users/reyzie29/Documents/Personal Projects/CSI-Hymns-Android-Native/automation/n8n`
- IMAGES repo (photos + `page-index.json`, remote `Reynold29/hymns-vault`, branch `main`): `/Users/reyzie29/Documents/Personal Projects/hymns-vault`
- DATA repo (the stored songs, READ ONLY, remote `Reynold29/csi-hymns-vault`): `/Users/reyzie29/Documents/Personal Projects/csi-hymns-vault`

IMAGES folders and the book each one means:
`hymns/` = Hymn, `keerthanes/` = Keerthane, `mangalore-tunes/` = M.T. Hymn,
`order-of-service/regular/` = Order of Service Regular, `order-of-service/festival/` = Order of Service Festival.
Photos are named `page<N>.jpg|jpeg|png` where N is the printed page number (for example `page12.jpg`).

## Hard rules
1. **Never print, log, echo, write to a file, commit or ask me for the Vision API key.** It must already be exported as
   `GOOGLE_VISION_API_KEY` in this terminal. Check only with `[ -n "$GOOGLE_VISION_API_KEY" ] && echo set || echo missing`.
   If it is missing, stop and tell me to run `export GOOGLE_VISION_API_KEY='...'` myself in this terminal.
2. **Always build the index for ALL books in one run (never pass `--books`).** The script rewrites `page-index.json`
   from scratch, so a partial run would delete the other books' entries. OCR results are cached in
   `tools/.ocr-cache/`, so re-running only pays for new or changed photos. Never delete that cache and never use `--fresh`.
3. Do not edit the DATA repo or any Android/iOS source. Do not change the tool scripts unless you find a bug; if you
   do, explain it and ask first.
4. No force pushes. Never delete photos. Only add photos and `page-index.json` to the IMAGES repo.
5. Do not retry a failing OCR call more than 3 times. If you see HTTP 403 or 429, stop and tell me (403 = API not
   enabled or key restricted; 429 = quota).

## Steps
1. **Preflight.** Check `node --version` (need 20 or newer). Run `git pull --ff-only` in the DATA repo and in the IMAGES repo.
   Confirm the key is set (rule 1). Run `git status` in IMAGES.
2. **Inspect the photos.** For each IMAGES folder, count images and list problems: wrong or missing page number in the
   name, duplicates such as both `page12.jpg` and `page12.png`, unsupported types (only jpg, jpeg and png work, so
   flag `.heic`, `.webp`, `.pdf`), photos in the wrong folder, and any file over 6 MB. Fix names only if it is
   unambiguous; for oversize photos, resize with `sips -Z 2600 --setProperty format jpeg --setProperty formatOptions 80 <file>`
   and tell me which ones. Ask me about anything else.
3. **Spot check.** For one photo per non-empty folder run, from TOOLS:
   `node tools/ocr-check.mjs "<IMAGES>/<folder>/<photo>" --book "<Book>" --data "<DATA>"`
   Report whether the page matched stored songs and how well. If a folder's sample matches nothing, stop and
   tell me (likely the wrong folder or a poor photo).
4. **Save the old index** (`cp page-index.json "$TMPDIR/page-index.before.json"`), then build, from TOOLS:
   `node tools/build-page-index.mjs --images "<IMAGES>" --data "<DATA>"`
5. **Review the result and write me a report** with, per book: photos read, songs placed on pages, songs not found
   (first 30), songs with coverage below 0.7 or spread over more than 3 photos, and photos that matched no song
   (covers, blank pages, indexes). Many songs will legitimately be missing while only part of a book has been
   photographed; say so, do not treat that as failure. A low-coverage song can also mean the stored lyrics have
   real errors, which is exactly what this project fixes, so list those rather than hiding them.
6. **Check nothing was lost.** Compare the old and new index: every song that was in the old index must still be
   there unless its photo was removed. Report any difference. Then run `node test-tools.mjs` in TOOLS.
7. **Publish** (only if steps 3 to 6 are clean): in IMAGES run `git add -A`, commit with the message
   `chore: add page photos and rebuild page index`, and `git push origin main` (no force). Then fetch
   `https://raw.githubusercontent.com/Reynold29/hymns-vault/main/page-index.json` and confirm it contains the new
   songs (this URL can lag by up to 5 minutes; retry once after a minute).
8. **Finish** with a short summary: counts per book, anything I need to fix or re-photograph, and the commit hash.

Cost note for me: Cloud Vision gives 1,000 free pages a month, then about USD 1.50 per 1,000. Tell me how many
photos were actually sent to Vision (not served from cache) before you start step 4 if it is more than 500.
