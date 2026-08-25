# Production Runbook: Move Docker ClickHouse Data To A DigitalOcean Volume [WIP]

> Status: Draft until every placeholder is completed and every prerequisite is checked.
>
> Risk level: High. This procedure stops a production database and changes the storage backing its Docker mount.
>
> Expected availability: ClickHouse is unavailable during the final synchronization and cutover. This is not a zero-downtime procedure.
>
> Primary safety rule: Never start the ClickHouse container unless its `/var/lib/clickhouse` container path is backed by the expected DigitalOcean Volume.

---

## Purpose

Use this runbook to migrate the production ClickHouse Docker named volume from the Droplet root disk to a dedicated DigitalOcean Volume.

Current production mapping:

```text
Docker named volume: clickhouse-setup_clickhouse_data
Container path:      /var/lib/clickhouse
Actual host data:    /var/lib/docker/volumes/clickhouse-setup_clickhouse_data/_data
```

Target production mapping:

```text
DigitalOcean Volume mounted on host: /mnt/clickhouse-volume
Host data directory:                 /mnt/clickhouse-volume/clickhouse-data
Container path:                      /var/lib/clickhouse
Docker Compose mapping:              /mnt/clickhouse-volume/clickhouse-data:/var/lib/clickhouse
```

The Droplet, public IP, ClickHouse ports, Docker image, ClickHouse database files, and application connection details remain unchanged.

---

## Architecture Correction

The old draft assumed ClickHouse runs directly on Ubuntu and that `/var/lib/clickhouse` exists on the Droplet root filesystem as the live data path. That is not the live architecture.

Actual architecture:

```text
DigitalOcean Droplet
└── 80 GB root disk
    ├── Ubuntu
    ├── Docker
    ├── Docker named volume
    │   └── clickhouse-setup_clickhouse_data
    │       └── actual ClickHouse data on the root disk
    └── ClickHouse container
        └── /var/lib/clickhouse
            └── mounted from clickhouse-setup_clickhouse_data
```

The path `/var/lib/clickhouse` is important because ClickHouse sees it inside the container. The data is not stored directly at `/var/lib/clickhouse` on the Droplet host.

This changes the migration plan:

- Do not stop or start `clickhouse-server.service` with `systemctl`; ClickHouse is managed by Docker Compose.
- Do not mount the DigitalOcean Volume at host path `/var/lib/clickhouse`.
- Copy data from the Docker named volume mountpoint, not from host `/var/lib/clickhouse`.
- Update Docker Compose so the ClickHouse service bind mounts a directory on the DigitalOcean Volume to container path `/var/lib/clickhouse`.
- Keep the old Docker named volume as the rollback copy until the retention deadline passes.

---

## Scope And Assumptions

This runbook assumes:

- ClickHouse runs in Docker from `clickhouse/clickhouse-server`.
- Docker Compose manages the ClickHouse container.
- The live ClickHouse data volume is confirmed as `clickhouse-setup_clickhouse_data`.
- The live mapping is `clickhouse-setup_clickhouse_data:/var/lib/clickhouse`.
- This is a single-Droplet storage migration.
- The new DigitalOcean Volume is attached to the same Droplet and same region.
- The new Volume uses an unpartitioned ext4 filesystem.
- The new Volume is dedicated to ClickHouse data.
- Analytics ingestion can be paused or durably buffered during downtime.
- The operator has root access on the Droplet and access to the correct DigitalOcean project.

Stop and extend this runbook before approval if any of these are true:

- ClickHouse is not using Docker Compose.
- The ClickHouse container uses multiple data mounts.
- The live ClickHouse volume name is not `clickhouse-setup_clickhouse_data`.
- ClickHouse Keeper or ZooKeeper data lives in the same Docker volume.
- ClickHouse is part of a replicated cluster.
- The app uses a separate production compose file not covered by this runbook.

---

## Change Record

Complete this table before scheduling the migration.

| Field | Required value |
| --- | --- |
| DigitalOcean project | `<PROJECT_NAME>` |
| Droplet name | `<DROPLET_NAME>` |
| Droplet ID | `<DROPLET_ID>` |
| Region | `<REGION>` |
| Current root-disk size | `80 GB` |
| Current Docker volume name | `clickhouse-setup_clickhouse_data` |
| Current Docker volume mountpoint | `<DOCKER_VOLUME_MOUNTPOINT>` |
| Current ClickHouse allocated size | `<SIZE_GIB>` |
| Current ClickHouse file count | `<FILE_COUNT>` |
| Current daily growth | `<GIB_PER_DAY>` |
| New DigitalOcean Volume name | `<VOLUME_NAME>` |
| New DigitalOcean Volume size | `<VOLUME_SIZE_GIB>` |
| Stable Volume device path | `/dev/disk/by-id/<VOLUME_DEVICE>` |
| Host Volume mount path | `/mnt/clickhouse-volume` |
| Target host data directory | `/mnt/clickhouse-volume/clickhouse-data` |
| Container data path | `/var/lib/clickhouse` |
| Live compose project directory | `<COMPOSE_PROJECT_DIR>` |
| Live compose file | `<COMPOSE_FILE>` |
| ClickHouse compose service | `<CH_SERVICE>` |
| Worker compose service | `<WORKER_SERVICE>` |
| Migration date | `<YYYY-MM-DD>` |
| Maintenance window | `<START_TIME> to <END_TIME> <TIMEZONE>` |
| Primary operator | `<NAME>` |
| Second reviewer | `<NAME>` |
| Rollback owner | `<NAME>` |
| Backup location | `<INDEPENDENT_BACKUP_LOCATION>` |
| Last restore-test date | `<YYYY-MM-DD>` |
| Expected downtime | `<MINUTES>` |
| Rollback observation period | `<HOURS_OR_DAYS>` |

---

## Prerequisite Checklist

