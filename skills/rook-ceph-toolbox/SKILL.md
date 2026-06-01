---
name: rook-ceph-toolbox
description: Use when running ceph CLI commands from the Rook toolbox pod — cluster health, OSD management, pool operations, RBD commands, CephFS management, RGW admin, and common diagnostic operations.
---

# Rook-Ceph Toolbox Operations

Toolbox pod: a privileged deployment with the `ceph` CLI inside the cluster. Defined in `rook-ceph-cluster` chart values as `toolbox.enabled: true`.

Image: `quay.io/ceph/ceph:v20.2.1` (must match cluster Ceph version)

```bash
# Access toolbox
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# Or run single command
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
```

## Cluster Health

```bash
ceph status                    # Overall cluster state (HEALTH_OK / WARN / ERR)
ceph health                    # Brief health string
ceph health detail             # Detailed health with reasons
ceph osd stat                  # OSD summary (up/in/out counts)
ceph mon stat                  # Monitor quorum status
ceph mgr stat                  # Manager status
ceph df                        # Pool usage (bytes, %, objects)
ceph osd df                    # Per-OSD utilization
```

## OSD Management

```bash
ceph osd tree                  # OSD hierarchy by node/rack
ceph osd ls                    # List OSD IDs
ceph osd stat                  # Per-OSD status (up/down, in/out)
ceph osd reweight-by-utilization  # Auto-rebalance OSD weights
ceph osd set <flag>            # noout/noscrub/nodeep-scrub/pause
ceph osd unset <flag>          # Remove flag

# Repair
ceph osd repair <osd.id>       # Trigger OSD repair
ceph osd deep-scrub <osd.id>   # Force deep scrub

# Mark OSD
ceph osd out <osd.id>          # Migrate data off OSD (pre-maintenance)
ceph osd in <osd.id>           # Bring OSD back into service
ceph osd down <osd.id>         # Mark OSD down
ceph osd rm <osd.id>           # Remove OSD from cluster (DESTRUCTIVE)
```

## Pool Operations

```bash
ceph osd lspools               # List pools
ceph osd pool stats <pool>     # Pool I/O stats
ceph osd pool set <pool> <key> <value>  # Change pool property
# size, min_size, pg_num, pg_autoscale_mode, target_size_bytes

ceph osd pool autoscale-status # PG autoscale recommendations

# PG management
ceph pg stat                   # PG distribution summary
ceph pg dump                   # Full PG dump (verbose)
ceph pg dump_stuck             # Stuck PGs (inactive/unclean/stale)
ceph pg repair <pgid>          # Repair inconsistent PG
```

## RBD (Block) Commands

```bash
# List images
rbd ls <pool>
rbd info <pool>/<image>        # Image details (size, features, snapshots)

# Create/delete
rbd create <pool>/<image> --size <size>  # e.g. --size 10G
rbd rm <pool>/<image>

# Snapshots
rbd snap ls <pool>/<image>
rbd snap create <pool>/<image>@<snap>
rbd snap rm <pool>/<image>@<snap>

# Clone from snapshot
rbd clone <pool>/<image>@<snap> <pool>/<clone>
rbd flatten <pool>/<clone>

# Resize
rbd resize <pool>/<image> --size <newsize>  # Use --allow-shrink to reduce

# Mirroring
rbd mirror pool info <pool>
rbd mirror pool status <pool>
rbd mirror image status <pool>/<image>

# Performance
rbd bench <pool>/<image> --io-type write --io-size 4M --io-total 1G
```

## CephFS Commands

```bash
ceph fs ls                     # List filesystems
ceph fs status <fs>            # Filesystem health + MDS rank info
ceph fs dump                   # Full filesystem configuration

# MDS management
ceph mds stat                  # MDS daemon status
ceph mds fail <fs>:<rank>      # Force MDS rank failover

# Subvolume management
ceph fs subvolume ls <fs>      # List subvolumes
ceph fs subvolume info <fs> <subvol>  # Subvolume details
ceph fs subvolume create <fs> <name> --size <size>
ceph fs subvolume rm <fs> <name>

# Quota
ceph fs subvolume resize <fs> <name> --size <newsize>
```

