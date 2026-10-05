# amnezia-tunneling-ru — operations reference

## What this is

A personal fork of [lib4u/amnezia-tunneling-ru](https://github.com/lib4u/amnezia-tunneling-ru),
forked to `whatever125/amnezia-tunneling-ru` and cloned to
`~/Development/amnezia-tunneling-ru` on 2026-09-04. Upstream builds daily,
auto-updated **Russian** domain/IP lists for Amnezia VPN's site-based split
tunneling — the goal being "RU sites go direct (fast, no VPN, no suspicious
foreign-IP logins to banks/gov), everything else through the tunnel." This
fork adds the same idea for **China**.

## How upstream works (read before touching anything)

Three scripts in `scripts/`, wired together by `.github/workflows/build.yml`,
sourced from `v2fly/domain-list-community` (geosite data) and `v2fly/geoip`:

1. **`extract_amnezia.py`** — walks ~28 hand-picked RU root categories from
   `domain-list-community/data`, recursively expands `include:` directives,
   dumps hostnames → `amnezia.json` (~1900 domains, no IPs yet).
2. **`fill_ips.py`** — resolves each domain via Russian DNS (77.88.8.8) + DoH,
   because **Amnezia only reads the `ip`/`ips` field on import — it never
   resolves domains itself unless a site is added by hand through the UI.**
   An imported domain with an empty `ip` field silently does nothing.
3. **`extract_ip.py`** — converts any plain CIDR list into Amnezia's JSON
   shape (`{"hostname": "<cidr-or-domain>", "ip": ""}`), collapsing adjacent
   subnets. Used both for the full `amnezia-ip.json` (v2fly/geoip's `ru.txt`,
   ~12,800 subnets, desktop-only) and for this fork's CN addition below.
4. **`derive_ip.py`** — the actual anti-bloat mechanism, used to build
   `amnezia-ip-lite.json` (~650 subnets, mobile-safe). For each domain's
   resolved IP: if it's on a *small* RU-hosted AS (≤ /18), pull in that AS's
   *entire* range (catches endpoints DNS alone misses — e.g. Ozon); if it's
   on a *large* carrier AS (Rostelecom etc.), take only the one covering
   prefix from `ru.txt` instead of exploding into the carrier's whole
   footprint; if the IP isn't even in RU (foreign CDN), add it as a `/32`.
   **The lever that keeps the list small is the AS-size cap, not domain
   curation** — worth remembering if this ever needs tuning.

Key gotcha already hit by upstream (see `fill_ips.py` comments): GitHub's
runners throttle outbound UDP/53, so DNS resolution during the build must go
over DoH (443) or it silently loses ~20% of domains. And full IP lists
(12,800 routes) simply fail to bring the tunnel up at all on Android/iOS —
too many routes for the mobile client; only the lite (~650) list works there.

## What was added: China (2026-09-04)

**Researched alternatives before picking a source** — see the conversation
that produced this (or ask the vps-repo Claude sessions) for the comparison,
but the short version:

| Source | Method | Routes |
|---|---|---|
| `v2fly/geoip` `cn.txt` (what naively extending the existing pipeline would use) | geoip DB dump | 14,735 |
| `gaoyifan/china-operator-ip` | live BGP, per-operator | 6,229 |
| **`misakaio/chnroutes2`** (chosen) | live BGP from 4 collectors, hourly, **route-aggregated** | **3,905** (from 61,623 raw BGP routes — supernetted 16×) |
| `fivesheep/chnroutes` (original, 2013) | static APNIC delegation | unmaintained since ~2018, Python 2 |

`chnroutes2` is not a raw dump — it's *already* the aggregation product we'd
otherwise have to build ourselves, and BGP-derived data is more accurate than
static registry allocation (catches blocks actually announced from China,
skips allocated-but-unused ones). At 3,905 routes it's smaller than this
fork's own existing `amnezia-ip.json` for Russia (12,800) — i.e. no worse
than what upstream already ships for desktop without complaint.

**Generated so far (manual, not yet wired into CI):**

```bash
curl -sSL "https://raw.githubusercontent.com/misakaio/chnroutes2/master/chnroutes.txt" -o /tmp/cn_cidrs.txt
python3 scripts/extract_ip.py /tmp/cn_cidrs.txt amnezia-ip-cn.json
```

Produces `amnezia-ip-cn.json` — 3,903 CIDRs, same Amnezia JSON shape as the
upstream files. `chnroutes2` updates hourly, so re-run this whenever a
refresh is wanted; nothing else in the repo needs to change for that.

**Not yet done, worth doing if this becomes more than personal-desktop use:**

- Wire the two-line generation above into `.github/workflows/build.yml` as a
  parallel step (same pattern as the existing `amnezia-ip.json` step, just
  pointed at `chnroutes2/chnroutes.txt` instead of `v2fly/geoip`'s `ru.txt`),
  so `amnezia-ip-cn.json` stays fresh automatically instead of via manual
  re-run.
- **3,905 routes is untested on mobile.** Upstream's own experience: 650
  works, 12,800 doesn't. If it turns out too many for a phone client to
  bring the tunnel up, the fix is to run chnroutes2's *already-aggregated*
  prefixes through `derive_ip.py`'s AS-size-cap logic instead of raw
  `cn.txt` (same lever upstream already uses for the RU lite list, just
  pointed at the smaller, better input) to shrink further.

## How to actually use it (personal, current state)

In Amnezia → Settings → Connection → Site-based split tunneling → mode
"Addresses from the list should not use VPN":
1. Import `amnezia-ip-lite.json` (RU, upstream, 650 subnets)
2. Import `amnezia-ip-cn.json` (CN, this fork, 3,903 subnets)

Import is additive — the two stack rather than one replacing the other.
