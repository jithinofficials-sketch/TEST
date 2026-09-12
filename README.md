# Plan v3: Migrate ClickHouse to a New Droplet, Add Log TTLs, Reconnect Monitoring

> Builds on the completed DO-Volume migration (`clickhouse-volume-migration-runbook-v3.md`), which is done and stable. **v3 fixes a real ordering bug in v2**: several steps said "do this on the new Droplet" before the step that actually creates the new Droplet. Every step below is now sequenced so nothing is asked of a machine that doesn't exist yet.

---

## Naming

| Machine | Container name | Role |
|---|---|---|
| Old Droplet (`bucks-analytics-clickhouse-server`) | `clickhouse-prod` | Original container, stopped, kept as rollback for the *first* migration |
| Old Droplet | `clickhouse-prod-new` | **Current live production** container, on the DO Volume |
| New Droplet | `clickhouse-prod-v2` | The container we are about to create |

---

## Step 0 — Finish password rotation FIRST, on the current live system (old Droplet)

Do this before anything else in this plan. User passwords live inside the data directory (`access/`), so whatever passwords are active on `clickhouse-prod-new` at the moment of the final data copy (Step 5) are exactly what gets carried to the new Droplet. Rotating afterward means redoing part of the copy.

- [ ] `analytics_prod_user`'s new password set and confirmed (`--user analytics_prod_user --password '<new>' --query "SELECT 1"` succeeds)
- [ ] `analytics_staging_user`'s new password set and confirmed
- [ ] Analytics backend's `.env` updated with new passwords, confirmed writing successfully
- [ ] `admin`'s password rotated too (its old value was found exposed in plaintext documentation)

---

## Step 1 — Create the new Droplet

In the DigitalOcean dashboard: 1 vCPU / 2GB RAM / 50GB NVMe SSD. Note its IP once created — every step from here on needs it. We'll refer to it as `<NEW_DROPLET_IP>`.

---

## Step 2 — Basic setup on the new Droplet (now that it exists)

SSH in for the first time from your laptop:
```bash
ssh root@<NEW_DROPLET_IP>
```
Then, on the new Droplet:
```bash
# Docker
curl -fsSL https://get.docker.com | sh

# tmux for a disconnect-proof session
apt-get install -y tmux
tmux new -s chmigrate2

# directories the later steps need
mkdir -p /var/lib/clickhouse-data
mkdir -p /etc/clickhouse-server/config.d
```

---

## Step 3 — Grant the OLD Droplet SSH access into the NEW Droplet

The final data copy (Step 5) runs *from* the old Droplet *to* the new one, so the old Droplet's key needs to be trusted here. This step needs both Droplets to exist, so it correctly comes after Step 1.