### Access And Approval

- [ ] The maintenance window is approved.
- [ ] Internal users and dependent services have been notified that analytics reads may fail during ClickHouse downtime.
- [ ] The primary operator has SSH access and confirmed `sudo` access.
- [ ] The operator can access the correct DigitalOcean project.
- [ ] A second person has verified the Droplet ID, region, and Volume ID.
- [ ] The rollback owner is available throughout the maintenance window.
- [ ] No unrelated infrastructure work is scheduled on the Droplet during the migration.

### Docker And ClickHouse Inventory

- [ ] The live ClickHouse container has been identified.
- [ ] The live Docker Compose project directory has been identified.
- [ ] The live Docker Compose file has been identified.
- [ ] The ClickHouse service name in Compose has been identified.
- [ ] The analytics worker service name in Compose has been identified.
- [ ] The current mapping is confirmed as `clickhouse-setup_clickhouse_data:/var/lib/clickhouse`.
- [ ] The host source path for the Docker named volume has been recorded.
- [ ] The current databases, tables, active parts, row totals, and representative query results have been recorded.
- [ ] The application-level query used to confirm the latest analytics event has been prepared.
- [ ] Pending mutations, merges, backups, and long-running queries have been reviewed.
- [ ] The ClickHouse version has been recorded.

### Data Protection

- [ ] A ClickHouse-aware backup exists outside the Droplet root disk and outside the new DigitalOcean Volume.
- [ ] The current Docker Compose file has been backed up separately.
- [ ] The current `.env` file used by Compose has been backed up separately.
- [ ] The backup has passed a restore test.
- [ ] The old Docker named volume will be retained after cutover.
- [ ] Nobody will run `docker compose down -v`, `docker volume rm`, or Docker prune commands during this migration.

Important: A live DigitalOcean Droplet snapshot is not guaranteed to be application-consistent. After migration, Droplet snapshots do not include the attached Volume. Snapshots complement backups but do not replace a tested ClickHouse backup.

### Traffic And Ingestion

- [ ] The team knows that ClickHouse reads and writes will fail during the final cutover.
- [ ] The analytics worker can be stopped so writes to ClickHouse stop.
- [ ] Storefront ingestion can continue writing to Redis, or the storefront ingestion path can be paused.
- [ ] Redis buffer retention exceeds the maximum maintenance window.
- [ ] Redis has enough free memory/disk for the expected event volume during downtime.
- [ ] Analytics API behavior during ClickHouse downtime has been tested or accepted.
- [ ] A gradual backlog replay procedure is available.
- [ ] The procedure for confirming that ingestion has caught up has been documented.

### Volume Capacity And Performance

- [ ] The DigitalOcean Volume is in the same region as the Droplet.
- [ ] The DigitalOcean Volume is attached only to the intended production Droplet.
- [ ] The Volume size includes current data, expected growth, and ClickHouse merge working space.
- [ ] Expected usage immediately after migration is below 70%.
- [ ] The Volume performance limits are acceptable for the Droplet plan.
- [ ] Insert latency, query latency, merge duration, and I/O wait baselines have been recorded.

### Tooling

Confirm that all required commands exist on the Droplet:

```bash
command -v docker
command -v rsync
command -v findmnt
command -v mountpoint
command -v lsblk
command -v blkid
command -v mkfs.ext4
command -v sudoedit
```

Confirm Docker Compose is available:

```bash
docker compose version
```

Confirm `rsync` supports the needed options:

```bash
rsync --version
```

Expected result:

- Every command returns a valid executable path or version.
- `rsync` supports archive mode, hard links, ACLs, xattrs, sparse files, and `--delete-delay`.
- Every `<PLACEHOLDER>` in this runbook has been replaced before cutover.

---

## Migration Variables

Open a root shell so variables remain available throughout the session:

```bash
sudo -i
```

Set these values after completing the discovery phase:

```bash
export CH_DOCKER_VOLUME="clickhouse-setup_clickhouse_data"
export CH_CONTAINER="<CLICKHOUSE_CONTAINER_NAME>"
export CH_SERVICE="<CLICKHOUSE_COMPOSE_SERVICE>"
export WORKER_SERVICE="<ANALYTICS_WORKER_COMPOSE_SERVICE>"
export COMPOSE_PROJECT_DIR="<COMPOSE_PROJECT_DIR>"
export COMPOSE_FILE="<COMPOSE_FILE>"
export DO_VOLUME_DEVICE="/dev/disk/by-id/<VOLUME_DEVICE>"
export DO_VOLUME_MOUNT="/mnt/clickhouse-volume"
export CH_NEW_DATA_DIR="$DO_VOLUME_MOUNT/clickhouse-data"
export CH_BASELINE_DIR="/root/clickhouse-volume-migration"
export CH_SOURCE_DIR="$(docker volume inspect "$CH_DOCKER_VOLUME" --format '{{ .Mountpoint }}')"
```

Confirm the values before continuing:

```bash
printf '%s\n' \
"CH_DOCKER_VOLUME=$CH_DOCKER_VOLUME" \
"CH_CONTAINER=$CH_CONTAINER" \
"CH_SERVICE=$CH_SERVICE" \
"WORKER_SERVICE=$WORKER_SERVICE" \
"COMPOSE_PROJECT_DIR=$COMPOSE_PROJECT_DIR" \
"COMPOSE_FILE=$COMPOSE_FILE" \
"DO_VOLUME_DEVICE=$DO_VOLUME_DEVICE" \
"DO_VOLUME_MOUNT=$DO_VOLUME_MOUNT" \
"CH_NEW_DATA_DIR=$CH_NEW_DATA_DIR" \
"CH_SOURCE_DIR=$CH_SOURCE_DIR" \
"CH_BASELINE_DIR=$CH_BASELINE_DIR"
```

Expected result:

