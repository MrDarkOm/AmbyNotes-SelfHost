# Amby Notes: run your own sync server

[Русская версия](README.ru.md)

This repository holds everything you need to run the **sync server** for [Amby Notes](https://github.com/MrDarkOm/amby-notes) on your own machine: a Docker Compose file that uses a ready-made image, a Caddy config that gets the HTTPS certificate for you, and these instructions. It contains no server code; the source lives in [`amby-notes/sync/`](https://github.com/MrDarkOm/amby-notes/tree/dev/sync) and the image `ghcr.io/mrdarkom/ambynotes-sync` is built from it.

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

## Accounts

By default only the accounts you add can sign in:

```bash
docker compose exec amby-sync amby-sync-server user add alice
```

It prints a **one-time code** valid for 24 hours. In the app, enter the server address, the user name and the code, then choose a password. The app creates the keys and shows a **recovery code: save it**. Running `user add` again for a name that has not set a password yet gives a new code and burns the old one.

To let people create an account themselves, set `AMBY_SYNC_OPEN_REGISTRATION=1` in `.env`. Attempts are limited per address, and `AMBY_SYNC_MAX_USERS` caps the number of accounts. Names you reserved with `user add` stay reserved.

```bash
docker compose exec amby-sync amby-sync-server user list
docker compose exec amby-sync amby-sync-server user remove alice --yes   # deletes everything of alice, for good
```

User names are 1 to 64 characters from `a-z 0-9 . _ -`, starting and ending with a letter or a digit; capital letters are turned into small ones.

## Plans and subscriptions

Without a plans file every account may sync and only the settings below limit it. A plans file gives accounts named sets of limits. Put it in the data volume as `/data/plans.json`:

```json
{
  "default_plan": "free",
  "plans": {
    "free": { "title": "Free", "sync": false },
    "notes": {
      "title": "Notes",
      "sync": true,
      "max_total_bytes": 1000000000,
      "max_file_bytes": 30416160,
      "max_history_bytes": 1000000000
    }
  }
}
```

- `sync: false` makes a plan **without sync**: the account exists (sign-in, password, names), the data already on the server can be downloaded, restored and deleted, but nothing new is sent. A missing `sync` means `true`.
- The other limits are optional: `max_spaces`, `max_total_bytes` (current files and unfinished uploads), `max_file_bytes`, `history_days`, `max_history_bytes` (old versions; over it the oldest are removed). Old versions do not count against `max_total_bytes`, so set `max_history_bytes` if accounts you do not know use the server.
- New accounts get `default_plan`. Give an account a plan, with an optional last day (inclusive, UTC), or take it back:

```bash
docker compose exec amby-sync amby-sync-server user plan set alice notes --until 2026-12-31
docker compose exec amby-sync amby-sync-server user plan set alice --default
docker compose exec amby-sync amby-sync-server user plan show alice
```

After the last day the account falls back to `default_plan` by itself; nothing is deleted. The app has a "Get a subscription" button; there is no payment yet, it tells the person to ask you.

The plans file is read at start. Check a new one before you switch, because a broken file stops the server:

```bash
docker compose cp plans.json amby-sync:/data/plans.new.json
docker compose exec amby-sync amby-sync-server plans check /data/plans.new.json
docker compose exec amby-sync mv /data/plans.new.json /data/plans.json
docker compose restart amby-sync
```

## Settings (`.env`)

| Variable                         | Default  | Meaning                                                               |
| -------------------------------- | -------- | --------------------------------------------------------------------- |
| `AMBY_SYNC_DOMAIN`               | required | your DNS name, without `https://`                                     |
| `AMBY_SYNC_VERSION`              | `latest` | image version to run, see [Pin a version](#pin-a-version)             |
| `AMBY_SYNC_OPEN_REGISTRATION`    | `0`      | `1`: accounts can be created from the app without a code              |
| `AMBY_SYNC_MAX_USERS`            | no limit | most accounts open registration may reach                             |
| `AMBY_SYNC_MAX_BLOB_BYTES`       | 512 MiB  | largest encrypted file; `30416160` matches the 30 MB limit of the app |
| `AMBY_SYNC_QUOTA_BYTES_PER_USER` | no limit | most encrypted bytes one user may keep (a ceiling over every plan)    |
| `AMBY_SYNC_MIN_FREE_BYTES`       | 1 GiB    | free disk space kept; below it new uploads and accounts are refused   |
| `AMBY_SYNC_RETAIN_VERSION_DAYS`  | `30`     | days to keep replaced and deleted versions                            |
| `AMBY_SYNC_MAX_LARGE_BODIES`     | `4`      | requests over 1 MiB read at once; more get "busy, try later"          |
| `AMBY_SYNC_BODY_TIMEOUT_SECS`    | `60`     | seconds a request may stop sending before it is refused (0.2.1)       |

`.env.example` explains each one. The complete list, the protocol and the formats are in the [sync format v1](https://github.com/MrDarkOm/amby-notes/blob/dev/docs/sync-format.md).

## Limits that protect the server

- When the disk has less free space than `AMBY_SYNC_MIN_FREE_BYTES`, the server refuses new accounts, new vaults and new uploads; people see that the server is running out of space. Reading, downloading and deleting keep working, so space can be freed.
- Sign-in requests are at most 64 KiB, and at most `AMBY_SYNC_MAX_LARGE_BODIES` large requests are read at once, so a stranger cannot fill the memory of the server.
- Since 0.2.1 a request that stops sending its body is refused after `AMBY_SYNC_BODY_TIMEOUT_SECS`, and one device keeps at most four event streams. Set limits in your proxy too.
- Failed sign-ins are limited per address and per name. Behind the bundled Caddy the server reads the real address from the header Caddy sets (`AMBY_SYNC_TRUST_PROXY=1` in `docker-compose.yml`); never publish port 8080 directly.

## Deleting data

- **From the app**: a person can delete the server copy of a vault ("Delete from server…" in the sync settings). Only the server copy and its old versions go; the files on every device stay, and the other devices stop syncing that vault by themselves.
- **As the administrator**:

```bash
docker compose exec amby-sync amby-sync-server space list alice                 # ids and sizes, names stay encrypted
docker compose exec amby-sync amby-sync-server space remove alice <space_id> --yes
docker compose exec amby-sync amby-sync-server user remove alice --yes
```

These cannot be undone: make a backup first.

## Back up

Everything lives in one Docker volume. This makes a consistent copy while the server is running:

```bash
docker compose exec amby-sync amby-sync-server backup /data/backups/today
docker cp "$(docker compose ps -q amby-sync)":/data/backups/today ./amby-backup-today
docker compose exec amby-sync rm -r /data/backups/today
```

The copy (`amby-sync.db` and, if there are attachments, `blobs/`) is as unreadable as the live data. Keep `plans.json` and your `.env` with it. Keep a copy on another machine too, and remember that **every device also holds a full copy of its notes**.

## Restore and roll back

Run these in the `sync` folder, with the backup in `./amby-backup-today`:

```bash
docker compose down
docker compose run --rm --no-deps --entrypoint sh \
  -v "$PWD/amby-backup-today:/restore:ro" amby-sync -c \
  'rm -rf /data/amby-sync.db /data/amby-sync.db-wal /data/amby-sync.db-shm /data/blobs && cp -r /restore/. /data/'
docker compose up -d
```

The `-wal` and `-shm` files belong to the database being replaced, so they must go. A restored server is older than what your devices have seen; they notice that it went back in time and do not delete anything to match it.

To go back to an older server version after an update, restore the backup made before the update and pin the old version in `.env` (`AMBY_SYNC_VERSION`). A server refuses to open a database written by a newer version, so an older image alone does not start. When a new version changes the database, it also keeps its own copy of the old one in `/data/backups/pre-v<N>-<time>.db`.

## Update

Make a backup first (see above), then:

```bash
cd AmbyNotes-SelfHost && git pull     # also updates this kit
cd sync
docker compose pull && docker compose up -d
docker compose ps                     # wait for "healthy"
```

The database updates itself on the first start of the new version, after making the copy described above. The features described here (plans with `sync`, `--until`, the disk and request limits) need server **0.2.0** or newer.

## Pin a version

By default the server runs `latest`. To update only when you choose, put a number in `.env`:

```
AMBY_SYNC_VERSION=0.2.1
```

and run `docker compose up -d`. The available versions are listed under _Packages_ on the [repository page](https://github.com/MrDarkOm/AmbyNotes-SelfHost). Caddy is pinned to its major version (`caddy:2`).

## Good to know

- `docker compose ps` shows the server as _healthy_ when `/v1/info` answers. `docker compose logs amby-sync` shows the log; it never contains tokens, keys, enrolment codes or request bodies.
- A server whose data was erased gets a new random identity, so a device that sees it stops instead of deleting its files to match an empty server.

## Source and license

The server source is in [`MrDarkOm/amby-notes`, folder `sync/`](https://github.com/MrDarkOm/amby-notes/tree/dev/sync). Every published image records the exact commit it was built from in the label `org.opencontainers.image.revision`.

Licensed under the [AGPL-3.0](LICENSE).
