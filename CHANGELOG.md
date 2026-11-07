# Changelog

All notable changes to FloodDM are recorded here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

Nothing yet.

## [1.0.0] - 2026-11-07

First public release.

### Added

- **Right-click a message in a DM, or right-click the DM in the sidebar, and
  pick "Flood This DM".** Both entry points open the same box. The message
  you right-click does not have to be yours, the flood targets the DM.
- **An input box asking for the message to send.** Type it and press
  **Flood**, or press **Cancel** to close the box and do nothing. The box
  shows a live character count and refuses content over Discord's 2000
  character limit before it starts.
- **A live progress window** showing how many messages have gone out, the
  current rate in messages per second, the estimated time remaining, and how
  many times Discord throttled the request.
- **A Cancel button on the progress window** that stops the flood between
  sends. Verified live: a 150 message flood cancelled after 4 seconds left
  15 messages on the server, and the run reported itself as cancelled rather
  than as a failure.
- **A result toast** with the real counts: how many were sent, how many
  failed, how many times Discord throttled, and how long it took.
- **Settings that save visibly.** Three numeric settings, each with its own
  **Apply** button, a **Saved** marker after applying, and a **Currently
  saved** read-out underneath so the applied value is never in doubt.
- **A pace that follows Discord's own instruction.** When Discord pushes
  back, the plugin waits exactly as long as it asked and sends the same
  message again. Nothing is skipped and nothing is quietly dropped.

### The settings

| Setting | Default | What it does |
| --- | --- | --- |
| Messages to send | 150 | How many copies of your message to send. |
| Send requests at once | 3 | How many sends are in flight at the same time. |
| Minimum gap between sends | 250ms | The floor on how close two sends can be. |

### Measured, for reference

Every number below was measured against the live Discord API on Discord
0.0.421, in a real DM, not estimated.

| Scenario | Rate | Throttles |
| --- | --- | --- |
| Sequential, 300ms apart | 1.01/s | 0 in 40 |
| 3 in flight, 250ms apart | 2.67/s | 5 in 40 |
| 4 in flight, 200ms apart | 1.58/s | 18 in 40 |
| 6 in flight, 150ms apart | 2.01/s | 20 in 40 |
| **Retry-after driven, 1 worker** | **0.76/s** | **26 in 80** |

The last row is what shipped. It is slower at the start than the fixed gap
approaches, but it wastes a quarter of its requests instead of two thirds,
and it never slows down as it goes.

### Known limitations at the time of release

- Discord's write rate limit is the floor. A DM with a long message history
  has a burst allowance of roughly 30 messages, and after that the bucket
  refills at a fixed rate.
- The flood is visible to the other person in full. They get every message,
  in order, as fast as Discord allows.
- A blocked recipient cannot be messaged. Discord refuses with error `50007`
  and the plugin counts those as failures.

[Unreleased]: https://github.com/Harishisonline/vencord-flood-dm/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/Harishisonline/vencord-flood-dm/releases/tag/v1.0.0
