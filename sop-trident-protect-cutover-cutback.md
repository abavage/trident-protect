# Trident Protect — Cutover, Resync & Cut-Back SOP

**Platform:** Red Hat OpenShift  
**Component:** NetApp Trident Protect with SnapMirror Replication  
**API Group:** `protect.trident.netapp.io/v1`

---

## Prerequisites

| Item | Details |
|------|---------|
| Clusters | Primary (Cluster A), Secondary (Cluster B) |
| Trident Protect | Installed on both clusters |
| AppVaults | Configured on both clusters (source + destination buckets) |
| AppMirrorRelationship | Established and replicating (Primary → Secondary) |
| Schedule | Running on Primary, creating snapshots on schedule |
| Application CR | Defined on both clusters |
| Storage classes | Matching on both clusters (use `storageClassMapping` if names differ) |

### Variable Reference

Throughout this SOP, replace placeholders with your environment values:

| Variable | Description | Example |
|----------|-------------|---------|
| `<app-namespace-primary>` | Application namespace on Primary | `app-prod` |
| `<app-namespace-secondary>` | Application namespace on Secondary | `app-dr` |
| `<amr-name>` | AppMirrorRelationship name | `my-app-amr` |
| `<schedule-name>` | Schedule CR name | `snap-sched1` |
| `<app-name>` | Application CR name | `one` |
| `<source-appvault>` | AppVault pointing to source bucket | `primary-vault` |
| `<dest-appvault>` | AppVault pointing to destination bucket | `secondary-vault` |

---

## Phase 1: Cutover (Primary → Secondary)

> **Goal:** Move the running application from Primary to Secondary with zero data loss.

### Step 1.1 — Disable the schedule on Primary

```bash
oc patch schedule <schedule-name> -n <app-namespace-primary> \
  --type=merge -p '{"spec":{"enabled":false}}'
```

**Verify:**

```bash
oc get schedule <schedule-name> -n <app-namespace-primary> -o jsonpath='{.spec.enabled}'
# Expected: false
```

### Step 1.2 — Create a ShutdownSnapshot on Primary

```bash
oc apply -f - <<EOF
apiVersion: protect.trident.netapp.io/v1
kind: ShutdownSnapshot
metadata:
  name: cutover-shutdown-snap
  namespace: <app-namespace-primary>
spec:
  appVaultRef: <source-appvault>
  applicationRef: <app-name>
EOF
```

**Wait for completion:**

```bash
oc get shutdownsnapshot cutover-shutdown-snap -n <app-namespace-primary> -w
```

### Step 1.3 — Get the ShutdownSnapshot basename

```bash
SNAP_PATH=$(oc get shutdownsnapshot cutover-shutdown-snap \
  -n <app-namespace-primary> \
  -o jsonpath='{.status.appArchivePath}' | awk -F/ '{print $3}')

SNAP_BASENAME=$(basename "$SNAP_PATH")

echo "Basename: $SNAP_BASENAME"
```

> **Record this value** — it is used in the next step.

### Step 1.4 — Promote the AMR on Secondary with the ShutdownSnapshot

```bash
oc patch amr <amr-name> -n <app-namespace-secondary> --type=merge \
  -p '{"spec":{"desiredState":"Promoted","promotedSnapshot":"${SNAP_BASENAME}"}}'
```

**Wait for Promoted state:**

```bash
oc get amr <amr-name> -n <app-namespace-secondary> -w
# Expected: STATE = Promoted
```

### Step 1.5 — Confirm pods are running on Secondary

```bash
oc get pods -n <app-namespace-secondary>
# All application pods should be Running/Ready
```

```bash
oc get deploy -n <app-namespace-secondary>
# Deployments should show desired replicas available
```

### Cutover Complete

The application is now running on **Secondary**. Primary is stopped (pods removed, PVCs intact).

---

## Phase 2: Resync (Establish replication Secondary → Primary)

> **Goal:** Set up replication in the reverse direction so Secondary (now running) replicates to Primary (now standby).

### Step 2.1 — Delete the AMR on Secondary

```bash
oc delete amr <amr-name> -n <app-namespace-secondary>
```

**Verify:**

```bash
oc get amr -n <app-namespace-secondary>
# No AMR should exist
```

### Step 2.2 — Clean up orphaned resources on Primary

TEST THIS -- The ShutdownSnapshot leaves Routes, Services, and other non-PVC resources on Primary. Clean them up:

```bash
# Delete all non-PVC resources (keeps PVCs for replication)
oc get all -n <app-namespace-primary> -o name | xargs oc delete -n <app-namespace-primary>

# Verify only PVCs remain
oc get pvc -n <app-namespace-primary>
```

> **Do NOT delete PVCs** — they are needed as SnapMirror destinations.

### Step 2.3 — Enable the schedule on Secondary