- Every variable is populated.
- `CH_SOURCE_DIR` ends with `/var/lib/docker/volumes/clickhouse-setup_clickhouse_data/_data`.
- `DO_VOLUME_DEVICE` contains the stable DigitalOcean device path.
- No value contains an unresolved `<PLACEHOLDER>`.

Stop if `CH_SOURCE_DIR` is `/var/lib/clickhouse`. That would mean this runbook does not match the host.

---

## Phase 1: Discover The Live Docker Setup

### 1. Identify The ClickHouse Container

List running containers:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Find the ClickHouse container:

```bash
docker ps --format '{{.Names}} {{.Image}}' | grep -E 'clickhouse/clickhouse-server|clickhouse-server'
```

Set the container variable manually from the output:

```bash
export CH_CONTAINER="<CLICKHOUSE_CONTAINER_NAME>"
```

Verify it:

```bash
docker inspect "$CH_CONTAINER" --format 'Name={{.Name}} Image={{.Config.Image}} Status={{.State.Status}}'
```

Expected result:

- The image is `clickhouse/clickhouse-server` or the approved production ClickHouse image.
- The status is `running` before migration.

### 2. Identify The Compose Project And Service Names

Read Docker Compose labels from the ClickHouse container:

```bash
docker inspect "$CH_CONTAINER" --format 'Project={{ index .Config.Labels "com.docker.compose.project" }}'
docker inspect "$CH_CONTAINER" --format 'WorkingDir={{ index .Config.Labels "com.docker.compose.project.working_dir" }}'
docker inspect "$CH_CONTAINER" --format 'ConfigFiles={{ index .Config.Labels "com.docker.compose.project.config_files" }}'
docker inspect "$CH_CONTAINER" --format 'Service={{ index .Config.Labels "com.docker.compose.service" }}'
```

Set the variables from that output:

```bash
export COMPOSE_PROJECT_DIR="<WORKING_DIR_FROM_LABEL>"
export COMPOSE_FILE="<CONFIG_FILE_FROM_LABEL>"
export CH_SERVICE="<SERVICE_FROM_LABEL>"
```

If the analytics worker runs in the same Compose project, list services:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" ps
```

Set the worker service name:

```bash
export WORKER_SERVICE="<ANALYTICS_WORKER_SERVICE>"
```

Expected result:

- `CH_SERVICE` is the service that runs ClickHouse.
- `WORKER_SERVICE` is the service that flushes Redis analytics data into ClickHouse.
- The compose file path is the live production file, not a stale copy.

### 3. Confirm The Current ClickHouse Data Mount

Inspect mounts on the ClickHouse container:

```bash
docker inspect "$CH_CONTAINER" --format '{{range .Mounts}}{{printf "Type=%s Name=%s Source=%s Destination=%s\n" .Type .Name .Source .Destination}}{{end}}'
```

Expected result:

```text
Type=volume Name=clickhouse-setup_clickhouse_data Source=/var/lib/docker/volumes/clickhouse-setup_clickhouse_data/_data Destination=/var/lib/clickhouse
```

Set and verify the Docker volume source path:

```bash
export CH_SOURCE_DIR="$(docker volume inspect "$CH_DOCKER_VOLUME" --format '{{ .Mountpoint }}')"
test -d "$CH_SOURCE_DIR"
printf 'CH_SOURCE_DIR=%s\n' "$CH_SOURCE_DIR"
du -sxh "$CH_SOURCE_DIR"
find "$CH_SOURCE_DIR" -xdev -type f | wc -l
```

Expected result:

- `CH_SOURCE_DIR` is the Docker volume mountpoint on the Droplet root disk.
- `du` reports the current allocated ClickHouse size.
- Standard ClickHouse directories such as `data`, `metadata`, `store`, or `metadata_dropped` are present.

Stop if the mount type is already `bind`, if the destination is not `/var/lib/clickhouse`, or if more than one mount targets ClickHouse data paths.

### 4. Confirm ClickHouse Health Through Docker

Run basic queries inside the container:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT version()"'
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT 1"'
```

Expected result:

- `SELECT 1` returns `1`.
- The ClickHouse version is recorded in the change record.

If the container does not expose `CLICKHOUSE_USER` and `CLICKHOUSE_PASSWORD`, use the existing secured production client command. Do not paste passwords into shared notes or shell history.

### 5. Check Current Workload State

Review running queries:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT elapsed, user, query_id, query FROM system.processes ORDER BY elapsed DESC LIMIT 20 FORMAT PrettyCompact"'
```

Review merges:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT database, table, elapsed, progress FROM system.merges ORDER BY elapsed DESC FORMAT PrettyCompact"'
```