## RGW (Object Store) Commands

```bash
# Gateway status (in Rook, check via status + radosgw-admin)
ceph status | grep rgw         # RGW in services section
radosgw-admin bucket list      # List all buckets
radosgw-admin bucket stats --bucket <name>  # Bucket details
radosgw-admin user list        # List all S3 users
radosgw-admin user info --uid <user>        # User + keys
radosgw-admin user create --uid <user> --display-name "<name>"
radosgw-admin user rm --uid <user>

# Bucket operations
radosgw-admin bucket rm --bucket <name>     # Delete bucket (DESTRUCTIVE)
radosgw-admin bucket limit check            # Check bucket limits
```

## CRUSH Map

```bash
ceph osd crush dump            # Dump CRUSH map (JSON)
ceph osd crush rule ls         # List CRUSH rules
ceph osd crush rule dump <rule>  # Rule details
ceph osd crush class ls        # List device classes (hdd, ssd, nvme)
ceph osd crush class create <class>  # Create custom device class
ceph osd crush set-device-class <class> <osd.id>  # Assign OSD to class
```

## Advanced Diagnostics

```bash
# Watch cluster events
ceph -w                        # Live event stream
ceph log last 100               # Last 100 log lines

# Monitor connection status
ceph mon dump                  # Monitor map
ceph mon enable-msgr2          # Enable msgr2 protocol on all mons

# Config
ceph config dump               # Effective Ceph configuration
ceph config get <who> <key>    # Get specific config (who: mon, osd, mgr, client)
ceph config set <who> <key> <value>  # Set config (persistent)

# Auth
ceph auth ls                   # List all auth keys
ceph auth get-or-create client.<name> <caps> -o <file>
ceph auth caps client.<name> <caps>

# Versions
ceph versions                  # Running daemon versions across cluster
```

## Common Operations

```bash
# Rebalance cluster
ceph osd reweight-by-utilization 100  # Target equal utilization

# Pause OSD operations for maintenance
ceph osd set noout
ceph osd set noscrub
ceph osd set nodeep-scrub
# ... do maintenance ...
ceph osd unset noout
ceph osd unset noscrub
ceph osd unset nodeep-scrub

# Full cluster recovery
ceph osd set norebalance       # Stop rebalancing
ceph osd set noout             # Prevent OSD marks as out
# Add new OSDs or bring back failed ones
ceph osd unset noout
ceph osd unset norebalance

# Check for inconsistent PGs
ceph pg dump_stuck unclean --format json-pretty
rados list-inconsistent-pg <pool>
ceph pg <pgid> query           # Detailed PG state

# Force OSD removal (when OSD won't come back)
ceph osd out <id>
ceph osd crush remove osd.<id>
ceph auth del osd.<id>
ceph osd rm <id>
```

## Common Mistakes

- **Mismatched toolbox image** — Toolbox Ceph version must match cluster Ceph version. Mismatch causes command failures.
- **`ceph osd rm` without drain** — Always `ceph osd out <id>` first, wait for data migration, then remove. Data loss otherwise.
- **PG stuck unclean** — Usually means a replica is missing. Check which OSDs are down and bring them back.
- **`ceph df` vs `ceph osd df`** — First shows pool-level usage, second shows per-OSD. Use both for full picture.
- **RBD image features mismatch** — Newer Ceph may set features older kernels don't support. Use `imageFeatures: layering` for max compatibility.
- **RGW bucket delete** — `radosgw-admin bucket rm` deletes the bucket and ALL objects. No recycle bin.
- **CephFS subvolume resize** — Can only increase, not decrease (no shrink support).
- **Ceph config changes** — `ceph config set` is persistent across restarts. Use `--id` flag to target specific daemon.
