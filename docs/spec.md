# music-migrate — specification

As of 2026-11-22.

## Goal

A free, open-source (Apache-2.0) command-line tool written in Rust that moves a music library from Yandex Music and Spotify to YouTube Music (YTM): liked tracks, playlists, saved albums (as whole albums) and followed artists.

## Decisions

1. **Three-step pipeline with resume.** `export` writes each source library to neutral JSON (which doubles as a backup), `match` finds YTM counterparts and writes a match cache plus reports, `apply` writes accepted matches to YTM and resumes after an interruption without repeating finished work.
2. **Spotify source = the account data download** ("Privacy → Download your data": `YourLibrary.json`, `Playlist*.json`). No Spotify Web API, no developer app, no Premium.
3. **Yandex Music source = the unofficial API** at `api.music.yandex.net`. `login yandex` runs the Yandex OAuth device-code flow (shows a code and `ya.ru/device`); a token can also be passed with `--token` or `MUSIC_MIGRATE_YANDEX_TOKEN`. The token is stored in the OS keyring.
4. **YTM auth = pasted browser headers.** `login ytmusic` reads a request to `music.youtube.com` copied from DevTools ("Copy as cURL" or raw `name: value` headers), keeps the cookie, `x-goog-authuser`, `user-agent` and `x-youtube-client-version`, and signs each call with `SAPISIDHASH`. Stored in the OS keyring. Cookies are bound to the public IP of the browser, so the tool must run where the browser runs.
5. **Liked tracks become real YTM likes**, deduplicated across both sources, applied oldest to newest so the newest end up on top. YTM keeps about 5,000 liked songs; whatever exceeds `LIKES_CAP = 5000` goes to a private playlist `Liked songs (overflow)`.
6. **Matching uses a score in [0, 1]** built from normalized and transliterated title and artist, duration (Yandex only; the Spotify export has none) and album. Score ≥ 0.85 is accepted automatically; 0.60–0.85 goes to `review.csv` with the top-3 candidates for the user to choose; below 0.60 goes to `not_found.csv`. `apply` uses only accepted matches.
7. **Playlists: all of them** (own and liked/followed) are copied as **PRIVATE** YTM playlists with the same title and track order; the description names the source. `--only <title>` / `--skip <title>` (repeatable) filter them. A re-run appends missing tracks to the playlist created earlier instead of creating a duplicate.
8. **Saved albums are added to the YTM library whole** (`like/like` on the album's `OLAK5uy_…` audio playlist id). An album with no confident match is only reported (`not_found.csv`/`review.csv`), never replaced with a track playlist. **Followed artists** are subscribed to.
9. **Distribution:** GitHub Releases with binaries for macOS (arm64, x86_64), Linux (x86_64, arm64) and Windows (x86_64) built by `dist` (cargo-dist) in GitHub Actions, plus shell/PowerShell installers, `cargo install music-migrate`, and a Homebrew formula published to `DimazzzZ/homebrew-tap`.
10. **Language:** all program output, docs, code and commits are in English.
11. **State lives in a workspace directory** (default `./music-migrate-data`, `--data-dir` to override) as JSON and CSV files that a person can read and edit. Every write is atomic (temp file + rename). Secrets never go there.

## Commands

```
music-migrate login yandex [--token <t>]
music-migrate login ytmusic [--file <path>]      # otherwise reads stdin
music-migrate logout <yandex|ytmusic>
music-migrate export yandex
music-migrate export spotify <path-to-unzipped-download>
music-migrate match [--accept-review]
music-migrate apply [--dry-run] [--only <title>]... [--skip <title>]... [--delay-ms <n>]
music-migrate status
```

## Workspace layout

```
music-migrate-data/
  export/yandex.json        # Library
  export/spotify.json       # Library
  matches.json              # source key -> MatchResult (cache; re-running match skips decided keys)
  review.csv                # doubtful matches; the user fills the `choice` column
  not_found.csv             # nothing usable found
  state.json                # apply progress: finished operations and created playlist ids
```

## Non-goals

YTM → anything; podcasts; uploading local files; a GUI; Spotify Web API; Yandex → Spotify.
