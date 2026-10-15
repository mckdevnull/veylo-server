# Using Veylo Server away from home

At home, your devices find the server by its address on your home network.
Away from home (on mobile data, at a hotel, at a friend's house) that address
does not work. This guide shows the two supported ways to reach the server from
outside.

Read the next section first. It is the most important thing on this page.

## Never forward the server's port from your router

Your router may have a setting called "port forwarding", "virtual server" or
"NAT". It can make a port on your home computer reachable from anywhere on the
internet.

**Do not forward port 8787, the server's port, to the internet. Ever.**

Why:

- The server also speaks plain `http` on that port. Your admin password and
  devices that have not updated would travel across the internet where anyone
  on the path could read them. Its built-in encrypted connection protects the
  devices that use it, but a server on the open internet is still not what it
  is built for.
- The server is built for a home network. It is not designed to be exposed to
  strangers who have all day to try it.

The two ways below keep the server hidden from the internet. Use one of them.

## Which one to use

| | Tailscale | Reverse proxy with https |
|---|---|---|
| Difficulty | Easy | Harder |
| Needs | A free Tailscale account, and the Tailscale app on the server's computer and on each device | A domain name, a certificate, and a second program (the proxy) |
| Visible to the internet | No | The proxy is |
| Recommended for most homes | Yes | Only if you already know how |

## One thing to know first: when the app accepts plain http

The Veylo app accepts a plain `http://` address without complaint only when
the address is **private**. These are private:

- `10.x.x.x`, `172.16.x.x` to `172.31.x.x`, and `192.168.x.x` (ordinary home
  network addresses)
- `100.64.x.x` to `100.127.x.x`, which includes every Tailscale address (they
  all start `100.`)
- `169.254.x.x` and `127.x.x.x`
- names ending `.local`, `.home.arpa`, `.internal` or `.localhost`, and
  single-word names with no dot, such as `veylo`

Everything else counts as **public**, including names ending `.lan` and
**`.ts.net`** (the names Tailscale gives your devices). The app cannot tell, by
looking at a name, that Tailscale protects it, so it treats the name as public.
So:

- `http://100.101.102.103:8787` is accepted (a Tailscale address).
- `https://anything` is accepted.
- `http://my-computer.tail1234.ts.net:8787` is **refused** by the server when
  you try to use it as the remote address, and the app would warn you that the
  connection is not encrypted. Use the `100.x.y.z` address instead.

## Reach it from outside your home

This is the easiest way to get an `https` address that every browser and phone
trusts with no warning, with Tailscale doing the encryption. It works on top of
Option 1 below: you still install Tailscale on the server's computer and on each
device.

1. In the Tailscale admin console (`https://login.tailscale.com/admin/dns`),
   turn on **MagicDNS** and **HTTPS Certificates**.
2. **Give the server's computer a neutral name** in Tailscale (for example
   `hub`, not `smiths-livingroom`). The certificate Tailscale gets for it is
   written to public Certificate Transparency logs, and anyone can read the
   name there.
3. On the server's computer, run:

   ```sh
   tailscale serve --bg 8787
   ```

   `--bg` keeps it running after a restart. Tailscale now answers on
   `https://<computer-name>.<your-tailnet>.ts.net` and passes the traffic to
   the server.
4. Tell the server this is its remote address, as in Step 3 of Option 1, but
   with the `https` name:

   ```yaml
         VEYLO_REMOTE_URL: "https://hub.your-tailnet.ts.net"
   ```

   then `docker compose up -d`. Devices paired after this learn both
   addresses from the QR code.

Devices accept this address because Tailscale's certificate is publicly
trusted. At home they keep using the server's own identity. If the computer is
asleep or Tailscale is off, there is no way in from outside. Do not use
Tailscale Funnel for this: it would put the server on the public internet.

## Option 1: Tailscale

Tailscale makes a private, encrypted network between your own devices, wherever
they are. Your server's computer and your phones join it, and then they can
reach each other as if they were on the same home network. The server stays
invisible to everyone else.

### What you need

- A Tailscale account (there is a free plan for personal use; see Tailscale's
  own site for current terms).
- The Tailscale app installed on the computer that runs the server, and on each
  phone, tablet or computer that you want to use away from home. Sign in to
  the same account on all of them.
- A device that cannot run Tailscale cannot use this route; it works only at
  home.

### Step 1: Install Tailscale on the server's computer

1. Go to `https://tailscale.com/download` and install Tailscale for your
   computer (Mac or Windows).
2. Open it and sign in.
3. Leave it running. If the Tailscale app on that computer is switched off,
   or the computer is asleep, you cannot reach the server away from home.

### Step 2: Find the computer's Tailscale address

1. Open the Tailscale app on the server's computer, and look at its own entry
   in the list of machines.
2. Find its address. It starts with `100.`, for example `100.101.102.103`.
   Write it down. This guide uses `100.101.102.103`; use your own.

### Step 3: Tell the server about it

The server needs to know this address, so that it can hand it out when you
pair a device.

