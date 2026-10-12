# Trident Protect — Kubernetes DR & Data Protection Reference

This repository contains a complete set of Kubernetes/OpenShift manifests for deploying and operating **NetApp Trident Protect** with SnapMirror-based disaster recovery across two clusters.

## Overview

The setup demonstrates:

- A sample **httpd web application** with both NFS and iSCSI persistent storage
- **NetApp Trident CSI** provisioning volumes from an FSxN (Amazon FSx for NetApp ONTAP) backend
- **Trident Protect** managing automated snapshots, backups, and cross-cluster replication
- **AppMirrorRelationship** (AMR) for SnapMirror-based DR with cutover and cut-back capability

## Architecture

```
┌──────────────────────────────┐         ┌──────────────────────────────┐
│      Primary Cluster         │         │       DR Cluster             │
│                              │         │                              │
│  ┌─────────────────────┐     │         │     ┌─────────────────────┐  │
│  │  Application "one"  │     │         │     │  Application "one"  │  │
│  │  (httpd + PVCs)     │     │  ONTAP  │     │  (standby)          │  │
│  └────────┬────────────┘     │ SnapMir │     └────────┬────────────┘  │
│           │                  │◄───────►│              │               │
│  ┌────────▼────────────┐     │         │     ┌────────▼────────────┐  │
│  │  Trident CSI        │     │         │     │  Trident CSI        │  │
│  │  (NFS + iSCSI)      │     │         │     │  (NFS + iSCSI)      │  │
│  └────────┬────────────┘     │         │     └────────┬────────────┘  │
│           │                  │         │              │               │
│  ┌────────▼────────────┐     │         │     ┌────────▼────────────┐  │
│  │  FSxN (ONTAP)       │─────┼─────────┼─────│  FSxN (ONTAP)       │  │
│  └─────────────────────┘     │         │     └─────────────────────┘  │
└──────────────────────────────┘         └──────────────────────────────┘
                         │                     │
                         └──────┬──────────────┘
                                │
                       ┌────────▼────────┐
                       │  AWS S3 Bucket  │
                       │  (AppVault)     │
                       └─────────────────┘
```

## Apply Order

```bash
# 1. Install Trident (operator must be installed first via OLM)
oc apply -f objects/01.tridentorchestrator.yaml

# 2. Create backend credentials
oc apply -f objects/02.secret-dr-backend-fsxn-secret.yaml

# 3. Configure storage backends
oc apply -f objects/03.tbc.dr-backend-fsxn-nfs.yaml
oc apply -f objects/04.tbc.dr-backend-fsxn-iscsi.yaml

# 4. Create StorageClasses and VolumeSnapshotClass
oc apply -f objects/05.storgeclass.yaml
oc apply -f objects/06.volumesnapshotclass.yaml

# 5. Create the AppVault
oc apply -f objects/06.appvault.lab-vault.yaml

# 6. Deploy the application (set namespace as needed)
oc apply -f objects/00.deploy.svc.route.yaml -n app-prod

# 7. Define the Application CR
oc apply -f objects/007.application.yaml -n app-prod

# 8. Start the snapshot/backup schedule
oc apply -f objects/07.schedule.yaml -n app-prod

# 9. Create the AppMirrorRelationship on the DR cluster
oc apply -f objects/08.appmirrorrelationship.yaml -n app-dr

# 10. (Optional) Create RBAC for automation
oc apply -f objects/09.serviceaccount-crb.yaml
```

---

## Object Reference

Detailed field-by-field documentation for every manifest in the `objects/` directory.

---

### `00.deploy.svc.route.yaml` — Application Stack

This multi-document YAML contains the complete application: a **Deployment**, a **Service**, an OpenShift **Route**, and two **PersistentVolumeClaims** (iSCSI + NFS). A commented-out **Namespace** block is included at the top.

#### Deployment (`apps/v1`)

