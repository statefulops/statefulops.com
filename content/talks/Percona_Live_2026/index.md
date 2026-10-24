---
title: "Schrödinger's Backup: It's only good until you try to restore it"
date: 2026-09-10
slug: schrodingers-backup
event: "Percona Live Amsterdam 2026"
slides: "schrodingers-backup-final.pdf"
# YouTube video ID (the part after watch?v=), added once the recording is published.
youtube: "fc63396aBho"
description: "A successful backup job tells you almost nothing. Why restores fail when you need them most, and how we automated restore testing across 1,200 Percona XtraDB clusters on Kubernetes."
tags: ["mysql", "percona", "backups", "disaster-recovery", "kubernetes"]
---

Until you try a restore, your backup is in a kind of superposition. It might be fine. It might be silently corrupt, incomplete, or built on assumptions about cluster state that no longer hold. You only find out when you open the box.

In March 2026, drone strikes took two of the three UAE availability zones in AWS's ME-CENTRAL-1 region offline at the same time. Our planned DR region shared the same blast radius, building a new environment in what was becoming a war zone wasn't an option, and data residency rules put lawyers and management straight into the recovery path. What saved us was that our backups and DR restore process had been tested, and documented, within the previous 24 hours.

This talk covers:

- **What a green tick doesn't tell you.** xtrabackup consistency, GTID and wsrep position drift, incomplete SSTs, bootstrap sequencing and the other failures that only show up mid-recovery.
- **How the platform is built.** Kubernetes operators running 1,200 Percona XtraDB Clusters, 600 Redis and 300 RabbitMQ clusters across 70+ Kubernetes clusters on OpenStack, AWS, GCP and Azure, with backups on shared NFS replicated to a DR region.
- **Automated restore testing.** Wiring the operator's backup schedule, restore CRD and a disposable single-node restore into a pipeline that runs on a schedule and can actually fail, then pointing the same loop at the DR region.
- **A practical checklist.** Restore to a fresh, isolated cluster, verify GTID and wsrep positions, simulate quorum loss, check the routing layer, pin image tags, and record every drill.

## Key takeaways

1. A backup job succeeding is not the same claim as a restore working.
2. Quorum loss and multi-zone failures are not edge cases anymore. Test for them.
3. A disposable, repeatable restore-testing process beats a heroic one-off every time.
