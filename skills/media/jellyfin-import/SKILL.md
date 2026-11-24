---
name: jellyfin-import
description: Land recent torlink downloads in the Jellyfin library, filed under the names Jellyfin reads.
disable-model-invocation: true
argument-hint: "title to import (default: every finished download the library lacks)"
---

# Jellyfin import

Take what `torlink` downloaded, move it into the Jellyfin media store, and name
it the way Jellyfin reads it. Neither library fetches online metadata
(`EnableInternetProviders: false`), so the file name **is** the metadata: a clean
`Title (Year)` is the difference between the film and an unmatched scene
release. Subtitles are the one thing that comes from the network: every film
leaves here with an English and a Russian track.

| | |
|---|---|
| downloads | `~/Downloads/torlink`, torlnk's folder |
| movies | `~/jellyfin/media/movies` — `/media/movies` in the container |
| shows | `~/jellyfin/media/TV shows` — `/media/TV shows` |
| torlink state | `~/.local/share/torlink/{queue,history,seeds}.json` |
| Jellyfin | `http://localhost:8096`, `JELLYFIN_API_KEY` in `~/.pi/agent/env.json` |

`media` is a loop-mounted ext4 image, mounted `:ro` in the container: write on
the host, then have the server rescan. A move crosses filesystems, so check that
`df -h ~/jellyfin/media` covers the largest file before starting.

## Steps

1. **Survey both sides.** Newest download first, every library folder beside it.

   ```bash
   find ~/Downloads/torlink -mindepth 1 -maxdepth 1 -printf '%TF %TH:%TM %10s %f\n' | sort -r
   ls ~/jellyfin/media/movies ~/jellyfin/media/"TV shows"
   ```

   In scope: a download whose title has no library folder. One title at a time,
   newest first, so a run that stops still leaves the library consistent.

2. **Tell finished from in flight.** torlnk preallocates every file to its final
   length and fills it in place, so the length is right from the first second and
   the allocated blocks are what has actually landed:

   ```bash
   stat -c %s FILE        # final length, already correct while downloading
   du -B1 FILE | cut -f1  # bytes really on disk
   ```

   Within a few MB of the length: finished. Short: still filling. The torrent's
   entry appearing in `~/.local/share/torlink/history.json` (`dir`,
   `completedAt`) confirms it, and a `torlnk serve` daemon answers live on
   `curl -s localhost:9161/downloads` (`status`, `progress` 0–1,
   `downloadedBytes`, `eta`) — that daemon knows only its own queue, the TUI in
   tmux (`torlnk attach`) keeps its own.

3. **Wait rather than guess.** Poll every 30–60 s while a file fills, reporting
   the rate and ETA. Keep waiting while the bytes grow. `paused`, `failed`, or no
   growth across two polls: say so and leave that folder alone. A partial file in
   the library is a film that breaks halfway; importing it later costs nothing.

