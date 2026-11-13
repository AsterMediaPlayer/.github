# Aster Media Player

**A free media player for your own Jellyfin, Emby and Plex servers**, on Apple TV, iPhone, iPad, Mac, Android, Android TV, Fire TV, Windows and Linux.

Aster is built for watching your own library on a TV, with the touches that make it feel like more than a file browser:

- **Disc menus.** DVD and Blu-ray style menus for your films and shows, with backgrounds, music, chapters and special features, shared through a community catalogue.
- **Channels.** Your video playlists, turned into scheduled TV channels with a programme guide, rotations, blocks and optional breaks with your own bumpers and commercials. Watching a channel leaves your Continue Watching alone.
- **Anime.** A dedicated tab, opening and ending song cards with the song, artist and series, playable openings and endings, and styled ASS/SSA subtitles.
- **Everything else you'd expect:** movies, TV, music, audiobooks and live TV, with theme music and videos on title pages.

Aster is free on every platform and always will be. It was built with extensive help from AI, and that's part of the reason it will never cost anything.

🌐 [astermediaplayer.com](https://astermediaplayer.com) · 🛟 [Support](https://astermediaplayer.com/support.html) · 💬 [Community](https://github.com/orgs/AsterMediaPlayer/discussions)

## The apps

| Platform | Repository | Get it |
|---|---|---|
| Apple TV, iPhone, iPad, Mac | [aster-apple](https://github.com/AsterMediaPlayer/aster-apple) | TestFlight now, App Store coming soon |
| Android, Android TV, Fire TV | [aster-android](https://github.com/AsterMediaPlayer/aster-android) | Amazon Appstore (Fire OS) |
| Windows | [aster-windows](https://github.com/AsterMediaPlayer/aster-windows) | Installer and zip from its Releases |
| Linux, Steam Deck, couch setups | [aster-linux](https://github.com/AsterMediaPlayer/aster-linux) | Coming soon; built for a controller and Steam's Big Picture |

> The app repositories are still private while they're prepared for release. Their source will be published; until then, those links show GitHub's "not found" page.

Each app plays what your server has as it is where the device can, and falls back to the server's conversion where it can't. Apple uses AetherEngine with Apple's own player as a fallback, Android uses Media3 with libass, and Windows and Linux use mpv.

## Server plugins

[**aster-companion-plugins**](https://github.com/AsterMediaPlayer/aster-companion-plugins): optional Jellyfin plugins, installed from one plugin repository. Aster works without them; each adds something.

| Plugin | What it adds |
|---|---|
| **Aster Disc** | Disc menus: an editor, and a catalogue to share and install them from. |
| **Aster OP-ED** | Anime openings and endings matched to your shows, with song details from AnimeThemes and copies every client can play. |
| **Aster Channels** | A Channels page in the server dashboard to make, edit and renumber channels. For Jellyfin and Emby. |
| **Aster Aurora** | Apple TV style touches for Jellyfin's own web interface: a home banner, trailer backgrounds and more. |
| **Aster Discovery and Tracking** | Each person's Discover page, scrobbling to their own Trakt, Simkl, AniList, MyAnimeList, Last.fm, ListenBrainz and Hardcover accounts, and requests through Seerr. |

To install, add this in Jellyfin under **Dashboard ▸ Plugins ▸ Repositories**:

```
https://raw.githubusercontent.com/AsterMediaPlayer/aster-companion-plugins/main/manifest.json
```

## Tools and sites

| Repository | What it is |
|---|---|
| [**discs**](https://github.com/AsterMediaPlayer/discs) | [discs.astermediaplayer.com](https://discs.astermediaplayer.com): the disc menu catalogue's website. |
| [**channel-builder**](https://github.com/AsterMediaPlayer/channel-builder) | [channels.astermediaplayer.com](https://channels.astermediaplayer.com): a page that writes the settings that turn a Plex playlist into an Aster channel. It runs entirely in your browser. |
| [**aster-skin**](https://github.com/AsterMediaPlayer/aster-skin) | A CSS theme that gives Jellyfin's web interface Aster's look. |

## Community

Aster is a small project, built in spare time, and the people using it shape where it goes. [**GitHub Discussions**](https://github.com/orgs/AsterMediaPlayer/discussions) is where that happens:

- 👋 **[Say hello](https://github.com/orgs/AsterMediaPlayer/discussions/8).** Tell us what you watch on Aster and what you run it on.
- 🛟 **Get help.** Each platform has its own Help category, plus one for the server plugins and web skin, so answers can be marked and found later.
- 🎬 **[Show and tell](https://github.com/orgs/AsterMediaPlayer/discussions/categories/show-and-tell).** Share your disc menus, channel lineups, home screens and setups.
- 💡 **[Ideas](https://github.com/orgs/AsterMediaPlayer/discussions/categories/ideas)** and **[Polls](https://github.com/orgs/AsterMediaPlayer/discussions/categories/polls).** Pitch features before they become issues, and [vote on what Aster builds next](https://github.com/orgs/AsterMediaPlayer/discussions/9).
- 📣 **[Announcements](https://github.com/orgs/AsterMediaPlayer/discussions/categories/announcements).** New releases, TestFlight builds and plugin updates.

New here? Start with the [welcome post](https://github.com/orgs/AsterMediaPlayer/discussions/1).

## Help and feedback

- **Questions and setup:** the [support page](https://astermediaplayer.com/support.html), or ask in [Discussions](https://github.com/orgs/AsterMediaPlayer/discussions).
- **Bugs:** open an issue in the repository for that app or plugin. Each has templates that ask for what's useful: your device, server and version.
- **Security problems:** please report privately, through **Security ▸ Report a vulnerability** on the affected repository, not in a public issue.

Specific reports help most: the device, the server and its version, and the file if it's a playback problem.

Thanks for giving Aster a try.

---

<sub>Aster isn't affiliated with or endorsed by Jellyfin, Emby or Plex.</sub>
