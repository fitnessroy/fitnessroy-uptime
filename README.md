# fitnessroy-uptime

Checks that [fitnessroy.com](https://fitnessroy.com) is answering, from outside
the house, and shouts if it is not. That is the whole repository.

## Why this exists at all

The blog is served from a k3s cluster at home, reached through a Cloudflare
tunnel. Two things already watch it, and both share a blind spot:

- **Uptime Kuma** runs *in* that cluster.
- **The tunnel watchdog** (`cloudflared/manifests/reconnect-watchdog.yaml` in
  `k3s-apps`) also runs in that cluster, and raises alerts through Alertmanager
  → RoyTools → Telegram.

Every one of those paths leaves the house through the same uplink. When the
uplink itself drops, the blog goes dark **and** every mechanism that would
report it goes dark at the same moment. The failure silences its own alarm.

So this runs on GitHub's machines instead. Nothing here touches the house.

## Why the repository is public

Actions minutes are metered on private repositories and free on public ones. A
five-minute check is about 8,600 billed minutes a month private, and nothing
public. Since the only thing this repo reveals is a domain name that is already
public, that is a cheap trade.

There is no secret in the code. `ROYTOOLS_API_KEY` is an encrypted repository
secret, and scheduled workflows never run from forks, so a pull request cannot
reach it.

## What happens when it fails

The probe asks for the front page six times, three seconds apart. Three or more
failures counts as down — the same "half the samples" rule the tunnel watchdog
uses, chosen because the fault this catches is usually partial rather than
total.

State lives in **an open issue labelled `outage`**, not in a file, so there is
nothing to commit and nothing to get out of sync:

| Probe | Open outage issue? | Action |
|---|---|---|
| down | no | open one, and push a notification |
| down | yes | nothing — already reported |
| up | yes | close it, and push a recovery notification |
| up | no | nothing |

That means one message when it breaks and one when it comes back, however long
it lasts in between. The issue history doubles as an outage log.

## Rehearsing it

An alarm nobody has watched fire is not known to work. Run the workflow by hand
with **`simulate_failure`** ticked and it forces a down reading: the issue opens
and the notification goes out, through the real code path rather than a mock.
Run it again without the tick and, since the site is up, it closes the issue and
sends the recovery notification.

Both are marked **`[ØVELSE]`** so a drill can never be mistaken for a real
outage later. An unmarked "site was down" entry that never happened would poison
the very history this repo exists to keep.

```sh
gh workflow run probe.yml --repo fitnessroy/fitnessroy-uptime -f simulate_failure=true
gh workflow run probe.yml --repo fitnessroy/fitnessroy-uptime
```

## Known limits

- **GitHub's scheduler is best-effort.** `*/5` is an upper bound, not a
  promise; under load runs are commonly several minutes late. This is a safety
  net, not a stopwatch.
- **Schedules are disabled after 60 days without repository activity.** An
  outage will wake it up, but if this repo goes quiet for two months, check that
  the workflow is still enabled.
- **One vantage point.** GitHub's runners land on one region, so a fault
  confined to a colo they do not use is invisible here — the same geometry
  problem documented in the tunnel watchdog. The watchdog's own external leg
  covers that case from inside.
- **Without `ROYTOOLS_API_KEY` set**, the GitHub issue is still opened, so the
  outage is still recorded and still notifies through GitHub — only the phone
  push is skipped.