4. **Resolve the title** from the release name: dots and underscores to spaces,
   cut the tags (`1080p`, `WEBRip`, `x265`, `YTS.MX`, `[TGx]`), keep the year.
   Drop the characters Jellyfin rejects — `: ? " < > | * / \`. Where the library
   already holds the title, copy that folder's spelling; otherwise use the
   canonical one: `How to Lose a Guy in 10 Days (2003)`, not
   `How To Lose A Guy In 10 Days (2003) [1080p]`.

5. **Move and rename**, one `mv` per file, straight to the final name.

   ```bash
   d="$HOME/jellyfin/media/movies/How to Lose a Guy in 10 Days (2003)"
   mkdir -p "$d"
   mv "How.To.Lose.A.Guy.In.10.Days.2003.1080p.BrRip.x264.YIFY.mp4" \
      "$d/How to Lose a Guy in 10 Days (2003).mp4"
   mv "How.To.Lose.A.Guy.In.10.Days.2003.1080p.BrRip.x264.YIFY.srt" \
      "$d/How to Lose a Guy in 10 Days (2003).en.srt"
   mv "WWW.YIFY-TORRENTS.COM.jpg" "$d/poster.jpg"
   ```

   Delete the scene junk on the way: `*.txt`, `.nfo`, samples, duplicate posters.
   A subtitle whose language the file or its name does not state stays untagged,
   `Film (Year).srt`.

6. **Rescan and verify.**

   ```bash
   K=$(jq -r .JELLYFIN_API_KEY ~/.pi/agent/env.json)
   H="Authorization: MediaBrowser Token=\"$K\""
   curl -s -o /dev/null -w '%{http_code}\n' -X POST -H "$H" localhost:8096/Library/Refresh
   curl -s -H "$H" 'localhost:8096/Items?Recursive=true&IncludeItemTypes=Movie&SearchTerm=How%20to%20Lose&Fields=Path,DateCreated'
   ```

   The realtime monitor usually sees the file within seconds; the refresh is the
   nudge. Done when the item's `Path` is the new file and its `Name` and
   `ProductionYear` are right. For an episode, ask for `IncludeItemTypes=Episode`
   and the `S01E01` code instead. Ask for `DateCreated` beside `Path`: an
   unchanged date means the server re-pointed the existing entry at the new file,
   a fresh one means it read the rename as a new file and the watch state is gone.

7. **Subtitles: English and Russian, every film.** Done means `<base>.en.srt` and
   `<base>.ru.srt` sit beside the video; two-letter tags, which Jellyfin reads as
   `eng` and `rus`.

   Harvest the release first: a `.srt` beside the video, or a `Subs/` folder.
   Read the language off the file name (`English.srt`, `2_English.srt`, `…rus…`)
   and rename into place.

   Then fetch what is missing, one API call per language into the server's Open
   Subtitles plugin, which holds the account (`$H` from step 6):

   ```bash
   id=$(curl -s -H "$H" 'localhost:8096/Items?Recursive=true&IncludeItemTypes=Movie&SearchTerm=How%20to%20Lose' | jq -r '.Items[0].Id')
   for lang in rus eng; do
     sid=$(curl -s -H "$H" "localhost:8096/Items/$id/RemoteSearch/Subtitles/$lang" | jq -r '.[0].Id // empty')
     [[ -n $sid ]] && curl -s -o /dev/null -w "$lang %{http_code}\n" -X POST -H "$H" \
       "localhost:8096/Items/$id/RemoteSearch/Subtitles/$sid"
   done
   ```

   `204` is a download. The plugin writes `<base>.rus.srt` beside the video, so
   rename it to `<base>.ru.srt`, rescan, and ask for the subtitle streams: two,
   `eng` and `rus`, is the check. An episode takes the same two calls on its own
   item id.

   ```bash
   curl -s -H "$H" 'localhost:8096/Items?Recursive=true&IncludeItemTypes=Movie&SearchTerm=How%20to%20Lose&Fields=Path,MediaSources' \
     | jq '.Items[0].MediaSources[0].MediaStreams[]|select(.Type=="Subtitle")|{Language,Path}'
   ```

   An empty search result means Open Subtitles has nothing in that language:
   leave the film without the track and name that title in the report. Never
   invent or machine-translate one.

8. **Empty the queue.** `rm -rf` the release folder, `.pad` leftovers included.
   Moving the file ends that torrent's seed, which is expected.

## Names

```text
movies/Film (Year)/
├── Film (Year).mp4              one video per folder
├── Film (Year).en.srt           both tracks, always; .forced / .sdh when they apply
├── Film (Year).ru.srt
└── poster.jpg                   local artwork — providers are off, so it lands here

TV shows/Series (Year)/Season 01/
└── Series S01E01 Episode Title.mkv
```

Two cuts of one film share the folder and differ by label, which Jellyfin reads
as versions of one entry: `Film (Year) - 2160p.mkv` beside `Film (Year) -
1080p.mp4`.

## Gotchas

- The API wants `Authorization: MediaBrowser Token="<key>"`. `?api_key=` and
  `X-Emby-Token` both answer 401.
- The subtitle fetch writes into the media folder, so the container mounts
  `./media:/media` rw. A `:ro` mount fails every download with
  `Read-only file system : '/media/…'` in `~/jellyfin/config/log/`.
- The Open Subtitles account allows about 20 downloads a day, so a whole-library
  backfill stops early; pick the titles that matter or wait out the quota.
- A season pack can be partly here. Move the finished episodes and leave the
  rest; a film waits for the whole file.
