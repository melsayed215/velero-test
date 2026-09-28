# Velero on Kubernetes (ArgoCD + MinIO)

Deploys [Velero](https://velero.io) through ArgoCD using the official Helm chart, with [MinIO](https://min.io) (S3-compatible) as the backup storage location.

## Repository layout

| File | Purpose |
|---|---|
| `velero-app.yaml` | ArgoCD `Application` (multi-source: Helm chart + this repo for values) |
| `velero-values.yaml` | Helm values: plugin, credentials, MinIO location, node agent, schedules |

## How it works

```
Git (velero-values.yaml) ─┐
                          ├─► ArgoCD ──► Velero + node-agent (ns: velero)
Helm chart (vmware-tanzu) ┘                    │
                                               ▼
                                  Backups (K8s objects + volume data via Kopia)
                                               │
                                               ▼
                                     MinIO (bucket: velero)
```

- **Velero** backs up Kubernetes objects.
- **node-agent** (DaemonSet) backs up volume data using file-system backup (Kopia).
- **AWS plugin** lets Velero talk to MinIO over the S3 protocol.

## Prerequisites

1. A running Kubernetes cluster with ArgoCD installed (namespace `argocd`).
2. A reachable MinIO server (reachable from **every node**, since node-agent runs on each one).
3. `kubectl`, `argocd` CLI, `helm`, `mc` (MinIO client), and optionally the `velero` CLI.

## Setup

### 1. Create the MinIO bucket

Velero does not create the bucket by itself.

```bash
mc alias set myminio http://<minio-ip>:9000 <access-key> <secret-key>
mc mb myminio/velero
```

### 2. Create the MinIO credentials Secret

The `minio-credentials` Secret holds the **MinIO access key and secret key**. They are written in AWS credentials-file format because Velero talks to MinIO through the S3 plugin.

- The Secret name must match `credentials.existingSecret` in the values file (`minio-credentials`).
- The key inside the Secret must be `cloud` (this one is fixed).

#### 2.1 (Recommended) Create a dedicated MinIO user for Velero

Avoid using the MinIO root credentials. Create a user limited to the `velero` bucket:

```bash
cat > velero-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": ["arn:aws:s3:::velero/*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": ["arn:aws:s3:::velero"]
    }
  ]
}
EOF

mc admin policy create myminio velero-policy velero-policy.json
mc admin user add myminio velero-user <strong-password>
mc admin policy attach myminio velero-policy --user velero-user
```

> On older `mc` versions use `mc admin policy add` and `mc admin policy set myminio velero-policy user=velero-user`.

For a quick test only, you can use the MinIO root user/password instead (`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`).

#### 2.2 Create the Secret

```bash
kubectl create namespace velero

cat > credentials-velero <<EOF
[default]
aws_access_key_id=velero-user
aws_secret_access_key=<strong-password>
EOF

kubectl create secret generic minio-credentials \
  -n velero --from-file=cloud=credentials-velero

rm credentials-velero velero-policy.json
```

Verify the Secret content:

```bash
kubectl get secret minio-credentials -n velero \
  -o jsonpath='{.data.cloud}' | base64 -d
```

> Never commit credentials to Git. For a Git-managed setup use Sealed Secrets, External Secrets, or SOPS.

### 3. Set the MinIO URL

In `velero-values.yaml`, replace the placeholder:

```yaml
s3Url: http://<minio-ip>:9000
```

### 4. Verify the chart version

`targetRevision` in `velero-app.yaml` is the **chart** version, not the Velero version. Check which Velero version it ships and that the AWS plugin version is compatible:

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
helm search repo vmware-tanzu/velero --versions | head
helm show chart vmware-tanzu/velero --version <chart-version> | grep appVersion
```

### 5. Register the private Git repo in ArgoCD

```bash
argocd repo add https://git.axisapp.io/axis/sysdevops/velero.git \
  --username <user> --password <access-token>
```

Use a GitLab Deploy Token or Project Access Token with `read_repository`.

### 6. Deploy

```bash
git add . && git commit -m "velero: initial setup" && git push

kubectl apply -f velero-app.yaml
argocd app get velero
```

## Verification

```bash
kubectl get pods -n velero                       # velero + one node-agent per node
kubectl get backupstoragelocation -n velero      # PHASE should be Available
kubectl get schedule -n velero
```

If the BackupStorageLocation is `Unavailable`:

```bash
kubectl describe backupstoragelocation default -n velero
kubectl logs deploy/velero -n velero
```

Common causes: bucket missing, wrong credentials, MinIO unreachable from the pods, wrong `s3Url`.

## Configuration reference (`velero-values.yaml`)

| Key | Description |
|---|---|
| `initContainers` | Installs the AWS plugin so Velero can use S3/MinIO |
| `credentials.existingSecret` | Pre-created Secret holding MinIO access/secret keys |
| `configuration.defaultVolumesToFsBackup` | Back up all pod volumes with Kopia by default |
| `configuration.backupStorageLocation` | Where backups are stored (MinIO bucket) |
| `s3ForcePathStyle: "true"` | Required for MinIO (path-style URLs) |
| `deployNodeAgent` | Deploys the node-agent DaemonSet for volume data backup |
| `snapshotsEnabled: false` | Disables volume snapshots (MinIO cannot snapshot disks) |
| `schedules` | Automatic backups (cron, namespaces, TTL) |

### Schedules

The default schedule is disabled. To enable it, set `disabled: false`:

```yaml
schedules:
  argocd-daily:
    disabled: false
    schedule: "0 2 * * *"      # every day at 02:00
    template:
      includedNamespaces:
        - argocd
      ttl: 168h                # keep for 7 days
```

## Usage

### Manual backup

```bash
velero backup create my-backup --include-namespaces <namespace>
velero backup get
velero backup describe my-backup --details
```

### Restore

```bash
velero restore create --from-backup my-backup
velero restore get
```

> Do not manage `Restore` objects through ArgoCD auto-sync; a restore is a one-time action.

### Inspect backup contents

```bash
velero backup download my-backup
tar -xzvf my-backup-data.tar.gz
```

## Known limitations

- **`hostPath` volumes are not supported** by file-system backup. Use `local-path`, NFS, Longhorn, or another real provisioner for PVCs you need backed up.
- **PV/PVC re-binding after restore:** with `Retain` reclaim policy, a restored PVC gets a new UID while the PV keeps the old `claimRef`, so it stays `Released`. Fix:

  ```bash
  kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
  ```

- The Secret is created manually and is not managed by GitOps.

## Troubleshooting

| Symptom | Check |
|---|---|
| ArgoCD `ComparisonError` / auth required | Repo not added to ArgoCD or token expired |
| CRD `metadata.annotations: Too long` | Ensure `ServerSideApply=true` in `syncOptions` |
| BSL `Unavailable` | Bucket exists, MinIO credentials/policy correct, network to MinIO, `s3Url` not a placeholder |
| `AccessDenied` in Velero logs | MinIO user policy does not cover the `velero` bucket |
| Backup `PartiallyFailed` | `velero backup logs <name>` |
| PVC stuck `Pending` after restore | PV `claimRef` (see limitations) |