Runs an httpd web server with an init container that seeds content.

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.annotations.deployment.kubernetes.io/revision` | `"3"` | Tracks the current revision number of the Deployment. Managed automatically by Kubernetes on each rollout. |
| `metadata.labels.app` | `one` | Application label used to identify and group all resources belonging to this application. |
| `metadata.name` | `one` | Name of the Deployment object. |
| `metadata.namespace` | _(commented)_ `app-prod` | Target namespace. Uncomment or pass `-n <ns>` when applying. |
| `spec.progressDeadlineSeconds` | `600` | Maximum time in seconds for the Deployment to make progress before it is considered failed. If a rollout stalls beyond this, the Deployment reports a `ProgressDeadlineExceeded` condition. |
| `spec.replicas` | `1` | Number of pod replicas to maintain. |
| `spec.revisionHistoryLimit` | `10` | Number of old ReplicaSets to retain for rollback. |
| `spec.selector.matchLabels.app` | `one` | Pods must carry this label to be managed by this Deployment. Must match `template.metadata.labels`. |
| `spec.strategy.type` | `RollingUpdate` | Update strategy. Options: `RollingUpdate` (gradually replace pods) or `Recreate` (kill all existing pods before creating new ones). |
| `spec.strategy.rollingUpdate.maxSurge` | `25%` | Maximum number or percentage of pods that can be created above the desired replica count during a rolling update. |
| `spec.strategy.rollingUpdate.maxUnavailable` | `25%` | Maximum number or percentage of pods that can be unavailable during a rolling update. |

##### Main Container — `httpd`

| Field | Value | Description |
|-------|-------|-------------|
| `image` | `registry.redhat.io/rhel8/httpd-24:1780548837` | Red Hat httpd 2.4 container image with a specific build tag. |
| `imagePullPolicy` | `IfNotPresent` | Pull the image only if it is not already present on the node. Options: `Always`, `IfNotPresent`, `Never`. |
| `name` | `httpd` | Container name within the pod. |
| `terminationMessagePath` | `/dev/termination-log` | File path inside the container where the termination message is written. |
| `terminationMessagePolicy` | `File` | How to populate the termination message. `File` reads from `terminationMessagePath`; `FallbackToLogsOnError` uses container logs if the file is empty. |
| `volumeMounts[0]` — `shared` at `/var/www/html` | | Mounts the ephemeral emptyDir volume (seeded by the init container) as the httpd document root. |
| `volumeMounts[1]` — `iscsi` at `/iscsi` | | Mounts the iSCSI PVC for persistent block storage. |
| `volumeMounts[2]` — `nfs` at `/nfs` | | Mounts the NFS PVC for persistent file storage. |

##### Init Container — `update`

| Field | Value | Description |
|-------|-------|-------------|
| `args` | `echo "today: RED GREEN BLUE" > /data/index.html` | Bash script that seeds the shared volume with a simple HTML page. Runs to completion before the main container starts. |
| `command` | `["/bin/bash", "-c"]` | Shell interpreter used to execute the `args` script. |
| `image` | `registry.redhat.io/ubi9/ubi:9.8-1780554162` | Red Hat UBI 9 base image. |
| `volumeMounts[0]` — `shared` at `/data` | | Writes to the shared emptyDir volume that the httpd container reads from `/var/www/html`. |

##### Pod-Level Settings

| Field | Value | Description |
|-------|-------|-------------|
| `dnsPolicy` | `ClusterFirst` | DNS resolution policy. `ClusterFirst` uses the cluster DNS service first, falling back to the node's upstream DNS. Other options: `Default`, `ClusterFirstWithHostNet`, `None`. |
| `resources.limits.cpu` | `500m` | Maximum CPU (500 millicores = 0.5 CPU core). The container is throttled if it exceeds this. |
| `resources.limits.memory` | `250Mi` | Maximum memory. The container is OOM-killed if it exceeds this. |
| `resources.requests.cpu` | `250m` | Guaranteed CPU allocation used for scheduling decisions. |
| `resources.requests.memory` | `125Mi` | Guaranteed memory allocation used for scheduling decisions. |
| `restartPolicy` | `Always` | Pod restart policy. `Always` (default for Deployments), `OnFailure`, or `Never`. |
| `schedulerName` | `default-scheduler` | Kubernetes scheduler used for pod placement. |
| `terminationGracePeriodSeconds` | `30` | Time in seconds given to the pod to shut down gracefully (SIGTERM) before being forcefully killed (SIGKILL). |

##### Volumes

| Name | Type | Description |
|------|------|-------------|
| `shared` | `emptyDir` | Ephemeral volume shared between the init container and the main httpd container. Populated on pod start, deleted when the pod is removed. |
| `iscsi` | `persistentVolumeClaim` (claimName: `iscsi`) | References the iSCSI PVC defined below. |
| `nfs` | `persistentVolumeClaim` (claimName: `nfs`) | References the NFS PVC defined below. |

#### Service (`v1`)

Exposes the Deployment as a cluster-internal endpoint.

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.labels.app` | `one` | Label matching the application. |
| `metadata.name` | `one` | Service name — used for in-cluster DNS: `one.<namespace>.svc.cluster.local`. |
| `spec.type` | `ClusterIP` | Service type. `ClusterIP` (internal only), `NodePort` (external via node port), `LoadBalancer` (cloud LB). |
| `spec.ports[0].name` | `8080-8080` | Port name — must be unique within the Service. |
| `spec.ports[0].port` | `8080` | Port the Service listens on within the cluster. |
| `spec.selector.app` | `one` | Routes traffic to pods matching this label. |

#### Route (`route.openshift.io/v1`)

OpenShift Route that exposes the Service externally with TLS.

