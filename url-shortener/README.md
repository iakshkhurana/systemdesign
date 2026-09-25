# URL Shortener (Amazon-style)

Internal shortener for marketing / affiliate links (`amzn.to/...`). Read-heavy, global, bursty; every click must be counted.

![Architecture](url-shortener/amazon-url-shortener.png)

## Functional requirements
- Create short link for an Amazon URL — optional alias, tracking tags (affiliate, campaign, source), expiry
- Redirect to original URL, preserving tracking tags
- Record every click (time, country, device, referrer)
- Show click stats to link owner
- Expire links when a campaign ends

## Non-functional requirements
- Read-heavy: ~1000 clicks per link created
- Redirect latency < 50 ms globally
- Highly available — a broken link = lost sales
- Never lose a click — affiliates are paid on them
- Absorb bursts (sales, viral posts)
- Secure: Amazon domains only, authenticated creation

## Back-of-envelope
| Item | Estimate |
|---|---|
| Links created | 1M/day → ~10 writes/sec |
| Clicks | 1B/day → ~10k/sec avg, **100k/sec peak** |
| Link storage | 1M × 500 B ≈ 500 MB/day ≈ 180 GB/yr (small) |
| Click storage | 1B × 100 B ≈ **100 GB/day ≈ 36 TB/yr** (analytics store) |
| Hot cache | 20% of links = 80% of traffic → few GB per edge |

Conclusion: reads → cache at the edge; click writes → async queue + separate analytics store.

## API
| Method | Path | Notes |
|---|---|---|
| `POST` | `/links` | `{ long_url, alias?, tags?, expires_at? }` → `{ short_url, code }` (auth required) |
| `GET` | `/{code}` | `302` → `long_url + tags`. 302 not 301 so every click is counted |
| `GET` | `/links/{code}/stats?from=&to=` | Aggregated clicks by hour / country / device |
| `DELETE` | `/links/{code}` | Disable link |

## Design
**Create path (low traffic):** Marketer → Create API (validates Amazon domain) → Link DB (sharded by code). Codes come from a replicated Key Generation Service that pre-generates unique codes; servers fetch them in batches.

**Redirect path (hot):** User → Edge/CDN cache → `302` with tags appended. On miss, fetch from Link DB and cache. Every click is pushed to a queue (Kafka/Kinesis); the redirect never writes to a DB synchronously. Consumers aggregate into an Analytics DB (per link, per hour, by country/device); raw events go to S3.

## Scaling & failure
- Viral link: edge absorbs reads, queue absorbs writes, primary DB sees nothing
- Queue down: buffer locally and retry — still redirect (lost click < broken link)
- KGS replicated, never on the hot path
- Expiry checked at redirect time, cleaned by a background job

> One line: a read-heavy key-value lookup at the edge, plus a write-heavy event stream for analytics — glued by the redirect.
