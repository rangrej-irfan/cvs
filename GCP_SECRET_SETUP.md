# GCP Service Account Key Setup Guide

This guide walks you through creating a Kubernetes Secret for your GCP service account key, mounting it in the RunnerDeployment, and validating it works.

## Prerequisites

- Access to your GKE cluster with `kubectl` configured
- GCP service account JSON key file (`.gcp-service-account-key.json`)
- RunnerDeployment already created (if not, use `setup-runner.sh` first)

## Step 1: Create Kubernetes Secret

### Option A: Using the Helper Script (Recommended)

If you have the key file locally and `kubectl` access:

```bash
cd privacera-manager
./actions-runner/apply-gcp-secret.sh
```

This script will:
- Validate your `.gcp-service-account-key.json` file
- Create the Kubernetes Secret
- Update the RunnerDeployment automatically

### Option B: Manual Creation

If you need to create it manually:

```bash
# Create the secret from your key file
kubectl create secret generic gcp-service-account-key \
    --namespace=actions-runner-system \
    --from-file=gcs.json=/path/to/your/.gcp-service-account-key.json

# Or if you want to use a different filename:
kubectl create secret generic gcp-service-account-key \
    --namespace=actions-runner-system \
    --from-file=service-account-key.json=/path/to/your/.gcp-service-account-key.json
```

**Note:** The filename you use here (`gcs.json` or `service-account-key.json`) must match what the workflow expects. The workflow checks for both.

### Option C: Create from Pasted JSON

If you can't access the file directly:

```bash
# Run the interactive script
./actions-runner/create-secret-manually.sh

# It will prompt you to paste your JSON key
# Then create the secret automatically
```

## Step 2: Mount Secret in RunnerDeployment

### Option A: Using the Helper Script (Recommended)

The `apply-gcp-secret.sh` script automatically updates the RunnerDeployment. If you created the secret manually, you still need to mount it.

### Option B: Manual Patch Command

```bash
kubectl patch runnerdeployment privacera-manager-deploy \
    -n actions-runner-system \
    --type='merge' \
    -p='
{
  "spec": {
    "template": {
      "spec": {
        "volumes": [
          {
            "name": "gcp-secrets",
            "secret": {
              "secretName": "gcp-service-account-key"
            }
          }
        ],
        "volumeMounts": [
          {
            "name": "gcp-secrets",
            "mountPath": "/etc/gcp-secrets",
            "readOnly": true
          }
        ]
      }
    }
  }
}'
```

### Option C: Edit RunnerDeployment Manually

```bash
# Edit the RunnerDeployment
kubectl edit runnerdeployment privacera-manager-deploy -n actions-runner-system
```

Add the following to `spec.template.spec`:

```yaml
volumes:
  - name: gcp-secrets
    secret:
      secretName: gcp-service-account-key
volumeMounts:
  - name: gcp-secrets
    mountPath: /etc/gcp-secrets
    readOnly: true
```

## Step 3: Wait for Runner Pod to Restart

After patching the RunnerDeployment, the runner pod will automatically restart:

```bash
# Watch the pod restart
kubectl get pods -n actions-runner-system -w

# You should see the pod restart with the new volume mount
```

Wait until the pod shows `Running` status (usually 30-60 seconds).

## Step 4: Validate the Secret is Mounted

### 4.1 Get the Runner Pod Name

```bash
# Get the pod name
RUNNER_POD=$(kubectl get pods -n actions-runner-system -o name | grep privacera-manager-deploy | head -1 | cut -d/ -f2)
echo "Runner pod: $RUNNER_POD"

# Or use the pod name directly if you see it:
# RUNNER_POD="privacera-manager-deploy-xxxxx-xxxxx"
```

### 4.2 Check if the Directory Exists

```bash
kubectl exec -n actions-runner-system $RUNNER_POD -- ls -la /etc/gcp-secrets/
```

**Expected output:**
```
total 4
drwxrwxrwt 3 root root  100 Dec  2 20:04 .
drwxr-xr-x 1 root root 4096 Dec  2 20:04 ..
drwxr-xr-x 2 root root   60 Dec  2 20:04 ..2025_12_02_20_04_18.751201444
lrwxrwxrwx 1 root root   31 Dec  2 20:04 ..data -> ..2025_12_02_20_04_18.751201444
lrwxrwxrwx 1 root root   15 Dec  2 20:04 gcs.json -> ..data/gcs.json
```

### 4.3 Verify the JSON File Exists

```bash
# Check if gcs.json exists (if you used that filename)
kubectl exec -n actions-runner-system $RUNNER_POD -- cat /etc/gcp-secrets/gcs.json | head -5

# Or check for service-account-key.json (if you used that filename)
kubectl exec -n actions-runner-system $RUNNER_POD -- cat /etc/gcp-secrets/service-account-key.json | head -5
```

**Expected output:**
```json
{
  "type": "service_account",
  "project_id": "your-project-id",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n..."
```

### 4.4 Validate JSON Format