| Field | Value | Description |
|-------|-------|-------------|
| `spec.host` | _(empty)_ | Hostname for the route. Leave empty for an auto-generated hostname from the cluster's default ingress domain. |
| `spec.port.targetPort` | `8080-8080` | Must match the Service port name. |
| `spec.tls.insecureEdgeTerminationPolicy` | `Redirect` | Behaviour for plain HTTP (port 80) traffic. `Redirect` sends a 301 to HTTPS. Options: `Allow` (serve HTTP too), `Redirect`, `None` (disable HTTP). |
| `spec.tls.termination` | `edge` | TLS termination type. `edge` (TLS terminates at the router), `passthrough` (TLS passes through to the pod), `reencrypt` (router decrypts then re-encrypts to pod). |
| `spec.to.kind` | `Service` | Backend resource type. |
| `spec.to.name` | `one` | Name of the Service to route traffic to. |
| `spec.to.weight` | `100` | Relative weight for traffic distribution (0–256). With a single backend, `100` sends all traffic here. |
| `spec.wildcardPolicy` | `None` | Wildcard DNS policy. `None` (exact hostname match) or `Subdomain` (matches all subdomains). |

#### PersistentVolumeClaim — `iscsi`

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.annotations.trident.netapp.io/snapshotDirectory` | `"true"` | Trident annotation that exposes the `.snapshot` directory inside the volume, allowing browsing of ONTAP snapshots from within the pod. |
| `metadata.name` | `iscsi` | PVC name — referenced by the Deployment's volume section. |
| `spec.accessModes` | `ReadWriteOnce` | Access mode. `ReadWriteOnce` (single node read-write), `ReadOnlyMany` (multi-node read-only), `ReadWriteMany` (multi-node read-write). iSCSI supports RWO only. |
| `spec.resources.requests.storage` | `2Gi` | Requested storage capacity. |
| `spec.storageClassName` | `storage-class-iscsi` | StorageClass to use. Must match a name defined in `05.storgeclass.yaml`. |

#### PersistentVolumeClaim — `nfs`

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.annotations.trident.netapp.io/snapshotDirectory` | `"true"` | Exposes the `.snapshot` directory inside the NFS volume. |
| `metadata.name` | `nfs` | PVC name — referenced by the Deployment's volume section. |
| `spec.accessModes` | `ReadWriteOnce` | Access mode. NFS also supports `ReadWriteMany` if needed. |
| `spec.resources.requests.storage` | `3Gi` | Requested storage capacity. |
| `spec.storageClassName` | `storage-class-nfs` | StorageClass to use. Must match a name defined in `05.storgeclass.yaml`. |

---

### `01.tridentorchestrator.yaml` — Trident Installation

Installs and configures NetApp Trident CSI on the cluster. The Trident operator (installed via OLM) watches this CR and reconciles the Trident controller Deployment, node DaemonSet, and related components.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `trident.netapp.io/v1` | Trident operator API group and version. |
| `kind` | `TridentOrchestrator` | Controls the Trident installation lifecycle. |
| `metadata.name` | `trident` | Name of the TridentOrchestrator CR. Typically `trident`. |
| `metadata.namespace` | `openshift-operators` | Namespace where the Trident operator is installed. OLM default for OpenShift. |
| `spec.IPv6` | `false` | Enable IPv6 support for Trident communication with storage backends. Set to `true` for IPv6-only or dual-stack clusters. |
| `spec.debug` | `false` | Enable debug-level logging for all Trident components. Useful for troubleshooting; generates high log volume so disable in production. |
| `spec.nodePrep` | `["iscsi"]` | List of automatic node-preparation tasks Trident performs on worker nodes. `iscsi` ensures iSCSI initiator tools (`iscsid`, `multipathd`) are installed and running. Other option: `nvme` for NVMe-oF connectivity. |
| `spec.imageRegistry` | `""` | Custom container image registry for Trident images. Leave empty to use the default public registry. Set to a private registry URL (e.g. `registry.example.com/trident`) for air-gapped environments. |
| `spec.k8sTimeout` | `30` | Timeout in seconds for Kubernetes API operations performed by the Trident operator during install and upgrade. |
| `spec.namespace` | `trident` | Target namespace where Trident pods (controller + node DaemonSet) are deployed. |
| `spec.silenceAutosupport` | `false` | When `true`, disables Trident's periodic AutoSupport bundles sent to NetApp support. Set to `true` in disconnected or air-gapped environments. |

---

### `02.secret-dr-backend-fsxn-secret.yaml` — Backend Credentials

Stores ONTAP management credentials for the FSxN backend. Referenced by TridentBackendConfig resources (`03`, `04`) via `spec.credentials.name`.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `v1` | Core API group. |
| `kind` | `Secret` | Stores sensitive data (credentials, keys, certificates). |
| `metadata.name` | `dr-backend-fsxn-secret` | Secret name — referenced by TridentBackendConfig `spec.credentials.name`. |
| `metadata.namespace` | `trident` | Must be in the same namespace as the TridentBackendConfig CRs (the Trident namespace). |
| `type` | `Opaque` | Secret type for arbitrary key-value data. Other types: `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`, `kubernetes.io/service-account-token`. |
| `stringData.username` | `vsadmin` | ONTAP management username. `vsadmin` for SVM-scoped access, `admin` for cluster-scoped. `stringData` accepts plaintext and is automatically base64-encoded by Kubernetes on creation. |
| `stringData.password` | `xxxxxxxxxx` | ONTAP management password. **Replace with the actual password before applying.** In production, use sealed-secrets, external-secrets, or a vault integration. |

