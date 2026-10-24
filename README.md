# Amby Notes: run your own sync server

[Русская версия](README.ru.md)

This repository holds everything you need to run the **sync server** for [Amby Notes](https://github.com/MrDarkOm/amby-notes) on your own machine: a Docker Compose file that uses a ready-made image, a Caddy config that gets the HTTPS certificate for you, and these instructions. It contains no server code; the source lives in [`amby-notes/sync/`](https://github.com/MrDarkOm/amby-notes/tree/main/sync) and the image `ghcr.io/mrdarkom/ambynotes-sync` is built from it.

## What the server can and cannot see

Your devices encrypt everything before sending it. The server stores your notes **encrypted** and is a blind store: the keys never reach it. It **cannot read**:

- the text of your notes and attachments,
- the names of your files and folders, and the folder structure,
- the names of your devices and vaults,
- your password, your recovery code and any key.

It **does see**, and this cannot be hidden: the user names, how many files you have and roughly how big they are, when they changed, which device changed them, and from which IP addresses the requests come.

Because the server cannot read your data, it cannot recover it: if you lose both your password and your recovery code, nobody can bring the data on the server back. The files on your own devices stay as they are.

## What you need

- A machine with [Docker](https://docs.docker.com/engine/install/) and the Compose plugin (`docker compose`).
- A DNS name (for example `sync.example.org`) that points at this machine.
- Ports **80 and 443** open from the internet. Caddy uses them to get and renew the HTTPS certificate.

## Quick start

```bash
git clone https://github.com/MrDarkOm/AmbyNotes-SelfHost
cd AmbyNotes-SelfHost/sync
cp .env.example .env
```

Open `.env` and set `AMBY_SYNC_DOMAIN` to your DNS name. Then:

```bash
docker compose up -d
curl https://sync.example.org/v1/info    # should answer with a short piece of JSON
```

In the app, enter `https://sync.example.org` as the server address.

The server speaks plain HTTP only inside the Compose network; port 8080 is not published. Do not publish it: passwords and notes must travel over HTTPS.

## Add users

```bash
docker compose exec amby-sync amby-sync-server user add alice
```

It prints a **one-time code** valid for 24 hours. In the app, enter the server address, the user name and the code, then choose a password. The app creates the keys and shows a **recovery code: save it**. Running `user add` again for a name that has not set a password yet gives a new code and burns the old one.

```bash
docker compose exec amby-sync amby-sync-server user list
docker compose exec amby-sync amby-sync-server user remove alice --yes   # deletes everything of alice, for good
```

User names are 1 to 64 characters from `a-z 0-9 . _ -`, starting and ending with a letter or a digit.

## Settings (`.env`)

| Variable                         | Default  | Meaning                                                      |
| -------------------------------- | -------- | ------------------------------------------------------------ |
| `AMBY_SYNC_DOMAIN`               | required | your DNS name, without `https://`                            |
| `AMBY_SYNC_VERSION`              | `latest` | image version to run, see [Pin a version](#pin-a-version)    |
| `AMBY_SYNC_QUOTA_BYTES_PER_USER` | no limit | most encrypted bytes one user may keep                       |

Other server settings (version retention, upload limits, …) are listed in the [server README](https://github.com/MrDarkOm/amby-notes/blob/main/sync/server/README.md#settings); add them under `environment:` of `amby-sync` in `docker-compose.yml` if you need them.

## Back up

Everything lives in one Docker volume. This makes a consistent copy while the server is running:

```bash
docker compose exec amby-sync amby-sync-server backup /data/backups/today
docker cp "$(docker compose ps -q amby-sync)":/data/backups/today ./amby-backup-today
docker compose exec amby-sync rm -r /data/backups/today
```

The copy (`amby-sync.db` and, if you have attachments, `blobs/`) is as unreadable as the live data. Keep it on another machine too, and remember that **every device also holds a full copy of your notes**.

## Restore

Run these in the `sync` folder, with the backup in `./amby-backup-today`:

```bash
docker compose down
docker compose run --rm --no-deps --entrypoint sh \
  -v "$PWD/amby-backup-today:/restore:ro" amby-sync -c \
  'rm -rf /data/amby-sync.db /data/amby-sync.db-wal /data/amby-sync.db-shm /data/blobs && cp -r /restore/. /data/'
docker compose up -d
```

The `-wal` and `-shm` files belong to the database being replaced, so they must go. A restored server is older than what your devices have seen; they notice that it went back in time and do not delete anything to match it.

## Update

Make a backup first (see above), then:

```bash
cd AmbyNotes-SelfHost && git pull     # also updates this kit
cd sync
docker compose pull && docker compose up -d
```

The database updates itself on the first start of the new version. A server refuses to open a database written by a newer version, so going back to an older version needs the backup.

## Pin a version

By default the server runs `latest`. To update only when you choose, put a number in `.env`:

```
AMBY_SYNC_VERSION=0.1.0
```

and run `docker compose up -d`. The available versions are listed under *Packages* on the [repository page](https://github.com/MrDarkOm/AmbyNotes-SelfHost). Caddy is pinned to its major version (`caddy:2`).

## Good to know

- `docker compose ps` shows the server as *healthy* when `/v1/info` answers. `docker compose logs amby-sync` shows the log; it never contains tokens, keys, enrolment codes or request bodies.
- A server whose data was erased gets a new random identity, so a device that sees it stops instead of deleting its files to match an empty server.

## Source and license

The server source is in [`MrDarkOm/amby-notes`, folder `sync/`](https://github.com/MrDarkOm/amby-notes/tree/main/sync). Every published image records the exact commit it was built from in the label `org.opencontainers.image.revision`.

Licensed under the [AGPL-3.0](LICENSE).
