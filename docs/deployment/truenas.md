# TrueNAS SCALE deployment

Run Engram as a TrueNAS SCALE custom app. TrueNAS manages the container
lifecycle (start, stop, update) from its Apps UI.

> **Replaces the TrueNAS MakeMKV app.** Engram takes over `/dev/sr0` (and the
> associated `/dev/sg*` device). Stop and remove the TrueNAS MakeMKV app before
> starting Engram, or the two will compete for the drive.

## Requirements

- TrueNAS SCALE with the Apps system enabled.
- An optical drive passed through to TrueNAS (e.g. `/dev/sr0`).
- A MakeMKV license key or the current beta key.
- A TMDB Read Access Token (v4 auth — the long `eyJ…` JWT from
  [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api)).

## Create the custom app

In the TrueNAS UI: **Apps → Discover Apps → Custom App**.

Paste the following YAML, filling in your values where noted:

```yaml
services:
  engram:
    image: ghcr.io/jsakkos/engram:latest
    container_name: engram
    restart: unless-stopped
    ports:
      - "8010:8000"          # Change 8010 to any free port on your TrueNAS host
    environment:
      PUID: "568"            # TrueNAS default apps user; adjust if different
      PGID: "568"
      TZ: "America/Chicago"  # Your timezone
      MAKEMKV_APP_KEY: ""    # Your MakeMKV license or beta key
      MAKEMKV_VERSION: "latest"
      TMDB_API_KEY: ""       # Your TMDB Read Access Token (eyJ… JWT)
      # LibreDrive SDF fix — see below if disc scans time out after ~10 minutes:
      # SDF_STOP: "ASUS_BW-16D1HT_3.11_212012011759_KLTO9CF4939"
    devices:
      - /dev/sr0:/dev/sr0    # Optical drive block device
      - /dev/sg4:/dev/sg4    # Blu-ray generic SCSI device — find with `ls -l /dev/sg*`
    volumes:
      - /mnt/tank/appdata/engram:/config:rw   # Persistent state: DB, MakeMKV, logs
      - /mnt/tank/media/staging:/staging:rw   # In-progress rips
      - /mnt/tank/media/movies:/media/movies:rw
      - /mnt/tank/media/tv:/media/tv:rw
```

> **`${VAR}` substitution does not work** in the TrueNAS custom app UI. Enter your
> MakeMKV key and TMDB token directly in the environment fields — do not use a
> `.env` file alongside the compose config.

### Finding the /dev/sg* device number

Blu-ray drives require both the block device (`/dev/sr0`) and the generic SCSI
interface. Find the `sg` number on your TrueNAS host:

```bash
ls -l /dev/sg*
```

The Blu-ray drive is typically the one with the highest number. If you're
unsure, check `dmesg | grep -i "sg\|sr"` after inserting a disc.

## Create required directories

From the TrueNAS shell or an SSH session:

```bash
mkdir -p /mnt/tank/appdata/engram
mkdir -p /mnt/tank/media/staging
mkdir -p /mnt/tank/media/movies
mkdir -p /mnt/tank/media/tv
chown -R apps:apps /mnt/tank/appdata/engram /mnt/tank/media/staging
```

## Deploy

Click **Save** in the TrueNAS Apps UI. TrueNAS pulls the image and starts the
container. First start is slow: MakeMKV compiles from source (~5 minutes) and
the speech-recognition model downloads on the first episode match (~465 MB).
Watch progress:

```bash
docker logs -f engram
```

Expected: `==> MakeMKV X.XX.X installed at /config/makemkv.` followed by the
uvicorn startup lines.

## Run the setup wizard

Open `http://<truenas-ip>:8010` and configure:

- **Staging directory** → `/staging`
- **Movies library** → `/media/movies`
- **TV library** → `/media/tv`
- **TMDB Read Access Token** → your `eyJ…` JWT

> **Wizard navigation:** Click the numbered section tabs at the top to move
> between pages. The Next button may not always advance the wizard.

## LibreDrive SDF hang fix

If disc scans time out after ~10 minutes, you're hitting a MakeMKV SDF lookup
hang. See the [Docker deployment docs](docker.md#libredrive--sdf-scan-hang) for
the full fix. For TrueNAS:

1. Find your drive model string and add `SDF_STOP` to the custom app environment.
2. Copy `sdf.bin` to `/mnt/tank/appdata/engram/data/sdf.bin` and restart the
   container so MakeMKV ingests it.

## Troubleshooting

- **UI is not reachable** — confirm the port mapping and that no other app uses
  the same host port. TrueNAS docker handles host binding; the "Allow LAN access"
  setting in the Engram wizard has no effect when running behind TrueNAS.
- **Drive not detected** — verify `/dev/sr0` and `/dev/sg*` paths in `devices:`.
  Run `docker exec engram ls /dev/sr0 /dev/sg4` to confirm they're visible inside
  the container.
- **Library files are root-owned** — set `PUID`/`PGID` to match the TrueNAS apps
  user (`568` by default).
- **Subtitle cache path error in logs** — the default `~/.engram/cache` path
  fails inside the container. Set the subtitle cache path to `/config/cache` in
  the wizard Preferences page.