---

### `03.tbc.dr-backend-fsxn-nfs.yaml` — NFS Backend Configuration

Configures a Trident storage backend using the `ontap-nas` driver to provision NFS volumes on FSxN.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `trident.netapp.io/v1` | Trident API group and version. |
| `kind` | `TridentBackendConfig` | Declarative configuration for a Trident storage backend. |
| `metadata.name` | `dr-backend-fsxn-nfs` | Name of the TridentBackendConfig CR. |
| `metadata.namespace` | `trident` | Must be in the Trident namespace. |
| `spec.version` | `1` | Schema version of the TridentBackendConfig spec. Always `1`. |
| `spec.backendName` | `dr-fsxn-nfs` | Friendly name for the backend. Appears in `tridentctl get backend` output and logs. |
| `spec.storageDriverName` | `ontap-nas` | Trident storage driver. Options: `ontap-nas` (NFS/SMB FlexVol), `ontap-nas-economy` (NFS qtrees, many volumes per FlexVol), `ontap-nas-flexgroup` (NFS FlexGroup, scale-out), `ontap-san` (iSCSI/NVMe block LUNs), `ontap-san-economy` (iSCSI LUNs in shared FlexVols). |
| `spec.managementLIF` | `10.10.0.8` | IP address or hostname of the ONTAP management LIF. Used for REST API calls (provisioning, snapshot, SnapMirror operations). |
| `spec.nasType` | `nfs` | NAS protocol type. Options: `nfs` or `smb`. Determines the data LIF protocol used for volume access. |
| `spec.storagePrefix` | `dr` | Prefix prepended to all volume names created by this backend on ONTAP. Helps identify volumes belonging to this cluster/environment on the storage system. |
| `spec.defaults.snapshotDir` | `"true"` | Whether to expose the `.snapshot` directory inside NFS volumes. Required for Trident Protect snapshot browsing and restore operations. Values: `"true"` or `"false"`. |
| `spec.defaults.nameTemplate` | `{{ .config.StoragePrefix }}_{{ .config.BackendName }}.{{ .volume.Namespace }}_{{ .volume.RequestName }}` | Go template for ONTAP volume names. Available variables: `.config.StoragePrefix` (the storagePrefix value), `.config.BackendName` (the backendName value), `.volume.Namespace` (Kubernetes PVC namespace), `.volume.RequestName` (PVC name). Example result: `dr_dr-fsxn-nfs.app-prod_nfs`. |
| `spec.credentials.name` | `dr-backend-fsxn-secret` | Reference to the Kubernetes Secret containing ONTAP username/password. |

---

### `04.tbc.dr-backend-fsxn-iscsi.yaml` — iSCSI Backend Configuration

Configures a Trident storage backend using the `ontap-san` driver to provision iSCSI block LUNs on FSxN.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `trident.netapp.io/v1` | Trident API group and version. |
| `kind` | `TridentBackendConfig` | Declarative configuration for a Trident storage backend. |
| `metadata.name` | `dr-backend-fsxn-iscsi` | Name of the TridentBackendConfig CR. |
| `metadata.namespace` | `trident` | Must be in the Trident namespace. |
| `spec.version` | `1` | Schema version. Always `1`. |
| `spec.backendName` | `dr-fsxn-iscsi` | Friendly name for the backend. |
| `spec.storageDriverName` | `ontap-san` | Trident storage driver. `ontap-san` provisions one LUN per FlexVol. See `03` above for all driver options. |
| `spec.managementLIF` | `10.10.0.8` | ONTAP management LIF IP address for API calls. |
| `spec.sanType` | `iscsi` | SAN protocol type. Options: `iscsi` or `nvme`. Determines the data protocol for block access. |
| `spec.storagePrefix` | `dr` | Prefix for ONTAP volume/LUN names. |
| `spec.defaults.snapshotDir` | `"true"` | Controls snapshot directory metadata for SAN backends. |
| `spec.defaults.nameTemplate` | `{{ .config.StoragePrefix }}_{{ .config.BackendName }}_{{ .volume.Namespace }}_{{ .volume.RequestName }}` | Go template for ONTAP volume/LUN names. Example result: `dr_dr-fsxn-iscsi_app-prod_iscsi`. |
| `spec.credentials.name` | `dr-backend-fsxn-secret` | Reference to the ONTAP credentials Secret. |

---

### `05.storgeclass.yaml` — Storage Classes

