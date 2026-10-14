# Release notes

What changed in each version of the Veylo Server. The newest is first. To
update, see [Updating](README.md#updating).

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
