Now I've got everything I need to give you a definitive answer. **I found the bug**, and it's not what either of us initially suspected — the timezone offset in the config is actually correct-ish. The real problem is a **midnight-wrap-around bug in the Painless script.**

## What the query does

The outer query filters the `fix_session_schedule` index for docs where:
- `group == "53002"` (matches your watcher's group)
- `alert_status == "enabled"`
- The inline Painless script returns `true`

The script's job is to answer: *"Is `now` (UTC) currently inside the alert window, after correcting for DST?"* If yes, the doc survives and gets aggregated into `hosts` buckets. Downstream, the watcher takes those `hosts` (session IDs) and checks the heartbeats index for missing docs.

The script's design intent (once you decode it):
1. Config stores start/end as fixed `-05:00` (EST) offsets — a hardcoded assumption "NY local time, winter."
2. In summer (US on EDT), NY is actually `-04:00`, so the stored hours are one hour too late. The script detects summer via `paris_offset == 2` (CEST) and subtracts 1 from both start and end hours to compensate.
3. Then it does a naive `now.getHour()` vs `start_hour` / `end_hour` comparison, with minute-level checks only at the boundary hours.

## Why it failed for TRUMID at 6:21 PM NY on June 30

Look at how ES normalized your TRUMID config's end time:

```
_source:  "end_alert_time": "19:00:00-05:00"
fields:   "end_alert_time": "1970-01-02T00:00:00.000Z"   ← midnight, NEXT day
```

19:00 EST = 24:00 UTC = **00:00 of the following day**. So `doc['end_alert_time'].value.getHour()` returns **0**, not 19 and not 23.

Now trace the script at 2026-06-30 22:21 UTC (= 18:21 EDT):

```
paris_offset       = 2       (CEST — summer branch taken)
now.getHour()      = 22      (UTC)
nowNY.getHour()    = 18
ny_time_diff       = 22 - 18 = 4

start_alert_time (var) = doc.start.getHour()  = 10   (10:02 UTC)
end_alert_time   (var) = doc.end.getHour()    = 0    (00:00 UTC next day)  ← the midnight wrap

// paris_offset == 2 branch fires:
start_alert_time = 10 - 1 = 9        ✓ (9:02 EDT — this is actually correct)
end_alert_time   =  0 - 1 = -1       ✗ (nonsense hour)
```

Then every branch of the comparison fails:

| Check | Values | Result |
|---|---|---|
| `now > start && now < end` | `22 > 9 && 22 < -1` | false |
| `now == start && start == end` | `22 == 9` | false |
| `now == start && ...` | `22 == 9` | false |
| `now == end && ...` | `22 == -1` | false |
| else | | **return false** |

The doc gets filtered out. `hosts` aggregation returns zero buckets. No missing-heartbeat check ever runs. No alert fires.

The alert was never going to fire for TRUMID between 09:02 UTC and 23:00 UTC (essentially the entire intended window) because `end_alert_time = -1` is unreachable.

## This also breaks your other enabled sessions

**`FIX.4.4:MAP_DIACAP_PROD->MAP_BLP_PROD`** — same `19:00:00-05:00` end time → same wrap → same bug. Alert will never fire.

**`FIX.4.4:PRDHSDMCP->PRDADROITDMCP`** — end is `23:55:00-05:00` = `1970-01-02T04:55:00.000Z`. Also wraps past midnight. `getHour() = 4`. Summer branch: `4 - 1 = 3`. Start becomes `5 - 1 = 4`. Window collapses to `hour > 4 && hour < 3` — impossible. Also broken.

So all three enabled sessions in your CSV are silently broken in summer. In winter (`paris_offset == 1`), the else branch runs and end stays at `0`, still busted.

## Why your ad-hoc test returned nothing

Same reason. When you ran the query in Kibana Dev Tools during the day (any UTC hour between 5 and 23), the script returned `false` for every doc because `end_alert_time` was either `0` (winter path) or `-1` (summer path). No hits, no aggregation buckets.

## Other latent bugs I noticed

- **`ny_time_diff` wraps at UTC midnight.** `nowUTC.getHour() - nowNY.getHour()` gives `-20` when it's 02:00 UTC and 22:00 NY the previous calendar day. Not currently triggering your failure but will bite you.
- **Using Europe/Paris as a proxy for US DST is fragile.** The US and EU switch DST on different dates. Between roughly March 8–29 and Nov 1–2 each year, `paris_offset == 1` AND NY is on EDT — the code has a specific `else if` for exactly this case (`paris_offset == 1 && ny_time_diff == 4`), which shows the author knew about it, but the whole approach is brittle. NY's DST status should be read directly.
- **Comment says "winter" but the branch is Paris summer.** Just misleading.
- **The hour-only comparison can never handle a window that crosses UTC midnight**, regardless of the DST fix.

## Recommended fix

The cleanest fix — don't decompose to hours at all, and don't use Paris as a proxy. Compare NY-local time directly:

```painless
ZonedDateTime nowNY = ZonedDateTime.ofInstant(
    Instant.ofEpochMilli(new Date().getTime()),
    ZoneId.of('America/New_York')
);
int nowMin = nowNY.getHour() * 60 + nowNY.getMinute();

// Interpret the stored ZonedDateTime in its ORIGINAL zone (-05:00),
// then treat those hour/minute components as if they were NY-local.
// This preserves the "5:02 NY wall clock" intent regardless of DST.
def s = doc['start_alert_time'].value.withZoneSameInstant(ZoneId.of('-05:00'));
def e = doc['end_alert_time'].value.withZoneSameInstant(ZoneId.of('-05:00'));
int startMin = s.getHour() * 60 + s.getMinute();
int endMin   = e.getHour() * 60 + e.getMinute();

if (startMin <= endMin) {
    return nowMin >= startMin && nowMin <= endMin;
} else {
    // window crosses local midnight
    return nowMin >= startMin || nowMin <= endMin;
}
```

Two things this fixes: (a) it treats the stored value as its original wall-clock instead of ES's UTC normalization, so 19:00 stays 19:00 and never becomes 0; (b) it collapses "am I inside the window" to a single monotonic minute-of-day compare that handles wrap-around explicitly.

**Even simpler if you're willing to change the schema:** store `start_alert_time_minutes_of_day` and `end_alert_time_minutes_of_day` as plain integers (0–1439) in NY local time. Then the whole script becomes three lines and DST is a non-issue because you're comparing NY-local minutes to NY-local minutes.

## What to do right now to verify

Before rewriting, prove the diagnosis in Kibana Dev Tools:

```json
GET fix_session_schedule/_search
{
  "size": 5,
  "query": { "term": { "target.keyword": "FIX.4.4:PRODHSDMCP->TRUMID" }},
  "script_fields": {
    "stored_end_hour":   { "script": "doc['end_alert_time'].value.getHour()" },
    "stored_start_hour": { "script": "doc['start_alert_time'].value.getHour()" }
  }
}
```

You should see `stored_end_hour: 0` and `stored_start_hour: 10`. That's the smoking gun.