Two cluster-scoped StorageClasses backed by Trident CSI. PVCs reference these by name in `spec.storageClassName`.

#### StorageClass — `storage-class-nfs`

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `storage.k8s.io/v1` | API group for StorageClass resources. |
| `kind` | `StorageClass` | Defines a class of storage with specific provisioner and parameters. Cluster-scoped (no namespace). |
| `metadata.name` | `storage-class-nfs` | StorageClass name — referenced by PVCs in `spec.storageClassName`. |
| `provisioner` | `csi.trident.netapp.io` | CSI provisioner. Always `csi.trident.netapp.io` for Trident. |
| `parameters.backendType` | `"ontap-nas"` | Trident parameter — selects backends using this driver type. Must match a TridentBackendConfig's `storageDriverName`. |
| `parameters.nasType` | `"nfs"` | NAS protocol type. Options: `nfs`, `smb`. |
| `allowVolumeExpansion` | `true` | Allows PVCs using this class to be resized after creation by editing the PVC's `spec.resources.requests.storage`. |

#### StorageClass — `storage-class-iscsi`

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.name` | `storage-class-iscsi` | StorageClass name. |
| `metadata.annotations.storageclass.kubevirt.io/is-default-virt-class` | `"true"` | KubeVirt annotation — marks this as the default StorageClass for virtual machine disks. Only relevant in OpenShift Virtualization environments. |
| `provisioner` | `csi.trident.netapp.io` | Trident CSI driver. |
| `parameters.backendType` | `"ontap-san"` | Selects backends with the `ontap-san` driver. |
| `parameters.sanType` | `"iscsi"` | SAN protocol type. Options: `iscsi`, `nvme`. |
| `parameters.fsType` | `ext4` | Filesystem type created on the block device. Options: `ext4`, `ext3`, `xfs`. **Note:** `xfs` can cause duplicate UUID issues with SnapMirror clones; prefer `ext4` for DR scenarios. |
| `mountOptions` | `["discard"]` | Mount options applied when the volume is mounted on a node. `discard` enables UNMAP/TRIM support, informing the storage backend when blocks are freed for better space reclamation on thin-provisioned LUNs. |
| `allowVolumeExpansion` | `true` | Allows online PVC expansion. |

---

### `06.appvault.lab-vault.yaml` — AppVault (Object Storage)

Defines an object storage bucket used by Trident Protect to store snapshot metadata, backup data, and replication state. Both clusters in a DR pair need access to the relevant AppVaults.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `protect.trident.netapp.io/v1` | Trident Protect API group and version. |
| `kind` | `AppVault` | Object storage location for Trident Protect data. |
| `metadata.name` | `lab-vault` | AppVault name — referenced by Schedules, Snapshots, AMRs via `appVaultRef`, `sourceAppVaultRef`, or `destinationAppVaultRef`. |
| `metadata.namespace` | `trident-protect` | Must be in the `trident-protect` namespace (where the Trident Protect controller runs). |
| `spec.dataMoverPasswordSecretRef` | `appvault-secret` | Name of a Secret containing the encryption password used by the Kopia data mover for backup encryption. Must exist in the `trident-protect` namespace. |

#### Provider Config

Only the section matching `providerType` is active; the others are left empty.

| Field | Value | Description |
|-------|-------|-------------|
| `spec.providerConfig.azure.accountName` | `""` | Azure Storage account name. |
| `spec.providerConfig.azure.bucketName` | `""` | Azure Blob container name. |
| `spec.providerConfig.azure.endpoint` | `""` | Custom Azure endpoint URL. Empty uses the default Azure public endpoint. |
| `spec.providerConfig.gcp.bucketName` | `""` | Google Cloud Storage bucket name. |
| `spec.providerConfig.gcp.projectID` | `""` | GCP project ID that owns the bucket. |
| `spec.providerConfig.s3.bucketName` | `bucket-q8c54-2` | S3 bucket name. Must already exist and be accessible with the provided credentials. |
| `spec.providerConfig.s3.endpoint` | `s3.us-east-1.amazonaws.com` | S3 endpoint URL. Format: `s3.<region>.amazonaws.com` for AWS. For S3-compatible stores (MinIO, StorageGRID), set the custom endpoint here. |

#### Provider Credentials

| Field | Value | Description |
|-------|-------|-------------|
| `spec.providerCredentials.accessKeyID.valueFromSecret.key` | `accessKeyID` | Key within the referenced Secret that holds the access key ID. |
| `spec.providerCredentials.accessKeyID.valueFromSecret.name` | `aws-s3-keys` | Name of the Secret in the `trident-protect` namespace. |
| `spec.providerCredentials.secretAccessKey.valueFromSecret.key` | `secretAccessKey` | Key within the Secret for the secret access key. |
| `spec.providerCredentials.secretAccessKey.valueFromSecret.name` | `aws-s3-keys` | Name of the Secret. |
| `spec.providerType` | `AWS` | Cloud provider type. Options: `AWS`, `Azure`, `GCP`, `GenericS3`. Determines which `providerConfig` section is used. |

---

### `06.volumesnapshotclass.yaml` — Volume Snapshot Class

Cluster-scoped resource required for Trident Protect to create CSI-level volume snapshots.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `snapshot.storage.k8s.io/v1` | API group for VolumeSnapshotClass (part of the CSI snapshot controller). |
| `kind` | `VolumeSnapshotClass` | Defines snapshot behaviour for a CSI driver. |
| `metadata.name` | `csi-trident-vsc` | VolumeSnapshotClass name. Trident Protect auto-discovers classes matching its CSI driver. |
| `driver` | `csi.trident.netapp.io` | CSI driver name. Must match the Trident CSI driver. |
| `deletionPolicy` | `Delete` | What happens to the underlying ONTAP snapshot when the VolumeSnapshot CR is deleted. `Delete` removes the ONTAP snapshot; `Retain` keeps it. |

---

### `007.application.yaml` — Application Definition

Defines the scope of a Trident Protect-managed application. This CR must exist before creating Schedules, Snapshots, Backups, ShutdownSnapshots, or AppMirrorRelationships that reference it.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `protect.trident.netapp.io/v1` | Trident Protect API group and version. |
| `kind` | `Application` | Defines the boundary of a protected application. |
| `metadata.name` | `one` | Application name — referenced by Schedules, Snapshots, AMRs, and ShutdownSnapshots via `applicationRef`. |
| `metadata.namespace` | _(commented)_ `app-prod` | Namespace where this Application CR lives — typically the application's own namespace. |
| `spec.includedNamespaces[0].namespace` | `application-api` | Namespace whose resources are part of this application. All resources in listed namespaces are included by default unless filtered. Multiple namespaces can be listed for multi-namespace applications. |
| `spec.resourceFilter` | `{}` | Empty object means include everything (no filtering). |

##### Resource Filter (commented example)

| Field | Description |
|-------|-------------|
| `spec.resourceFilter.resourceMatchers[].kind` | Kubernetes resource kind to match (e.g. `Secret`, `ConfigMap`, `Deployment`). |
| `spec.resourceFilter.resourceMatchers[].version` | API version of the resource (e.g. `v1`). |
| `spec.resourceFilter.resourceMatchers[].name[]` | List of resource names to match. |
| `spec.resourceFilter.resourceSelectionCriteria` | `Exclude` removes matched resources from protection; `Include` protects only matched resources (everything else is excluded). |

---

### `07.schedule.yaml` — Snapshot/Backup Schedule (Production)

Automated snapshot and backup schedule for the production cluster.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `protect.trident.netapp.io/v1` | Trident Protect API group and version. |
| `kind` | `Schedule` | Automated recurring snapshot and backup policy. |
| `metadata.name` | `snap-sched1` | Schedule name. Must be unique within the namespace. |
| `metadata.namespace` | _(commented)_ `app-prod` | Application namespace. |
| `spec.appVaultRef` | `lab-vault` | Name of the AppVault CR where snapshot metadata and backup data are stored. |
| `spec.applicationRef` | `one` | Name of the Application CR that defines which resources to protect. |
| `spec.backupRetention` | `"5"` | Number of Backup CRs (full backups to object storage) to retain. Oldest are deleted when exceeded. Set to `"0"` to disable backups entirely (snapshot-only mode — useful to avoid iSCSI/XFS UUID issues with Kopia). |
| `spec.enabled` | `true` | Whether the schedule is active (`true`) or paused (`false`). Set to `false` during cutover/cut-back to prevent snapshot conflicts. |
| `spec.granularity` | `Custom` | Schedule granularity. Options: `Hourly`, `Daily`, `Weekly`, `Monthly`, `Custom`. `Custom` enables the `recurrenceRule` field. |
| `spec.recurrenceRule` | _(multiline)_ | iCalendar (RFC 5545) recurrence rule. Only used when `granularity` is `Custom`. |
| — `DTSTART:20250430T000000Z` | | Anchor date/time in UTC for recurrence calculation. The schedule fires at intervals relative to this start time. |
| — `RRULE:FREQ=MINUTELY;INTERVAL=5` | | Recurrence rule definition. `FREQ` options: `SECONDLY`, `MINUTELY`, `HOURLY`, `DAILY`, `WEEKLY`, `MONTHLY`, `YEARLY`. `INTERVAL` sets the frequency multiplier. Additional options: `COUNT` (max occurrences), `UNTIL` (end date), `BYDAY`, `BYHOUR`, etc. |
| `spec.snapshotRetention` | `"5"` | Number of Snapshot CRs (CSI-level snapshots on ONTAP) to retain. Oldest are deleted when exceeded. |

---

### `07.dr.schedule.yaml` — Snapshot/Backup Schedule (DR)

Identical to the production schedule but applied on the DR cluster. Enabled after a cutover when the DR cluster becomes the active source.

All fields are the same as `07.schedule.yaml` above. The only difference:

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.namespace` | _(commented)_ `app-dr` | DR application namespace instead of `app-prod`. |

