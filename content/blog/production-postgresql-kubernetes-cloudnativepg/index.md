---
title: "Production PostgreSQL on Kubernetes with CloudNativePG"
date: 2026-04-04T07:35:09
draft: true
description: "Deploy a production-grade PostgreSQL HA cluster on Kubernetes using CloudNativePG with AZ isolation, declarative users, and automatic failover."
tags: ["postgresql", "kubernetes", "cloudnativepg"]
---

Running a production database inside Kubernetes used to mean a lot of compromises. Stateful workloads were an afterthought in the early ecosystem, and most teams either kept their databases on traditional VMs or accepted significant operational overhead running StatefulSet-based Helm charts.

CloudNativePG (CNPG) changes the picture meaningfully. It is a Kubernetes operator built specifically around PostgreSQL's native streaming replication model — primary/standby, physical WAL-based, battle-tested. This post walks through setting up a full three-node HA cluster on **kind** with real availability-zone isolation, then explores declarative user management and what happens when the primary disappears.

## What CloudNativePG actually does

CNPG is not a thin wrapper around StatefulSets. The operator manages PersistentVolumeClaims directly, controls the lifecycle of each PostgreSQL pod individually, and speaks directly to the Kubernetes API server to maintain cluster state.

Under the hood, the HA story is straightforward PostgreSQL streaming replication: one primary accepts all writes, one or more hot-standby replicas replay WAL and stay read-ready. CNPG handles automated failover, promotion of the most up-to-date standby, updating the `-rw` service endpoint transparently, re-joining the former primary as a new replica, rolling updates with zero downtime, and declarative management of PostgreSQL roles.

{{< callout >}}
CNPG does not use consensus algorithms (Raft, Paxos) or multi-primary writes. It is a deliberately opinionated implementation of PostgreSQL's own replication. That simplicity is a feature — the failure modes match 25+ years of PostgreSQL operational knowledge.
{{< /callout >}}

## The target architecture

We want three PostgreSQL nodes, each pinned to a separate availability zone. The constraint must be *hard* — if the scheduler can't honour it, we want the pod to stay pending rather than two nodes landing in the same zone.

```text
                Kubernetes cluster — 3 worker nodes

   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │  ● primary       │  │  ● replica       │  │  ● replica       │
   │  AZ-A            │  │  AZ-B            │  │  AZ-C            │
   │  kind-worker     │  │  kind-worker2    │  │  kind-worker3    │
   └──────────────────┘  └──────────────────┘  └──────────────────┘
                   ↕  WAL streaming replication  ↕

   pg-cluster-rw (primary) · pg-cluster-ro (replicas) · pg-cluster-r (all)
```

## Setting up the kind cluster

kind nodes do *not* automatically carry `topology.kubernetes.io/zone` labels — we must add them in the cluster config. This is the same label the Kubernetes scheduler uses in real cloud environments.

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
    labels:
      topology.kubernetes.io/zone: AZ-A
  - role: worker
    labels:
      topology.kubernetes.io/zone: AZ-B
  - role: worker
    labels:
      topology.kubernetes.io/zone: AZ-C
```

```shell
$ kind create cluster --name pg-ha --config kind-cluster.yaml
$ kubectl get nodes -L topology.kubernetes.io/zone
```

## Installing the CNPG operator

The operator lives in its own namespace (`cnpg-system`) and installs via a single manifest, registering all CRDs and starting the controller manager.

```shell
$ kubectl apply --server-side -f \
    https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.0.yaml

$ kubectl rollout status deployment/cnpg-controller-manager -n cnpg-system
```

## Deploying the HA cluster

**topologySpreadConstraints** tells the Kubernetes scheduler to refuse scheduling a pod if it would cause the zone spread to exceed `maxSkew: 1`. With three zones and three pods, this means exactly one pod per zone — hard stop. **affinity.enablePodAntiAffinity** with `podAntiAffinityType: required` is CNPG's own mechanism generating a `requiredDuringScheduling` anti-affinity rule keyed on the zone label. Belt and suspenders.

```yaml
# pg-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
spec:
  instances: 3
  storage:
    size: 10Gi

  # Hard AZ enforcement
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          cnpg.io/cluster: pg-cluster

  affinity:
    enablePodAntiAffinity: true
    topologyKey: topology.kubernetes.io/zone
    podAntiAffinityType: required
