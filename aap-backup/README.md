# AAP Backup to GitHub

This Helm chart automates the backup of Ansible Automation Platform (AAP) inventories to a GitHub repository. Backups are scheduled via a CronJob that exports inventories and pushes them to GitHub, with automatic rotation of old backups (keeping the latest 20).

## Overview

- **What it backs up**: Only AAP inventories (not all assets)
- **Storage**: GitHub repository (no PVC required)
- **Schedule**: Configurable via CronJob (default: every hour)
- **Retention**: Keeps the latest 20 backup files
- **Authentication**: Uses Kubernetes secrets for GitHub credentials

## Prerequisites

1. OpenShift/OKD cluster with appropriate permissions
2. GitHub repository (private or public) - create it before deployment
3. GitHub personal access token (PAT) or password with repository write permissions
4. Helm 3.x installed
5. Access to the `aap` namespace (or modify the namespace in values.yaml)
6. AAP Controller credentials (username and password)

## Implementation Flow

### Step 1: Prepare GitHub Repository

1. Create a new GitHub repository (or use an existing one)
2. Note the repository URL (e.g., `https://github.com/your-username/aap-backups.git`)
3. Generate a GitHub Personal Access Token (PAT):
   - Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Generate a new token with `repo` scope (or `public_repo` for public repositories)

### Step 2: Create GitHub Credentials Secret

Create a Kubernetes secret containing your GitHub username and token/password:

```bash
oc create secret generic github-credentials \
  --from-literal=username='your-github-username' \
  --from-literal=password='your-github-token-or-password' \
  -n aap
```

**Note**: Replace:
- `your-github-username` with your actual GitHub username
- `your-github-token-or-password` with your GitHub PAT or password
- `aap` with your namespace if different

Verify the secret was created:
```bash
oc get secret github-credentials -n aap
```

### Step 3: Verify AAP Controller Secret Exists

Ensure the AAP controller admin password secret exists. If it doesn't, create it:

```bash
oc create secret generic aap-controller-admin-password \
  --from-literal=password='your-aap-admin-password' \
  -n aap
```

Verify:
```bash
oc get secret aap-controller-admin-password -n aap
```

### Step 4: Configure values.yaml

Edit `values.yaml` to match your environment:

```yaml
namespace: aap  # Change if using a different namespace
schedule: "0 * * * *"  # Cron schedule (default: every hour)
backupFilenamePrefix: export-results  # Prefix for backup files
# GitHub Configuration
githubSecretName: "github-credentials"  # Should match the secret created in Step 2
githubRepoUrl: "https://github.com/your-username/your-repo.git"  # UPDATE THIS!
# Controller Configuration
controllerHost: "https://aap-controller-aap.apps.yb-ocp4.rh-igc.com"  # Your AAP controller URL
controllerUsername: "admin"  # Your AAP controller username
controllerAdminSecretName: "aap-controller-admin-password"  # Should match Step 3
```

**Important**: Update `githubRepoUrl` with your actual GitHub repository URL.

### Step 5: Deploy the Helm Chart

Install or upgrade the Helm chart:

```bash
# From the aap-backup directory
cd aap/aap-backup

# Install/Upgrade the chart
helm upgrade --install aap-backup . \
  --namespace aap \
  --create-namespace \
  -f values.yaml
```

**Alternative**: If you want to override specific values at install time:

```bash
helm upgrade --install aap-backup . \
  --namespace aap \
  --create-namespace \
  --set githubRepoUrl="https://github.com/your-username/your-repo.git" \
  --set controllerHost="https://your-controller-url" \
  --set schedule="0 * * * *"
```

### Step 6: Verify Deployment

Check that all resources were created:

```bash
# Check CronJob
oc get cronjob aap-backup -n aap

# Check ConfigMap
oc get configmap aap-backup-playbook -n aap

# Check PrometheusRule (if monitoring is enabled)
oc get prometheusrule aap-backup-failure -n aap
```

### Step 7: Test the Backup (Manual Trigger)

You can manually trigger a backup job without waiting for the schedule:

```bash
# Create a job from the CronJob
oc create job --from=cronjob/aap-backup aap-backup-manual-$(date +%s) -n aap
```

