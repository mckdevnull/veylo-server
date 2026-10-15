# Veylo Server

A small server for your home that keeps the Veylo app in step across your
phones, tablets and televisions. One container per household. Nothing leaves
your network unless you set that up yourself.

It also records. Ask the app to record a programme and the server does it,
whether or not a phone or television is switched on, and plays it back to any
of them — including while it is still recording. And it can show a live
channel through itself, which is what lets an Apple TV play a provider it
otherwise cannot.

## Start it

The server is published as a Docker image, `ghcr.io/mckdevnull/veylo-server`,
for 64-bit Intel/AMD and ARM computers (including Apple silicon Macs and
Raspberry Pi 4/5). You need Docker Desktop (Mac or Windows) or Docker on Linux.

The install guides below take you through it step by step. In short, save this
as `compose.yaml` in a folder of its own, replacing `192.168.1.20` with the
address of the computer running Docker as other devices on your network see it:

```yaml
services:
  veylo-server:
    image: ghcr.io/mckdevnull/veylo-server:latest
    restart: unless-stopped
    ports:
      - "8787:8787"
    environment:
      VEYLO_PUBLIC_URL: "http://192.168.1.20:8787"
    volumes:
      - veylo-data:/data
      - ./recordings:/recordings
    mem_limit: 512m   # 2g if you turn on Recording size (admin page)
    pids_limit: 256
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

volumes:
  veylo-data:
    name: veylo-data
```

Then, in that folder:

```sh
docker compose up -d
docker compose logs veylo-server
```

The log shows an admin password **once**, the first time the server starts.
Open `http://192.168.1.20:8787/admin` from any device on your network, log in
with it, and choose your own password. The one from the log can do nothing
else, and stops working the moment you do. If nobody uses it within 24 hours
it expires; restart the server and the log shows a new one.

Then press **Create a pairing code**. On a phone or tablet, scan the square
with the camera. On a television, type the address and the eight-character
code into the app. A code works once and lasts ten minutes.

## Guides

Step-by-step pages for people who have not used Docker before:

- [Install on a Mac](docs/install-mac.md)
- [Install on Windows](docs/install-windows.md)
- [Using the server away from home](docs/remote-access.md)
- [Troubleshooting](docs/troubleshooting.md)
- [Release notes](CHANGELOG.md): what changed in each version

## Updating

New versions are published as `ghcr.io/mckdevnull/veylo-server`, with
`:latest` always the newest. The [release notes](CHANGELOG.md) say what each
one changes.

1. On the admin page, check that **Live now** and **Recording now** are
   empty. Updating restarts the server, which stops recordings and channels
   being watched through it.
2. Back up the `/data` volume (the install guides show how).
3. In the folder that holds your `compose.yaml`:

   ```sh
   docker compose pull
   docker compose up -d
   ```

