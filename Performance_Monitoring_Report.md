# Performance Monitoring Report — Carburoam Application VM

## 1. System metrics collected

### Baseline (idle)

```bash
htop
```

| Metric | Value |
|---|---|
| Load average (1/5/15 min) | 0.29 / 0.28 / 0.26 |
| Memory used | 4.84G / 11.7G |
| Swap used | 0K / 12.7G |
| CPU per core | 0.4%–2.2% across 10 cores |

Interpretation: system is lightly loaded at rest, consistent with a single Streamlit app plus monitoring agents.

## 2. PostgreSQL performance test on a significant data volume

### Setup

```bash
sudo apt install -y postgresql postgresql-contrib
sudo -u postgres createuser --pwprompt perftest
sudo -u postgres createdb -O perftest monitoring_lab
```

Two related tables were created and populated using `generate_series` (no external dataset needed):

```sql
CREATE TABLE stations (
    id SERIAL PRIMARY KEY, name TEXT, city TEXT, postal_code TEXT
);
CREATE TABLE price_readings (
    id SERIAL PRIMARY KEY, station_id INTEGER REFERENCES stations(id),
    fuel_type TEXT, price NUMERIC(5,3), reading_date TIMESTAMP
);
```

| Table | Rows | Notes |
|---|---|---|
| `stations` | 5,000 | Simulated fuel stations across 5 cities |
| `price_readings` | 2,000,000 | Fuel price readings referencing a random station, over a 365-day window |

```bash
psql -c "SELECT pg_size_pretty(pg_database_size('monitoring_lab'));"
# → 166 MB
```

### Query comparison: simple vs. joined

**Simple query (no join)** — aggregate on a single table:

```sql
EXPLAIN ANALYZE
SELECT fuel_type, AVG(price)
FROM price_readings
WHERE fuel_type = 'SP95'
GROUP BY fuel_type;
```

Result: `Execution Time: 40.536 ms`. Plan used a parallel sequential scan (2 workers) with a filter on `fuel_type`, discarding ~555k non-matching rows out of ~666k scanned per worker.

**Joined query** — same aggregate, joined against the `stations` table to group by city:

```sql
EXPLAIN ANALYZE
SELECT s.city, AVG(p.price)
FROM price_readings p
JOIN stations s ON p.station_id = s.id
WHERE p.fuel_type = 'SP95'
GROUP BY s.city;
```

Result: `Execution Time: 55.472 ms`. Plan added a `Hash Join` step (building a hash table from all 5,000 stations) before the same parallel scan/filter/aggregate pipeline.

### Comparison summary

| Query | Execution time | Overhead vs. simple |
|---|---|---|
| Simple (single table) | 40.5 ms | baseline |
| Joined (two tables) | 55.5 ms | **+37%** |

The join overhead here comes almost entirely from the added `Hash` build step and the `Sort`/`Gather Merge` stage needed to group by a column from the joined table — the underlying scan of `price_readings` itself is nearly identical in cost between the two plans (both dominated by the same parallel sequential scan with the `fuel_type` filter).

### Sustained load test

```bash
export PGPASSWORD='<perftest_db_password>'  # retrieved from secure storage, never hardcoded
time (for i in {1..50}; do
  psql -h localhost -U perftest -d monitoring_lab -P pager=off \
    -c "SELECT s.city, AVG(p.price) FROM price_readings p JOIN stations s ON p.station_id = s.id GROUP BY s.city;" > /dev/null 2>&1
done)
```

**Note on credential handling**: the `perftest` database password used throughout this exercise is intentionally not reproduced in this report. It is stored in macOS Keychain (consistent with the practice already applied to the Wazuh dashboard credentials — see the Wazuh Security Monitoring Report) rather than committed to any document or version-controlled file.

Result: `real 0m5.344s` for 50 sequential runs (~107 ms per run including `psql` connection/process overhead, versus 55.5 ms for the query execution itself measured in isolation — the difference is connection setup cost per invocation, not query cost).

## 3. Confrontation with the existing monitoring server (Zabbix)

The load test above was run while cross-checking the Zabbix Agent2 (already deployed on this VM, per the Iteration 2 monitoring setup) via `htop` and the Zabbix web interface (**Monitoring → Latest data**, filtered to `carburoam-vm`).

**Observation**: the query load generated in this test (50 short JOIN queries over ~5 seconds) was too brief and too light (peak load average only reached 0.44, well under the configured 90%-utilization CPU trigger) to register as a visible spike in Zabbix's per-minute metric collection. This is itself a useful finding: **Zabbix's default polling interval (1 min for CPU/memory items) is not fine-grained enough to catch short-lived load spikes like a batch of quick queries** — only sustained load over at least one full polling interval would reliably show up. A tool like `pg_stat_statements` or PostgreSQL's own slow-query log would be a better complement to Zabbix for this kind of short-burst database performance monitoring, since Zabbix here is confirmed as sufficient for sustained-resource trends (as already validated with `stress-ng` in Iteration 2) but not for sub-minute query-level analysis.

## 4. Unrelated but significant finding during this exercise

While observing `htop` during the load test, an active `cloudflared` tunnel process was discovered running on this VM (`systemctl status cloudflared` → `active (running)`, connected to Cloudflare's network with live registered connections). Investigation confirmed this was leftover from a prior module (Infrastructure & Cloud) after the WordPress/nginx-proxy-manager containers it originally served were removed during the Iteration 2 cleanup — the tunnel service itself had kept running independently, orphaned, continuing to expose this VM to the public internet with no backend service behind it.

**Corrective action taken**:
```bash
sudo systemctl stop cloudflared
sudo systemctl disable cloudflared
```

This is flagged here rather than only in the security reports because it was discovered specifically through routine performance-monitoring observation (`htop`) — reinforcing that performance monitoring and security monitoring are complementary practices that surface different classes of issues from the same tools.

## Conclusion

- PostgreSQL performance on a 2M-row, 166 MB dataset is fast in absolute terms (tens of milliseconds), with joins adding a measurable but moderate ~37% overhead versus single-table queries at this scale.
- The existing Zabbix monitoring setup is well-suited to sustained resource trends but has a blind spot for short, bursty database load — a gap worth addressing if query performance becomes an operational concern as data volume grows.
- Routine system observation during this performance exercise surfaced an unrelated, orphaned internet-facing service, underscoring the value of periodically reviewing `htop`/`systemctl` output even when the stated goal of a session is something else entirely.
