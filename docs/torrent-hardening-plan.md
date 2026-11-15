# Torrent stack hardening plan

This plan covers the five hardening changes and the cleanup from the source note `research/2026-10-15 Torrent stack hardening.md` in the Claude vault. It exists for issue #21. Writing it changed nothing on the stack. Applying the changes is separate work, filed after Julian approves this plan.

## Background

On 2026-10-15, 18 Windows PE executables were found in `/mnt/docker/media/downloads` on docker-manager-00 and deleted, about 20 GiB in total. They were `.exe` and `.scr` files padded to 1.0 to 1.36 GB, named like episodes of The Boys S05, Rick and Morty S09, Invincible S04, The Pitt S02 and Silo S03. Sonarr grabbed them from public indexers (LimeTorrents, TorrentDownload, Uindex, TorrentGalaxyClone) between April and July 2026. It never imported them, and nobody ran them.

Sonarr only warns "Caution: Found executable file" by default. The download fails only when an indexer has "Fail Downloads (Executables and Potentially Dangerous)" ticked. That option is off by default and Prowlarr does not sync it.

## How to read this plan

Each numbered section has the same parts: who applies the change, the exact setting or file, the steps, how to check it worked, the risk, and whether it needs a decision from Julian.

| # | Change | Who applies it | Decision needed |
|---|---|---|---|
| 1 | Tick "Fail Downloads" on every Sonarr and Radarr indexer | Julian, by hand | No |
| 2 | Prune public indexers or raise minimum seeders | Julian, by hand | Yes |
| 3 | Keep the downloads folder off SMB, show file extensions on Windows | Julian, by hand | No |
| 4 | qBittorrent "Excluded file names" | Julian, by hand, later | Yes, when to start |
| 5 | Magic-byte check or Cleanuparr malware blocker (optional) | Agent, after a decision | Yes |
| 6 | Remove the six leftover qBittorrent torrents | Julian or an agent | Yes, one yes/no |

Recommended order is the note's order, with section 6 after section 1. Section 6 may trigger a re-search in Sonarr, and section 1 should be in place before that happens.

### Facts to confirm first

The source note recorded versions on 2026-10-15: qBittorrent 5.1.4, Sonarr 4.0.17.2952, Radarr 6.0.4.10291 and Prowlarr 2.3.0.5236. Three things in this repo suggest those numbers may be stale.

- `media-stack-gluetun.yml` runs `lscr.io/linuxserver/qbittorrent:latest`, `sonarr:latest`, `radarr:latest` and `prowlarr:latest`, so every image pull moves them.
- The qBittorrent exporter added in commit 11e409c authenticates with a WebUI API key, and the README section "Create the qBittorrent API key" says that needs qBittorrent 5.2.0 or later. If the exporter works, the container is no longer on 5.1.4 and is on 5.2.0 or later. That puts it in or just past the 5.2.0 through 5.2.3 range from section 4, so the exact version matters.
- The README says the compose file pins versions, but the arr and qBittorrent lines do not.

Before section 4, read the version from each app's UI (System, then Status; qBittorrent's `GET /api/v2/app/version` returns it). From the host, the LinuxServer images also carry the running app version in a label:

```sh
for c in qbittorrent sonarr radarr prowlarr; do
  docker inspect "$c" --format '{{.Name}} {{index .Config.Labels "build_version"}}'
done
```

### A check that works for every section

Executables that landed in the downloads folder show up with this command on docker-manager-00:

```sh
find /mnt/docker/media/downloads -type f \( -iname '*.exe' -o -iname '*.scr' -o -iname '*.lnk' -o -iname '*.bat' -o -iname '*.cmd' -o -iname '*.msi' -o -iname '*.vbs' -o -iname '*.ps1' \)
```

Empty output is the goal. Re-run it every few weeks. A hit is a file to look at, not proof of a fake: it can be an old leftover or an installer bundled in a real release. This check is extension-based, so a PE file named `.mkv` does not show up. Section 5 covers that.

### Reading API keys without printing them

Several checks below call the Sonarr, Radarr or Prowlarr API. Each app keeps its key in its `config.xml`, at `/mnt/docker/sonarr/config.xml`, `/mnt/docker/radarr/config.xml` and `/mnt/docker/prowlarr/config.xml`. Read the key into a variable and send it through a header file on stdin, so it never appears in the command line or the terminal:

```sh
KEY=$(grep -oP '(?<=<ApiKey>)[^<]+' /mnt/docker/sonarr/config.xml)
printf 'X-Api-Key: %s\n' "$KEY" | curl -fsS -H @- http://localhost:8989/api/v3/indexer
```

All `curl` calls here use `-f`, so an HTTP error such as a bad key prints an error instead of an empty result. The qBittorrent WebUI key is `QBITTORRENT_API_KEY` in `stack.env` on the host. The registry in agent-config lists it by name and location, and this plan adds no new credential.

