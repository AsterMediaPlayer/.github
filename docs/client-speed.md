# Making the Aster clients fast

Android 0.10.8 went from "it feels very slow" to opening Home in about a second on an onn. 4K Pro (an Amlogic TV box,
the slowest device we test on). This page is what made the difference, in order of how much it mattered, so the other
clients can check themselves against it. It also covers what the Apple apps' `speed` branch, the official Jellyfin
Android TV app, Wholphin and Findroid do.

Each section says what to do, why, what it measured on Android, and where the code is in Android
(`aster-android`, `app/src/main/java/com/astermediaplayer/astermediaplayer/`) and, where it exists, in the Apple app.

**Android, cold launch on the onn. 4K Pro:**

| | Before (0.10.7, debug build) | After (0.10.8, release build) |
|---|---|---|
| App window on screen | 4.4 s | **0.38 s** |
| Last time's Home on screen | 16–19 s (a spinner until the server answered) | **≈ 1.1 s** |
| Fresh Home from the server | 5–10 s | **1.7–1.9 s** |
| Home's size on the wire and on disk | 1 MB | **300 KB** |

---

## 0. Measure on a release build, on the slowest device

**Time things on a release build.** A debug build of a Compose app runs unoptimised code with no startup profiles,
and on the onn it was several times slower at everything. Its timings mislead, and "it feels slow" on a debug build
may be mostly the build. Android's window took 4.4 s on a debug build and 0.38 s on a release build of the same code.

Log the time each step takes, then read the log from a cold start. Android logs these lines (`adb logcat -s Aster:I`):

```
[Home] Last time's Home: shown in 1083 ms (the file in 259 ms)
[Home] Libraries in 1322 ms
[Home] From the server in 1895 ms
```

Also watch for Android's own `Choreographer: Skipped N frames`, which means the main thread was blocked for N/60
seconds. Every big win below began with one of these lines being far larger than expected.

## 1. Never decode the server's answers on the main thread

**The biggest single fix on Android.** The Jellyfin Kotlin SDK reads and decodes each response **on whichever thread
made the call**: `request(...).createResponse()` runs in the caller's context. Android's screens call the
repository from the UI thread, so every list of `BaseItemDto` was parsed on the thread that draws the screen. The
screen froze while Home's rows were decoded, and the network calls themselves queued behind the frozen thread. This
is the "Libraries in 5190 ms" that turned into 300 ms.

- **Android:** every server call in `data/AsterRepository.kt` runs inside `io { … }`, which is
  `withContext(Dispatchers.IO)`. Calls that change the signed-in state (sign-in, sign-out) are the exceptions.
- **Findroid and Wholphin** do the same: Findroid wraps every repository call in `withContext(Dispatchers.IO)`, and
  Wholphin runs its loaders on IO.
- **Apple:** the same trap exists if a `@MainActor` view model awaits a call whose `JSONDecoder` work runs on the main
  actor. Decode in a nonisolated function or a detached task.
- **Windows and Linux:** make sure `JsonSerializer.Deserialize` for item lists doesn't run on the UI thread, and
  prefer streaming from the response stream.

## 2. Ask the server for less

Home asked for `MediaSources` on every list, for one thing: the HD/4K badge. That returned every file, path and
audio and subtitle track of every title. **Three quarters of Home was track lists** (665 KB of 900 KB). Slimming the
lists made Home smaller to send, to decode and to keep, and made "From the server" go from 5.2 s to 2–3 s on its own.

**For lists** (shelves, grids, search, related), use the fields below:

| Parameter | Value |
|---|---|
| `Fields` | `Overview,Genres,Width,Height` (plus `SortName` for grids, `DateCreated` where sorting needs it) |
| `EnableImageTypes` | `Primary,Backdrop,Thumb,Logo` |
| `ImageTypeLimit` | `1` |

- **Never** put `MediaSources`, `MediaStreams`, `People`, `Chapters` or `Trickplay` on a list.
- `Width` gives the video's width for the badge: 3800 or more is 4K, 1280 or more is HD.
- `ImageTypeLimit=1` stops a title with 20 backdrops from sending 20 tags.

**For a title page,** fetch the single item with all its fields: `MediaSources` for audio and subtitles, `People`,
`Chapters` and so on. That's one item, not a hundred.

**Worth adding next:**
- `EnableTotalRecordCount=false` on shelves that don't page. The official TV app and Wholphin both do this.
- Ask for the total only on a grid's first page.

Android: `listFields` and `listImages` in `data/AsterRepository.kt`. Apple: `listFields` and `detailFields` in
`JellyfinClient.swift`.

