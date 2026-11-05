# Aster Tracking: how the Aster clients work with it

The [Aster Tracking](https://github.com/AsterMediaPlayer/aster-companion-plugins/tree/main/AsterTracking) Jellyfin
plugin holds everything about tracking and discovery on the **server**, so every Aster app gets it the same way:

- **Seerr, through the server.** Aster's request features show only when the server turns them on, and the apps
  reach Seerr through the plugin as the signed-in person. **No app signs in to Seerr.**
- **Linked accounts.** Each person's Simkl, AniList, MyAnimeList, Last.fm, ListenBrainz and Hardcover accounts,
  linked once for every app. Trakt is still to come.
- **Scrobbling.** The server sends what people watch and listen to, from any app. The apps don't send it themselves.
- **Discover lists.** Watchlists, watching, recommendations, trending and this season's anime from those accounts, in
  one shape, with TMDB ids for Seerr and the title on the server when it's there.

This page is the contract the clients follow: iOS, tvOS and macOS (Aster-Media-Player), Android, Windows and Linux.
Aster Linux is the worked example ([commit f043ddc](https://github.com/AsterMediaPlayer/aster-linux/commit/f043ddc)).
The plugin's own README has the server side.

All routes are on the Jellyfin server, with the signed-in person's normal Jellyfin authorization
(`Authorization: MediaBrowser Token="…"`). JSON comes back in PascalCase; read it case-insensitively.

---

## 1. Is the plugin there?

Ask once after sign-in, and again when the app comes back to the foreground after a long while:

```
GET /AsterTracking/Features
→ 200 { "Seerr": { "Enabled": true, "Proxy": "/AsterTracking/Seerr/api/v1" },
        "Services": ["simkl", "anilist", "myanimelist", "lastfm", "listenbrainz", "hardcover"] }
```

| Answer | Meaning |
|---|---|
| 200 | The plugin is installed. `Services` lists what people can link on this server. |
| 404 | No plugin, which counts as **Seerr off** and **no linking, scrobbling or Discover lists** from the server. |
| Anything else, or no answer | Treat it like 404 for now, but don't forget a 200 you got earlier in this session. |

Plex servers never have the plugin. Emby will get its own port later, at the same routes.

## 2. Seerr: through the server, never signed in to

**Decided (2026-11-04): no Aster app signs in to Seerr.** The plugin holds Seerr's address and API key, set on the
dashboard by the server admin, and passes the apps' Seerr calls through, calling Seerr **as the signed-in person**.
Seerr's `X-Api-User` is matched to the Seerr user linked to that Jellyfin user. So Seerr's own permissions, quotas
and auto-approval apply as if the person had signed in. Someone without a Seerr account is imported from Jellyfin on
first use, with Seerr's defaults for new users.

- **When it shows:** Seerr's features (Discover's Seerr rows, Request buttons, the Requests page) show **only** when
  `Seerr.Enabled` is true. Without the plugin, a Plex server for example, they stay hidden.
- **How to call it:** Seerr's own API, unchanged (same paths, queries, bodies and replies), at
  `{server}{Seerr.Proxy}/{Seerr's path}`, for example `GET {server}/AsterTracking/Seerr/api/v1/discover/trending`.
  Send the **Jellyfin authorization header** and no Seerr cookie. `auth/me` returns the person's Seerr user.
- **What passes:** browsing, titles, search and requests: `auth/me`, `movie|tv|person|collection/…`, `discover/…`,
  `search`, `request…` (create, edit, delete, approve, decline, retry, which Seerr checks the person may do),
  `genres`, `studio`, `network`, `settings/discover`, `service/radarr|sonarr`, and the person's own
  `user/{id}/quota|requests|watchlist`.
- **What doesn't:** everything else (Seerr's settings, user management, other people's data) is **403**. A 503 means
  requests are off. A 502 means the server couldn't reach Seerr: say "requests aren't available right now", never
  "sign in".
- **Remove the apps' Seerr sign-in:** Settings ▸ Requests (address, password, Quick Connect) and the `seerr` entry in
  the `trackers` vault are retired. A sign-in an app saved before is simply no longer used.

Aster Linux: `SeerrAccounts.UsePluginAsync` makes the "account" the plugin's address plus the Jellyfin header. The
rest of its Seerr code didn't change. Settings ▸ Requests is gone
([commit 7433fd3](https://github.com/AsterMediaPlayer/aster-linux/commit/7433fd3)).

## 3. Accounts: linked only through the plugin

**Decided (2026-11-04):** tracking accounts live **only** in the plugin. The apps no longer keep their own Trakt,
Simkl, AniList or MyAnimeList sign-ins for tracking, and no longer write them to the `trackers` vault in the display
preferences. **Seerr's sign-in stays in the vault**, as it does today.

### Routes

Each route acts for the signed-in person. An administrator may add `?userId=` to act for someone else.

```
GET    /AsterTracking/Accounts
POST   /AsterTracking/Accounts/{service}/Link
POST   /AsterTracking/Accounts/{service}/Link/Complete      body for paste services: { "Code": "…" }
PUT    /AsterTracking/Accounts/{service}/Settings           { "Scrobble": false }
DELETE /AsterTracking/Accounts/{service}
```

The services are `simkl`, `anilist`, `myanimelist`, `lastfm`, `listenbrainz` and `hardcover`.

`GET /AsterTracking/Accounts` returns, for each service, `{ Available, Linked, Name, LinkedAt, Expires, Scrobble }`,
plus `Waiting` (sends to be retried) and `Log` (the last sends, newest first: `{ At, Service, Title, Result }`).
`Available` false means the server has no app for that service yet, so don't offer it. `Expires` is set only where
the person has to link again by a date: AniList, after a year. The others renew themselves.

### How each service links

`POST …/Link` returns `{ Url, Code, CompleteUrl, Paste, Interval, ExpiresIn }`. There are three ways it goes:

| Service | Flow | What the app shows | Then |
|---|---|---|---|
| Simkl, Hardcover | **Code** (`Code` set) | "Go to `Url` and enter `Code`", plus a QR code of `CompleteUrl` | Call `…/Link/Complete` every `Interval` seconds |
| Last.fm | **Approve** (no `Code`, `Paste` false) | Open `Url` (on a TV, show it as a QR code). The person approves Aster there | Call `…/Link/Complete` every `Interval` seconds |
| AniList, MyAnimeList, ListenBrainz | **Paste** (`Paste` true) | Open `Url`. The person signs in or copies their token, then pastes the code into the app | `…/Link/Complete` with `{ "Code": "…" }` |

`…/Link/Complete` answers:

| Status | Meaning |
|---|---|
| 200 | Linked. The body is the account: `{ Linked, Name, … }`. |
| 202 | Not entered or approved yet. Keep calling. |
| 400 | A paste service was sent no code. |
| 409 | The server has no app for this service. |
| 410 | The code ran out or was refused. Start again. |
| 502 | The service couldn't be reached. Say so and offer to try again. |

Things the Aster Linux and dashboard work taught us:

- **Check again when the person comes back to the app.** Phones and TVs pause an app in the background, and polling
  stops while the person is on the service's page. Last.fm also shows an error page *after* the person approves, but
  the approval still counts: tell people to ignore it and come back.
- **On a TV, offer only the code and approve flows** (Simkl, Hardcover, Last.fm). For the paste flows, say "link on
  your server's dashboard (Plugins ▸ Aster Tracking) or in Aster on your phone", since nobody pastes a token with a
  remote. MyAnimeList's code is very long.
- A MyAnimeList code can be pasted as the whole address too (`…?code=…`). The server takes either.
- A pending link lives in the server's memory: it's lost if the server restarts, which shows as 410. Start again.

### Moving off the vault

On the first run of a version with this, when the plugin is there:

1. Read the vault as now. If it has tracking accounts (anything but `seerr`), show once: "Tracking now lives on your
   server. Link your accounts again in Settings ▸ Tracking". Then stop using them and remove them from the vault, but
   keep `seerr`.
2. Settings ▸ Tracking lists `Services` from `/Features` with each account's state from `/Accounts`: Link, Unlink and
   the "send what I watch" switch (`Settings` → `Scrobble`). Show the `Log` too: it's how people see it working.

**Without the plugin** (a Plex server, or a Jellyfin server without it), an app may keep its own tracking as it does
today. Never run both for the same person: with the plugin there, the app's own tracking is off.

## 4. Scrobbling: the server does it, and the app reports playback well

The plugin listens to Jellyfin's playback events, so it works for every app, Aster or not. What the server sends:

| Media | When | Where |
|---|---|---|
| Films, episodes | Stopped past **85%**, or marked played | Simkl history. Anime also moves AniList and MyAnimeList progress |
| Songs | Played **half way or 4 minutes** (longer than 30 s), with "now playing" at the start | Last.fm, ListenBrainz |
| Audiobooks | Progress every 15 min while playing and on stop, Read at **98%** or marked played | Hardcover |

So **an app must not send any of this itself** when the plugin lists the service. Instead, it must **report playback
to Jellyfin properly**, or nothing is sent:

- `POST /Sessions/Playing` when playback really starts, `/Sessions/Playing/Progress` about every 10 s, and
  `/Sessions/Playing/Stopped` at the end. Each carries `ItemId`, `PlaySessionId` and **`PositionTicks`**.
- **Report the stop before the next item loads,** with the position the item reached. Aster Linux's music player
  reported a song stopped at position 0 once the next one had loaded. The server now also uses the furthest progress
  report, but don't rely on that.
- Report audiobooks and songs exactly like videos: start, progress and stop. Report on Sonos and other remote outputs
  too, not only when playing locally.
- "Marked played" goes through Jellyfin's normal played toggle. That counts for films, episodes and audiobooks, but
  not songs.

A quick check that an app reports well: play something past the rule, stop, and look at **Recent sends** on the
dashboard page (or `GET /AsterTracking/Accounts`).

## 5. Discover: the person's lists

```
GET /AsterTracking/Discover
→ [ { "Service": "simkl", "List": "watchlist", "Kind": "Watchlist", "Title": "Simkl Watchlist", "Personal": true }, … ]

GET /AsterTracking/Discover/{service}/{list}?page=1&pageSize=30        (pageSize at most 60)
→ { "Items": [ … ], "Page": 1, "TotalItems": 22, "HasMore": false }
```

Only the lists of services the person has linked come back, in the order to show them:

| Kind | Lists |
|---|---|
| Watchlist | Simkl Watchlist · AniList Planning · MyAnimeList Plan to Watch |
| Watching | on Simkl · on AniList · on MyAnimeList, each item with `Progress` ("Episode 4 of 12") |
| Recommended | Recommended by AniList · Suggested by MyAnimeList (none already on the person's list) |
| Trending | Simkl (films, shows, anime) · AniList trending anime · MyAnimeList top airing |
| This Season | This season's anime (AniList's, or MyAnimeList's without AniList) |

Each item:

```json
{ "Title": "Frieren: Beyond Journey's End", "Year": 2023, "MediaType": "tv", "Anime": true,
  "Poster": "https://…", "Overview": "…", "Progress": "Episode 4 of 12",
  "Ids": { "tmdb": "209867", "tvdb": "424536", "imdb": "tt22248376", "anilist": "154587", "mal": "52991", "anidb": "17617", "tmdbSeason": "1" },
  "InLibrary": { "ItemId": "…", "Name": "Frieren: Beyond Journey's End", "Type": "Series" } }
```

- `MediaType` is `movie` or `tv`, the way Seerr and TMDB split titles.
- `Ids.tmdb` opens Seerr's page for the title, Request button included. Nearly every item has one: anime get theirs
  through ARM. Leave out items without one when Seerr is the way in.
- `Poster` is a full address from the tracking site, not a TMDB path. Show it as it is.
- `InLibrary` is the film or show on this server, among what this person may see: offer **Play** or show an "On Server"
  badge. A later anime season matches the whole show, so the season itself may not be there yet.

How to show them (Aster Linux's `DiscoverViewModel.DiscoverRowsAsync`):

1. **Personal lists first** (Watchlist, Watching, Recommended), **in place of Seerr's Watchlist slider (type 3)**.
   That slider is Plex's watchlist only, empty for Jellyfin people.
2. **Seerr's own sliders**, in its admin's order.
3. **Trending and This Season after Seerr's Trending** (and In Theaters).
4. Each item is the app's usual **Seerr card**, built from the item, *not* looked up in Seerr one by one: twelve lists
   of 30 would be hundreds of Seerr calls each time Discover opens. Use the item's poster, `Progress` (or year and kind)
   as the subtitle, and "On Server" as the badge for `InLibrary`. Opening it fetches the full title from Seerr by TMDB
   id, as any Seerr card does.
5. A list that fails leaves the others. 410 means that service needs linking again: say so in Settings ▸ Tracking,
   not on Discover.

The server keeps each person's lists for 10 minutes. The first load of a list takes 1–3 s, so load the rows in
parallel and show each as it comes.

**Without Seerr** (switch off), the lists can still show: cards whose `InLibrary` is set open the library item, and the
rest show without a Request button. Aster Linux doesn't do that yet: its Discover needs Seerr.

## 6. Where each app stands (2026-11-04)

| | Seerr switch (§2) | Linking in Settings (§3) | Vault tracking retired (§3) | No app-side scrobbling (§4) | Playback reports (§4) | Discover lists (§5) |
|---|---|---|---|---|---|---|
| **Linux** | ✓ through the server, no sign-in | – (it never had tracking) | n/a | ✓ | Music ✓; audiobooks: check on Sonos | ✓ (Seerr only) |
| **iOS / tvOS / macOS** | to do | to do | to do: it keeps Trakt, Simkl, AniList and MAL in the vault (`AsterFIN/Trackers`) | to do: it sends Simkl, AniList and MAL itself | check | to do (`AsterFIN/Seerr/SeerrDiscover.swift`) |
| **Android** | to do (no Seerr yet) | to do | check | check | check | to do |
| **Windows** | to do | to do | check | check | check | to do |

On iOS the Seerr work is on the `seerr-discover` branch. The tracker code to retire behind the plugin is
`AsterFIN/Trackers/*`: keep it for servers without the plugin (§3, last paragraph).

## 7. Testing against the home server

- **The dashboard page** (Plugins ▸ Aster Tracking) links and unlinks for anyone and shows **Recent sends**. It's the
  quickest way to see whether an app's playback reached the services.
- **Seerr switch:** turn Requests off on that page and check the app hides Seerr. Turn it back on and check a request
  goes through as the person, shown under their own name in Seerr.
- **Discover:** a linked account's lists show within a few seconds. The "On Server" badges match what's in the
  library.
- **Don't test on titles people are in the middle of.** Hardcover, AniList and MAL progress is real. Use a title the
  account owner has OK'd, and put it back afterwards.
