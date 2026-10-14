# Troubleshooting Veylo Server

Find the thing you see, and follow the steps under it. Commands are typed into
Terminal (Mac) or PowerShell (Windows), inside the `veylo-server` folder. If
you have closed the window, open it again and type `cd ~/veylo-server` (Mac)
or `cd $HOME\veylo-server` (Windows) first.

Two commands help with almost everything:

```sh
docker compose ps
docker compose logs --tail 50 veylo-server
```

The first shows whether the server is running (`Up`, then `(healthy)` after
half a minute). The second shows the last 50 lines of what the server has said.

## Contents

- [I cannot reach the admin page](#i-cannot-reach-the-admin-page)
- [The admin page will not let me log in](#the-admin-page-will-not-let-me-log-in)
- [The pairing code is refused](#the-pairing-code-is-refused)
- [My phone or TV cannot find the server](#my-phone-or-tv-cannot-find-the-server)
- [The admin page says "This server is not recording"](#the-admin-page-says-this-server-is-not-recording)
- [A recording did not happen, or only part of it did](#a-recording-did-not-happen-or-only-part-of-it-did)
- [The disk is filling up](#the-disk-is-filling-up)
- [I lost the admin password, or it expired](#i-lost-the-admin-password-or-it-expired)
- [The server keeps restarting](#the-server-keeps-restarting)
- [I want to move the server to another computer](#i-want-to-move-the-server-to-another-computer)

## I cannot reach the admin page

The browser says it cannot connect, or the page never loads.

1. **Is the server running?** Type `docker compose ps`. If the list is empty
   or the status is not `Up`, start it with `docker compose up -d`. If Docker
   itself says it cannot connect, open Docker Desktop and wait for it to say
   it is running.
2. **Is the address exactly right?** It must look like
   `http://192.168.1.20:8787/admin`: `http` (not `https`), your computer's
   address, `:8787`, then `/admin`. With `https` you will get an error,
   because the server itself speaks plain `http`.
3. **Is the device on the same network as the computer?** Same Wi-Fi or same
   router. A guest Wi-Fi network is often kept apart from the rest of the
   home, and devices on it cannot see the computer. Some routers also have a
   setting that stops Wi-Fi devices seeing each other (often called "client
   isolation" or "AP isolation").
4. **Has the computer's address changed?** Routers can give out a new address
   after a restart. Check the computer's address again (Step 4 of the
   install guide) and compare it with `VEYLO_PUBLIC_URL` in `compose.yaml`. If
   it has changed, edit that line, then run `docker compose up -d`. To stop it
   happening again, reserve the address in your router.
5. **Can the computer itself open it?** On the computer that runs the
   server, open `http://localhost:8787/admin`. If that works but other
   devices cannot get in, a firewall is probably in the way: see
   [My phone or TV cannot find the server](#my-phone-or-tv-cannot-find-the-server).
6. **Is the computer awake?** A computer that is asleep answers nobody.
7. **Did the server refuse to start?** Look at the log with
   `docker compose logs --tail 50 veylo-server`. A line starting
   `invalid configuration` says which setting is wrong and why. Fix it in
   `compose.yaml`, then run `docker compose up -d`.
8. **Does the page say it does not know the server's name?** If you see a
   message such as "This server is not known by the name in the Host header"
   (an error 421), you used a *name* for the computer, such as
   `my-mac.local`, that the server was not told about. Either use the number
   address, or put that name in `VEYLO_PUBLIC_URL`.

## The admin page will not let me log in

- **"That password is not right."** On the very first visit, the password is
  the long one from the log (`docker compose logs veylo-server`), not one you
  chose. After you have chosen your own password, use that.
- **The long password from the log no longer works.** It stops working the
  moment you choose your own password, and it also expires after 24 hours if
  unused. See [I lost the admin password, or it expired](#i-lost-the-admin-password-or-it-expired).
- **You are told to wait.** The server
  allows at most 10 wrong passwords a minute across the whole household, to
  slow down anyone guessing. Wait a minute and try again. A browser that has
  logged in before (the server remembers it for 90 days) is not locked out by
  other people's guesses.
- **Your new password is refused.** It must be at least 10 characters.

## The pairing code is refused

A pairing code has eight letters and numbers. It works **once**, and lasts
**ten minutes**. The most common causes:

1. **It has expired or been used.** Press **Create a pairing code** again on
   the admin page, and use the new one. Only the last five codes you made stay
   valid; a sixth cancels the oldest.
2. **A typing mistake.** Codes never contain the letters I or O, or the
   numbers 0 or 1; if you typed one, you misread a character. Capital and small
   letters are the same, and the dash is optional. Check the code letter by
   letter on the admin page.
3. **You tried again too quickly after a mistake.** After a wrong code the
   server makes everyone wait a few seconds before the next try (4 seconds,
   then longer after each further mistake, up to 30 seconds). Wait half a
   minute and try once, carefully.
4. **The address is wrong.** The code only works with the server it came from.
   Check the **Server address** in the app matches the address under the code on
   the admin page, character for character, including `http://` and `:8787`.
5. **The server has all the devices it allows.** A server holds at most 32
   devices. If so, the app says there are too many. On the admin page, under
   **Devices**, remove a device you no longer use (**Forget**, or **Remove**).
6. **Another device is already using the code.** Codes work once. Create a
   fresh code for each device.

If the QR code did nothing, point the **ordinary camera app** at it, not a
scanner app, and tap the link. Or type the address and code by hand instead.

## My phone or TV cannot find the server

The admin page works on the computer, but the app on a phone, tablet or
television says it cannot reach the server.

1. **Same network.** Is the phone on the home Wi-Fi, not on mobile data and not
   on a guest network? Turn mobile data off to be sure.
2. **A VPN on the phone.** Some VPN apps stop the phone reaching devices at
   home. Switch the VPN off to test.
3. **Try the admin page on the phone.** In the phone's web browser, open
   `http://192.168.1.20:8787/admin` (with your own address). If that does not
   load, the problem is the network or the computer, not the app. If it does
   load, check the address typed in the app.
4. **The computer's firewall.**
   - **Windows:** the first time, Windows asks whether to allow Docker through
     the firewall. If you chose "Cancel" or only ticked "Public networks",
     phones are blocked. Open **Windows Security**, **Firewall & network
     protection**, **Allow an app through firewall**, choose **Change
     settings**, find **Docker Desktop** (or **Docker Desktop Backend**), and
     tick **Private**.
   - **Mac:** if you have turned the macOS firewall on (System Settings,
     Network, Firewall), make sure it allows incoming connections for Docker.
5. **The computer's address changed**, as above. Reserve it in your router.
6. **Did you open the admin page as `localhost`?** Pairing codes use the
   address you opened the page with, and `localhost` only works on the
   computer itself. The admin page warns you about this under **Addresses**.
   Open the admin page by the computer's network address, or set
   `VEYLO_PUBLIC_URL` in `compose.yaml`, then create a new pairing code.

## The admin page says "This server is not recording"

Sync still works, but nothing can be recorded, because the server could not
use its recording tools or could not write to its recordings folder. Ask the
log which:

```sh
docker compose logs veylo-server
```

Look for a line that says `recording is off`:

- `recording is off: the server cannot write to its recordings folder` and a
  folder name. The server runs as an ordinary user (number 10001), not as an
  administrator, and that folder must be one it may write to.
  - On a **Mac or Windows with Docker Desktop**, any folder you can write to
    works. If you chose a folder on another disk, check the disk is plugged in
    and switched on, that Docker Desktop is allowed to use it (Docker Desktop's
    Settings, under Resources, File sharing), and that the path in
    `compose.yaml` is spelt exactly right. The simplest test is to go back to
    the default folder, then restart with `docker compose up -d`.
  - On **Linux**, make the folder and give it to the server before the first
    start: `mkdir -p recordings && sudo chown 10001:10001 recordings`.
- `recording is off: no remuxer`. The recording program inside the server could
  not be found. This should not happen with the published server. Update
  (`docker compose pull`, then `docker compose up -d`) and, if it persists,
  report it.

After fixing it, restart with `docker compose up -d` (or `docker compose
restart veylo-server`). The message goes away on the admin page.

## A recording did not happen, or only part of it did

On the admin page, the **Recordings** section has a list called **Did not
record in full**. Each line gives the channel, the programme, and the reason
in plain words. The same reason shows in the app. These are the reasons, and
what to do about each:

| What it says | What happened | What to do |
|---|---|---|
| "The computer was asleep or the server was stopped." | The computer slept, Docker Desktop was closed, or the computer restarted while the programme was on. | Stop the computer sleeping; see "Keep the Mac awake" or "Keep the PC awake" in the install guide. If it woke up during the programme, the part after waking is recorded and the part in the middle is marked missing. |
| "The server restarted during the recording." | The server was restarted (for example by an update). | Avoid updating or restarting while something is recording. |
| "The provider had no free connection: other recordings were using them." | Your IPTV provider allows only a fixed number of connections at once, often one or two, and they were all in use. | See the section below. |
| "The provider's stream would not play, or stopped." | The provider's channel did not work, or dropped. | Check the channel plays in the app. Providers have outages; the server tries again while the programme is on. |
| "The provider refused the login." | The username or password for the provider no longer works. | Fix the provider's details in the app; they sync to the server. |
| "The provider no longer lists this channel." | The provider removed or changed the channel (for example renumbered it, or two channels now share a name). The server finds a channel by its provider number first, then by exact name or guide id, and gives up rather than record the wrong channel if the match is not clear. | In the app, find the channel again and schedule the recording again. |
| "The provider it was to record from is no longer set up." | You removed the provider. | Set it up again, or choose another channel. |
| "The stream is in a format the server cannot record." | The provider's stream is one the recorder cannot handle. | Try a different version of the channel (for example another quality). |
| "The recordings disk was too full." | Free space was below the limit. | See [The disk is filling up](#the-disk-is-filling-up). |
| "It started late." | The recording began after the programme did, so the start is missing. | Check the other reasons on this list; the rest of the programme is recorded. |

### When the provider's connections are all in use

Your provider limits how many channels you can watch or record at once. A
recording needs one connection. The server never lets a recording take a
connection from **another recording**. But when a recording needs a connection
and none is free, it **does** take one from a live channel you are watching
through the server. The app warns you before that happens and offers to keep
watching the channel as it is being recorded, instead of stopping it.

If recordings fail with "no free connection", two programmes were due at the
same time on a provider that allows too few. Either schedule fewer at once, or
use a provider plan with more connections.

### When you were watching the channel

If you are watching a live channel through the server and a recording needs
that channel's provider connection, the recording wins: it takes the
connection from your live channel. The app warns you first and offers to keep
watching the channel as it is recorded. If you stopped a recording yourself,
it says "It was stopped on purpose." and the part before is kept.

## The disk is filling up

Recordings are kept exactly as broadcast, so they are large: roughly 1 to 2 GB
an hour for HD and 5 to 8 GB an hour for 4K.

**To see how much space is used:** on the admin page, in the **Storage**
section, **Recordings** shows the space used and how many recordings there
are, and **Free space** shows what is left.

**The server never deletes a recording to make room.** When free space falls
below a limit you set, it stops recording and says the disk is full. The limit
is 5 GB unless you change it: the setting **Stop recording when free space
falls below (GB)**.

To keep the disk from filling, set clean-up rules in **Storage**, then press
**Save**:

| Setting | What it does |
|---|---|
| **Keep only the newest recordings of a programme** | Keeps the newest few of the same programme and deletes the older ones. Leave empty to keep all. |
| **Delete recordings after this many days** | Deletes each recording that many days after it ended. Leave empty to keep them. |
| **Delete a recording a day after it has been watched** | Deletes it 24 hours after anyone has first watched it. If everyone marks it unwatched again, it is kept. |
| **Start recording early by**, **Keep recording after the end for** (minutes) | Padding around every programme. More padding means bigger recordings. |

How the rules behave:

- **Nothing is deleted on less than 24 hours' notice.** After you change a rule,
  anything it newly condemns is given at least a day before it goes. Turning on
  "delete after 7 days" does not wipe a month-old library overnight. The app
  shows each recording's deletion date in advance.
- **A recording you mark as protected in the app is never deleted by a rule.**
  Neither is a recording that is still being made.
- The clean-up runs once an hour. It does not run in the first five minutes
  after the server starts, nor for an hour after the computer wakes from sleep,
  and one pass deletes at most a fifth of the recordings that are eligible. If
  a rule seems not to have worked, wait for the next hour.
- **You can always delete a recording yourself in the app**, which happens at
  once.

If the disk is completely full, even the small database can no longer be
written. The server keeps a small reserve for this. You can still remove a
device from the admin page; everything else may be refused until you make room.
Free space by deleting recordings in the app, or by clearing other files from
the same disk, and the server recovers by itself.

## I lost the admin password, or it expired

The admin password can be reset from the computer that runs the server, which
is how the server knows it is really you. Your devices are not affected; they
keep syncing.

1. Open Terminal (Mac) or PowerShell (Windows), and go into the `veylo-server`
   folder.
2. Remove the marker file that says a password was chosen:

   ```sh
   docker compose exec veylo-server rm /data/admin-password-set
   ```

3. Restart the server:

   ```sh
   docker compose restart veylo-server
   ```

4. Show the new password:

   ```sh
   docker compose logs --tail 20 veylo-server
   ```

   Find the box "Veylo Server admin password (shown this once)", copy the
   password, and follow Step 10 of the install guide to choose a new one. It
   expires after 24 hours if you do not use it; restarting the server again
   gives you another.

If the first password from the very first start has expired unused, you do not
need step 2: just run steps 3 and 4. A restart always gives a new password
while the first one has not been replaced.

## The server keeps restarting

Look at the log: `docker compose logs --tail 50 veylo-server`.

- A line starting `invalid configuration` says which setting is wrong. Fix it
  in `compose.yaml` and run `docker compose up -d`. The most common one is
  `VEYLO_REMOTE_URL` set to an `http://` address that is not private; use an
  `https://` address or a Tailscale `100.x.y.z` address (see
  [Using the server away from home](remote-access.md)).
- The word "out of memory". The server is limited to 512 MB so that a fault
  cannot take over the computer. Open `compose.yaml`, change `mem_limit: 512m`
  to `mem_limit: 1g`, and run `docker compose up -d`.

## I want to move the server to another computer

You keep everything: devices, synced data and settings. Recordings are
separate and are not moved by this.

1. **On the old computer**, make a backup of the server's data, as described in
   "Back up and restore" in the install guide, and copy the backup file to the
   new computer. Do not start anything yet.
2. **On the new computer**, follow the install guide from Step 1 up to the end of
   Step 6 (install Docker Desktop, make the folder, write `compose.yaml` with
   the *new* computer's address). Do not skip Step 6: the address in it must be
   the new computer's.
3. **Restore the backup** there, as described under "Restore a backup" in the
   install guide.
4. **Open the admin page** at the new address. Log in with your usual password
   (the restore brings it with it). Review the **Devices** list and remove any
   device you had already removed from the old server, because a restore pairs
   it again.
5. **Switch the old server off** with `docker compose down` in its folder, so
   that your devices do not find two. Do not use `-v` yet. When you are sure
   everything works on the new computer, remove the old one using the install
   guide's "Uninstalling".
6. **Tell your devices the new address.** If the new computer took over the old
   address (you can arrange that in your router), nothing more is needed. If
   not, each device still remembers the old address. In the app, pair with the
   server again: choose **Pair with a server**, enter the new address, and a
   new pairing code from the new admin page. Because the new computer holds the
   same server data, the device keeps its place in the list and its synced
   data. The app shows the server's name and address and asks you to confirm;
   read it, check the address is the new computer's, and confirm.
7. **Recordings:** copy the `recordings` folder (or the Docker storage area)
   across yourself if you want to keep them. The server finds recordings it
   did not write in the folder, but it does not guess at what they were.

If the move fails, the old computer still works: start it again with
`docker compose up -d` and your devices carry on as before.

## Still stuck

Keep the output of `docker compose ps` and
`docker compose logs --tail 100 veylo-server`. The log never contains your
provider passwords or device tokens, so it is safe to share when asking
someone for help. The admin password shown the first time is the one exception; if you
share an old log, check it does not still contain a password that works.