Your data, devices, pairing and recordings are kept, and paired devices
reconnect by themselves. The admin page's **Version** line shows the version
now running. The install guides for
[Mac](docs/install-mac.md#updating) and [Windows](docs/install-windows.md#updating)
also cover going back to an earlier version, and servers started with
`docker run` instead of compose.

## Settings

| Variable | Default | What it is |
|---|---|---|
| `VEYLO_PUBLIC_URL` | not set | The address devices at home use, for example `http://192.168.1.20:8787`. Pairing codes carry it. If you reach the server by a name rather than a number, it must be set here, or the server will refuse requests made to that name |
| `VEYLO_REMOTE_URL` | not set | An optional second address for use away from home. It must be `https://`, or a private address — which includes a Tailscale `100.x` address, but not a `.ts.net` name over plain `http://` |
| `VEYLO_LISTEN` | `:8787` | The port inside the container |
| `VEYLO_DATA_DIR` | `/data` | The database and its key |
| `VEYLO_RECORDINGS_DIR` | `/recordings` | Where recordings will go |
| `VEYLO_TRUST_PROXY` | `0` | Set to `1` only if every request reaches the server through your own reverse proxy and that proxy sets `X-Forwarded-Proto`. Otherwise leave it: the header is one anyone can send |
| `VEYLO_MAX_CONNECTIONS` | `256` | How many connections the server holds at once. When they are all in use, the one that has been idle longest is closed to let a new one in |
| `VEYLO_HEALTH_LISTEN` | `127.0.0.1:8788` | Where the container's own health check is answered. It must be a loopback address; change it only if that port is taken, which can happen with host networking |

A setting the server cannot make sense of stops it at start, with a message
that says which one and why.

## What you should know

**On your home network the connection is not encrypted.** A certificate for a
private address is not something a household can reasonably be asked to
arrange, so the server speaks plain HTTP by default. Anyone who can watch
traffic on your network can read what syncs — including your TV provider
logins — and the pairing code while it is valid. If that worries you, put the
server behind HTTPS (a reverse proxy with a certificate) or reach it over
Tailscale. Away from home, only those two are supported.

**The server holds your provider logins.** They are encrypted where they are
stored, with a key kept beside them (`secret.key` in the data volume). That
protects a copy of the database that ends up somewhere it should not. It does
not protect against someone who can read the whole data volume: that person is
the administrator of the computer, and is trusted.

**Back up the whole data volume, not one file.** The database is unreadable
without `secret.key`, and the server will not start with a key that does not
match its database.

**After restoring any backup, open the admin page and review your devices.**
A backup from last month does not know that you removed a device last week:
after a restore, that device is paired again and its access works. The server
cannot undo that for you, and **it cannot always tell you either.** Remove any
device you had removed before.

The server keeps a small file called `generation` beside the database. If the
database is older than that file when the server starts, the admin page shows
"Restored from backup: review devices" until you have looked. But a backup of
the whole volume contains that file too, so restoring the whole volume puts
back a matching pair and **no notice appears**. If you want the automatic
notice, leave the `generation` file out of your backups (back up everything
else in the volume, including `secret.key`); the server then sees a database
without its file, or beside a newer one, and says so at the next start. Either
way, do not wait for a notice: check the device list after every restore.

**If you lose the admin password**, remove the file `admin-password-set` from
the data volume and restart. The log shows a new one:

```sh
docker compose exec veylo-server rm /data/admin-password-set
docker compose restart veylo-server
docker compose logs --tail 20 veylo-server
```

**Revoke and forget are different.** *Revoke* signs a device out and keeps
what it had synced. *Forget* signs it out and deletes what only it had synced.
The server keeps the synced data of the 32 most recently revoked devices;
revoking one more removes the data of the one revoked longest ago. The admin
page marks that device beforehand, tells you when you are about to do it, and
names it afterwards. Normally nothing is lost by it: your other devices hold
copies of everything they synced from it.

**Your own browser is remembered.** After you log in, the admin page sets a
cookie that lasts 90 days and marks the browser as one that has logged in
before. It does not log you in — you still need the password — but it means
that someone else on your network guessing passwords cannot lock *you* out of
your own admin page. Ten wrong passwords in ten minutes from a remembered
browser make the server forget it; it is remembered again at its next
successful login. Changing the password forgets every such browser.

## Recording

Recordings are kept as they were broadcast: the picture is never re-encoded,
so a recording looks exactly like the channel did and takes about the same
space — roughly 1 to 2 GB an hour for HD, 5 to 8 GB an hour for 4K.

**Where they go.** `compose.yaml` puts them in a folder called `recordings`
next to it. The server runs as an ordinary user (number 10001), not as an
administrator, so that folder has to be one it may write to:

- **Mac and Windows (Docker Desktop):** nothing to do. Any folder you can
  write to works.
- **Linux:** create the folder yourself before the first start and hand it to
  the server, or Docker creates it owned by root and the server cannot use it:

  ```sh
  mkdir -p recordings && sudo chown 10001:10001 recordings
  ```

If the server cannot write there it still starts, and sync still works, but it
does not record. The log says so in one line that names the folder, and the
admin page says "This server is not recording". Fix the folder and restart.

**The computer has to be awake.** A recording is made by this computer. If it
sleeps, or Docker is stopped, the recording stops; when it wakes inside the
programme it carries on, and the recording says that part is missing and why.
Set the computer not to sleep if you record overnight.

**Connections.** Your provider allows a fixed number of connections at once,
often one or two. A recording uses one. When a recording needs a connection
and none is free, it takes one from a live channel being watched through the
server — never from another recording. The app warns before that happens and
offers to keep watching the channel as it is recorded.

**Disk space.** The admin page's Storage section shows what is used and lets
you set clean-up rules: keep only the newest few recordings of a programme,
delete a recording a day after it has been watched, or delete after so many
days. Nothing is deleted on less than 24 hours' notice after you change a
rule, and a recording you mark as protected in the app is never deleted by a
rule. **The server never deletes a recording to make room.** When free space
falls below the limit you set (5 GB unless you change it) it stops recording
and says the disk is full.

**What it will not do yet:** record a whole series from one request, or keep
subtitles.

## Secure connection

The server makes its own identity (a key and a certificate, kept in the `tls`
folder of the data volume) the first time it starts, and answers encrypted and
plain connections on the same port. Apps that know about it pair using the
identity in the QR code, then talk to the server over `https` on that same
address. Older apps keep working over plain `http`.

- The admin page shows the identity, how many devices are encrypted and which
  are not, and has a switch, **Allow unencrypted connections from older
  apps**, that is on by default. Turn it off once every device shows as
  encrypted.
- When you type the pairing code instead of scanning it, the device shows a
  **check code**. It should match the one on the admin page.
- Your browser will warn the first time you open the `https` address of the
  admin page, because the identity is made by this server rather than bought.
  That is expected: check that the identity the browser shows matches the one
  on the page.
- Video still plays over the plain address, because Apple's player cannot check
  the server's identity.
- **Replace identity…** (admin page) is for the case that someone copied the
  server's data folder. Every device is signed out and must be paired again.
- To keep the identity when you move the server to a new computer, move the
  whole data volume, `tls` folder included. A backup of the data volume is a
  copy of the key: keep it private.
- For an address that works away from home with no browser warning, see
  [Reach it from outside your home](docs/remote-access.md#reach-it-from-outside-your-home).

## Keeping it private

`compose.yaml` publishes port 8787 on **every network interface** of the
computer running Docker, so that the other devices in your home can reach it.
That is also every network that computer is connected to.

- **Never forward this port from your router.** The server speaks plain HTTP
  and is built for a home network. To use it away from home, use one of the
  two supported ways: Tailscale, or a reverse proxy with a certificate. A
  reverse proxy must pass the `Host` header through unchanged.
- **On Linux, Docker bypasses `ufw`.** A port Docker publishes is opened in
  the firewall by Docker itself, ahead of `ufw`'s rules, so `ufw deny 8787`
  does not close it. If the computer is on a network you do not trust, bind
  the port to one address instead, for example `"192.168.1.20:8787:8787"`, or
  to `"127.0.0.1:8787:8787"` when a reverse proxy on the same machine is the
  only thing that should reach it.

## Limits

`compose.yaml` gives the container at most 512 MB of memory and 256
processes, so that a fault cannot take the computer with it. Measured on the
machine this was built on, in a container with those limits:

| | Memory | Processes and threads |
|---|---|---|
| Idle | 82 MB | |
| Two recordings and one live channel | 167 MB at most | 66 |
| Eight live channels and two recordings (every limit reached) | 300 MB at most | 122 |
| One 4K recording at 12 Mb/s | 64 MB | |

In a full household of 32 devices, each with a 4 MB file, an upload needs
roughly 100 MB more while it is merged. So 512 MB holds the worst case of both
together, with about 100 MB to spare, and neither default has been raised. If
you see the container restarting with "out of memory", raise `mem_limit` in
`compose.yaml` (for example to `1g`) and run `docker compose up -d` again.
The server itself aims to stay under 256 MB of that, leaving the rest to the
programs that do the recording; if you raise `mem_limit`, you can raise that
aim with it by setting `GOMEMLIMIT` (for example `GOMEMLIMIT=512MiB`).

Making recordings smaller (Recording size: Smaller or Smallest) needs more:
the encoder alone is about 650 MB. Set `mem_limit: 2g` if you use it. The
server reads its memory limit at start, and under 1.5 GB it does not shrink
anything: recordings wait as recorded, and the admin page says why.

Four devices each playing a recording, keeping a long-poll open and calling
the API held 14 connections between them at most (one player holds three or
four), against a limit of 256.

Recording costs little processor time, because the picture is copied, not
re-encoded: two recordings and a live channel together used about a tenth of
one processor core, and one 4K recording about a seventh.

The server sets aside 8 MB in a file called `reserve` in the data volume. If
the disk fills up completely, that file is given up so that you can still
remove a device from the admin page. Only removing a device from the admin page
uses it. Once you have made room, the server sets the 8 MB aside again by
itself within a few minutes.

## About this repository

This repository holds the Veylo Server's documentation and release notes. The
server's source code is not published here.