```

```shell
$ kubectl apply -f pg-cluster.yaml
$ kubectl get cluster pg-cluster -w

NAME         INSTANCES   READY   STATUS                    PRIMARY
pg-cluster   3           3       Cluster in healthy state  pg-cluster-1
```

```shell
$ kubectl get pods -l cnpg.io/cluster=pg-cluster -o wide

NAME           NODE           ZONE
pg-cluster-1   kind-worker    AZ-A   # primary
pg-cluster-2   kind-worker2   AZ-B   # replica
pg-cluster-3   kind-worker3   AZ-C   # replica
```

## Declarative user management

The `.spec.managed.roles` stanza provides fully declarative PostgreSQL role management via Kubernetes. The operator reconciles any drift: direct `ALTER ROLE` changes in the database are reverted on the next reconciliation cycle.

### Step 1 — Create the credential Secret

```yaml
# app-user-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-app-user
  labels:
    cnpg.io/reload: "true"
type: kubernetes.io/basic-auth
stringData:
  username: app_user
  password: supersecretpassword
```

### Step 2 — Reference it from the Cluster spec

```yaml
spec:
  managed:
    roles:
      - name: app_user
        ensure: present
        login: true
        superuser: false
        createdb: false
        inherit: true
        connectionLimit: 100
        passwordSecret:
          name: pg-app-user

      - name: readonly_user
        ensure: present
        login: true
        connectionLimit: 20
        inRoles:
          - pg_read_all_data
        passwordSecret:
          name: pg-readonly-user
```

{{< callout type="warning" title="Important" >}}
The Secret must exist before the Cluster is applied — the operator does not auto-generate passwords. Credential lifecycle belongs to your secrets management pipeline (Vault, External Secrets Operator, Sealed Secrets, etc.).
{{< /callout >}}

{{< callout type="tip" title="Tip — Rotating passwords" >}}
Update the Secret content and the operator applies the new password on next reconciliation. For immediate rotation: `kubectl annotate cluster pg-cluster cnpg.io/reconcile=$(date +%s) --overwrite`
{{< /callout >}}

## Simulating primary failure

When the primary pod disappears — node failure, OOM kill, or deliberate eviction — the operator promotes the most up-to-date replica and updates service endpoints automatically.

```shell
$ kubectl delete pod pg-cluster-1 --grace-period=0 --force
```

| Time | Event |
| --- | --- |
| T+0s | Primary pod deleted — hard kill |
| T+2s | Operator detects missing primary via Kubernetes watch API |
| T+5s | Cluster status → Failing over |
| T+8s | Most up-to-date replica promoted — WAL position determines winner |
| T+10s | pg-cluster-rw service endpoint updated to new primary |
| T+15s | Former primary pod re-created as a replica |
| T+30s | Cluster status → Cluster in healthy state |

```shell
$ kubectl exec -it pg-cluster-2 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
 f   -- false = this is now the primary

$ kubectl exec -it pg-cluster-1 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
 t   -- true = rejoined as replica
```

## Service topology at a glance

| Service | Routes to | Purpose |
| --- | --- | --- |
| `pg-cluster-rw` | Primary only | Read/write connection for your application. Updated on every failover. |
| `pg-cluster-ro` | Replicas only | Read scaling — reporting, analytics, read-heavy microservices. |
| `pg-cluster-r` | All instances | Load-balanced across primary + all replicas. |

## Summary

What we've built is a genuinely production-grade PostgreSQL deployment: three nodes each isolated in a separate availability zone, with hard scheduling constraints that prevent co-location even under pressure, automatic failover completing in under 30 seconds, and fully declarative user management where credentials are Kubernetes-native objects.

Because CNPG is entirely declarative and backed by the Kubernetes API, this entire setup lives in version-controlled YAML. Changes go through the same review process as application code. Drift is automatically corrected. There's no separate database administration tool with its own state to manage.

{{< callout title="Further reading" >}}
The official CloudNativePG documentation at [cloudnative-pg.io](https://cloudnative-pg.io) is thorough and well-maintained. Particularly worth reading: the scheduling page for deeper affinity options, the replication page for synchronous replication configuration, and the bootstrap page for recovery from existing backups.
{{< /callout >}}