```bash
oc patch schedule <schedule-name> -n <app-namespace-secondary> \
  --type=merge -p '{"spec":{"enabled":true}}'
```

If no schedule exists on Secondary, create one:

```bash
oc apply -f - <<EOF
apiVersion: protect.trident.netapp.io/v1
kind: Schedule
metadata:
  name: <schedule-name>
  namespace: <app-namespace-secondary>
spec:
  appVaultRef: <dest-appvault>
  applicationRef: <app-name>
  backupRetention: "5"
  snapshotRetention: "5"
  enabled: true
  granularity: Custom
  recurrenceRule: |-
    DTSTART:20260901T000200Z
    RRULE:FREQ=MINUTELY;INTERVAL=5
EOF
```

### Step 2.4 — Get the Application UID on Secondary

```bash
APP_UID=$(oc get application <app-name> -n <app-namespace-secondary> \
  -o jsonpath='{.metadata.uid}')

echo "Application UID: $APP_UID"
```

### Step 2.5 — Create the AMR on Primary (reverse direction)

```bash
oc apply -f - <<EOF
apiVersion: protect.trident.netapp.io/v1
kind: AppMirrorRelationship
metadata:
  name: <amr-name>
  namespace: <app-namespace-primary>
spec:
  desiredState: Established
  destinationAppVaultRef: <source-appvault>
  namespaceMapping:
    - destination: <app-namespace-primary>
      source: <app-namespace-secondary>
  recurrenceRule: |-
    DTSTART:20260901T000200Z
    RRULE:FREQ=MINUTELY;INTERVAL=5
  sourceAppVaultRef: <dest-appvault>
  sourceApplicationName: <app-name>
  sourceApplicationUID: "${APP_UID}"
  storageClassMapping:
    - source: storage-class-iscsi
      destination: storage-class-iscsi
    - source: storage-class-nfs
      destination: storage-class-nfs
EOF
```

> **Note:** Adjust `storageClassMapping` to match your environment. Omit the field entirely if storage class names are identical and you want the default behaviour.

### Step 2.6 — Wait for Established and first successful replication

```bash
oc get amr <amr-name> -n <app-namespace-primary> -w
# Wait until STATE = Established
```

**Confirm snapshots are replicating:**

```bash
oc get amr <amr-name> -n <app-namespace-primary> \
  -o jsonpath='{.status.lastTransferredSnapshot}' | jq
```

If no snapshot has replicated yet and you don't want to wait for the schedule, create a manual snapshot on Secondary:

```bash
oc apply -f - <<EOF
apiVersion: protect.trident.netapp.io/v1
kind: Snapshot
metadata:
  name: manual-resync-snap
  namespace: <app-namespace-secondary>
spec:
  applicationRef: <app-name>
  appVaultRef: <dest-appvault>
EOF
```

### Step 2.7 — Confirm application still running on Secondary

```bash
oc get pods -n <app-namespace-secondary>
# All pods should still be Running/Ready
```

### Resync Complete

Replication is now flowing **Secondary → Primary**. The application runs on Secondary.

---

## Phase 3: Cut-Back (Secondary → Primary)

> **Goal:** Move the application back to Primary with zero data loss (reverse of Phase 1).

### Step 3.1 — Disable the schedule on Secondary

```bash
oc patch schedule <schedule-name> -n <app-namespace-secondary> \
  --type=merge -p '{"spec":{"enabled":false}}'
```

**Verify:**

```bash
oc get schedule <schedule-name> -n <app-namespace-secondary> -o jsonpath='{.spec.enabled}'
# Expected: false
```

### Step 3.2 — Create a ShutdownSnapshot on Secondary

```bash
oc apply -f - <<EOF
apiVersion: protect.trident.netapp.io/v1
kind: ShutdownSnapshot
metadata:
  name: cutback-shutdown-snap
  namespace: <app-namespace-secondary>
spec:
  appVaultRef: <dest-appvault>
  applicationRef: <app-name>
EOF
```

**Wait for completion:**

```bash
oc get shutdownsnapshot cutback-shutdown-snap -n <app-namespace-secondary> -w
```

### Step 3.3 — Get the ShutdownSnapshot basename

```bash
SNAP_PATH=$(oc get shutdownsnapshot cutback-shutdown-snap \
  -n <app-namespace-secondary> \
  -o jsonpath='{.status.appArchivePath}' | awk -F/ '{print $3}')

SNAP_BASENAME=$(basename "$SNAP_PATH")

echo "Basename: $SNAP_BASENAME"
```

### Step 3.4 — Promote the AMR on Primary with the ShutdownSnapshot

```bash
oc patch amr <amr-name> -n <app-namespace-primary> --type=merge \
  -p '{"spec":{"desiredState":"Promoted","promotedSnapshot":"${SNAP_BASENAME}"}}'
```

**Wait for Promoted state:**

```bash
oc get amr <amr-name> -n <app-namespace-primary> -w
# Expected: STATE = Promoted
```