## 1. Tick "Fail Downloads" on every Sonarr and Radarr indexer

**Who applies it.** Julian, by hand. It is a setting inside the Sonarr and Radarr UI, and Prowlarr does not sync it, so there is no file in this repo to edit.

**Setting.** In Sonarr: Settings, then Indexers, open an indexer, click Show Advanced, and tick "Fail Downloads (Executables and Potentially Dangerous)". Repeat for every indexer in Sonarr, then repeat all of it in Radarr (port 7878). Sonarr 4.0.12 added the option and 4.0.13 added `.scr`, so Sonarr 4.0.17 has both. The source note says the same option exists in Radarr 6.0.4. Confirm it appears in the Radarr form before counting on it.

**Why first.** It costs no compute and would have failed all 18 single-file fakes. A failed download is blocklisted, and Sonarr re-searches it when Redownload Failed is on, which is the default.

**Steps.**

1. Open `http://docker-manager-00:8989`, then Settings, Indexers, and switch Show Advanced on.
2. For each indexer, open it, tick the option, and save.
3. Do the same in Radarr.
4. Note any indexer you add later. A new indexer starts with the option off, so this step recurs.

**Check it worked.** List the field for every indexer through the API. The field name below is my best guess from the label, so the filter matches any field whose name contains "fail", case-insensitive:

```sh
KEY=$(grep -oP '(?<=<ApiKey>)[^<]+' /mnt/docker/sonarr/config.xml)
printf 'X-Api-Key: %s\n' "$KEY" | curl -fsS -H @- http://localhost:8989/api/v3/indexer \
  | jq '.[] | {name, fail: [.fields[] | select(.name|test("fail";"i")) | {name, value}]}'
```

Every indexer should show the option ticked in `value`. If `fail` is empty for all of them, the field has another name, so read one full indexer object and adjust the filter. Run the same command with port 7878 and the Radarr key for Radarr. After the next Prowlarr sync (or a manual "Sync App Indexers" in Prowlarr), run it again to confirm the sync did not reset the box.

**Risk.** Low. The worst case is a false positive: a legitimate release that carries a flagged file (for example a bundled `.exe` installer in a game or software torrent) gets failed and blocklisted. For TV and movie indexers that is rare, and Sonarr re-searches. The option does not clean anything already in the queue, and the box can be unticked. One gap remains: as the note describes it, Sonarr flags files by extension. A PE file named `.mkv` would pass this check, which is an argument for section 5.

**Decision from Julian.** None.

## 2. Prune the public indexers

**Who applies it.** Julian, by hand, after he decides. Prowlarr holds 17 public indexers, all enabled at priority 25 with a minimum seeders value of 1.

**Setting.** In Prowlarr at `http://docker-manager-00:9696`, Indexers, open an indexer to change "Enable", "Priority" (1 to 50, lower wins) or "Minimum Seeders". The seeders field may sit on the Prowlarr indexer, on the app sync profile, or on the copy in Sonarr and Radarr. Find where it lives in the UI before editing, and note that a value changed only in Sonarr can be overwritten by the next sync.

**Options.**