---

### `001.shutdownsnapshot.yaml` — Shutdown Snapshot

Takes a crash-consistent snapshot of the application and then gracefully shuts down all workload resources (Deployments, StatefulSets, ReplicaSets, etc.). PVCs are preserved. Used during a DR cutover — the resulting snapshot path (from `.status.appArchivePath`) is passed to the AMR's `promotedSnapshot` field.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `protect.trident.netapp.io/v1` | Trident Protect API group and version. |
| `kind` | `ShutdownSnapshot` | Creates a snapshot then scales down all workloads to zero. |
| `metadata.name` | `replication-shutdown-snapshot-1` | Unique name for this ShutdownSnapshot. Choose a descriptive name per cutover event. |
| `metadata.namespace` | _(commented)_ `app-prod` | Namespace of the application to snapshot. |
| `spec.appVaultRef` | `lab-vault` | Name of the AppVault CR that stores the snapshot metadata and data. Must exist in the `trident-protect` namespace. |
| `spec.applicationRef` | `one` | Name of the Application CR that defines which resources belong to this application. |

**After completion**, check `.status.appArchivePath` to get the snapshot path:

```bash
oc get shutdownsnapshot <name> -n <ns> -o jsonpath='{.status.appArchivePath}'
```

Extract the **basename** (after the last `/`) to use as the AMR's `promotedSnapshot` value.

