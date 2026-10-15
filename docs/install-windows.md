# Install Veylo Server on Windows

This guide takes you from nothing to a working Veylo Server on a Windows 10 or
Windows 11 PC, using Docker Desktop. Allow about an hour, most of it waiting
for downloads and one restart.

You do not need to know anything about Docker. Where you have to type
something, it is written out for you to copy.

## What you are installing

Veylo Server is a small program that runs in the background on one computer in
your home. It does two jobs:

- It keeps the Veylo app in step across your phones, tablets and televisions
  (your providers, favourites, merged channels and so on).
- It records live channels, even when no phone or television is switched on,
  and plays the recordings back to any of them.

Docker Desktop is the program that runs Veylo Server on your PC. It uses a
Windows feature called **WSL 2** (Windows Subsystem for Linux), which is a
small, built-in Linux inside Windows. Docker Desktop sets that up for you.

## Before you start

- A PC that can stay switched on and awake, and that is on your home network
  all the time. A recording is made by this PC. A PC that is asleep records
  nothing (see "Keep the PC awake" below).
- Windows 10 (version 22H2) or Windows 11, 64-bit. Docker Desktop's own
  download page lists the exact requirements, including that the PC's
  virtualisation feature is switched on (it usually is).
- Free disk space for recordings: roughly 1 to 2 GB for every hour of HD, and
  5 to 8 GB an hour for 4K. The Recording size setting can shrink HD
  recordings to about a quarter of that after they end (see the admin page);
  while it does, the server needs about 1 GB more memory, so give Docker at
  least 2 GB and set `mem_limit` in `compose.yaml` to `2g` (the default, 512m,
  is too small: the server then leaves recordings as recorded and says so).
- Your phone or television on the same home network (the same Wi-Fi or the
  same router) as the PC.

## What the server stores, and who can read it

Read this once before you go on.

- **Your IPTV provider logins.** To keep your devices in step, and to record,
  the server holds the logins your devices give it. They are encrypted on the
  disk, with a key that is kept next to them (a file called `secret.key`).
  That protects a stray copy of the database. It does **not** protect against
  anyone who controls this PC: whoever can read the server's data can read
  your provider logins. Only install the server on a computer that you, and
  people you trust, control.
- **Recordings are ordinary video files**, not encrypted.
- **On your home network the connection is not encrypted.** The server speaks
  plain `http`, not `https`. When you pair a device you are trusting your home
  network: anyone who can watch the traffic on it (a guest on your Wi-Fi, for
  example) could read the pairing code while it is valid, and later the
  device's access to the server and what syncs, including provider logins. If
  that worries you, use a network you trust, and read [Using the server away
  from home](remote-access.md), which covers encrypted options.

## Step 1: Install Docker Desktop

1. In a web browser, go to the Docker Desktop download page at
   `https://www.docker.com/products/docker-desktop/` and download the
   Windows version.
2. Open the downloaded installer. If it offers a choice between **WSL 2** and
   **Hyper-V**, choose **WSL 2**.
3. When the installer finishes, it may ask you to sign out or restart. Do so.
4. After the restart, open **Docker Desktop** from the Start menu. Accept the
   licence terms.
5. Wait until Docker Desktop says that it is running. You should see a small
   whale icon near the clock, and the Docker Desktop window should show no
   error. It may take a few minutes the first time.

You can skip any offer to create a Docker account or to answer a survey.

## Step 2: Make Docker start by itself

Without this, the server would stay off after a restart until you opened
Docker by hand.

1. Open Docker Desktop, then open **Settings** (the gear icon at the top).
2. On the **General** tab, tick **Start Docker Desktop when you sign in to
   your computer**, then choose **Apply** (or **Apply and restart**).

Docker Desktop only starts when someone has signed in to Windows. If the PC
restarts by itself (after a Windows update, say), the server is back as soon
as you sign in. If you want it back without anyone signing in, Docker Desktop
alone cannot do that.

## Step 3: Open PowerShell

PowerShell is a window where you type instructions to the PC. Click the Start
button, type `PowerShell`, and press **Enter**.

How to use it in this guide:

- Each grey box below is something to type or paste into PowerShell.
- After you paste or type, press **Enter** to run it.
- Nothing happens until you press Enter.
- Copy the text exactly, including spaces.

## Step 4: Find your PC's address

Other devices at home find the server by your PC's address on the home
network. It looks like four numbers separated by dots, for example
`192.168.1.20`.

1. In PowerShell, type this and press Enter:

   ```powershell
   ipconfig
   ```