1. Open Terminal (Mac) or PowerShell (Windows), and go into the server's
   folder: `cd ~/veylo-server` on a Mac, `cd $HOME\veylo-server` on Windows.
2. Open `compose.yaml` in a plain-text editor (`nano compose.yaml` on a Mac,
   `notepad compose.yaml` on Windows).
3. Under `environment:`, below the `VEYLO_PUBLIC_URL` line, add a line for the
   remote address. Keep the spaces at the start exactly lined up with the line
   above it:

   ```yaml
         VEYLO_REMOTE_URL: "http://100.101.102.103:8787"
   ```

4. Save the file. Then apply the change:

   ```sh
   docker compose up -d
   ```

5. Open the admin page. Under **Addresses**, **Away from home** should now show
   your address instead of "Not set".

If the server does not start after this change, read its message with
`docker compose logs veylo-server`. A message such as
`VEYLO_REMOTE_URL: must be https:// unless it names a private or tailnet
address` means the address was not accepted: check you used the `100.` number
and not a `.ts.net` name.

### Step 4: Pair, or tell the app

A device paired after Step 3 learns both addresses automatically: the QR code
carries them both. For a television, type the home address in **Server
address** and the Tailscale address in **Remote address (optional)**.

A device that was paired **before** you did this knows only the home address.
Either type the new address into its **Remote address** field if the app
offers it in its server settings, or pair it again with a new code.

### Step 5: Try it

1. On the phone, open the Tailscale app and make sure it is connected.
2. Turn off Wi-Fi on the phone, so that it is using mobile data.
3. Open Veylo. It should reach the server and sync.

## Option 2: a reverse proxy with https

A reverse proxy is a program that sits in front of the server. People on the
internet talk to the proxy over an encrypted connection (https), and the proxy
passes their requests on to the server. This needs more setup and more
knowledge than Tailscale. If this paragraph is unfamiliar, use Tailscale.

What you need:

- A domain name, for example `veylo.example.com`, pointing at your home's
  internet address.
- A reverse proxy that obtains a certificate for that name. Caddy does this
  automatically; Nginx and Traefik can too.
- Your router forwarding port **443**, and only 443, to the **proxy**. Never
  port 8787.

What to do:

1. Make sure the proxy passes the `Host` header through unchanged. The server
   uses it to check who the request was meant for, and refuses a request made
   to a name it does not know. Many proxies do this by default; check yours.
2. Let the proxy wait for long requests. Devices ask the server "has anything
   changed?" and the server may hold the request open for up to 55 seconds
   before answering. A proxy that gives up after less than about a minute will
   cut these off.
3. Point the proxy at the server: `http://127.0.0.1:8787` if the proxy runs on
   the same computer.
4. Optional, and only if the proxy runs on the same computer: make the server
   reachable **only** from the proxy. In `compose.yaml`, change the line
   `- "8787:8787"` to:

   ```yaml
         - "127.0.0.1:8787:8787"
   ```

   Other devices on your network can then no longer use port 8787 directly, so
   your home devices must use the https address too (set `VEYLO_PUBLIC_URL` to
   it as well). If you would rather keep the plain-http address working at
   home, skip this step.
5. Set the remote address in `compose.yaml` to your https name:

   ```yaml
         VEYLO_REMOTE_URL: "https://veylo.example.com"
   ```

6. Leave `VEYLO_TRUST_PROXY` alone (do not add it) unless every request to the
   server reaches it through your own proxy and the proxy sets the
   `X-Forwarded-Proto` header itself. Set to `1` when the conditions do not
   hold, it lets any visitor claim the connection is encrypted.
7. Apply with `docker compose up -d`.

An example for Caddy, where `Caddyfile` is the whole file:

```
veylo.example.com {
    reverse_proxy 127.0.0.1:8787
}
```

This example has not been tested against the published server; check your
proxy's own documentation.

## What "Remote address" means in the app

When you pair a device, the app asks for two addresses:

- **Server address**: the address that works at home, for example
  `http://192.168.1.20:8787`. This is the address shown on the admin page.
- **Remote address (optional)**: a second address for use away from home, for
  example your Tailscale address or your https name.

The app tries the server at the first address and, if that does not answer,
at the second. That is why you never need to switch anything yourself when you
leave home: at home the first answers, away from home the second does.

Leave **Remote address** empty if you do not use the server away from home.

If you pair with the QR code, both fields are filled in for you from what the
server knows. That is why it matters to set `VEYLO_REMOTE_URL` before you
make the pairing code.

Away from home, the same safety applies as at home: the app first checks that
the machine at the address really is your server before it sends anything
private, so another network that happens to use `192.168.1.20` cannot collect
your login.

## If you are not sure it is safe

You can always leave remote access switched off. The server keeps working at
home, and your devices keep working away from home, using their own copies of
your providers and settings. They just do not sync, or reach recordings, until
they are back on your home network.

## Where to go next

- [Troubleshooting](troubleshooting.md)
- [Install on a Mac](install-mac.md) or [Install on Windows](install-windows.md)