## 3. Show last time's screen at once, then refresh it in place

Keep the last answer of Home, and of Discover, on disk. Show it the moment the screen opens, then load the fresh one
and swap it in **row by row**, keeping focus. The person sees their Home in about a second instead of a spinner.

- **One file per server, user and screen,** named by a hash of `serverUrl|userId|screen`, so another server's or
  person's never shows. Delete them all on sign-out.
- **No expiry.** Always ask again after showing it; it's only ever a first picture.
- **Read and write it off the main thread,** and write it whole or not at all (a temporary file, then a rename).
- **Store it so it decodes as fast as the server's answer.** Android first wrote one JSON tree and read it back with
  `parseToJsonElement` then `decodeFromJsonElement`. That took 8–10 s on the onn: slower than asking the server. It
  now writes one line per row, each line the row's items as the server sent them, decoded straight from the string,
  the same streaming decode as a server answer.
- **Keep it small** by asking for less (§2): Home's file went from 1 MB to 300 KB.
- If the refresh fails (offline), keep showing the old screen rather than an error.
- **Discover:** keep the server's page as it came (`/AsterTracking/Discover/Home`), plus which sections there were and
  Seerr's slider list and their titles. Rows shown from the snapshot are asked for again once Seerr answers, without a
  spinner over them.

Android: `data/ResponseCache.kt`; `HomeContent.encode`/`decode` and `HomeViewModel` in `ui/home/HomeScreen.kt`;
`DiscoverViewModel.restore`/`keep` in `ui/discover/DiscoverModels.kt`. Apple: `Server/ResponseCache.swift`, and
`HomeView`'s `restore()` and `keep()`.

## 4. Do things side by side, and don't wait for what isn't needed yet

- **Home:**
  - resume, next up and the libraries go out together;
  - then the latest of every library at once.
  - Wholphin caps these at 4 at a time; with the shared client in §6 there's room for more.
- **Discover's checks, side by side:**
  - the plugin's `/Features`, Seerr's `auth/me` and the Upcoming check;
  - the server's Discover page as soon as `/Features` says the plugin is there, without waiting for Seerr.
  - Android did these one after another before 0.10.8.
- **Several screens asking for `/Features` at once** share one request; Android uses a mutex around it.
- **Title page:** the item, its disc menu, and for a show its seasons and next episode, all at once. They were one
  after another.
- **Playback:**
  - Ask the server for the stream **without** first waiting for skip markers (`MediaSegments`) or the next episode.
  - Load those in the background; Skip Intro and Next Episode aren't needed in the first seconds.
  - The Apple apps load them after `play()` too.

## 5. Open a title page on the card's item

The card already has the title's backdrop tag, logo, name, year and overview. Keep the items the lists showed (an
in-memory cache of a few hundred, by id), and open the page on that item at once: art, logo, meta line, overview. The
full item, seasons and buttons fill in a moment later.

**On a TV, hold the buttons back** until the full item arrives, and show a small spinner in their place. Then move focus
to Play. Otherwise the remote lands on a button that moves, or on Play for a show whose next episode isn't known yet.

Android: `repo.noteSeen`/`seen`, `DetailContent.preview` in `ui/detail/DetailScreen.kt`. Apple: `ItemDetailView`
renders `model.item ?? item`.

## 6. Pictures

| What | Value | Why |
|---|---|---|
| **One HTTP client** for the API, the SDK and images | shared connection pool and dispatcher | Android had three clients, each opening its own connections to the same server. The official TV app and Wholphin share one. |
| **Requests at once per server** | 12, up from OkHttp's 5 (or URLSession's 6) | a screen of posters arrives 2–4× faster (the Apple apps measured this) |
| **Size** | ask at the size drawn: posters about 400–480 px wide, landscape cards 600–800, backdrops 1920 | decoding a full-size JPEG is slow on a TV box's CPU. Wholphin and the official TV app pass the card's pixel size. |
| **Quality** | `quality=80` up to 800 px wide, 90 above | about 40% smaller with no visible difference at card size |
| **Always pass `tag=`** | the server's image tag | the address changes when the picture does, so a cached copy is good forever |
| **Disk cache** | 512 MB | a relaunch shows every poster with no network |
| **Fade-in** | off on a TV | a fade on every image is a lot of frames when a screen of them arrives at once |
| **Warm ahead** | each shelf's posters past the edge, into the disk cache only, 4 at a time | the next cards are there when scrolled to, and 8 of the 12 connections stay free for what's on screen |