2. A long list appears. Find the section called **Wireless LAN adapter
   Wi-Fi** (if you use Wi-Fi) or **Ethernet adapter Ethernet** (if you use a
   cable). Inside it, find the line called **IPv4 Address**. The numbers on
   that line are your PC's address.
3. **Ignore** sections whose names start with `vEthernet`, or mention `WSL`,
   `Docker` or `Hyper-V`. They are internal to the PC; other devices cannot
   use them.
4. Write the address down. This guide uses `192.168.1.20`; yours will be
   different. From here on, wherever you see `192.168.1.20`, use your own.

**Keep the address from changing.** Most routers hand out addresses that can
change after a restart. If this one changes, your devices will no longer find
the server. In your router's settings, look for a way to "reserve" or "fix"
the address for your PC (often called "DHCP reservation" or "static lease").
Every router is different; your router's manual or provider will say how.

## Step 5: Create a folder for the server

Type this and press Enter. It makes a folder called `veylo-server` in your
user folder, and moves PowerShell into it:

```powershell
mkdir $HOME\veylo-server; cd $HOME\veylo-server
```

PowerShell shows the new folder. From now on, every instruction in this guide
assumes PowerShell is in that folder. If you close PowerShell and open it
again later, type `cd $HOME\veylo-server` first.

## Step 6: Write the settings file

Docker is told what to run by a small text file called `compose.yaml`.

1. Type this and press Enter. Notepad opens:

   ```powershell
   notepad compose.yaml
   ```

2. Notepad says it cannot find the file and asks whether to create it.
   Choose **Yes**.
3. Paste the following into Notepad. **Change the address on the
   `VEYLO_PUBLIC_URL` line** to your own, keeping `http://` before it and
   `:8787` after it.

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
         - veylo-recordings:/recordings
       # Recording size Smaller or Smallest? Change this to 2g.
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
     veylo-recordings:
       name: veylo-recordings
   ```

4. Choose **File**, then **Save**, and close Notepad. Check the file is
   really named `compose.yaml` and not `compose.yaml.txt`: in PowerShell,
   type `dir` and press Enter. If you see `compose.yaml.txt`, type
   `ren compose.yaml.txt compose.yaml` and press Enter.

Spaces matter in this file. If the lines do not line up as shown, paste it
again.

What the important lines mean:

| Line | What it does |
|---|---|
| `image: ghcr.io/mckdevnull/veylo-server:latest` | Which program to download and run |
| `restart: unless-stopped` | Start it again by itself after a restart or a crash |
| `"8787:8787"` | The server answers on port 8787 of your PC. A port is like a numbered door |
| `VEYLO_PUBLIC_URL` | The address your devices use at home. Pairing codes carry it |
| `veylo-data:/data` | A storage area, managed by Docker, holding the database and its key |
| `veylo-recordings:/recordings` | A second storage area, managed by Docker, where recordings go (see Step 7) |
| The rest (`mem_limit`, `pids_limit`, `read_only`, `cap_drop`, `no-new-privileges`) | Safety limits so that a fault in the server cannot take over the PC. Leave them alone, with one exception: if you turn on Recording size (Smaller or Smallest), change `mem_limit` to `2g` |

## Step 7: Choose where recordings go

The place for recordings is chosen in this file, not on the admin page. (The
admin page later shows how much space they use and how much is free, and lets
you set clean-up rules.)

**The default, which this guide recommends, is a storage area managed by
Docker** (`veylo-recordings`, in the file above). It lives inside the space
Docker Desktop keeps for WSL 2, not in a normal Windows folder. You do not see
the recordings as files in File Explorer, but you do not need to: you play
them in the Veylo app.

Why not a normal Windows folder such as `C:\Recordings`? Docker runs inside
WSL 2, which is a separate Linux system, and every file it reads or writes in
a Windows folder has to cross from one system to the other. That crossing is
slower than using space inside WSL 2, and recording writes video all day. A
Windows folder works, but it is the slower choice.

**If you want a normal Windows folder anyway** (to see the files, or to use
another disk):

1. Make the folder, for example `D:\Veylo recordings`.
2. In `compose.yaml` (`notepad compose.yaml`), change the line
   `- veylo-recordings:/recordings` so that the left half is the folder, written
   with forward slashes and in quote marks:

   ```yaml
         - "D:/Veylo recordings:/recordings"
   ```

   Do not change `/recordings` on the right.
3. Delete the two lines at the bottom that mention `veylo-recordings`
   (`veylo-recordings:` and `name: veylo-recordings`), since the storage area
   is no longer used.
4. Docker Desktop must be allowed to use that drive. If the server later says
   it is not recording, see
   [Troubleshooting](troubleshooting.md#the-admin-page-says-this-server-is-not-recording).

If you already use Ubuntu (or another Linux) under WSL and are comfortable
running these commands there, running them inside it and keeping the
`recordings` folder in its home directory is also fast. This guide does not
cover that.

Recordings are not encrypted. Anyone who can use this PC's Docker can read
them.

## Step 8: Start the server

1. Make sure Docker Desktop is running (the whale icon near the clock).
2. In PowerShell, in the `veylo-server` folder, type this and press Enter:

   ```powershell
   docker compose up -d
   ```

3. The first time, Docker downloads the server. That can take a few minutes.
   The `-d` means "keep running in the background". When it finishes you
   should see a line ending in `Started`.

### The firewall prompt

The first time something at home tries to reach the server, or when it
starts, Windows may show a window titled **Windows Security Alert** or
**Windows Defender Firewall has blocked some features of this app**, naming
Docker or `com.docker.backend`.

1. Tick **Private networks** (home or work networks).
2. Do **not** tick **Public networks**.
3. Choose **Allow access**.

If you closed the window by mistake, or later find that phones cannot reach
the server, see
[Troubleshooting](troubleshooting.md#my-phone-or-tv-cannot-find-the-server).

### Check that it is running

```powershell
docker compose ps
```

You should see a line for `veylo-server` with a status that starts with `Up`
and, after about half a minute, says `(healthy)`.

## Step 9: Find the admin password

The first time the server starts, it makes a password for the admin page and
shows it **once**, in its log. Type this and press Enter:

```powershell
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

