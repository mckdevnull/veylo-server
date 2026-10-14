# Install Veylo Server on a Mac

This guide takes you from nothing to a working Veylo Server on a Mac, using
Docker Desktop. It works on Macs with Apple silicon (M1 and later) and on
Intel Macs. Allow about half an hour, most of it waiting for downloads.

You do not need to know anything about Docker. Where you have to type
something, it is written out for you to copy.

## What you are installing

Veylo Server is a small program that runs in the background on one computer in
your home. It does two jobs:

- It keeps the Veylo app in step across your phones, tablets and televisions
  (your providers, favourites, merged channels and so on).
- It records live channels, even when no phone or television is switched on,
  and plays the recordings back to any of them.

Docker Desktop is the program that runs Veylo Server on your Mac. You install
Docker Desktop once; after that Veylo Server looks after itself.

## Before you start

- A Mac that can stay switched on and awake, and that is on your home network
  all the time. A recording is made by this Mac. A Mac that is asleep records
  nothing (see "Keep the Mac awake" below).
- Free disk space for recordings: roughly 1 to 2 GB for every hour of HD, and
  5 to 8 GB an hour for 4K.
- Your phone or television on the same home network (the same Wi-Fi or the
  same router) as the Mac.

## What the server stores, and who can read it

Read this once before you go on.

- **Your IPTV provider logins.** To keep your devices in step, and to record,
  the server holds the logins your devices give it. They are encrypted on the
  disk, with a key that is kept next to them (a file called `secret.key`).
  That protects a stray copy of the database. It does **not** protect against
  anyone who controls this Mac: whoever can read the server's data can read
  your provider logins. Only install the server on a computer that you, and
  people you trust, control.
- **Recordings are ordinary video files**, not encrypted, in a folder on this
  Mac.
- **On your home network the connection is not encrypted.** The server speaks
  plain `http`, not `https`. When you pair a device you are trusting your home
  network: anyone who can watch the traffic on it (a guest on your Wi-Fi, for
  example) could read the pairing code while it is valid, and later the
  device's access to the server and what syncs, including provider logins. If
  that worries you, use a network you trust, and read [Using the server away
  from home](remote-access.md), which covers encrypted options.

## Step 1: Install Docker Desktop

1. Find out which kind of Mac you have. Click the Apple menu at the top left
   of the screen and choose **About This Mac**. Next to "Chip" you will see
   either "Apple M1" (or M2, M3 and so on), or "Intel".
2. In a web browser, go to the Docker Desktop download page at
   `https://www.docker.com/products/docker-desktop/`.
3. Download the Mac version that matches your chip: "Apple silicon" for M1 and
   later, "Intel chip" for Intel.
4. Open the downloaded file and drag the Docker icon onto **Applications**,
   as the window shows.
5. Open **Docker** from your Applications folder. Accept the licence terms and
   enter your Mac password if it asks.
6. Wait until Docker Desktop says that it is running. You should see a small
   whale icon in the menu bar at the top of the screen. It may take a minute
   or two the first time.

You can skip any offer to create a Docker account or to answer a survey.

## Step 2: Make Docker start by itself

Without this, the server would stay off after a restart until you opened
Docker by hand.

1. Click the whale icon in the menu bar and choose **Settings** (the gear
   icon in the Docker Desktop window does the same).
2. On the **General** tab, tick **Start Docker Desktop when you sign in to
   your computer**, then choose **Apply** if the window offers it.

Docker Desktop only starts when you are signed in to the Mac. If the Mac
restarts by itself (after an update, say), the server is back as soon as you
sign in.

## Step 3: Open Terminal

Terminal is a window where you type instructions to the Mac. Press
**Command and Space**, type `Terminal`, and press **Return**.

How to use it in this guide:

- Each grey box below is something to type or paste into Terminal.
- After you paste or type, press **Return** to run it.
- Nothing happens until you press Return.
- Copy the text exactly, including spaces. Capital letters matter.