Review pending mutations:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT database, table, mutation_id, command, is_done, latest_fail_reason FROM system.mutations WHERE NOT is_done FORMAT PrettyCompact"'
```

Expected result:

- No unexpected backup, mutation, or long-running operation will block shutdown.
- Any intentionally active operation is documented.

---

## Phase 2: Provision And Validate The DigitalOcean Volume

### 6. Create And Attach The Volume

In the DigitalOcean Control Panel:

1. Open the project recorded in the change record.
2. Create a Volume in the same region as the production Droplet.
3. Use the approved Volume name and size.
4. Attach it to the production Droplet.
5. Record the Volume ID and stable device path.

Verify attachment from the Droplet:

```bash
test -b "$DO_VOLUME_DEVICE"
ls -l "$DO_VOLUME_DEVICE"
readlink -f "$DO_VOLUME_DEVICE"
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL
```

Expected result:

- The stable device path exists.
- The reported size matches the new Volume.
- The device is not mounted anywhere unexpected.
- The device corresponds to the Volume created for this migration.

Do not use `/dev/sda`, `/dev/sdb`, or another transient kernel device name in `/etc/fstab`.

### 7. Check Before Formatting

Check whether the device is mounted or already formatted:

```bash
findmnt --source "$DO_VOLUME_DEVICE" || true
blkid "$DO_VOLUME_DEVICE" || true
```

Interpretation:

- If `findmnt` returns a mount, stop and investigate.
- If `blkid` shows an expected ext4 filesystem created during DigitalOcean provisioning, do not format it again.
- If the verified new Volume has no filesystem, format it in the next command.
- If it contains any unexpected filesystem or data, stop and investigate.

Destructive checkpoint:

- [ ] Primary operator confirmed the device.
- [ ] Second reviewer confirmed the device.
- [ ] The device is the newly created dedicated Volume.
- [ ] The device does not contain required data.

Format only an unformatted, verified new Volume:

```bash
mkfs.ext4 -L clickhouse-data "$DO_VOLUME_DEVICE"
```

Verify:

```bash
blkid "$DO_VOLUME_DEVICE"
```

Expected result:

- `blkid` reports `TYPE="ext4"`.

### 8. Mount The Volume On The Host

Create the host mountpoint and mount the Volume:

```bash
mkdir -p "$DO_VOLUME_MOUNT"
mount -o defaults,discard,noatime "$DO_VOLUME_DEVICE" "$DO_VOLUME_MOUNT"
findmnt --mountpoint "$DO_VOLUME_MOUNT"
df -hT "$DO_VOLUME_MOUNT"
```

Verify the exact device:

```bash
EXPECTED_DEVICE="$(readlink -f "$DO_VOLUME_DEVICE")"
ACTUAL_DEVICE="$(readlink -f "$(findmnt -nro SOURCE --mountpoint "$DO_VOLUME_MOUNT")")"
if [ "$ACTUAL_DEVICE" != "$EXPECTED_DEVICE" ]; then
printf 'ERROR: expected %s but found %s\n' "$EXPECTED_DEVICE" "$ACTUAL_DEVICE"
exit 1
fi
```

Expected result:

- The command produces no error.
- The mountpoint is `/mnt/clickhouse-volume`.
- The filesystem is ext4.
- Available space is sufficient.

Create the target data directory on the mounted Volume:

```bash
mkdir -p "$CH_NEW_DATA_DIR"
```

Expected result:

- `$CH_NEW_DATA_DIR` exists inside the mounted DigitalOcean Volume.
- `$CH_NEW_DATA_DIR` is empty before the first copy.

### 9. Confirm Capacity

Compare current ClickHouse size with the new Volume capacity:

```bash
du -sxBG "$CH_SOURCE_DIR"
df -BG "$DO_VOLUME_MOUNT"
```

Go/no-go rule:

```text
Source allocated size < available Volume space
Expected post-copy Volume usage < 70%
```

Stop if the Volume will exceed 70% immediately after migration.

---

## Phase 3: Online Pre-Copy

### 10. Run The Initial Copy While ClickHouse Is Online

The initial copy reduces downtime but is not a consistent backup. ClickHouse continues changing files while this copy runs.

```bash
rsync -aHAXS --numeric-ids --exclude='/lost+found' --info=progress2 --stats "$CH_SOURCE_DIR/" "$CH_NEW_DATA_DIR/"
```

Expected result:

- Most data is copied while ClickHouse remains available.
- `rsync` may report files that vanished because ClickHouse merged or removed parts.
- The destination must not be used to start ClickHouse yet.

If the pre-copy affects production query or insert latency, stop it with `Ctrl+C`, select an approved bandwidth limit, and rerun:

```bash
rsync -aHAXS --numeric-ids --exclude='/lost+found' --bwlimit='<KIB_PER_SECOND>' --info=progress2 --stats "$CH_SOURCE_DIR/" "$CH_NEW_DATA_DIR/"
```

### 11. Measure The Expected Final Delta

Run another online pass shortly before the maintenance window:

```bash
rsync -aHAXS --numeric-ids --exclude='/lost+found' --info=progress2 --stats "$CH_SOURCE_DIR/" "$CH_NEW_DATA_DIR/"
```

Record:

| Measurement | Value |
| --- | --- |
| Scan time | `<MINUTES>` |
| Bytes transferred | `<BYTES>` |
| Effective throughput | `<MIB_PER_SECOND>` |
| Files transferred | `<COUNT>` |
| Estimated final delta | `<GIB>` |
| Revised expected downtime | `<MINUTES>` |

---

## Phase 4: Maintenance Window And Baseline

### 12. Pause Writes To ClickHouse

Stop the analytics worker so Redis stops flushing new analytics data into ClickHouse:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" stop "$WORKER_SERVICE"
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" ps
```

Expected result:

- The worker service is stopped.
- ClickHouse remains running.
- Storefront ingestion either continues buffering into Redis or is paused by the approved app procedure.

Verify no new writes are being flushed to ClickHouse:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT count() FROM system.processes WHERE query_kind = '\''Insert'\''"'
```

Expected result:

- The count is `0`, or any active insert is understood and allowed to finish before shutdown.

Do not stop ClickHouse until write pause is confirmed.

### 13. Capture The Final Baseline

Create the baseline directory:

```bash
mkdir -p "$CH_BASELINE_DIR"
```

Record table inventory:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT database, name, engine FROM system.tables WHERE database NOT IN ('\''system'\'', '\''INFORMATION_SCHEMA'\'', '\''information_schema'\'') ORDER BY database, name FORMAT TSVWithNames"' > "$CH_BASELINE_DIR/tables-before.tsv"
```

Record active MergeTree row totals:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT database, table, sum(rows) AS rows FROM system.parts WHERE active GROUP BY database, table ORDER BY database, table FORMAT TSVWithNames"' > "$CH_BASELINE_DIR/rows-before.tsv"
```

Record configured disks:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT name, path, type FROM system.disks ORDER BY name FORMAT TSVWithNames"' > "$CH_BASELINE_DIR/disks-before.tsv"
```