```bash
# Validate the JSON is properly formatted
kubectl exec -n actions-runner-system $RUNNER_POD -- python3 -m json.tool /etc/gcp-secrets/gcs.json > /dev/null && echo "✅ JSON is valid" || echo "❌ JSON is invalid"
```

### 4.5 Test GCP Authentication (Optional)

```bash
# Install gcloud in the pod (if not already installed)
kubectl exec -n actions-runner-system $RUNNER_POD -- bash -c "
  if ! command -v gcloud &> /dev/null; then
    echo 'Installing gcloud...'
    echo 'deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main' | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
    apt-get update && apt-get install -y google-cloud-cli
  fi
  
  # Test authentication
  gcloud auth activate-service-account --key-file=/etc/gcp-secrets/gcs.json
  gcloud auth list
  echo '✅ GCP authentication successful'
"
```

## Step 5: Verify in GitHub Actions Workflow

Run your GitHub Actions workflow. The workflow will:

1. **Find the key file** at `/etc/gcp-secrets/gcs.json` (or `service-account-key.json`)
2. **Validate the JSON format**
3. **Authenticate to GCP** after `gcloud` is installed
4. **Use the credentials** for all GCP operations

Look for these log messages in the workflow:

```
✅ Found key file in mounted secret: /etc/gcp-secrets/gcs.json
✅ Key file found and validated: /etc/gcp-secrets/gcs.json
✅ GCP authentication successful
```

## Troubleshooting

### Secret Not Found

**Error:** `secret "gcp-service-account-key" not found`

**Solution:**
```bash
# Check if secret exists
kubectl get secret gcp-service-account-key -n actions-runner-system

# If not found, create it (see Step 1)
```

### File Not Mounted

**Error:** `ls: cannot access '/etc/gcp-secrets/gcs.json': No such file or directory`

**Solution:**
1. Check if volumeMount is in RunnerDeployment:
   ```bash
   kubectl get runnerdeployment privacera-manager-deploy -n actions-runner-system -o yaml | grep -A 10 volumeMounts
   ```

2. If missing, add it (see Step 2)

3. Restart the runner pod:
   ```bash
   kubectl delete pod -n actions-runner-system -l runner-template
   ```

### Wrong Filename

**Error:** Workflow finds the directory but not the file

**Solution:**
The workflow checks for both `gcs.json` and `service-account-key.json`. Make sure your secret has the file with one of these names:

```bash
# Check what files are in the secret
kubectl get secret gcp-service-account-key -n actions-runner-system -o jsonpath='{.data}' | jq 'keys'

# If you need to recreate with correct filename:
kubectl delete secret gcp-service-account-key -n actions-runner-system
kubectl create secret generic gcp-service-account-key \
    --namespace=actions-runner-system \
    --from-file=gcs.json=/path/to/your/key.json
```

### Pod Not Restarting

**Solution:**
```bash
# Force delete the pod to trigger recreation
kubectl delete pod -n actions-runner-system -l runner-template

# Or restart the RunnerDeployment
kubectl rollout restart runnerdeployment privacera-manager-deploy -n actions-runner-system
```

## Quick Reference Commands

```bash
# 1. Create secret
kubectl create secret generic gcp-service-account-key \
    --namespace=actions-runner-system \
    --from-file=gcs.json=/path/to/key.json

# 2. Mount in RunnerDeployment
kubectl patch runnerdeployment privacera-manager-deploy -n actions-runner-system --type='merge' -p='{"spec":{"template":{"spec":{"volumes":[{"name":"gcp-secrets","secret":{"secretName":"gcp-service-account-key"}}],"volumeMounts":[{"name":"gcp-secrets","mountPath":"/etc/gcp-secrets","readOnly":true}]}}}}'

# 3. Verify
RUNNER_POD=$(kubectl get pods -n actions-runner-system -o name | grep privacera-manager-deploy | head -1 | cut -d/ -f2)
kubectl exec -n actions-runner-system $RUNNER_POD -- ls -la /etc/gcp-secrets/
kubectl exec -n actions-runner-system $RUNNER_POD -- cat /etc/gcp-secrets/gcs.json | head -5
```

## Security Best Practices

1. ✅ **Never commit** the JSON key file to git (it's in `.gitignore`)
2. ✅ **Use Kubernetes Secrets** instead of files in the repository
3. ✅ **Mount as read-only** (already configured with `readOnly: true`)
4. ✅ **Rotate keys regularly** - delete old secrets and create new ones
5. ✅ **Use least privilege** - only grant the IAM roles needed for the workflow
6. ✅ **Monitor secret access** - use GCP audit logs to track key usage

## Updating the Secret

If you need to update the key file:

```bash
# Delete the old secret
kubectl delete secret gcp-service-account-key -n actions-runner-system

# Create new secret with updated key
kubectl create secret generic gcp-service-account-key \
    --namespace=actions-runner-system \
    --from-file=gcs.json=/path/to/new-key.json

# The pod will automatically pick up the new secret (Kubernetes updates mounted secrets)
# Or force restart:
kubectl delete pod -n actions-runner-system -l runner-template
```

