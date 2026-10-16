# Release notes

What changed in each version of the Veylo Server. The newest is first. To
update, see [Updating](README.md#updating).

## 0.5.0 — 15 October 2026

- **Skip the provider's "channel offline" screen in a recording.** When a
  recording caught the provider's stand-in screen, the server now says
  exactly where it is in the recording, so an app can offer to jump past it.
  Nothing is cut from the recording.
- **Animated "channel offline" screens are noticed too**, when they loop
  silently for 30 seconds. A still or animated screen that plays music or
  speech is still not noticed; a sudden change of picture size is noticed
  with or without sound.
- **A sudden change of picture size is acted on sooner**: after 8 seconds
  instead of 20. A still, silent screen is still given 20 seconds.

For developers: the server API is now 1.6.0. Each entry of a recording's
`stand_ins` has `start_ms` and `end_ms` (media time), `signal` can be
`looping_silent`, a recording's media playlists carry each stand-in as an
`EXT-X-DATERANGE` with `CLASS="tv.veylo.standin"`, and `/info` lists the
capability `standin_spans` (`docs/server-api.md` §9.1, §11.2). Older apps see
no change.

## 0.4.0 — 15 October 2026

- **Choose which source of a merged channel plays.** When a channel comes from
  more than one provider or stream, an app can now ask the server to play a
  particular one. If only you are watching, the server switches to it; if
  someone else is watching the same channel on another device, or it is being
  recorded, you join what is already playing and the app says which source
  that is. Switching never takes a provider over its connection limit: when
  there is no room, the channel keeps playing as it was.

For developers: the server API is now 1.5.0. `POST /api/v1/live` takes an
optional `preferred_source`, and `/info` lists the capability
`live_source_choice` (`docs/server-api.md` §10.2). Older apps send neither and
see no change.

- A remuxer that has already exited is never signalled again (its process id could belong to another program by then), and each leftover temporary file the start-up sweep removes is logged.

## 0.3.0 — 15 October 2026

- **A recording no longer quietly records a "channel offline" screen as the
  programme.** When the channel's picture stands still and its sound goes
  silent for 20 seconds, or the picture suddenly changes size and stays that
  way, the server takes it for the provider's stand-in screen. If the
  recording has another source, it switches to it and tries the first one
  again after a few minutes. If it has none, it keeps recording (nothing is
  ever stopped, deleted or cut) and notes when the stand-in was shown. A still
  picture with sound, such as a radio channel, is never taken for one. Moving
  or animated "offline" screens are not detected yet.
- **A channel recorded from one provider can now fall back to another.** If
  the scheduled source fails and another of your providers carries a channel
  with exactly the same guide id, the server records from that one instead,
  within that provider's connection limit. The scheduled source is always
  tried first.
- The check costs little: the server looks at one picture every second or two,
  shrunk to a few pixels, and the sound once a second. In our measurement a
  1080p50 channel at 8 Mbit/s took about 2 to 4 % of one CPU core for it.
- **Smaller recordings.** A recording is the broadcast as sent: about 4 GB an
  hour for a 1080p60 channel. A new **Recording size** setting on the admin
  page (and in the apps, through the API) can make each recording smaller
  once it has ended: **Smaller** re-encodes the picture to HEVC at the same
  resolution (about a quarter of the size), **Smallest** also scales it to
  720p (about an eighth). Sound is never re-encoded. **Original**, the
  default, keeps recordings exactly as they are now. It applies to
  recordings that finish after the change.
- Shrinking is slow on a small computer: a one-hour recording can take one to
  three hours on a NAS or mini PC. The server shrinks one recording at a
  time, at the lowest priority, and stops whenever it is recording or
  someone is watching live, carrying on afterwards. Every part of a
  recording is re-encoded and checked before any of it replaces the
  original, and only while nobody is playing it; if any part fails, the
  whole recording is kept as it was. Pictures taller than 1080 lines and HDR
  are kept as recorded.
- While it re-encodes, the server uses up to about 1 GB of memory more than
  usual. If you run it with a memory limit, allow for that. The shipped
  `mem_limit: 512m` is too small: the server reads its limit at start and,
  under 1.5 GB, does not start the encoder (it would be killed every time).
  Recordings wait as recorded, nothing fails, and the admin page says "Shrinking
  is paused" next to Recording size. Change `mem_limit` to `2g` in
  `compose.yaml` if you turn Recording size on.
- The Storage card shows how much shrinking has saved, the recording being
  shrunk with its progress, and the last few results.
- **Record from: HD (720p) when available.** For a channel that has both an
  HD and an FHD source, recordings can use the HD one: smaller, at no extra
  work for the server.
- Measured on 60 s of 1080p60 inside the server image, on a 16-core host that
  was heavily loaded: Smaller (x265 `fast`, CRF 24) encoded at about 0.7×
  real time, Smallest (720p, CRF 26) at about 1.4×, using about 650 MB and
  460 MB of memory (x265 with two frame threads and a 20-frame lookahead; the
  encoder's address space is capped at 3 GB). The encoder's thread pool is sized from the CPUs the
  container may use: left to itself inside Docker Desktop it ran 2.5 times
  slower.

For developers: the server API is now 1.4.0. It adds `has_fallback` to
schedules, `stand_ins` and `shrink` to recordings, and the recording settings
`recording_size` and `source_quality` (`docs/server-api.md` §8.1, §9.1, §13).
All are additive; older apps ignore them, and a settings write from an older
app keeps the new settings as they are.

## 0.2.0 — 15 October 2026

Use it with Veylo for Android 0.1.23 or later and Veylo for iPhone, iPad and
Apple TV 1.0 (22) or later. Older apps keep working, unencrypted, until they
are updated.

- **Your devices now talk to the server over an encrypted connection.** The
  server makes its own identity the first time it starts (nothing to install,
  buy or configure) and answers encrypted and plain connections on the same
  port, so no compose file, port or firewall needs to change. Provider logins,
  the pairing token and synced settings no longer cross your home network in
  plain text, once the apps that use it are updated. Older apps carry on over
  plain http.
- **The admin page** shows a lock and the secure address at the top, a check
  code next to the pairing code, and a new **Secure connection** card: the
  server's identity, how many devices are encrypted, which ones are not, a
  switch to refuse unencrypted connections once every device is encrypted
  (it is on by default, so nothing stops working), and **Replace identity…**
  for the rare case that the server's data folder was copied. Replacing the
  identity signs every device out; each must be paired again.
- Your browser will warn the first time you open the https address, because
  the server made the identity itself. That is expected. See
  [Secure connection](README.md#secure-connection).
- The admin page has new colours, in the Veylo logo palette, on every section.
- Video still plays over the plain address for now: Apple's player cannot check
  this server's identity.
- A new guide, [Reach it from outside your home](docs/remote-access.md#reach-it-from-outside-your-home),
  shows how to get an https address with no browser warning using Tailscale.

For developers: the server API is now 1.3.0. It adds `tls_cert_sha256` to
`GET /api/v1/info`, the pairing link parameter `f`, the error `tls_required`,
the setting `allow_plain_http`, and `encrypted` on admin device entries
(`docs/server-api.md` §15.7). Media links are absolute `http://` links when
asked for over TLS.

## 0.1.3 — 14 October 2026

- **Rewind live TV further.** A channel watched through the server can now be
  rewound by 5 minutes, 30 minutes or 1 hour, chosen under **Storage** on the
  admin page. The default is 30 minutes (it was always 5). The apps need no
  update: they play whatever the server keeps.
- **Storage** now shows how much space the live rewind buffers take, next to
  the number of channels open.
- If free space runs low, live rewind buffers are shortened first, well before
  the point at which recordings stop. A recording is never cut to make room
  for them.

For developers: the server API is now 1.2.0. It adds the recording setting
`live_rewind_s`; `window_s` on a live session now follows it.

## 0.1.2 — 14 October 2026

- Some channels played on Android but not on iPhone, iPad or Apple TV, which
  said the format was not supported. Their sound was in an older kind of AAC
  that Apple devices cannot decode. The server now converts any sound that is
  not standard AAC into standard AAC as it records or shows a channel, so
  every device can play it. The picture is untouched, and channels that
  already used standard AAC are passed through exactly as before.

## 0.1.1 — 14 October 2026

**Admin page**

- A new **Live now** card lists who is watching what through the server, how
  long for, and how many of each provider's connections are in use. A
  **Stop** button next to each viewer ends that channel and frees the
  connection: useful when a device was left playing. Recordings appear in the
  list too, but are stopped from the **Recordings** card.
- The page now refreshes itself every few seconds while it is open.
- Setting up a new server asks for the one-time password from the log only
  once, and accepts it even if you copied a space before or after it.
- **Free space** under **Storage** no longer shows an impossibly large number
  on Docker Desktop. If the disk reports something that cannot be right, the
  page says *Unknown* instead, and recording carries on.

**Watching and recording**

- Changing channel through the server when every connection to a provider is
  in use no longer waits about ten seconds: the server lets go of your own
  previous channel straight away.
- Lowering a provider's connection limit now applies at once, including to a
  channel that was already waiting to connect.
- Leftover temporary files more than a day old are tidied away when the
  server starts.

**Apps**

- For **Stop** to keep a channel stopped, a device needs Veylo for Android
  0.1.21 or Veylo for iPhone, iPad and Apple TV 1.0 (20) or newer. An older
  app restarts the channel by itself a moment later.
- Older apps otherwise keep working with this server. Nothing needs pairing
  again.

For developers: the server API is now 1.1.0. It adds one live-session close
reason, `stopped_by_admin`.

## 0.1.0 — 13 October 2026

First release: sync between devices, recording, and watching live channels
through the server.