Run and record the approved application-level analytics query:

```text
<READ_ONLY_QUERY_THAT_RETURNS_THE_LATEST_EXPECTED_ANALYTICS_EVENT>
```

Checklist:

- [ ] Table inventory captured.
- [ ] Active part row totals captured.
- [ ] Disk configuration captured.
- [ ] Latest expected analytics event result captured.
- [ ] Representative analytics API result captured.

---

## Phase 5: Stop ClickHouse And Final Sync

### 14. Stop The ClickHouse Container Cleanly

Stop ClickHouse through Compose:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" stop "$CH_SERVICE"
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" ps
```

Confirm no ClickHouse server process remains in a running container:

```bash
docker ps --filter "name=$CH_CONTAINER" --format '{{.Names}} {{.Status}}'
pgrep -af '[c]lickhouse-server' || true
```

Inspect shutdown logs:

```bash
docker logs --since 15m "$CH_CONTAINER"
```

Expected result:

- The ClickHouse service is stopped.
- No running ClickHouse server process remains.
- Logs do not show shutdown timeout, filesystem error, or unhandled exception.

Do not run the final copy while ClickHouse is still running.

### 15. Review Final Deletions

The online copy may contain obsolete parts that ClickHouse removed after the first pass. The final copy must reconcile those deletions.

Run a dry run first:

```bash
rsync -aHAXS --numeric-ids --delete-delay --exclude='/lost+found' --itemize-changes --dry-run "$CH_SOURCE_DIR/" "$CH_NEW_DATA_DIR/"
```

Confirm before continuing:

- [ ] The source is the Docker volume mountpoint for `clickhouse-setup_clickhouse_data`.
- [ ] The destination is `/mnt/clickhouse-volume/clickhouse-data/`.
- [ ] The destination is on the mounted DigitalOcean Volume.
- [ ] Deletions are limited to stale files on the dedicated destination.
- [ ] `lost+found` is excluded.
- [ ] No unexpected directory is being deleted.

### 16. Run The Final Authoritative Copy

Run the final sync while ClickHouse is stopped:

```bash
rsync -aHAXS --numeric-ids --delete-delay --exclude='/lost+found' --info=progress2 --stats "$CH_SOURCE_DIR/" "$CH_NEW_DATA_DIR/"
```

Check the exit status immediately:

```bash
printf 'rsync exit status: %s\n' "$?"
```

Expected result:

```text
rsync exit status: 0
```

Preserve the source directory ownership and mode on the target root:

```bash
chown --reference="$CH_SOURCE_DIR" "$CH_NEW_DATA_DIR"
chmod --reference="$CH_SOURCE_DIR" "$CH_NEW_DATA_DIR"
sync
```

Run a final comparison:

```bash
rsync -aHAXS --numeric-ids --delete-delay --exclude='/lost+found' --itemize-changes --dry-run "$CH_SOURCE_DIR/" "$CH_NEW_DATA_DIR/"
```

Expected result:

- No ClickHouse data differences are listed.
- `lost+found` may remain on the Volume and is intentionally ignored.

Stop if the final synchronization or comparison reports unexplained differences.

---

## Phase 6: Docker Compose Cutover

### 17. Back Up The Live Compose Files

Back up the live Compose file and environment file before editing:

```bash
cp --archive "$COMPOSE_FILE" "$COMPOSE_FILE.pre-clickhouse-do-volume-$(date +%Y%m%d%H%M%S)"
if [ -f "$COMPOSE_PROJECT_DIR/.env" ]; then
cp --archive "$COMPOSE_PROJECT_DIR/.env" "$COMPOSE_PROJECT_DIR/.env.pre-clickhouse-do-volume-$(date +%Y%m%d%H%M%S)"
fi
```

Expected result:

- Backup files exist beside the production compose files.

### 18. Update The ClickHouse Volume Mapping

Open the live Compose file:

```bash
sudoedit "$COMPOSE_FILE"
```

Replace the ClickHouse service volume mapping.

Old mapping:

```yaml
volumes:
- clickhouse_data:/var/lib/clickhouse
```

New recommended mapping:

```yaml
volumes:
- type: bind
    source: /mnt/clickhouse-volume/clickhouse-data
    target: /var/lib/clickhouse
    bind:
    create_host_path: false
```

If the file has this top-level named volume declaration, leave it in place during the rollback observation period unless a reviewer approves removing it:

```yaml
volumes:
clickhouse_data:
```

Reason:

- Keeping the old named volume preserves rollback data.
- Do not run `docker compose down -v`; that can delete named volumes.

Validate the Compose config:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" config > "$CH_BASELINE_DIR/compose-after-cutover.yml"
grep -n '/mnt/clickhouse-volume/clickhouse-data' "$CH_BASELINE_DIR/compose-after-cutover.yml"
grep -n '/var/lib/clickhouse' "$CH_BASELINE_DIR/compose-after-cutover.yml"
```

Expected result:

- The ClickHouse service maps `/mnt/clickhouse-volume/clickhouse-data` on the host to `/var/lib/clickhouse` in the container.
- The old named volume is no longer mounted into the ClickHouse service.

### 19. Add Boot Safety For The Host Mount

Back up `/etc/fstab`:

```bash
cp --archive /etc/fstab "/etc/fstab.pre-clickhouse-volume-$(date +%Y%m%d%H%M%S)"
```

Open `/etc/fstab`:

```bash
sudoedit /etc/fstab
```

Add this line after replacing `<VOLUME_DEVICE>` with the verified stable identifier:

```fstab
/dev/disk/by-id/<VOLUME_DEVICE> /mnt/clickhouse-volume ext4 defaults,nofail,discard,noatime 0 2
```

Validate the file:

```bash
findmnt --verify --verbose
```

Expected result:

- No parsing or mount errors are reported.

If a systemd service starts this Compose app on boot, add `RequiresMountsFor=/mnt/clickhouse-volume` to that service before enabling automatic startup. If Docker restart policies are used, confirm they cannot start ClickHouse against an unmounted host path.

Check restart policy:

```bash
docker inspect "$CH_CONTAINER" --format 'RestartPolicy={{.HostConfig.RestartPolicy.Name}}'
```

Expected result:

- The restart policy is understood and documented.
- If the policy is `always` or `unless-stopped`, a reviewer confirms boot ordering before reboot testing.

### 20. Verify The Target Host Path Before Starting

Confirm the DigitalOcean Volume is mounted and contains ClickHouse data:

```bash
mountpoint -q "$DO_VOLUME_MOUNT"
findmnt --mountpoint "$DO_VOLUME_MOUNT"
test -d "$CH_NEW_DATA_DIR"
ls -la "$CH_NEW_DATA_DIR" | head
du -sxh "$CH_NEW_DATA_DIR"
```

Verify exact backing device:

```bash
EXPECTED_DEVICE="$(readlink -f "$DO_VOLUME_DEVICE")"
ACTUAL_DEVICE="$(readlink -f "$(findmnt -nro SOURCE --mountpoint "$DO_VOLUME_MOUNT")")"
if [ "$ACTUAL_DEVICE" != "$EXPECTED_DEVICE" ]; then
printf 'ERROR: expected %s but found %s\n' "$EXPECTED_DEVICE" "$ACTUAL_DEVICE"
exit 1
fi
```

Expected result:

- The mountpoint check succeeds.
- The exact device check produces no error.
- ClickHouse files are visible under `$CH_NEW_DATA_DIR`.

Stop if the DigitalOcean Volume is not mounted.

---

## Phase 7: Start And Verify

### 21. Recreate And Start The ClickHouse Container

Start ClickHouse with the updated Compose mapping:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" up -d "$CH_SERVICE"
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" ps
```

Refresh the container variable if Compose recreated it with a new container ID or name:

```bash
docker ps --format '{{.Names}} {{.Image}}' | grep -E 'clickhouse/clickhouse-server|clickhouse-server'
export CH_CONTAINER="<NEW_OR_EXISTING_CLICKHOUSE_CONTAINER_NAME>"
```

Confirm the new mount mapping:

```bash
docker inspect "$CH_CONTAINER" --format '{{range .Mounts}}{{printf "Type=%s Source=%s Destination=%s\n" .Type .Source .Destination}}{{end}}'
```

Expected result:

```text
Type=bind Source=/mnt/clickhouse-volume/clickhouse-data Destination=/var/lib/clickhouse
```

Inspect startup logs:

```bash
docker logs --since 10m "$CH_CONTAINER"
```

Expected result:

- No filesystem, permission, metadata, missing-part, corruption, or read-only error is present.

Run basic queries:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT 1"'
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT version()"'
```

Expected result:

- Both queries succeed.

### 22. Compare The Data Baseline

Capture post-migration table inventory:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT database, name, engine FROM system.tables WHERE database NOT IN ('\''system'\'', '\''INFORMATION_SCHEMA'\'', '\''information_schema'\'') ORDER BY database, name FORMAT TSVWithNames"' > "$CH_BASELINE_DIR/tables-after.tsv"
```

Capture active MergeTree row totals:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT database, table, sum(rows) AS rows FROM system.parts WHERE active GROUP BY database, table ORDER BY database, table FORMAT TSVWithNames"' > "$CH_BASELINE_DIR/rows-after.tsv"
```

Compare results:

```bash
diff -u "$CH_BASELINE_DIR/tables-before.tsv" "$CH_BASELINE_DIR/tables-after.tsv"
diff -u "$CH_BASELINE_DIR/rows-before.tsv" "$CH_BASELINE_DIR/rows-after.tsv"
```

Expected result:

- No databases or tables are missing.
- Row totals match while the worker remains stopped.
- Any difference is understood and approved before traffic resumes.

Verify ClickHouse sees the new disk capacity:

```bash
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT name, path, type, formatReadableSize(total_space) AS total, formatReadableSize(free_space) AS free FROM system.disks FORMAT PrettyCompact"'
```

Expected result:

- The default disk path remains `/var/lib/clickhouse/` inside the container.
- Total and free space correspond to the new DigitalOcean Volume.

Run the approved representative analytics query:

```text
<READ_ONLY_QUERY_THAT_RETURNS_THE_LATEST_EXPECTED_ANALYTICS_EVENT>
```

Expected result:

- The result includes the expected pre-migration event ID and timestamp.

### 23. Resume The Worker Gradually