## Step 4: Find your Mac's address

Other devices at home find the server by your Mac's address on the home
network. It looks like four numbers separated by dots, for example
`192.168.1.20`.

1. Make sure the Mac is connected to your home network, by Wi-Fi or a cable.
2. Open **System Settings**, choose **Wi-Fi** (or **Ethernet** if you use a
   cable), and choose **Details** next to the network you are on. Find the
   line called **IP address**. The exact wording differs a little between
   versions of macOS.
3. Write the address down. This guide uses `192.168.1.20`; yours will be
   different. From here on, wherever you see `192.168.1.20`, use your own.

If you use Wi-Fi, you can also ask Terminal. Type this and press Return; if it
prints four numbers, that is the address:

```sh
ipconfig getifaddr en0
```

**Keep the address from changing.** Most routers hand out addresses that can
change after a restart. If this one changes, your devices will no longer find
the server. In your router's settings, look for a way to "reserve" or "fix"
the address for your Mac (the setting is often called "DHCP reservation" or
"static lease"). Every router is different; your router's manual or provider
will say how.

## Step 5: Create a folder for the server

Type this and press Return. It makes a folder called `veylo-server` in your
home folder, and moves Terminal into it:

```sh
mkdir -p ~/veylo-server && cd ~/veylo-server
```

You should see no message. That means it worked. From now on, every
instruction in this guide assumes Terminal is in that folder. If you close
Terminal and open it again later, type `cd ~/veylo-server` first.

## Step 6: Write the settings file

Docker is told what to run by a small text file called `compose.yaml`. You
will create it with two steps.

First, tell Terminal your address. Type this with your own address in place of
`192.168.1.20`, and press Return. Type only the four numbers, with no `http://`
and no `:8787`:

```sh
VEYLO_ADDRESS=192.168.1.20
```

Then paste all of the following at once and press Return. It writes the file:

```sh
cat > compose.yaml <<EOF
services:
  veylo-server:
    image: ghcr.io/mckdevnull/veylo-server:latest
    restart: unless-stopped
    ports:
      - "8787:8787"
    environment:
      VEYLO_PUBLIC_URL: "http://$VEYLO_ADDRESS:8787"
    volumes:
      - veylo-data:/data
      - ./recordings:/recordings
    mem_limit: 512m
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
EOF
```

Check that it worked. Type this and press Return:

```sh
cat compose.yaml
```

You should see the file you just pasted, with the line `VEYLO_PUBLIC_URL` showing
your own address, for example `"http://192.168.1.20:8787"`. If the address is
missing or wrong, type the `VEYLO_ADDRESS=` line again and paste the block
again; it replaces the old file.

What the important lines mean:

| Line | What it does |
|---|---|
| `image: ghcr.io/mckdevnull/veylo-server:latest` | Which program to download and run |
| `restart: unless-stopped` | Start it again by itself after a restart or a crash |
| `"8787:8787"` | The server answers on port 8787 of your Mac. A port is like a numbered door |
| `VEYLO_PUBLIC_URL` | The address your devices use at home. Pairing codes carry it |
| `veylo-data:/data` | A storage area, managed by Docker, holding the database and its key |
| `./recordings:/recordings` | Your recordings go in a folder called `recordings`, next to this file |
| The rest (`mem_limit`, `pids_limit`, `read_only`, `cap_drop`, `no-new-privileges`) | Safety limits so that a fault in the server cannot take over the Mac. Leave them alone |

## Step 7: Choose where recordings go

The default is the `recordings` folder inside `veylo-server` in your home
folder (`~/veylo-server/recordings`). Docker creates it the first time the
server starts. If that suits you, go straight to Step 8.

The place for recordings is chosen in this file, not on the admin page. (The
admin page later shows how much space they use and how much is free, and lets
you set clean-up rules.)

To keep recordings on another disk, such as an external drive:

1. Plug the drive in, and find its name in Finder. This example uses a drive
   called `Media`.
