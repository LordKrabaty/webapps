# Workout music

Drop two MP3 files here (e.g. exported from Suno) and push to GitHub:

| File | Plays during |
|---|---|
| `work.mp3` | exercises (work / manual phases) |
| `rest.mp3` | pauses (rest between sets/exercises, prep countdown, L↔R side switch) |

Both tracks loop. Each keeps its position while the other plays, so the work
track resumes where it left off after a rest. Toggle with the ♪ button next to
▶ start workout (per-device preference, stored in localStorage).

Notes:
- Keep files reasonably small (a few MB) — they are fetched from GitHub Pages
  on first play and then cached by the browser.
- The repo is public, so these files are downloadable by anyone with the URL.
- Missing files are harmless: the app shows a one-time "not found" status and
  plays no music; toggling ♪ off/on retries after you push the files.