Start the analytics worker:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" up -d "$WORKER_SERVICE"
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" ps
```

Verify:

- [ ] New events reach ClickHouse.
- [ ] A new test event can be queried.
- [ ] The Redis buffered backlog is decreasing.
- [ ] No unexpected duplicate events are observed.
- [ ] Insert errors remain at the normal baseline.
- [ ] Query errors remain at the normal baseline.
- [ ] Merge backlog remains controlled.
- [ ] CPU, memory, I/O wait, and disk usage remain acceptable.

Record the first confirmed post-migration event:

| Field | Value |
| --- | --- |
| Event ID | `<EVENT_ID>` |
| Event timestamp | `<TIMESTAMP>` |
| Insert confirmation time | `<TIMESTAMP>` |
| Query confirmation time | `<TIMESTAMP>` |

---

## Phase 8: Reboot Safety Test

A migration is not complete until mount ordering survives a reboot or an approved follow-up reboot test is scheduled.

Before rebooting:

```bash
findmnt --verify --verbose
mountpoint -q "$DO_VOLUME_MOUNT"
findmnt --mountpoint "$DO_VOLUME_MOUNT"
docker inspect "$CH_CONTAINER" --format '{{range .Mounts}}{{printf "Type=%s Source=%s Destination=%s\n" .Type .Source .Destination}}{{end}}'
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT 1"'
```

Reboot only inside the approved maintenance window:

```bash
systemctl reboot
```

After reconnecting:

```bash
mountpoint -q /mnt/clickhouse-volume
findmnt --mountpoint /mnt/clickhouse-volume
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
docker inspect "$CH_CONTAINER" --format '{{range .Mounts}}{{printf "Type=%s Source=%s Destination=%s\n" .Type .Source .Destination}}{{end}}'
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT 1"'
```

Expected result:

- The DigitalOcean Volume mounts at `/mnt/clickhouse-volume`.
- The ClickHouse container maps `/mnt/clickhouse-volume/clickhouse-data` to `/var/lib/clickhouse`.
- ClickHouse queries succeed.
- No writes were made to the old Docker named volume or to an unmounted root-disk path.

---

## Rollback

### Rollback Conditions

Initiate rollback if any of the following occurs:

- The final `rsync` cannot complete successfully.
- The DigitalOcean Volume cannot mount persistently.
- The exact mounted device cannot be verified.
- The ClickHouse container does not start.
- ClickHouse reports missing or corrupt metadata or data parts.
- Database, table, or row-total verification fails.
- Application queries fail after startup.
- Volume latency or throughput causes unacceptable production behavior.
- Data correctness is uncertain.

### Rollback Before Worker Resumes

Use this procedure only if ClickHouse has not accepted new production writes on the DigitalOcean Volume.

Stop ClickHouse:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" stop "$CH_SERVICE"
```

Open the live Compose file:

```bash
sudoedit "$COMPOSE_FILE"
```

Restore the original ClickHouse service mapping:

```yaml
volumes:
- clickhouse_data:/var/lib/clickhouse
```