Check the job status:
```bash
# List jobs
oc get jobs -n aap

# View job logs
oc logs job/aap-backup-manual-<timestamp> -n aap

# Or follow the pod logs
oc get pods -n aap -l job-name=aap-backup-manual-<timestamp>
oc logs <pod-name> -n aap
```

### Step 8: Verify Backup in GitHub

1. Go to your GitHub repository
2. Check that backup files are being created with the format: `export-results-YYYYMMDD-HHMMSS.yaml`
3. Verify commits are being made with messages like "Add backup for YYYYMMDD-HHMMSS"

## Monitoring

### Check CronJob Status

```bash
# View CronJob details
oc describe cronjob aap-backup -n aap

# List all jobs created by the CronJob
oc get jobs -n aap | grep aap-backup
```

### View Logs

```bash
# Find the latest backup pod
oc get pods -n aap -l job-name

# View logs
oc logs <pod-name> -n aap

# Follow logs in real-time
oc logs -f <pod-name> -n aap
```

### Check Job History

```bash
# View successful jobs (keeps last 2)
oc get jobs -n aap | grep aap-backup | grep Complete

# View failed jobs (keeps last 2)
oc get jobs -n aap | grep aap-backup | grep Error
```

## Troubleshooting

### Job Fails to Start

**Issue**: CronJob not creating jobs
- Check CronJob status: `oc describe cronjob aap-backup -n aap`
- Verify schedule syntax: Use [cron expression validator](https://crontab.guru/)

### Authentication Errors

**Issue**: GitHub authentication fails
- Verify secret exists: `oc get secret github-credentials -n aap`
- Check secret keys: `oc get secret github-credentials -n aap -o jsonpath='{.data}'`
- Ensure token has `repo` scope for private repos or `public_repo` for public repos

### AAP Controller Connection Errors

**Issue**: Cannot connect to AAP controller
- Verify controller URL is correct in `values.yaml`
- Check AAP controller secret: `oc get secret aap-controller-admin-password -n aap`
- Test connectivity from a pod: `oc run -it --rm test --image=curlimages/curl -- curl -k <CONTROLLER_URL>`

### Git Push Fails

**Issue**: Cannot push to GitHub
- Verify repository URL is correct
- Check that the GitHub token has write permissions
- Review pod logs for specific error messages
- Ensure repository exists and is accessible

### No Backup Files in GitHub

**Issue**: Job runs but no files appear
- Check pod logs for errors during export
- Verify AAP controller export is working: `ansible.controller.export` module
- Check if git commit/push steps are executing
- Verify file permissions in the repository

### Export Fails

**Issue**: Inventory export fails
- Check AAP controller is accessible
- Verify controller credentials are correct
- Check if inventories exist in AAP
- Review retry logic (retries 5 times with 15s delay)

## Configuration Reference

### Schedule Format

The `schedule` field uses standard cron syntax:
- `"0 * * * *"` - Every hour at minute 0
- `"0 */2 * * *"` - Every 2 hours
- `"0 0 * * *"` - Daily at midnight
- `"0 0 * * 0"` - Weekly on Sunday at midnight

### Backup File Naming

Files are named: `<backupFilenamePrefix>-<timestamp>.yaml`

Example: `export-results-20240128-143000.yaml`

### Retention Policy

- Keeps the latest **20 backup files**
- Older files are automatically deleted and the deletion is committed to GitHub

## Uninstallation

To remove the backup solution:

```bash
# Delete the Helm release
helm uninstall aap-backup -n aap

# Optionally delete secrets (be careful!)
oc delete secret github-credentials -n aap
oc delete secret aap-controller-admin-password -n aap
```

## Security Considerations

1. **Secrets**: GitHub credentials are stored in Kubernetes secrets (encrypted at rest in etcd)
2. **Token Permissions**: Use a GitHub PAT with minimal required permissions (`repo` scope only)
3. **Repository**: Consider using a private GitHub repository for sensitive inventory data
4. **Logs**: The playbook uses `no_log: true` to prevent credentials from appearing in logs
5. **Network**: Ensure proper network policies if restricting pod-to-pod communication

## Support

For issues or questions:
1. Check pod logs: `oc logs <pod-name> -n aap`
2. Review CronJob events: `oc describe cronjob aap-backup -n aap`
3. Verify all secrets and configurations are correct