**On the OLD Droplet**, get its public key (generate one for `root` if it doesn't have one yet):
```bash
ls ~/.ssh/id_ed25519.pub 2>/dev/null || ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```
Copy that full output.

**On the NEW Droplet** (you're already there from Step 2), paste it in:
```bash
mkdir -p ~/.ssh
echo "<paste the old Droplet's public key here>" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**Test it from the OLD Droplet:**
```bash
ssh root@<NEW_DROPLET_IP> "echo connected ok"
```
Should print `connected ok` with no password prompt.

---

## Step 4 — Create the TTL config file on the new Droplet

Still on the **new** Droplet (directory already created in Step 2):
```bash
nano /etc/clickhouse-server/config.d/log_ttl.xml
```
Paste:
```xml
<clickhouse>
    <trace_log>
        <database>system</database>
        <table>trace_log</table>
        <ttl>event_date + INTERVAL 7 DAY DELETE</ttl>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </trace_log>
    <text_log>
        <database>system</database>
        <table>text_log</table>
        <ttl>event_date + INTERVAL 7 DAY DELETE</ttl>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </text_log>
    <part_log>
        <database>system</database>
        <table>part_log</table>
        <ttl>event_date + INTERVAL 14 DAY DELETE</ttl>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </part_log>
    <metric_log>
        <database>system</database>
        <table>metric_log</table>
        <ttl>event_date + INTERVAL 14 DAY DELETE</ttl>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </metric_log>
    <asynchronous_metric_log>
        <database>system</database>
        <table>asynchronous_metric_log</table>
        <ttl>event_date + INTERVAL 14 DAY DELETE</ttl>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </asynchronous_metric_log>
    <query_log>
        <database>system</database>
        <table>query_log</table>
        <ttl>event_date + INTERVAL 30 DAY DELETE</ttl>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_log>
</clickhouse>
```
Save (`Ctrl+O`, Enter), exit (`Ctrl+X`). Verify:
```bash
cat /etc/clickhouse-server/config.d/log_ttl.xml
```

**Scope reminder:** this only affects the six named `system.*` tables (server-wide, not per-database). It does not touch `bucks_analytics_prod.*` or `analytics_staging.*`, and it is not a "whole storage" setting.

`config.xml` itself needs no action — it ships inside the ClickHouse image already.

---

## Step 5 — Copy the data from the old Droplet to the new one

Run from the **OLD** Droplet, now that Step 3 has made this possible.

**Optional online pre-copy (ClickHouse stays running, harmless, saves a little time later):**
```bash
export CH_NEW_DATA_DIR="/mnt/clickhouse_storage/clickhouse-data"
rsync -aHAXS --numeric-ids --exclude='/lost+found' --info=progress2 --stats \
  -e "ssh -o StrictHostKeyChecking=accept-new" \
  "$CH_NEW_DATA_DIR/" root@<NEW_DROPLET_IP>:/var/lib/clickhouse-data/
```

**Final, authoritative copy — this is the actual downtime window:**
```bash
docker stop clickhouse-prod-new
rsync -aHAXS --numeric-ids --delete-delay --exclude='/lost+found' --info=progress2 --stats \
  -e "ssh -o StrictHostKeyChecking=accept-new" \
  "$CH_NEW_DATA_DIR/" root@<NEW_DROPLET_IP>:/var/lib/clickhouse-data/
echo "rsync exit status: $?"     # must be 0
```

---

## Step 6 — Start ClickHouse on the new Droplet

Run on the **NEW** Droplet. Requires Step 4's file and Step 5's data copy to already be done — both are now satisfied.

```bash
docker run -d \
  --name clickhouse-prod-v2 \
  --restart unless-stopped \
  -p 8123:8123 -p 9000:9000 \
  -v /var/lib/clickhouse-data:/var/lib/clickhouse \
  -v /etc/clickhouse-server/config.d/log_ttl.xml:/etc/clickhouse-server/config.d/log_ttl.xml:ro \
  -e TZ=UTC \
  -e CLICKHOUSE_CONFIG=/etc/clickhouse-server/config.xml \
  clickhouse/clickhouse-server:26.2
```

**Verify:**
```bash
docker ps --filter name=clickhouse-prod-v2
docker logs clickhouse-prod-v2 --tail 50
docker exec clickhouse-prod-v2 clickhouse-client --query "SELECT 1"
docker exec clickhouse-prod-v2 clickhouse-client --query "SHOW DATABASES"
docker exec clickhouse-prod-v2 clickhouse-client --query "SELECT database, table, sum(rows) AS rows FROM system.parts WHERE active AND database IN ('bucks_analytics_prod','analytics_staging') GROUP BY database, table ORDER BY database, table"
docker exec clickhouse-prod-v2 clickhouse-client --query "SELECT name FROM system.users"
docker exec clickhouse-prod-v2 clickhouse-client --query "SHOW CREATE TABLE system.trace_log"   # confirm TTL clause present
```
From your laptop:
```bash
curl http://<NEW_DROPLET_IP>:8123/ping   # expect "Ok."
```
Confirm the already-rotated passwords work here too (they came across automatically with the data):
```bash
docker exec clickhouse-prod-v2 clickhouse-client --user analytics_prod_user --password '<new password>' --query "SELECT 1"
```

---

## Step 7 — Cut over the application

1. Stage the new `CLICKHOUSE_HOST=<NEW_DROPLET_IP>` value in the backend's `.env` — don't restart yet.
2. Confirm Step 6's verification is fully green.
3. Update `.env`, restart the backend process.
4. Update DBeaver (and any other client) to the new IP.
5. Confirm writes resume:
```bash
docker exec clickhouse-prod-v2 clickhouse-client --query "SELECT database, table, sum(rows) AS rows FROM system.parts WHERE active AND database='bucks_analytics_prod' GROUP BY table"
```
Row counts should keep climbing.

---

## Rollback reference

| Stage | How to roll back | Data loss risk |
|---|---|---|
| Before Step 5's final rsync | Nothing to do — old setup untouched | None |
| After stopping `clickhouse-prod-new`, before new Droplet verified | `docker start clickhouse-prod-new` on the old Droplet | None |
| New Droplet verified, app still points at old IP | `docker start clickhouse-prod-new` on the old Droplet; new Droplet sits idle | None |
| App's `CLICKHOUSE_HOST` updated, new Droplet receiving writes | Revert `.env` to old IP, restart backend. Any writes landed only on the new Droplet in the gap need a manual reverse-rsync to preserve; Redis buffering means nothing was silently dropped either way | Minimal, only if reverse-copy skipped |

**Never run `docker rm clickhouse-prod-new`, remove the DO Volume, or destroy the old Droplet until Step 9's stability window is complete.**

---

## Step 8 — Reconnect monitoring (Prometheus + Grafana)

Prometheus/Grafana run on a separate server (`192.241.140.192`). Two exporters that feed it (`node-exporter`, `clickhouse-exporter`) were found `Exited (255)` on the old Droplet, 7 weeks ago — unrelated to migration, monitoring has simply been blind. Redeploy fresh on the new Droplet.

On the **NEW** Droplet:
```bash
docker run -d --name node-exporter --restart unless-stopped \
  -p 9100:9100 \
  prom/node-exporter

docker run -d --name clickhouse-exporter --restart unless-stopped \
  -p 9363:9363 \
  -e CLICKHOUSE_URL=http://<NEW_DROPLET_IP>:8123 \
  f1yegor/clickhouse-exporter
```
Confirm the exporter's auth requirements against whatever `default`'s current password status is; add `CLICKHOUSE_USER`/`CLICKHOUSE_PASSWORD` env vars if needed.
```bash
docker logs node-exporter --tail 50
docker logs clickhouse-exporter --tail 50
```

On the **Prometheus server** (`192.241.140.192`), edit `prometheus.yml`:
```yaml
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["<NEW_DROPLET_IP>:9100"]
  - job_name: "clickhouse"
    static_configs:
      - targets: ["<NEW_DROPLET_IP>:9363"]
```
```bash
sudo systemctl restart prometheus
```
Verify: `http://192.241.140.192:9090/targets` (both jobs `UP`), then check `http://192.241.140.192:3000/dashboards`.

**Also check firewall rules on the new Droplet** for ports `8123`, `9000`, `9100`, `9363`:
```bash
ufw status verbose
```

**Rotate Grafana's password too** (`helixoGrafana@123` was found in plaintext documentation, same pattern as other exposed credentials).

---

## Step 9 — Decommission the old Droplet

Only after: new Droplet stable for several days with zero errors, app confirmed writing/reading correctly, monitoring confirmed working, and a fresh independent backup taken from the new Droplet's data.

1. `docker rm clickhouse-prod-new` and `docker rm clickhouse-prod` on the old Droplet
2. Detach and destroy the old DO Volume (`clickhouse-storage`, 150GB)
3. Remove the never-used `clickhouse_data` volume (645MB leftover from the drifted compose YAML)
4. Destroy the old Droplet
5. Update documentation with the new Droplet's IP/specs
6. Retire or correct the stale `/root/clickhouse-setup/docker-compose.yml`

---

## Tracked separately, not yet acted on

- `admin`'s exposed password — rotate (see Step 0)
- `analytics_user` — a sixth user in `system.users`, distinct from `analytics_prod_user`; check its grants/usage before deciding whether to keep it
- `metrics` user — likely used by `clickhouse-exporter`; re-verify after Step 8's redeploy
- Monitoring itself having silently failed for 7 weeks with nobody noticing — worth considering a meta-alert (e.g., a dead-man's-switch check) once the new setup is stable

---

## Summary checklist

- [ ] Password rotation (prod, staging, admin) completed and verified — Step 0
- [ ] New Droplet created — Step 1
- [ ] Docker, tmux, directories set up on new Droplet — Step 2
- [ ] Old Droplet's SSH key trusted by new Droplet, tested — Step 3
- [ ] `log_ttl.xml` created and verified on new Droplet — Step 4
- [ ] Data copied, final pass with old ClickHouse stopped — Step 5
- [ ] `clickhouse-prod-v2` started and fully verified (tables, rows, users, TTL, external curl) — Step 6
- [ ] App `.env` and DBeaver updated, writes confirmed resuming — Step 7
- [ ] Rollback table understood, old Droplet left untouched until confident
- [ ] Exporters redeployed, Prometheus targets updated, Grafana confirmed, firewall checked — Step 8
- [ ] Grafana password rotated — Step 8
- [ ] `admin`/`analytics_user`/`metrics` reviewed — tracked separately
- [ ] Stable for several days, then old Droplet + Volume decommissioned — Step 9
