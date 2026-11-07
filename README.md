# FloodDM

A Vencord plugin that sends the same message many times in a DM, paced to stay
inside Discord's write rate limit.

Discord has no bulk endpoint for this. Every message is one POST to
`/channels/{id}/messages`, and that endpoint is rate limited per channel with
a shared scope, so the plugin paces the sends itself and honours exactly what
Discord asks for when it pushes back.

## What it does

- Right-click a message in a DM, or right-click the DM in the sidebar, and
  pick **Flood This DM**.
- A box opens asking for the message to send. Type it and press **Flood**, or
  press **Cancel** to close the box without doing anything.
- The message is sent the configured number of times, spaced out so Discord
  does not throttle the account.
- A progress window shows how many have gone out, the current speed, the
  estimated time left, and how many times Discord throttled the request.
  Press **Cancel** at any point to stop it between sends.

## Install

Vencord has no runtime plugin folder. A user plugin has to be compiled into the
Vencord bundle, so installing this means rebuilding Vencord once. The script
below does the whole thing.

### Option 1: the script (recommended)

```bash
git clone https://github.com/Harishisonline/vencord-flood-dm.git
cd vencord-flood-dm
./install.sh
```

The script detects whether you use the regular Vencord installer or the
standalone Vencord that ships with Discord for Mac. Force it if the detection
is wrong:

```bash
./install.sh --standalone   # Discord for Mac (standalone Vencord)
./install.sh --regular      # regular Vencord (Windows, Linux, Mac installer)
```

Requirements: `git`, `node` (20 or newer), and `pnpm`. See
[Vencord's install guide](https://docs.vencord.dev/installing) for how to get
those. On macOS, `corepack enable` gives you `pnpm` if you do not have it.

Restart Discord afterwards. The plugin appears under **Settings > Plugins >
FloodDM**, and it starts **enabled**.

**After a Vencord update, re-run the script.** Vencord's auto-updater
replaces `renderer.js` with the official build, which does not contain this
plugin, so the plugin disappears until you install it again. The script prints
this reminder every time it runs.

### Option 2: manual

1. Clone and build Vencord following the
   [official guide](https://docs.vencord.dev/installing).
2. Copy `src/userplugins/floodDm/index.tsx` from this repo into your Vencord
   checkout, at `src/userplugins/floodDm/index.tsx`.
3. Run `pnpm build` (or `pnpm build --standalone` for Discord for Mac).
4. Install the build with `pnpm inject`, or copy `dist/renderer.js` over the one
   in your Vencord install directory.

## Using it

1. Open the DM. You do not need to scroll to anything.
2. Right-click **a message in the DM**, or right-click **the DM in the
   sidebar**.
3. Pick **Flood This DM**.
4. Type the message, then press **Flood**.

## Settings

| Setting | Default | What it does |
| --- | --- | --- |
| Messages to send | 150 | How many copies of your message to send in one flood. |
| Send requests at once | 3 | How many sends are in flight at the same time. |
| Gap between sends | 250 | Milliseconds between the start of each send. |
| Ask for confirmation | on | Ask before flooding. |

Each numeric setting has its own **Apply** button, shows **Saved** after you
apply it, and shows the value that is currently saved underneath it, so you
can always see what the plugin is actually using.

## Limits and behaviour worth knowing

**There is a hard limit.** Discord limits how fast a channel can be written
to. The measured numbers against the live API on Discord 0.0.421:

| Setup | Rate | Throttles in 40 |
| --- | --- | --- |
| Sequential, 300ms apart | 1.01/s | 0 |
| 3 in flight, 250ms apart | 2.67/s | 5 |
| 4 in flight, 200ms apart | 1.58/s | 18 |
| 6 in flight, 150ms apart | 2.01/s | 20 |

Pushing harder does not make it faster, it just makes Discord refuse more.
The plugin therefore paces itself, widens the gap when Discord pushes back,
and never speeds up again on its own, because Discord's refill rate is fixed
and waiting earns no credit.

**Discord's rate limit is per channel.** Flooding one DM does not slow down
sends in another, but the limit is shared across the account, so heavy
sending anywhere counts against you.

**A flood is visible to the other person.** They get every message, in order,
at the speed the plugin manages. There is no undo and no recall once Discord
has accepted a message.

**Blocked people cannot be messaged.** If the other person has blocked you,
or you have blocked them, Discord refuses the send with error `50007`. The
plugin reports those as failures and stops counting them as sent.

## Safety

- The plugin only sends to the DM you right-clicked.
- The message is sent exactly as typed. There is no formatting or rewriting.
- Discord's "delete" is a hard delete. There is no undo and no trash folder.
- You can cancel a run at any point from the progress window.

## Changelog and development notes

- [CHANGELOG.md](CHANGELOG.md) lists every change between releases.
- [betterment.md](betterment.md) explains what was slow or broken, the
  measurements behind each fix, and what is still on the list.

## Disclaimer

This is an unofficial plugin and is not affiliated with Discord or Vencord.
Sending many messages is a normal Discord operation that the client itself
allows, but you are responsible for your account. Use it on your own account
and only where it is wanted.

## Uninstall

Remove the plugin and rebuild, or reinstall the stock Vencord build. The
`install.sh` script keeps a `renderer.js.bak` next to the file it replaced in a
standalone install.