### Step 3.5 — Confirm pods are running on Primary

```bash
oc get pods -n <app-namespace-primary>
# All application pods should be Running/Ready
```

```bash
oc get deploy -n <app-namespace-primary>
# Deployments should show desired replicas available
```

### Cut-Back Complete

The application is now running on **Primary** again. Secondary is stopped.

---

## Phase 4: Resync (Re-establish replication Primary → Secondary)

> **Goal:** Restore the original replication direction (Primary → Secondary).

### Step 4.1 — Delete the AMR on Primary

```bash
oc delete amr <amr-name> -n <app-namespace-primary>
```

### Step 4.2 — Clean up orphaned resources on Secondary

TEST THIS BIT

```bash
oc get all -n <app-namespace-secondary> -o name | xargs oc delete -n <app-namespace-secondary>

# Verify only PVCs remain
oc get pvc -n <app-namespace-secondary>
```

> desiredState**Do NOT delete PVCs.**

### Step 4.3 — Enable the schedule on Primary

```bash
oc patch schedule <schedule-name> -n <app-namespace-primary> \
  --type=merge -p '{"spec":{"enabled":true}}'
```

### Step 4.4 — Get the Application UID on Primary

```bash
APP_UID=$(oc get application <app-name> -n <app-namespace-primary> \
  -o jsonpath='{.metadata.uid}')

echo "Application UID: $APP_UID"
```

### Step 4.5 — Create the AMR on Secondary (original direction)

```bash
oc apply -f - <<EOF
apiVersion: protect.trident.netapp.io/v1
kind: AppMirrorRelationship
metadata:
  name: <amr-name>
  namespace: <app-namespace-secondary>
spec:
  desiredState: Established
  destinationAppVaultRef: <dest-appvault>
  namespaceMapping:
    - destination: <app-namespace-secondary>
      source: <app-namespace-primary>
  recurrenceRule: |-
    DTSTART:20260901T000200Z
    RRULE:FREQ=MINUTELY;INTERVAL=5
  sourceAppVaultRef: <source-appvault>
  sourceApplicationName: <app-name>
  sourceApplicationUID: "${APP_UID}"
  storageClassMapping:
    - source: storage-class-iscsi
      destination: storage-class-iscsi
    - source: storage-class-nfs
      destination: storage-class-nfs
EOF
```

### Step 4.6 — Wait for Established and first successful replication

```bash
oc get amr <amr-name> -n <app-namespace-secondary> -w
# Wait until STATE = Established
```

**Confirm snapshots are replicating:**

```bash
oc get amr <amr-name> -n <app-namespace-secondary> \
  -o jsonpath='{.status.lastTransferredSnapshot}' | jq
```

If needed, create a manual snapshot on Primary:

```bash
oc apply -f - <<EOF
apiVersion: protect.trident.netapp.io/v1
kind: Snapshot
metadata:
  name: manual-resync-snap
  namespace: <app-namespace-primary>
spec:
  applicationRef: <app-name>
  appVaultRef: <source-appvault>
EOF
```

### Step 4.7 — Confirm application still running on Primary

```bash
oc get pods -n <app-namespace-primary>
# All pods should be Running/Ready
```

### Resync Complete — Original State Restored

Replication is flowing **Primary → Secondary** as originally configured.

---

## Quick Reference: State at Each Phase

| Phase | Primary | Secondary | Replication |
|-------|---------|-----------|-------------|
| **Initial state** | Running (source) | Standby (destination) | Primary → Secondary |
| **After Cutover** | Stopped | Running | None (AMR promoted) |
| **After Resync** | Standby (destination) | Running (source) | Secondary → Primary |
| **After Cut-Back** | Running | Stopped | None (AMR promoted) |
| **After Final Resync** | Running (source) | Standby (destination) | Primary → Secondary |

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| AMR stuck in `Promoting` with `No snapshot.json found` | `promotedSnapshot` set to full path instead of basename | Patch AMR with only the basename (everything after the last `/`) |
| Pods not created after promotion | AMR promoted before first snapshot replicated | Wait for `Established` + at least one `lastTransferredSnapshot` before promoting |
| PVCs on destination have wrong StorageClass | Single `storageClassName` used with mixed NFS/iSCSI | Use `storageClassMapping` to map each source class to destination class |
| Kopia backup pods stuck with mount error | XFS duplicate UUID on iSCSI clone | Set `backupRetention: "0"` for snapshot-only, or use `ext4` for iSCSI StorageClass |
| Route still exists on old source after ShutdownSnapshot | ShutdownSnapshot only removes workload resources (pods/deployments) | Manually delete Routes and Services on old source (Phase 2 Step 2.2 / Phase 4 Step 4.2) |
| Schedule not creating snapshots | Stuck backup CRs blocking the queue | Delete failed/stuck Backup CRs: `oc delete backup <name> -n <ns>` |