- A. Disable LimeTorrents, TorrentDownload and Uindex. These served the identified fakes, and Uindex has public 2026 reports of the same scam (https://github.com/ehulle117/tars/issues/107 and #114). This is the strongest cut and loses the most real releases.
- B. Keep all three and raise the minimum seeders on them. A fake with one seeder passes today. Seeder counts on public trackers are self-reported, so a determined fake can inflate them, and this is a weak filter by itself.
- C. Keep all indexers enabled but set a lower priority number (higher rank) on indexers that have not served fakes, so Sonarr prefers them when several offer the same release.

My recommendation is A for Uindex alone (it has outside reports), B and C for the other two, then watch the Sonarr history for a few weeks. That recommendation is a lean, not a verdict. Julian knows which of these indexers find releases he cannot get elsewhere.

**Check it worked.** Indexers disabled or reprioritized show in the Prowlarr indexers list. For a command check:

```sh
KEY=$(grep -oP '(?<=<ApiKey>)[^<]+' /mnt/docker/prowlarr/config.xml)
printf 'X-Api-Key: %s\n' "$KEY" | curl -fsS -H @- http://localhost:9696/api/v1/indexer \
  | jq -r '.[] | [.name, .enable, .priority] | @tsv'
```

The chosen indexers should show `false` in the second column or the new priority in the third. Confirm in Sonarr's indexer list that the disabled ones disappeared after the sync.

**Risk.** Medium for content coverage. Pruning also loses real releases, so a show that only one pruned indexer carried stops finding new episodes. It does not remove files. Reversible by re-enabling.

**Decision from Julian.** Yes. Which indexers to disable, and whether to change seeders or priority instead.

## 3. Keep the downloads folder off SMB and show file extensions

**Who applies it.** Julian, by hand. Nothing in this repo defines an SMB share, so the share lives on the host or on another machine that exports `/mnt/docker/media`. I could not tell from the repo whether the downloads folder is shared today, so the first step is to find out.

**Settings.**

- On whichever host exports the share, limit the share to the library folders (`/mnt/docker/media/movies` and `/mnt/docker/media/tv`), not `/mnt/docker/media` itself, because `downloads` sits under it.
- On every Windows machine that opens the share, turn on File name extensions in Explorer (View, Show, File name extensions), so `Episode.S05E01.mkv.exe` shows its real extension.

**Steps.**

1. Find the export. On docker-manager-00, run `sudo testparm -s 2>/dev/null | grep -E '^\[|path'` for Samba, and `exportfs -v` for NFS. If the media share is on TrueNAS, check it under Shares, SMB.
2. If a share path is `/mnt/docker/media` or `downloads`, change it to the library folders.
3. On each Windows client, enable File name extensions.

**Check it worked.** The `testparm` output no longer lists a path that contains `downloads`. From a Windows client, opening `\\<host>\` shows only library shares, and an attempt to browse to a `downloads` folder fails. A `.txt` file renamed `test.txt.exe` in Explorer shows the `.exe`.

**Risk.** Low to medium. Anything that reads `/mnt/docker/media/downloads` over SMB (a Windows tool, a backup job) loses access. Confirm there is none before narrowing the share. Showing extensions has no downside.

**Decision from Julian.** None, unless step 1 shows a consumer of the downloads share he wants to keep.

## 4. Add qBittorrent "Excluded file names", later

**Who applies it.** Julian, by hand, when the preconditions below hold. This is not for now.

**Setting.** qBittorrent Options, Downloads, "Excluded file names", or the preference `excluded_file_names_enabled` with `excluded_file_names` in the WebUI API. The conf keys are `ExcludedFileNamesEnabled` and `Session\ExcludedFileNames`, inside `/mnt/docker/qbittorrent/qBittorrent/qBittorrent.conf` (the LinuxServer image keeps the config at that path under the `qb_config` volume). Use the UI, not a file edit, because qBittorrent rewrites its conf on exit. The value from the note is:

```
*.exe, *.scr, *.lnk, *.bat, *.cmd, *.msi, *.vbs, *.ps1, *.zipx
```

**Preconditions. Do not apply before all of these are true.**

1. Sonarr is on 5.x, where a torrent whose files are all excluded is failed. The note says Sonarr 5.0.0.2608 is a pre-release that does this. On Sonarr 4, such a torrent completes empty, Sonarr only warns, and the episode gets stuck with no blocklist and no re-search. That is worse than today.
2. Radarr fails an all-excluded torrent too. This is unconfirmed, because the note covers Sonarr only. Check Radarr's behaviour before relying on it.
3. qBittorrent is 4.5.0 or later and outside 5.2.0 through 5.2.3. Those versions ignore the setting for WebUI adds, and Sonarr adds torrents through the WebUI API, so the setting would do nothing. The fix is targeted for 5.3.

Because the compose file runs `:latest`, precondition 3 can change without anyone editing the repo, and an image pull that lands on 5.2.0 through 5.2.3 would silently turn this setting into a no-op, and the README suggests the container may already be there. If Julian wants this change, pin the qBittorrent tag in `media-stack-gluetun.yml` first (a small agent task). Pinning is a decision about how the stack updates, so I am raising it rather than doing it.

**Check it worked.** Not applicable until applied. After applying, add a throwaway test torrent containing only an `.exe` in the qBittorrent UI and confirm the file is skipped (priority "Do not download"). To test Sonarr's reaction, the torrent must come from a Sonarr grab, because a torrent added by hand has no Sonarr history and Sonarr marks nothing failed. On Sonarr 5, confirm Sonarr then marks that download failed. Use a harmless test file you create yourself, not a real fake.

**Risk.** High if applied early (stuck episodes, as above). Low once the preconditions hold. Excluding `.msi`, `.zipx` and `.scr` also stops legitimate software torrents, which are not on this stack.

**Decision from Julian.** Yes. Whether to wait for stable Sonarr 5, and whether to pin the qBittorrent tag now.

## 5. Optional: magic-byte check or Cleanuparr malware blocker

**Who applies it.** An agent, after Julian picks one of the two options. This is optional and I recommend skipping it until sections 1 to 3 have run for a while.

**Option A. On-finish hook.** In qBittorrent Options, Downloads, "Run external program on torrent finished", call a script that reads the first 1 MiB of each file with `file` and flags any file whose type is a Windows or Linux executable. Set it in the UI. The conf key names for this setting differ between qBittorrent versions and I did not verify them, so read them from a live `qBittorrent.conf` after saving the setting once. The script would live under `/mnt/docker/qbittorrent/scripts/`, mounted inside the container at `/config/scripts/`. Details to settle before building:

- The LinuxServer qBittorrent image may not ship `file`. Check with `docker exec qbittorrent which file`. If it is missing, either install it through a LinuxServer mod or use a short Python check of the first bytes (`MZ` for PE, `\x7fELF` for ELF).
- What the hook does on a hit. Logging alone leaves the torrent in the queue. Making Sonarr blocklist and re-search needs an API call to Sonarr, which means the script needs the Sonarr key, so it holds a credential. That adds an entry to the credential registry.
- The hook catches what section 1 cannot, such as a PE renamed to `.mkv`.

**Option B. Cleanuparr.** A separate service with a malware blocker (https://cleanuparr.github.io/docs/configuration/malware-blocker/). It would blocklist and remove bad downloads without custom scripting. The note names it but I did not verify what it matches on (file names or file contents), and whether it catches a renamed PE. Adding a service to the stack is an architecture decision for Julian, plus it needs a new config directory and an entry for its own API keys.

**Check it worked.** For Option A, add a test torrent with a harmless file whose first bytes are `MZ` and whose name ends in `.mkv`, and confirm the hook logs or acts on it. For Option B, the Cleanuparr UI lists the blocked download and Sonarr's History shows it blocklisted.

**Risk.** Low cost either way. Option A runs after the torrent completes and does not hold it back, so a broken script fails silently and the fake stays in the queue. It needs logging and an alert on errors. Option B adds a service and its credentials to maintain.

**Decision from Julian.** Yes. Whether to do this at all, and A or B.

## 6. Remove the six leftover qBittorrent torrents

**Who applies it.** Julian in the UI, or an agent through the qBittorrent API after Julian confirms. The files are already deleted, so removing these entries touches no media.

**What to remove.** Six torrents are still listed in qBittorrent with no files:

- Rick and Morty S09E02, E03, E04, E06 and E07, all SYLiX
- Silo S03E03, HiggsBoson

**Setting.** qBittorrent at `http://docker-manager-00:8080`, torrent list. Filter by `SYLiX` and `HiggsBoson`, select the six, right-click, Remove. Leave "Also remove the content files" ticked or not, since there is nothing to delete. Section 1 should already be applied.

**Steps.**

1. Do section 1 first. Otherwise a re-search can fetch another fake.
2. Check Sonarr's Activity, Queue for these six. If they appear there, remove them from the Queue with "Remove from download client" and "Add to blocklist" both ticked. Sonarr then removes the torrent for you and will not grab that release again.
3. If they are not in Sonarr's queue, remove them in qBittorrent. Then open Sonarr History, find the grab entries for the six episodes and use Mark as Failed. That should blocklist the release, but the behaviour depends on the Sonarr version and the failed-download settings, so open the blocklist (Activity, Blocklist) and confirm each of the six releases is listed. If one is missing, add it from History.
4. Expect Sonarr to search again for the missing episodes. This is the wanted outcome, and a real release replaces the fake.

**Check it worked.** No torrent named like the fakes remains:

```sh
KEY=$(grep '^QBITTORRENT_API_KEY=' stack.env | cut -d= -f2-)
printf 'Authorization: Bearer %s\n' "$KEY" | curl -fsS -H @- http://localhost:8080/api/v2/torrents/info \
  | jq -r '.[].name' | grep -i -E 'SYLiX|HiggsBoson'
```

Run it from the directory that holds `stack.env`. `curl -f` fails loudly on an HTTP error, so an empty result after a successful call means no matching torrent, and `grep` then exits 1. If curl prints an error instead, nothing was checked. If curl reports an auth error, the UI check serves instead: the filter box shows zero rows for each of the two names. In Sonarr, Wanted shows the six episodes as missing or newly grabbed, and Settings, Indexers shows no new warnings.

**Risk.** Low. Without blocklisting, Sonarr can grab the same fake release again, which is why step 2 or 3 matters. The re-search could pull another bad release if section 1 is not in place, which is why it comes first.

**Decision from Julian.** One yes or no, to confirm the six torrents may go. There is also a side question from the source note. The source note says qBittorrent sends RSS and miscellaneous traffic through an HTTP proxy at `192.168.4.64:8888` while torrent traffic stays bound to `tun0` through gluetun. I could not verify that split. Julian needs to say whether he recognizes that proxy. If not, clear it under Options, Connection, Proxy Server (type None) and confirm that torrents still connect through gluetun. The proxy setting lives in the qBittorrent config volume, and a search of this repo finds no `192.168.4.64`.