---

### `08.appmirrorrelationship.yaml` — AppMirrorRelationship (DR Side)

Created on the **destination (DR) cluster**. Establishes SnapMirror-based replication from the primary cluster. Trident Protect uses ONTAP SnapMirror to replicate PV data and the AppVault to replicate Kubernetes resource definitions.

| Field | Value | Description |
|-------|-------|-------------|
| `apiVersion` | `protect.trident.netapp.io/v1` | Trident Protect API group and version. |
| `kind` | `AppMirrorRelationship` | Manages cross-cluster application replication via SnapMirror. |
| `metadata.name` | `application-api-amr` | AMR name. Must be unique within the namespace. |
| `metadata.namespace` | _(commented)_ `app-prod` | Namespace on the destination cluster. |
| `spec.desiredState` | `Established` | Desired replication state. `Established` — actively replicating from source to destination. `Promoted` — break replication and start workloads on this (destination) cluster. |
| `spec.promotedSnapshot` | _(commented, empty)_ | Basename of the snapshot to restore when promoting. Obtained from a ShutdownSnapshot's `.status.appArchivePath` — use only the basename (after the last `/`). Leave empty during normal `Established` operation. |
| `spec.destinationAppVaultRef` | `lab-vault` | Name of the AppVault on the destination cluster. Stores replicated metadata. |
| `spec.namespaceMapping[0].source` | `application-api` | Namespace on the source (primary) cluster where the application runs. |
| `spec.namespaceMapping[0].destination` | `application-api` | Namespace on the destination (DR) cluster where replicated resources are created. Can differ from source if needed. |
| `spec.recurrenceRule` | `DTSTART:20240901T000200Z` / `RRULE:FREQ=MINUTELY;INTERVAL=5` | iCalendar recurrence rule controlling how often SnapMirror transfers occur. |
| `spec.sourceAppVaultRef` | `lab-vault` | Name of the AppVault on the source cluster. Contains snapshot metadata to replicate from. |
| `spec.sourceApplicationName` | `one` | Name of the Application CR on the source cluster. |
| `spec.sourceApplicationUID` | `27a608f1-cceb-4b37-bb1d-0f6a4dee84b0` | UID (`.metadata.uid`) of the Application CR on the source cluster. Ensures the AMR binds to the correct application instance. Obtain with: `oc get application <name> -n <ns> -o jsonpath='{.metadata.uid}'`. |
| `spec.storageClassName` | _(commented)_ `storage-class-iscsi` | **Deprecated.** Single StorageClass applied to all destination PVCs. Use `storageClassMapping` instead for mixed storage. |
| `spec.storageClassMapping[]` | `storage-class-iscsi` → `storage-class-iscsi`, `storage-class-nfs` → `storage-class-nfs` | Maps source StorageClass names to destination StorageClass names. Required when the application uses multiple StorageClasses or when names differ between clusters. Each entry has a `source` and `destination` field. |

---

### `08.primary.appmirrorrelationship.yaml` — AppMirrorRelationship (Primary Side)

Same resource type as the DR-side AMR above, but created on the **primary cluster** — used after a resync when replication direction is reversed (DR→Primary). All fields are the same as `08.appmirrorrelationship.yaml` with these differences:

| Field | Value | Description |
|-------|-------|-------------|
| `spec.sourceApplicationUID` | `6aca5133-d3b8-426d-8f6b-3749cfe0b0ce` | UID of the Application CR on the source cluster (which is now the DR cluster after a direction reversal). |
| `spec.promotedSnapshot` | _(commented)_ `snapshots/20261008060349_replication-shutdown-snapshot-1_2c43516d-ddff-4de4-b82e-10b4c5e72b2e` | Example of a full snapshot path from a ShutdownSnapshot. When promoting, **only the basename** (everything after the last `/`) should be used: `20261008060349_replication-shutdown-snapshot-1_2c43516d-ddff-4de4-b82e-10b4c5e72b2e`. Uncomment and set during cut-back operations along with `desiredState: Promoted`. |

---

### `09.serviceaccount-crb.yaml` — RBAC for Automation

Creates a **ServiceAccount**, a long-lived **token Secret**, and a **ClusterRoleBinding** with `cluster-admin` privileges. Typically used for automation, CI/CD pipelines, or cross-cluster operations.

#### ServiceAccount

| Field | Value | Description |
|-------|-------|-------------|
| `kind` | `ServiceAccount` | Provides an identity for pods and automated processes interacting with the cluster API. |
| `metadata.name` | `netapp-replicator` | ServiceAccount name. Referenced by the Secret annotation and ClusterRoleBinding subject. |
| `metadata.namespace` | _(commented)_ `cpaas-admin` | Namespace for the ServiceAccount. |

#### Secret (ServiceAccount Token)

In Kubernetes 1.24+, ServiceAccount tokens are no longer automatically created as Secrets. This Secret explicitly requests a long-lived token.

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.name` | `netapp-replicator-serviceaccount-secret` | Secret name. |
| `metadata.annotations.kubernetes.io/service-account.name` | `netapp-replicator` | Links this Secret to the ServiceAccount. The token controller populates the `token`, `ca.crt`, and `namespace` data fields automatically. |
| `type` | `kubernetes.io/service-account-token` | Secret type that triggers automatic token generation. Other common types: `Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`. |

#### ClusterRoleBinding

| Field | Value | Description |
|-------|-------|-------------|
| `kind` | `ClusterRoleBinding` | Binds a ClusterRole to subjects cluster-wide (not namespace-scoped). |
| `metadata.name` | `netapp-replicator-cluster-admin` | Binding name. Must be unique cluster-wide. |
| `roleRef.apiGroup` | `rbac.authorization.k8s.io` | RBAC API group. Always `rbac.authorization.k8s.io`. |
| `roleRef.kind` | `ClusterRole` | Type of role. `ClusterRole` (cluster-scoped) or `Role` (namespace-scoped). |
| `roleRef.name` | `cluster-admin` | Built-in ClusterRole granting unrestricted access to all resources. **In production, create a custom role with least-privilege permissions.** |
| `subjects[0].kind` | `ServiceAccount` | Subject type. Options: `ServiceAccount`, `User`, `Group`. |
| `subjects[0].name` | `netapp-replicator` | ServiceAccount name to grant the role to. |
| `subjects[0].namespace` | `tooling` | Namespace where the ServiceAccount lives. Must match the SA's actual namespace. An alternative `app-dr` namespace is commented out. |

---

## DR Operations

For full cutover, resync, and cut-back procedures, see:

- **[Cutover / Resync / Cut-Back SOP](sop-trident-protect-cutover-cutback.md)** — step-by-step runbook with verification commands and troubleshooting guide.

### Quick Reference

| Phase | Primary | Secondary | Replication Direction |
|-------|---------|-----------|----------------------|
| Initial state | Running (source) | Standby (destination) | Primary → Secondary |
| After Cutover | Stopped | Running | None (AMR promoted) |
| After Resync | Standby (destination) | Running (source) | Secondary → Primary |
| After Cut-Back | Running | Stopped | None (AMR promoted) |
| After Final Resync | Running (source) | Standby (destination) | Primary → Secondary |

## Customisation

- **Namespaces** — Most files have `#namespace:` commented out. Either uncomment or pass `-n <namespace>` when applying.
- **Storage Prefix** — Change `storagePrefix` in TridentBackendConfig files to distinguish volumes from different clusters on the same ONTAP system.
- **Schedule Frequency** — Adjust `RRULE:FREQ=MINUTELY;INTERVAL=5` in Schedule and AMR CRs to change the snapshot/replication cadence.
- **Backup Retention** — Set `backupRetention: "0"` in Schedules to disable full backups (snapshot-only mode), avoiding iSCSI/XFS UUID issues with Kopia.
- **StorageClass Mapping** — Update `storageClassMapping` in AMR CRs if source and destination clusters use different StorageClass names.
- **Filesystem Type** — Prefer `ext4` over `xfs` for iSCSI StorageClasses in DR scenarios to avoid duplicate UUID issues with SnapMirror clones.