Validate Compose config:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" config > "$CH_BASELINE_DIR/compose-rollback.yml"
grep -n 'clickhouse_data' "$CH_BASELINE_DIR/compose-rollback.yml"
```

Start and verify ClickHouse from the old Docker named volume:

```bash
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" up -d "$CH_SERVICE"
docker compose --project-directory "$COMPOSE_PROJECT_DIR" -f "$COMPOSE_FILE" ps
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT 1"'
```

Run the same database, table, row-total, and application-level checks before restoring the worker.

### Rollback After Worker Resumes

Warning: The old Docker named volume is stale after ClickHouse accepts writes on the DigitalOcean Volume. Switching directly to it would lose post-cutover events and metadata changes.

If the DigitalOcean Volume is healthy and rollback is only due to performance or operational concerns:

1. Stop the analytics worker again.
2. Wait until ClickHouse inserts stop.
3. Stop the ClickHouse container.
4. Verify the Droplet root disk has enough free space for the current data.
5. Dry-run a reverse synchronization from the DigitalOcean Volume path to the old Docker named volume mountpoint.
6. Synchronize the current stopped Volume data back to the old Docker named volume.
7. Verify the reverse copy.
8. Restore the Compose mapping to `clickhouse_data:/var/lib/clickhouse`.
9. Start ClickHouse and run all acceptance checks.
10. Resume the worker gradually.

Dry-run reverse synchronization:

```bash
rsync -aHAXS --numeric-ids --delete-delay --exclude='/lost+found' --itemize-changes --dry-run "$CH_NEW_DATA_DIR/" "$CH_SOURCE_DIR/"
```

Perform the reverse synchronization only after the dry run is reviewed:

```bash
rsync -aHAXS --numeric-ids --delete-delay --exclude='/lost+found' --info=progress2 --stats "$CH_NEW_DATA_DIR/" "$CH_SOURCE_DIR/"
```

If data corruption is suspected, do not reverse-sync potentially corrupt data. Restore from the tested ClickHouse backup or recover from a healthy replica.

---

## Downtime Estimate

For this Docker migration with online pre-copy:

```text
Estimated downtime = worker stop and drain + ClickHouse container stop + final rsync scan + final changed bytes / measured throughput + Compose edit validation + container recreate/start + blocking verification
```

The online pre-copy is performed before the maintenance window and is not included in ClickHouse downtime.

Planning examples:

| Data size | Full-copy outage | Pre-copy cutover outage |
| --- | --- | --- |
| 40 GiB | Approximately 7-20 minutes | Approximately 3-10 minutes |
| 60 GiB | Approximately 9-27 minutes | Approximately 3-10 minutes |
| 80 GiB | Approximately 11-34 minutes | Approximately 3-11 minutes |

These are planning examples, not guarantees. Use the measured second pre-copy pass to set the production maintenance window.

Final estimate:

| Component | Measured or approved duration |
| --- | --- |
| Worker stop and insert drain | `<MINUTES>` |
| ClickHouse container shutdown | `<MINUTES>` |
| Final rsync scan | `<MINUTES>` |
| Final data transfer | `<MINUTES>` |
| Compose edit and validation | `<MINUTES>` |
| ClickHouse container start | `<MINUTES>` |
| Blocking verification | `<MINUTES>` |
| Safety buffer | `<MINUTES>` |
| Total planned downtime | `<MINUTES>` |

---

## Post-Migration Monitoring

Monitor for at least `<OBSERVATION_PERIOD>` after the worker resumes.

| Metric | Baseline | Warning threshold | Owner |
| --- | --- | --- | --- |
| DigitalOcean Volume usage | `<VALUE>` | `70%` | `<NAME>` |
| Critical Volume usage | `<VALUE>` | `80%` | `<NAME>` |
| Insert error rate | `<VALUE>` | `<VALUE>` | `<NAME>` |
| Query error rate | `<VALUE>` | `<VALUE>` | `<NAME>` |
| Insert p95 latency | `<VALUE>` | `<VALUE>` | `<NAME>` |
| Query p95 latency | `<VALUE>` | `<VALUE>` | `<NAME>` |
| I/O wait | `<VALUE>` | `<VALUE>` | `<NAME>` |
| Active merges | `<VALUE>` | `<VALUE>` | `<NAME>` |
| Ingestion backlog | `<VALUE>` | `<VALUE>` | `<NAME>` |

After migration:

- Configure separate DigitalOcean Volume snapshots if required.
- Continue ClickHouse-aware backups to independent storage.
- Keep the live Docker Compose file and `.env` in configuration backups.
- Test backup restoration regularly.
- Do not rely on Droplet snapshots to protect the attached Volume.
- Keep the old Docker named volume until the rollback retention deadline passes.

---

## Future Volume Expansion

DigitalOcean Volumes can be increased but cannot be decreased.

Before expanding:

- [ ] Take a current ClickHouse-aware backup.
- [ ] Take a Volume snapshot if required by the recovery policy.
- [ ] Schedule a maintenance window.
- [ ] Stop the analytics worker.
- [ ] Stop the ClickHouse container.
- [ ] Unmount the DigitalOcean Volume as recommended by DigitalOcean.
- [ ] Increase the Volume size in DigitalOcean.
- [ ] Expand the ext4 filesystem.
- [ ] Verify the new filesystem size.
- [ ] Mount the Volume.
- [ ] Verify the exact device.
- [ ] Start and validate ClickHouse.
- [ ] Start the analytics worker.

For an unpartitioned ext4 Volume, filesystem expansion uses:

```bash
resize2fs /dev/disk/by-id/<VOLUME_DEVICE>
```

Verify expanded capacity:

```bash
df -hT /mnt/clickhouse-volume
docker exec "$CH_CONTAINER" sh -lc 'clickhouse-client --user "$CLICKHOUSE_USER" --password "$CLICKHOUSE_PASSWORD" --query "SELECT name, path, formatReadableSize(total_space), formatReadableSize(free_space) FROM system.disks FORMAT PrettyCompact"'
```

Do not run resize commands against an unverified device.

---

## Completion Checklist

### Storage

- [ ] The expected DigitalOcean Volume is mounted at `/mnt/clickhouse-volume`.
- [ ] `findmnt` confirms the exact expected device.
- [ ] `/etc/fstab` passes `findmnt --verify --verbose`.
- [ ] The Volume mounts automatically after reboot.
- [ ] The ClickHouse container bind mounts `/mnt/clickhouse-volume/clickhouse-data` to `/var/lib/clickhouse`.
- [ ] Docker does not use the old named volume for the live ClickHouse service.
- [ ] Volume usage is below 70%.
- [ ] The old Docker named volume remains intact for rollback.
- [ ] The rollback copy has a documented retention deadline.
- [ ] No rollback data was deleted during the migration.

### ClickHouse

- [ ] The ClickHouse container is running.
- [ ] `SELECT 1` succeeds through `docker exec`.
- [ ] Startup logs contain no unexplained errors.
- [ ] All expected databases are present.
- [ ] All expected tables are present.
- [ ] Active MergeTree row totals match the baseline.
- [ ] The expected pre-migration analytics event is queryable.
- [ ] ClickHouse reports the new disk capacity.
- [ ] Pending mutations are healthy.
- [ ] Replication is healthy, if applicable.
- [ ] A reboot test completed successfully or is scheduled with approval.

### Traffic

- [ ] The analytics worker resumed successfully.
- [ ] A new post-migration event was inserted and queried.
- [ ] The Redis ingestion backlog is decreasing or fully drained.
- [ ] No unexpected duplicate events were detected.
- [ ] Query and insert error rates returned to baseline.
- [ ] Application analytics functionality passed its smoke test.
- [ ] Traffic restoration was approved by the operator and application owner.

### Recovery And Ownership

- [ ] The independent ClickHouse backup remains available.
- [ ] The live Docker Compose file and `.env` remain backed up.
- [ ] Post-migration Volume snapshot policy is configured.
- [ ] Disk-usage alerts are configured at 70% and 80%.
- [ ] The monitoring owner has acknowledged responsibility.
- [ ] The rollback owner confirms the old Docker named volume remains usable.
- [ ] The maintenance timeline and command outputs are attached to this page.
- [ ] The change record contains the final downtime.
- [ ] The migration has received final sign-off.

---

## Final Sign-Off

| Role | Name | Decision | Timestamp |
| --- | --- | --- | --- |
| Primary operator | `<NAME>` | `<APPROVED/REJECTED>` | `<TIMESTAMP>` |
| Second reviewer | `<NAME>` | `<APPROVED/REJECTED>` | `<TIMESTAMP>` |
| Application owner | `<NAME>` | `<APPROVED/REJECTED>` | `<TIMESTAMP>` |
| Rollback owner | `<NAME>` | `<APPROVED/REJECTED>` | `<TIMESTAMP>` |

Final outcome:

```text
Migration result: <SUCCESS / ROLLED_BACK / PARTIAL>
Actual ClickHouse downtime: <MINUTES>
Actual ingestion interruption: <MINUTES>
Backlog catch-up duration: <MINUTES>
Final Volume usage: <PERCENT>
Rollback Docker volume retention deadline: <YYYY-MM-DD>
Incident or follow-up link: <URL_OR_NOT_APPLICABLE>
```

---

## References

- DigitalOcean: Mount and unmount Volumes
- DigitalOcean: Increase Volume size
- DigitalOcean: Volume features and performance
- DigitalOcean: Droplet snapshots
- DigitalOcean: Volume snapshots
- ClickHouse: Backup and restore
- Docker: Bind mounts
- Docker Compose: Long syntax volumes
- rsync manual
