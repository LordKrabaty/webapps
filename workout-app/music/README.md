# Workout music

Drop two MP3 files here (e.g. exported from Suno) and push to GitHub:

| File | Plays during |
|---|---|
| `work.mp3` | exercises (work / manual phases) |
| `rest.mp3` | pauses (rest between sets/exercises, prep countdown, L↔R side switch) |

Cardio intervals: an interval whose name contains rest / pause / recover / cool
automatically plays `rest.mp3` even though it is a running interval. The ♪W/♪R
button on each interval row overrides the auto-detection.

Both tracks loop. Each keeps its position while the other plays, so the work
track resumes where it left off after a rest. Toggle with the ♪ button next to
▶ start workout (per-device preference, stored in localStorage).

Any slot — work track, rest track, or one exercise's override in its ⚙
settings — can be set to **— no music —**: that part of the workout stays
silent (the alert beeps still sound).

Notes:
- Keep files reasonably small (a few MB). The app downloads each file whole on
  first use (Cache Storage, `workout-music-v1`) and from then on plays it from
  the device — nothing streams mid-workout. Streaming broke on every push:
  each Pages deploy changes the ETag of all files, and the next range request
  of a playing track then failed silently.
- Replacing a file under the same name: the app notices the different size on
  the server and downloads the new version in the background; it plays from
  the next app start.
- If another app (or a call, the media notification, unplugged headphones)
  pauses the music, the workout leaves the audio alone — a `♪ ▶` button appears
  next to ♪ to resume it. The ♪ modal has "copy music log" for debugging.
- The repo is public, so these files are downloadable by anyone with the URL.
- Missing files are harmless: the app shows a one-time "not found" status and
  plays no music; toggling ♪ off/on retries after you push the files.