- **Android:** `sharedHttp` in `data/AsterRepository.kt`, Coil set up in `AsterApp.kt`, `picture()` for quality,
  `ui/components/ImageWarmer.kt`. The warm-ahead decodes each picture at 8×8, since Coil 3 always decodes; the disk
  cache still keeps the whole file.
- **Apple:** `JellyfinClient.imageSession`, `RemoteImage.warm(urls, concurrently: 4)`.

## 7. Playback start

- **Start at the resume point when loading,** not by seeking after the player is ready. Otherwise the server's first
  bytes are for the opening.
  - Android: `setMediaItem(item, startPositionMs)`.
  - Apple: `seek(to:)` before `replaceCurrentItem`.
- **Start playing after 1.5 s of buffer,** not ExoPlayer's 2.5 s.
  - Android uses `DefaultLoadControl.setBufferDurationsMs(30_000, 90_000, 1_500, 5_000)`; a stall still refills to 5 s.
  - Neither the official TV app nor Wholphin lowers it.
- **Nothing extra on the way to the first frame** (§4): skip markers, next episode, theme cards and artwork all come
  after.
- **Work out the device's decoders once, in the background.** On Android, `MediaCodecList` queries took over a second
  on the onn. Android now does them in the background at launch, so the first video doesn't wait, and keeps the
  answers. Wholphin builds its device profile once on a background thread and keeps it until a setting changes.

Measure with Android's start timing in the debug report (Settings ▸ About), which logs each step from the item to
the first picture. Before these changes a direct play took 11.5 s, most of it buffering to the first picture.

## 8. Launch

- **Nothing on the main thread before the first frame** but what draws it.
  - Android was checking audio passthrough, the video decoders and the input devices there: 1.3 s and 77 dropped frames
    on the onn.
  - The official TV app restores the session on IO through App Startup.
- **Restore the sign-in from local storage** and go straight to Home, with no server round trip first.
- **Shrink the release build.** Android uses R8. Wholphin also turns on R8 full mode and optimised resource
  shrinking.

## 9. Not done yet, in any Aster app

- **Wait for focus to rest before loading the big backdrop.**
  - Wholphin clears the backdrop and applies the new one after 500 ms if the card is still focused.
  - The official TV app cancels the load on every focus change and keeps 800 ms between changes.
  - Fast scrolling then loads no backdrops at all.
- **A startup profile shipped with the app** (`baseline-prof.txt` and `profileinstaller`). None of the apps we
  looked at have one, so it's free speed none of them have.
- **Constant-bitrate seeking** in the extractor (`DefaultExtractorsFactory().setConstantBitrateSeekingEnabled(true)`).
  The official TV app and Wholphin both set it.
- **`EnableTotalRecordCount=false`** on rows (§2).
- **Ask for PlaybackInfo when a title page opens,** so Play has it already. Nobody does this yet; keep it to one
  title at a time.
- **On Android, one remaining 1.2 s stall** right after the saved Home appears, most likely the first draw of Home
  itself. Profile it next.

## 10. Two bugs Android hit that the other apps may share

- **Upcoming showed episodes a day early.** Jellyfin sends a premiere date with no time as midnight UTC
  (`2026-11-06T00:00:00Z`). Converted to local time west of UTC, that's the evening before. Read a midnight date
  **as the calendar date written**; convert only dates that carry a real time. Jellyfin's upcoming list also starts
  yesterday, so drop days before today. Android: `UpcomingSchedule.airDay`.
- **YouTube trailers didn't open on Android TV.** The TV's YouTube app ignores `vnd.youtube:ID`. Open
  `https://www.youtube.com/watch?v=ID` in the package `com.google.android.youtube.tv`, then `com.amazon.firetv.youtube`
  (Fire TV), then `com.google.android.youtube` (phones), then any browser.

## 11. Where each app stands (2026-11-07)

| | Off-main decoding (§1) | Slim lists (§2) | Last screen at once (§3) | Side by side (§4) | Page from card (§5) | Pictures (§6) | Playback (§7) | Launch (§8) |
|---|---|---|---|---|---|---|---|---|
| **Android** | ✓ 0.10.8 | ✓ 0.10.8 | ✓ Home, Discover | ✓ | ✓ | ✓ | ✓ | ✓ |
| **iOS / tvOS / macOS** | check | ✓ (`speed` branch) | ✓ Home, Anime, Movies/TV heroes; not Discover | ✓ mostly; the title page still loads one thing after another | ✓ | ✓ | ✓ seek before load; check the extras | ✓ |
| **Linux** | check | check | to do | Discover ✓ | check | check | check | check |
| **Windows** | check | check | to do | check | check | check | check | check |
