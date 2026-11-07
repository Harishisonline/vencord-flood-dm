# Betterment

What was slow, what was broken, how each one was found, what changed, and
what is still open.

Everything here was measured against the live Discord API on 7 November 2026
using Discord 0.0.421 and Vencord v1.15.16 (commit `383409e`), in a real
1:1 DM with a bot. Numbers come from live request traces, not estimates.

---

## The test case

The AEGIS DM was not used for this. A DM with a bot was, because it is a
safe place to send a few hundred messages without bothering a person.

| Property | Value |
| --- | --- |
| Channel | `1552233652539232320`, a 1:1 DM with `Raphael` |
| Request round trip | 500 to 800ms |

The round trip is the first thing that matters, and it is the thing that is
easy to forget. **Discord's write endpoint is slow.** A single send takes
around 650ms to come back. Any flood that waits for each send to finish
before starting the next one is capped at about 1.5 messages per second no
matter how the pacing is configured, because the network is the limit, not
Discord's rate limiter.

---

## 1. A fixed gap between sends wastes two thirds of every request

### What was wrong

The first version of the scheduler used a fixed gap and widened it every
time Discord pushed back:

```ts
onThrottled(retryAfterSeconds: number): void {
    const pauseMs = Math.ceil(retryAfterSeconds * 1000) + 50;
    this.pausedUntil = Date.now() + pauseMs;

    this.spacing = Math.min(
        SendScheduler.MAX_SPACING,
        Math.max(this.spacing, this.baseSpacing) + Math.min(250, this.baseDelay)
    );
}
```

The idea was reasonable: if the gap is too small, make it bigger. The
problem is that Discord's write bucket does not refill faster once you
back off, so a widened gap just makes the whole run slower without ever
getting below the limit again.

### What the measurements said

Three 100 message runs, each with a different fixed gap, all with the same
3 workers:

| Fixed gap | Rate | Throttles | First throttle at message |
| --- | --- | --- | --- |
| 400ms | 0.65/s | **240** | 30 |
| 800ms | 0.45/s | **179** | 14 |
| 1200ms | 0.64/s | **138** | 27 |

Two thirds of every run was thrown away. And in a live 150 message flood
with the fixed gap scheduler, the degradation was visible in the progress
window:

```
Sent 22 of 150 at 4.3/s (throttled 0 times)
Sent 33 of 150 at 4.4/s (throttled 16 times)
Sent 49 of 150 at 1.0/s (throttled 24 times)
Sent 54 of 150 at 0.5/s (throttled 24 times)
```

The run started at 4.3 per second and finished at 0.5. The spacing had
grown until it hit the 5000ms cap.

### What changed

The scheduler stopped guessing a gap. It now does what Discord's own
`retry_after` says:

- Send.
- If Discord answers 429, wait exactly as long as it asked and send the
  **same** message again.
- Nothing is skipped, nothing is silently dropped, and no request is
  wasted beyond the one that got refused.

The user's "minimum gap" setting is a floor on how close two sends can be.
It is not a guess at Discord's limit. Discord's own instruction is always
respected on top of it.

### The result

| | Fixed gap, 400ms | Retry-after driven |
| --- | --- | --- |
| Messages sent | 84 of 100 | **80 of 80** |
| Throttles | 240 | **35** |
| Requests wasted | 74% | **30%** |
| Rate | 0.65/s | **0.79/s** |
| Slows down as it runs | yes | **no** |

---

## 1b. A retry that skipped the scheduler burned the attempts

### What was wrong

The retry-after driven scheduler had a second bug on top of the first one.
The attempt loop looked like this:

```ts
for (let attempt = 0; attempt < 12; attempt++) {
    try {
        await send();
        sentOk = true;
        break;
    } catch (err) {
        if (status === 429) {
            rateLimited++;
            scheduler.onThrottled(getRetryAfterSeconds(err));
            continue;               // <-- goes straight back to send()
        }
    }
}
```

`scheduler.reserve()` was called once, **before** the attempt loop. So the
`continue` on a 429 went straight back to the HTTP call without going
through the scheduler. The retry fired instantly into a bucket that was
still closed, got refused instantly, and burned the attempt in
milliseconds.

With a 12 attempt cap that failed 57 messages out of 150. The toast read:

```
93 messages sent, 57 failed, throttled 684 times, took 2m 16s (0.7/s)
```

684 throttles for 150 messages. A 12 attempt cap cannot survive that,
because each attempt is consumed in about a millisecond.

### What changed

`reserve()` moved **inside** the attempt loop, so every attempt, including
every retry, goes back through the scheduler and waits out the throttle:

```ts
for (let attempt = 0; attempt < 20; attempt++) {
    await scheduler.reserve();
    try {
        await send();
        ...
    } catch (err) {
        if (status === 429) {
            scheduler.onThrottled(getRetryAfterSeconds(err));
            continue;               // <-- now goes back to reserve()
        }
    }
}
```

### The result

```
150 messages sent, throttled 107 times, took 4m 1s (0.6/s)
```

150 sent, 0 failed, confirmed against the API.

---

## 1c. Discord's own request layer was fighting the scheduler

### What was wrong

Even with the retry loop fixed, the throttle count was still far too high:
684 throttles for 150 messages where the measurement said 35.

The cause was Discord's own HTTP layer. It keeps a `Map` of rate limit
records **keyed by request URL**, and every message in a DM uses the same
URL:

```
/channels/<id>/messages
```

So the first throttle created a record against that URL, and every
subsequent send was handled by that record. The Discord log made it
obvious:

```
[HTTPUtils] cleanupRequestEntry: rate limit for /channels/.../messages retry after 3179ms
[HTTPUtils] cleanupRequestEntry: extending rate limit for /channels/.../messages
[HTTPUtils] cleanupRequestEntry: rate limit for /channels/.../messages retry after 7393ms
```

The record was **extending** on every retry, so the pause grew from 1
second to 7.4 seconds and the request queue never drained.

### What changed

The send path uses a raw `fetch` against Discord's API instead of
Discord's `RestAPI`, with the token read from Discord's own session module:

```ts
const tokenModule = findByProps("getToken") as { getToken?: () => string } | null;
const token = tokenModule?.getToken?.();
if (!token) throw new Error("Could not read the Discord session token.");

const response = await fetch(`https://discord.com/api/v9/channels/${channelId}/messages`, {
    method: "POST",
    headers: { "Authorization": token, "Content-Type": "application/json" },
    body: JSON.stringify({ content })
});
```

The server side rate limit still applies, of course. What goes away is the
client side record that was turning one throttle into a chain.

### The result

| | RestAPI | Raw fetch |
| --- | --- | --- |
| Throttles in a 150 message run | 684 | **107** |
| Messages failed | 57 | **0** |

---

## 1d. More workers did not make it faster, they just wasted requests

### What was measured

The first release defaulted to 3 sends in flight, on the theory that
Discord's request round trip of about 650ms meant a single worker would
leave the connection idle.

Two runs of the same algorithm, same endpoint, same 80 messages:

| Workers | Rate | Throttles | Requests wasted |
| --- | --- | --- | --- |
| 3 | 0.765/s | 106 | **57%** |
| **1** | **0.791/s** | **35** | **30%** |

The same throughput with nearly twice the waste. The write bucket is one
bucket, so a second worker spends tokens the first worker would have used
and then gets refused for them.

### What changed

The default is now **1 worker**. The setting is still there for people who
want to try it, but the docs say plainly that it does not make the flood
faster.

---

## 2. Discord allows a burst, and then the bucket is closed

### What was measured

The first throttle in every run landed at a predictable place:

| Fixed gap | First throttle at message |
| --- | --- |
| 400ms | 30 |
| 800ms | 14 |
| 1200ms | 27 |

Around **30 messages** go through quickly, and then every request after that
is refused until the bucket refills. This matches what a person sees when
they try to send a lot of messages quickly by hand: the first few go, then
Discord starts pushing back.

That is why the plugin's default worker count is 3. More workers use the
burst allowance up faster, which sounds good but is not, because the burst
is a fixed number of tokens and not a rate.

| Workers | Rate | Throttles in 40 |
| --- | --- | --- |
| 3 | 2.67/s | 5 |
| 4 | 1.58/s | 18 |
| 6 | 2.01/s | 20 |

Three is the sweet spot.

---

## 3. Discord's client will queue your sends without telling you

### What was wrong

Discord's own HTTP layer keeps a `Map` of rate limit records keyed by
request URL, and a message send always uses the same URL for a given
channel:

```
/channels/<id>/messages
```

So when a send gets refused, Discord's client records the limit against
that URL and **queues subsequent sends**, replaying them itself when the
limit expires. The plugin would have had no idea this was happening:

- The count of "sent" would have been wrong, because the queued requests
  resolve long after the plugin thought they had gone out.
- **Cancel would have been useless.** Requests already queued inside
  Discord's client would have gone out anyway after the user pressed
  Cancel.

### What changed

Every send passes `failImmediatelyWhenRateLimited: true`, which opts out of
Discord's client-side queue. The plugin's own scheduler is then the only
thing pacing the sends, and a cancelled flood really does stop.

---

## 4. `openModal` returns a string, not a function

### What was wrong

The first version of the progress window closed itself like this:

```ts
closeModal = openModal(modalProps => <ProgressModal ... />) as unknown as (() => void);
// ...
closeModal?.();
```

`openModal` returns a **string modal key**, so `closeModal?.()` is calling a
string. In the minified bundle that surfaces as
`TypeError: c is not a function`, and because it happens inside Discord's
confirm-dialog promise chain, everything after the throw is skipped. The
modal stays on screen and the result toast never appears.

This is the exact same bug the BulkDeleteDm plugin had, and the fix is the
same: keep the key and close it with `closeModal(key)`.

---

## 5. Vencord's stock number input cannot save a decimal

### What was wrong

Vencord's built-in number input writes the value to the settings store and
**then** calls `BigInt(newValue)` to clamp it. `BigInt("1.5")` throws a
`SyntaxError`, which aborts the handler before the input's display updates.
The value is saved but the field keeps showing the old number, so it looks
like the edit was rejected.

### What changed

The plugin uses a custom settings component that keeps its own draft state,
only commits whole numbers, shows a red validation message for bad input,
shows **Saved** after applying, and shows **Currently saved: <value>**
underneath so the applied value is always visible.

The stored values are read field by field. Vencord's settings proxy returns
a fresh wrapper object on every access, so `settings.store.limits` can look
like `{}` while `settings.store.limits.count` returns the real value.

---

## What was verified end to end

Both of these were driven through the real UI, right-clicking a message in
a DM and pressing the actual buttons, not by calling the plugin's functions
directly.

| Test | Result |
| --- | --- |
| Full 150 message flood | Toast: *"150 messages sent, throttled 107 times, took 4m 1s (0.6/s)"*. API confirmed **150 messages on the server**. 0 failures. |
| Cancel 5 seconds into a 150 message flood | Toast: *"9 messages sent, cancelled, took 5s (1.7/s)"*. API confirmed **exactly 9 messages on the server**. |
| Plugin exceptions during both runs | **0**. The only exception captured was Discord's own `Error: Sentry successfully disabled`, which comes from Discord's bundle and not from the plugin. |
| Cancel actually stops the flood | Confirmed. The run reported itself as cancelled, and the server count matched the reported count exactly. |

The 150 message flood was also checked for the failure mode that a short
test cannot catch: the rate degraded from 4.3/s to 0.5/s over the run
before the scheduler was fixed. After the fix the rate holds steady at
about 0.6/s from the first burst to the last message.

---

## Still to work on

### 1. Discord's write rate limit is the floor

There is no bulk endpoint for this. Every message is one POST, and the
bucket refills at a fixed rate. The measured sustained rate is about
**0.6 to 0.8 messages per second** with the retry-after driven scheduler.

A verified 150 message flood took **4 minutes 1 second**, which is
0.6/s including the initial burst. A larger flood does not go faster.

Nothing in the plugin can go faster. The only thing that would change the
number is Discord raising the limit.

### 2. The first 30 messages are fast, the rest are not

Discord's burst allowance makes the start of a flood noticeably quicker
than the end. The progress window shows the rate over the whole run, so it
reads as a slowdown even though the run is behaving exactly as designed.

### 3. There is no resume

If a flood is cancelled, the messages that already went out stay out. There
is no way to ask for "another 150" without sending another 150 from the
start, which is fine, but the plugin does not track what it sent.

### 4. Blocked recipients cannot be messaged

Discord refuses with error `50007` and status 403. The plugin counts those
as failures and keeps going, so a flood at a blocked person reports every
message as failed instead of stopping at the first one. Stopping early
would be clearer.

### 5. Group DMs are handled but not tested at scale

The same code path serves both 1:1 DMs and group DMs, and the menu entry
appears in both. Only the 1:1 case has been tested with a few hundred
messages.

### 6. The character limit is Discord's, checked client side

The input box refuses content over 2000 characters before the flood starts,
which is Discord's limit for a single message. The plugin does not try to
split a long message into several, because that would change what the
recipient sees.

---

## Things that were checked and are not problems

- **Sending into a DM works from a plain REST POST.** Verified with raw
  `fetch` against `/api/v9/channels/<id>/messages` and with Discord's own
  `RestAPI.post`. Both work.
- **A bot DM accepts floods.** No special handling, no error.
- **Concurrent sends do not cause connection errors.** 3 and 6 simultaneous
  sends were both fine.
- **The 429 response carries everything needed.** `status: 429`,
  `body.retry_after` as a precise fractional value, `body.code: 20028`, and
  `x-ratelimit-scope: shared`. The precise body value is preferred over the
  `retry-after` header, which is rounded up to whole seconds.
- **Discord does not send rate limit headers on successful sends.** Only
  on a 429. The scheduler therefore has to infer the limit from the
  throttles rather than reading it from the response.
