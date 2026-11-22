# music-migrate

A free, open-source command-line tool that moves your music library from Yandex Music and Spotify to YouTube Music: liked tracks, playlists, saved albums (added to the library as whole albums) and followed artists.

> Status: early development. Nothing to install yet.

## How it will work

1. `export` reads your library from Yandex Music (through its API) and Spotify (from the account data download, no Premium or developer app needed) into plain JSON files. These files double as a backup.
2. `match` searches YouTube Music for every track, album and artist. Confident matches are accepted; doubtful ones go to `review.csv` for you to check, and anything not found goes to `not_found.csv`.
3. `apply` writes the accepted matches to YouTube Music and can resume after an interruption.

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