2. In `compose.yaml`, the line `- ./recordings:/recordings` has two halves
   separated by a colon. Change only the left half, to the folder you want, for
   example:

   ```yaml
         - "/Volumes/Media/Veylo recordings:/recordings"
   ```

   Keep the quote marks if the name has a space in it. Do not change
   `/recordings` on the right.
3. Make the folder first, so that it exists. In Terminal:

   ```sh
   mkdir -p "/Volumes/Media/Veylo recordings"
   ```

You can change the file with any plain-text editor. In Terminal,
`nano compose.yaml` opens a simple one: move with the arrow keys, press
**Control and O** then **Return** to save, and **Control and X** to leave.

Docker Desktop must be allowed to use that disk. If the server later says it
is not recording, see [Troubleshooting](troubleshooting.md#the-admin-page-says-this-server-is-not-recording).
An external drive that is unplugged or asleep stops recordings.

Whichever folder you choose, anyone who can open it can play the recordings.

## Step 8: Start the server

Type this and press Return:

```sh
docker compose up -d
```

The first time, Docker downloads the server. That can take a few minutes. The
`-d` means "keep running in the background". When it finishes you should see a
line ending in `Started`.

Now check that it is running:

```sh
docker compose ps
```

You should see a line for `veylo-server` with a status that starts with `Up`
and, after about half a minute, says `(healthy)`.

## Step 9: Find the admin password

The first time the server starts, it makes a password for the admin page and
shows it **once**, in its log. Type this and press Return:

```sh
docker compose logs veylo-server
```

Look for a box like this:

```
  ============================================================
   Veylo Server admin password (shown this once):

       <a long password of letters and numbers>

   Open /admin, log in with it, and choose your own password.
   It works for nothing else, stops working when you do, and
   expires unused after 24 hours (a restart issues a new one).
  ============================================================
```

Select the password between the lines of equals signs and copy it. Be careful
to take all of it.

This password can only be used to choose your own. If you do not use it within
24 hours, it stops working; if that happens, see
[Troubleshooting](troubleshooting.md#i-lost-the-admin-password-or-it-expired).

## Step 10: Open the admin page and choose your password

The admin page is a web page served by the server. Open it from any device on
your home network, including the Mac itself.

1. Open a web browser (Safari, for example).
2. In the address bar type your address with `:8787/admin` on the end, for
   example `http://192.168.1.20:8787/admin`, and press Return. Use your Mac's
   address, not the word `localhost`: pairing codes copy the address you used,
   and `localhost` only works on the Mac itself.
3. You should see a **Log in** page. Paste the password from the log and
   press **Log in**.
4. You should now see **Choose a password**. Paste the password from the log
   again into **Password from the log**, and type a password of your own in
   **New password** (at least 10 characters; a few unrelated words work well).
   Press **Set password**.
5. You should now see the main page, with sections called **Addresses**,
   **Pair a device**, **Devices**, **Recordings**, **Storage** and
   **Password**.

Check two things on the page:

- Under **Addresses**, **At home** shows your address, for example
  `http://192.168.1.20:8787`. If it says "Not set", or a yellow warning
  mentions `VEYLO_PUBLIC_URL`, go back to Step 6.
- Under **Recordings**, you should not see "This server is not recording".
  If you do, see [Troubleshooting](troubleshooting.md#the-admin-page-says-this-server-is-not-recording).

You can also give the server a name under **Name shown to devices**, then
press **Rename**.

## Step 11: Pair your first device

Pairing connects an app to the server. The admin page makes a one-time
**pairing code**. A code works once and lasts ten minutes.

1. On the admin page, under **Pair a device**, press **Create a pairing
   code**.
2. A code appears, written like `K7QF-2M9X` (eight letters and numbers), with
   the address under it and a square QR code beside it. You will see a
   countdown.

**On a phone or tablet (iPhone, iPad, Android):**

1. Install and open Veylo, if you have not already.
2. Point the phone's normal camera at the QR code on the screen. Do not use a
   special scanner; the ordinary camera app is the right one.
3. Tap the link that appears. Veylo opens. It shows the server's name and
   address and asks you to confirm that it is your server. Check the address
   is yours, then confirm.

**On a television (Apple TV, Android TV):**

A television cannot scan the code, so you type it.

1. In Veylo on the television, choose the option to pair with a server (it is
   called **Pair with a server**).
2. Enter the **Server address**: the address written under the code on the
   admin page, for example `http://192.168.1.20:8787`.
3. Leave **Remote address (optional)** empty for now. It is explained in
   [Using the server away from home](remote-access.md).
4. Enter the **Pairing code**. You can leave out the dash, and capital or
   small letters both work. A code never contains the letters I or O or the
   numbers 0 or 1.
5. Give the television a name, such as "Living room TV". The admin page shows
   this name in its list of devices.
6. Choose **Find server**, then confirm.

The words on screen may differ a little in your version of the app.

After pairing, the device appears under **Devices** on the admin page. Pair
your other devices the same way, making a new code for each.

If the app refuses the code, see
[Troubleshooting](troubleshooting.md#the-pairing-code-is-refused).

## Keep the Mac awake

This matters if you record. A recording is made by this Mac. If the Mac goes
to sleep, or Docker Desktop is closed, the recording stops. If the Mac wakes
up again during the programme, the server carries on, and the recording says
that the part in between is missing and why: "The computer was asleep or the
server was stopped." A recording that never started because the Mac was
asleep shows the same reason.

To stop the Mac sleeping:

1. Open **System Settings**.
2. On a laptop, open **Battery** (or **Lock Screen**). On a desktop Mac, open
   **Energy** (on some versions **Displays** or **Lock Screen**).
3. Turn on **Prevent automatic sleeping on power adapter when the display is
   off**, or set the sleep timers to **Never**. The names differ between
   versions of macOS.

A laptop with its lid closed will normally sleep whatever you set. For
recording, use a Mac that stays open or is a desktop, plugged in.

Keep Docker Desktop open, too (it can sit in the menu bar). If you quit it,
the server stops.

## Updating

New versions of the server are published from time to time. The
[release notes](../CHANGELOG.md) say what changed in each, and which app
versions a new feature needs.

### Before you update

1. Open the admin page. Check that **Live now** and **Recording now** (in
   the **Recordings** card) are empty. Updating restarts the server, and a
   restart stops a recording and every channel being watched through it.
2. [Make a backup](#make-a-backup). An update keeps your data, but a backup
   is what lets you go back if something goes wrong.

### Update

In Terminal:

```sh
cd ~/veylo-server
docker compose pull
docker compose up -d
```

The first command downloads the newest version. The second restarts the
server on it, which takes a few seconds. Your data, devices, pairing and
recordings are kept, and paired devices reconnect by themselves.

### Check it worked

On the admin page, the **Version** line under **Addresses** shows the version
now running. It should match the newest one in the release notes.

### Going back to an earlier version

If a new version misbehaves:

1. Open `compose.yaml` in TextEdit and change `:latest` at the end of the
   `image:` line to the version you had before, for example
   `ghcr.io/mckdevnull/veylo-server:0.1.0`. Save it.
2. Run `docker compose up -d` in `~/veylo-server`.
3. If the earlier version will not start, [restore the backup](#restore-a-backup)
   you made before updating.

Put `:latest` back when you want updates again.

### If you started the server with `docker run`

These steps assume you followed this guide, which uses `compose.yaml`. If you
started the server with a `docker run` command instead, run that same command
again after removing the old container:

```sh
docker pull ghcr.io/mckdevnull/veylo-server:latest
docker stop veylo-server
docker rm veylo-server
docker run ...   # the same command you used the first time
```

Use your container's name in place of `veylo-server` (`docker ps` lists it).
Your data is kept as long as the `docker run` command mounts the same volume
or folder at `/data`. If you no longer have the command, `docker inspect
veylo-server` shows the settings it was started with: the volumes under
`Mounts`, the port under `PortBindings` and the settings under `Env`.

## Back up and restore

The server's database and its key are in the Docker storage area called
`veylo-data` (mounted as `/data` inside the server). **Back up the whole of
it, not just one file.** The database cannot be read without the key
(`secret.key`), and the server will not start with a key that does not match
its database.

Recordings are **not** in this backup. They are in the `recordings` folder
(or the folder you chose in Step 7); copy that folder yourself if you want to
keep them.

### Make a backup

1. Stop the server, so that its files are not changing while they are copied:

   ```sh
   cd ~/veylo-server
   docker compose stop
   ```

2. Make the backup file. This creates `veylo-data-backup.tar.gz` in the
   current folder:

   ```sh
   docker run --rm -v veylo-data:/data -v "$PWD":/backup alpine tar czf /backup/veylo-data-backup.tar.gz --exclude=./generation -C /data .
   ```

   (`alpine` is a very small program that Docker downloads for this one job.
   `--exclude=./generation` leaves out a small file that lets the server notice
   a restore; see "After restoring" below.)

3. Start the server again:

   ```sh
   docker compose start
   ```

4. Copy `veylo-data-backup.tar.gz` somewhere safe, away from this Mac (another
   disk or a cloud drive). The file contains your provider logins and the key
   that decrypts them, so treat it like a password.

### Restore a backup

Use this on the same Mac after a problem, or on a new Mac after you have
done Steps 1 to 6 there. The backup file must be in the `veylo-server` folder.

1. Start the server once, so that Docker creates its storage area correctly,
   then stop it:

   ```sh
   cd ~/veylo-server
   docker compose up -d
   docker compose stop
   ```

2. Replace the contents with the backup, and give the files back to the
   server's own user:

   ```sh
   docker run --rm -v veylo-data:/data -v "$PWD":/backup alpine sh -c 'rm -rf /data/* /data/.[!.]* ; tar xzf /backup/veylo-data-backup.tar.gz -C /data && chown -R 10001:10001 /data && chmod 700 /data'
   ```

3. Start the server:

   ```sh
   docker compose start
   ```

### After restoring: review the device list

Do this after **every** restore, even if the server shows no warning.

A backup does not know what you did after it was made. If you removed a
device last week and restore a backup from last month, that device is paired
again, and it works. The server cannot undo that, and it cannot always tell
you either.

1. Open the admin page and log in. If the page shows **Restored from backup:
   review devices**, that is the server telling you. (Because the backup
   command above leaves out the `generation` file, a restore will normally
   show this notice. A backup that included it would not.)
2. Look through the list under **Devices**. For each one you had removed, press
   **Revoke** (signs it out, and keeps what it synced) or **Forget** (signs it
   out and deletes what only it had synced).
3. If the notice is showing, press **I have reviewed the devices**.

## Uninstalling

Uninstalling removes your synced data from the server. Each of your devices
keeps its own copy of everything, so nothing is lost on the devices.

1. In each app, choose **Disconnect from server**. If you cannot (the server is
   already gone), nothing is harmed.
2. In Terminal, stop and remove the server but keep its data, in case you
   change your mind:

   ```sh
   cd ~/veylo-server
   docker compose down
   ```

3. To also delete the server's database, key and device list for good:

   ```sh
   docker compose down -v
   ```

4. To free the space the downloaded server takes:

   ```sh
   docker image rm ghcr.io/mckdevnull/veylo-server:latest
   ```

5. Recordings are not deleted by any of the above. Delete the `recordings`
   folder (or the folder you chose) yourself if you no longer want them.
6. To remove Docker Desktop itself, quit it and drag it from Applications to
   the Bin. Then delete the `veylo-server` folder in your home folder.

## Where to go next

- [Using the server away from home](remote-access.md)
- [Troubleshooting](troubleshooting.md)
