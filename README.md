# BGP Hijacking Detection System

A real-time anomaly detection system that monitors live internet routing data (BGP) and flags suspicious changes that could indicate route hijacking, leaks, or misconfigurations — built on a live feed of over 75,000 real BGP announcements.

## Why This Matters

BGP (Border Gateway Protocol) is the system that tells internet traffic which path to take across the globe. It was built in the 1980s on a foundation of trust — routers simply believe whatever route announcements they receive, with no built-in verification of who actually owns an IP address block. This makes BGP hijacking possible: an attacker (or a misconfigured network) can announce ownership of someone else's IP space and redirect their traffic. Real-world incidents have taken down YouTube, rerouted traffic through foreign countries, and been used to steal cryptocurrency. This project builds a working detector for that exact class of attack.

## Results

| Metric | Value |
|--------|-------|
| Live BGP announcements captured | 75,098 |
| Unique IP prefixes tracked | 22,437 |
| Unique origin ASNs observed | 4,920 |
| Average path length | 4.54 hops |
| Total anomalies detected | 327 |
| Short-path anomalies | 319 |
| Origin-change anomalies (high severity) | 8 |

![BGP Anomaly Types Detected](images/anomaly_types.png)
*Breakdown of the two anomaly types flagged by the detection engine across all 75,098 announcements.*

![BGP Path Length Distribution](images/path_length_distribution.png)
*Distribution of AS path lengths across all captured announcements, with the alert threshold marked at 1 hop.*

## How It Works

1. The system connects to **RIPE RIS Live**, a free real-time stream of global BGP routing data, via websocket.
2. Over a 60-second window, every route announcement is logged — capturing the IP prefix, origin ASN (the network claiming ownership), and AS path length.
3. A trusted baseline is built from the captured data: for each IP prefix, the most commonly observed origin ASN is treated as "expected."
4. Every announcement is then re-checked against that baseline. Two anomaly types are flagged:
   - **Origin change** — the prefix is being announced by a different ASN than expected (the strongest hijack signal)
   - **Short path** — the announcement has an unusually short AS path, a pattern sometimes used to make a hijacked route look more attractive to other routers
5. High-severity alerts (origin changes) are cross-checked against RIPE's public ASN registry to identify the real organizations involved, then trigger a live desktop notification.

## Key Findings

- Out of 75,098 live announcements, only 8 showed an origin ASN change — confirming that real anomalies are rare relative to total BGP traffic, exactly as expected.
- Manually investigating those 8 alerts using RIPE's ASN lookup API showed most were explainable as **multi-homing or corporate ASN ownership overlaps** (e.g. two ISPs under a shared parent company), not confirmed hijacks — an important distinction, since anomaly detection flags *suspicious*, not *malicious*, activity.
- One pairing (Eastern New Mexico University vs. an unrelated hosting provider, CHECS) stood out as the most genuinely unexplained mismatch and would warrant deeper investigation in a real monitoring setup.
- The majority of flagged anomalies (319 of 327) were short-path alerts, reflecting how common single-hop announcements are for networks announcing their own address space — a reminder that detection thresholds need tuning to reduce noise in a production system.
- The system goes beyond static analysis by triggering real-time desktop notifications the moment a high-severity anomaly is detected, demonstrating a full detect-to-alert pipeline rather than just offline analysis.

## Tech Stack

- Python 3.14
- `websocket-client` — live connection to RIPE RIS Live
- Pandas — data wrangling and baseline construction
- Matplotlib, Seaborn — visualization
- `requests` — RIPE ASN ownership lookups
- `plyer` — real-time desktop alerting

## How To Run It

```bash
pip install pandas numpy matplotlib seaborn websocket-client requests plyer jupyter ipykernel
```

Then open `bgp_monitor.ipynb` and run all cells in order. The notebook connects live to RIPE RIS Live for a 60-second capture window, builds a baseline, runs detection, and fires real desktop notifications for any high-severity alerts found.

## Data Source

[RIPE RIS Live](https://ris-live.ripe.net/) — a free, public, real-time stream of global BGP routing announcements provided by the RIPE NCC, one of the five Regional Internet Registries that manage IP address allocation worldwide.
