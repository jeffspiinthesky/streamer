# youtube-udp-streamer

A systemd template service that resolves a YouTube live stream with
[yt-dlp](https://github.com/yt-dlp/yt-dlp) and relays it to a UDP
(multicast) address with ffmpeg — for feeding a live source into something
like [ffplayout](https://github.com/ffplayout/ffplayout).

Run any number of named instances, each configured by its own file, the
same way `openvpn@<name>` or `wg-quick@<name>` work:

```
systemctl start streamer@eagles
```

reads `/etc/streamer/eagles.conf`.

## Why it restarts itself

yt-dlp's resolved googlevideo URLs are signed and expire after a few
hours, and ffmpeg has no way to refresh them mid-stream. The service is
built around that: `Restart=always` means it's expected to exit
periodically and come straight back up with a freshly resolved URL,
rather than trying to keep a single connection alive forever.

## Install

```
sudo install -o root -g root -m 755 youtube-udp-streamer /usr/local/bin/youtube-udp-streamer
sudo install -o root -g root -m 644 systemd/streamer@.service /etc/systemd/system/streamer@.service
sudo mkdir -p /etc/streamer
sudo systemctl daemon-reload
```

Then, for each stream you want to relay, copy `etc/streamer/example.conf`
to `/etc/streamer/<name>.conf` and fill it in (see the comments in that
file for the required fields — a YouTube URL, a UDP destination, and the
yt-dlp video/audio format codes for that stream, found with
`yt-dlp -F <url>`), then make sure it's world-readable:

```
sudo cp etc/streamer/example.conf /etc/streamer/<name>.conf
sudo chmod 644 /etc/streamer/<name>.conf
```

The service runs as an unprivileged user (`pi` by default — see
Requirements below), not root, so a config file created with a plain
`sudo cp`/`sudo nano` (root:root, mode 600 or 640) won't be readable by
it and the service will crash-loop with "Config not found or not
readable" in the journal. There's nothing secret in these files — just
a public stream URL, a UDP target, and format codes — so world-readable
is the simplest fix, not just a workaround.

```
sudo systemctl start streamer@<name>
sudo systemctl enable streamer@<name>   # survive reboots
journalctl -u streamer@<name> -f        # watch it
```

## Requirements

- `yt-dlp` (path configurable per-instance via `YTDLP_BIN`, default
  `$HOME/src/yt-dlp/yt-dlp`)
- `ffmpeg`
- A user for the service to run as (the shipped unit uses `pi`; change
  `User=`/`Group=` in the unit file to suit your system)