Select the password between the lines of equals signs and copy it (in
PowerShell, select with the mouse and press **Enter** to copy). Be careful to
take all of it.

This password can only be used to choose your own. If you do not use it within
24 hours, it stops working; if that happens, see
[Troubleshooting](troubleshooting.md#i-lost-the-admin-password-or-it-expired).

## Step 10: Open the admin page and choose your password

The admin page is a web page served by the server. Open it from any device on
your home network, including the PC itself.

1. Open a web browser.
2. In the address bar type your address with `:8787/admin` on the end, for
   example `http://192.168.1.20:8787/admin`, and press Enter. Use your PC's
   address, not the word `localhost`: pairing codes copy the address you used,
   and `localhost` only works on the PC itself.
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

## Keep the PC awake

This matters if you record. A recording is made by this PC. If the PC goes to
sleep, or Docker Desktop is closed, the recording stops. If the PC wakes up
again during the programme, the server carries on, and the recording says that
the part in between is missing and why: "The computer was asleep or the server
was stopped." A recording that never started because the PC was asleep shows
the same reason.

To stop the PC sleeping:

1. Open **Settings**, then **System**, then **Power** (Windows 11: **Power &
   battery**). On Windows 10: **System**, then **Power & sleep**.
2. Under **Screen and sleep** (Windows 10: **Sleep**), set **When plugged in,
   put my device to sleep after** to **Never**. Turning the screen off is fine.
3. A laptop also sleeps when you close the lid. In the Start menu search for
   **Choose what closing the lid does**, and set **When I close the lid** to
   **Do nothing** for "Plugged in".

The names differ a little between versions of Windows.

Keep Docker Desktop running (it can sit near the clock). If you quit it, the
server stops. Windows updates that restart the PC also interrupt recordings.

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

In PowerShell:

```powershell
cd $HOME\veylo-server
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

1. Open `compose.yaml` in Notepad and change `:latest` at the end of the
   `image:` line to the version you had before, for example
   `ghcr.io/mckdevnull/veylo-server:0.1.0`. Save it.
2. Run `docker compose up -d` in `$HOME\veylo-server`.
3. If the earlier version will not start, [restore the backup](#restore-a-backup)
   you made before updating.

Put `:latest` back when you want updates again.

### If you started the server with `docker run`

These steps assume you followed this guide, which uses `compose.yaml`. If you
started the server with a `docker run` command instead, run that same command
again after removing the old container:

```powershell
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

Recordings are **not** in this backup. They are in `veylo-recordings` (or the
Windows folder you chose in Step 7). Back them up separately if you want them.

### Make a backup

1. Stop the server, so that its files are not changing while they are copied:

   ```powershell
   cd $HOME\veylo-server
   docker compose stop
   ```

2. Make the backup file. This creates `veylo-data-backup.tar.gz` in the
   current folder:

   ```powershell
   docker run --rm -v veylo-data:/data -v "${PWD}:/backup" alpine tar czf /backup/veylo-data-backup.tar.gz --exclude=./generation -C /data .
   ```

   (`alpine` is a very small program that Docker downloads for this one job.
   `--exclude=./generation` leaves out a small file that lets the server notice
   a restore; see "After restoring" below.)

3. Start the server again:

   ```powershell
   docker compose start
   ```

4. Copy `veylo-data-backup.tar.gz` somewhere safe, away from this PC (another
   disk or a cloud drive). The file contains your provider logins and the key
   that decrypts them, so treat it like a password.

### Restore a backup

Use this on the same PC after a problem, or on a new PC after you have done
Steps 1 to 6 there. The backup file must be in the `veylo-server` folder.

1. Start the server once, so that Docker creates its storage area correctly,
   then stop it:

   ```powershell
   cd $HOME\veylo-server
   docker compose up -d
   docker compose stop
   ```

2. Replace the contents with the backup, and give the files back to the
   server's own user:

   ```powershell
   docker run --rm -v veylo-data:/data -v "${PWD}:/backup" alpine sh -c 'rm -rf /data/* /data/.[!.]* ; tar xzf /backup/veylo-data-backup.tar.gz -C /data && chown -R 10001:10001 /data && chmod 700 /data'
   ```

3. Start the server:

   ```powershell
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
2. In PowerShell, stop and remove the server but keep its data, in case you
   change your mind:

   ```powershell
   cd $HOME\veylo-server
   docker compose down
   ```

3. To also delete the server's database, key and device list, **and every
   recording in the `veylo-recordings` storage area**, for good:

   ```powershell
   docker compose down -v
   ```

4. To free the space the downloaded server takes:

   ```powershell
   docker image rm ghcr.io/mckdevnull/veylo-server:latest
   ```

5. If you chose a Windows folder for recordings, it is not deleted by any of
   the above. Delete it yourself if you no longer want the recordings.
6. To remove Docker Desktop itself, use **Settings**, **Apps**, find
   **Docker Desktop**, and choose **Uninstall**. Then delete the
   `veylo-server` folder in your user folder.

## Checklist: did it work?

Work down this list. Each line says what you should see. Stop at the first
line that does not match, and look in [Troubleshooting](troubleshooting.md).
Replace `192.168.1.20` with your PC's address.

1. In PowerShell, `docker --version` prints a version number.
2. In the `veylo-server` folder, `docker compose config` prints your file back
   and no error. (It checks the file is correctly written.)
3. `docker compose ps` shows `veylo-server` as `Up`, then `(healthy)` after
   about half a minute.
4. `docker compose logs veylo-server` shows the box "Veylo Server admin
   password" the first time, and a line containing `veylo-server listening`.
   After you choose your password, a line says the bootstrap password "has been
   replaced by a chosen one and no longer works".
5. On the PC, a browser opens `http://192.168.1.20:8787/admin` and shows the
   **Log in** page.
6. On a phone on the same Wi-Fi (not the PC), the same address shows the
   **Log in** page too. If this fails but step 5 works, it is the firewall:
   see Troubleshooting.
7. After logging in, the **Addresses** section shows your address next to
   **At home**, with no warning under it.
8. The **Recordings** section does not say "This server is not recording", and
   the **Storage** section is shown, with a **Free space** figure. (That proves
   the server found its recording tools and can write to the recordings area.)
9. In a browser, `http://192.168.1.20:8787/api/v1/info` shows a block of text
   that includes `"capabilities"` listing `sync`, `recording` and `live`. (If
   `recording` is missing, recording is off; the log says why.)
10. **Create a pairing code** shows a code and a QR code, and a phone that
    scans it ends up with a device listed under **Devices**.
11. Restart the PC and sign in **without opening Docker yourself**. After a
    couple of minutes step 5 works again. (This checks Step 2.)
12. Set the PC to sleep (as it would overnight), wake it, and check the admin
    page still loads. (A recording that overlapped the sleep should show "The
    computer was asleep or the server was stopped.")
13. Make a backup as above. `veylo-data-backup.tar.gz` exists and is more
    than a few kilobytes. In PowerShell, `tar tzf veylo-data-backup.tar.gz`
    lists files including `secret.key`.
14. `docker compose pull` followed by `docker compose up -d` finishes without
    an error, and the admin page still loads and still lists your devices.

## Where to go next

- [Using the server away from home](remote-access.md)
- [Troubleshooting](troubleshooting.md)
