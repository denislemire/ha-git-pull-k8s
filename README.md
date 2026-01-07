# Home Assistant Git Pull Add-on for Kubernetes

This repository contains Kubernetes manifests to run the Git Pull add-on functionality as a pod in your Kubernetes cluster. This allows you to automatically sync your Home Assistant configuration from a Git repository, just like the official Home Assistant add-on.

## Features

- ✅ Pulls configuration from a Git repository
- ✅ Supports both `pull` and `reset` git commands
- ✅ Optional automatic Home Assistant restart on config changes
- ✅ Configurable file ignore list for restart triggers
- ✅ Runs as a CronJob (periodic) or Deployment (continuous)
- ✅ Supports SSH and HTTPS authentication
- ✅ Uses the same PVC as your Home Assistant deployment

## Prerequisites

- Kubernetes cluster with Home Assistant already deployed
- Home Assistant running in the `homeassistant` namespace
- PVC named `homeassistant-config` (or update the manifest)
- `kubectl` configured to access your cluster

## Quick Start

### 1. Configure Git Repository

Edit `k8s-git-pull.yaml` and set your repository URL:

```yaml
- name: REPOSITORY
  value: "https://github.com/yourusername/your-config-repo.git"  # CHANGE THIS
```

Or for SSH:

```yaml
- name: REPOSITORY
  value: "git@github.com:yourusername/your-config-repo.git"
```

### 2. Set Up Authentication

#### Option A: SSH Key (Recommended)

1. Generate an SSH key (if you don't have one):
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/git-pull-key -N ""
   ```

2. Add the public key to your Git provider (GitHub, GitLab, etc.)

3. Create a Kubernetes secret:
   ```bash
   kubectl create secret generic git-pull-ssh-key \
     --from-file=id_ed25519=~/.ssh/git-pull-key \
     --from-file=known_hosts=<(ssh-keyscan github.com) \
     -n homeassistant
   ```

4. Update `k8s-git-pull.yaml` to mount the secret (see SSH configuration section below)

#### Option B: Username/Password (Less Secure)

```bash
kubectl create secret generic git-pull-credentials \
  --from-literal=username=your-username \
  --from-literal=password=your-password \
  -n homeassistant
```

Then update the repository URL to include credentials:
```yaml
- name: REPOSITORY
  value: "https://username:password@github.com/user/repo.git"
```

**Note:** For GitHub, personal access tokens are recommended over passwords.

### 3. Configure Options

Edit the environment variables in `k8s-git-pull.yaml`:

```yaml
- name: GIT_BRANCH
  value: "master"  # or "main", or your branch name

- name: GIT_COMMAND
  value: "pull"  # or "reset" for hard reset (WARNING: overwrites local changes)

- name: AUTO_RESTART
  value: "true"  # Set to "true" to auto-restart HA on changes

- name: RESTART_IGNORE
  value: "ui-lovelace.yaml,.gitignore"  # Files that won't trigger restart
```

### 4. Adjust Schedule (CronJob mode)

Edit the schedule in `k8s-git-pull.yaml`:

```yaml
schedule: "*/5 * * * *"  # Every 5 minutes
# schedule: "*/30 * * * *"  # Every 30 minutes
# schedule: "0 * * * *"  # Every hour
```

### 5. Deploy

```bash
# Apply service account and RBAC
kubectl apply -f service-account.yaml

# Apply the git pull CronJob
kubectl apply -f k8s-git-pull.yaml
```

### 6. Verify

Check the logs:

```bash
# View recent jobs
kubectl get jobs -n homeassistant

# View logs of the latest job
kubectl logs -n homeassistant -l app=git-pull --tail=50

# Or follow logs
kubectl logs -n homeassistant -f job/git-pull-<timestamp>
```

## SSH Configuration

To use SSH authentication, update `k8s-git-pull.yaml` to mount the SSH secret:

Add to the container's `volumeMounts`:
```yaml
- name: ssh-key
  mountPath: /root/.ssh
  readOnly: true
```

Add to the pod's `volumes`:
```yaml
- name: ssh-key
  secret:
    secretName: git-pull-ssh-key
    defaultMode: 0600
```

And add to the container's command to set up SSH:
```yaml
command:
- /bin/sh
- -c
- |
  # Set up SSH
  mkdir -p /root/.ssh
  cp /root/.ssh/id_ed25519 /root/.ssh/id_ed25519 2>/dev/null || true
  chmod 600 /root/.ssh/id_ed25519
  chmod 644 /root/.ssh/known_hosts
  
  # Install kubectl and run script
  apk add --no-cache curl
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x kubectl && mv kubectl /usr/local/bin/
  
  cp /scripts/git-pull.sh /tmp/git-pull.sh
  chmod +x /tmp/git-pull.sh
  /tmp/git-pull.sh
```

## Deployment Mode vs CronJob Mode

### CronJob (Default)
- Runs on a schedule (e.g., every 5 minutes)
- More resource-efficient
- Recommended for most use cases

### Deployment (Continuous)
- Runs continuously, checking at intervals
- Uncomment the Deployment section in `k8s-git-pull.yaml`
- Comment out the CronJob section
- Useful if you want immediate updates

## Configuration Options

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `REPOSITORY` | **required** | Git repository URL |
| `GIT_BRANCH` | `master` | Branch to pull from |
| `GIT_COMMAND` | `pull` | `pull` or `reset` (reset overwrites local changes) |
| `GIT_REMOTE` | `origin` | Remote name |
| `GIT_PRUNE` | `false` | Prune deleted remote branches |
| `CONFIG_DIR` | `/config` | Path to HA config directory |
| `AUTO_RESTART` | `false` | Auto-restart HA on config changes |
| `RESTART_IGNORE` | - | Comma-separated list of files to ignore |
| `REPEAT_ACTIVE` | `false` | Enable repeat mode (Deployment only) |
| `REPEAT_INTERVAL` | `300` | Interval in seconds (Deployment only) |

## Troubleshooting

### Pod can't access the config directory
- Verify the PVC name matches: `kubectl get pvc -n homeassistant`
- Check the pod has the correct volume mount

### Git authentication fails
- For SSH: Verify the secret is created and mounted correctly
- For HTTPS: Check credentials in the repository URL or secret
- View pod logs: `kubectl logs -n homeassistant -l app=git-pull`

### Home Assistant doesn't restart
- Verify `AUTO_RESTART=true`
- Check the service account has permissions: `kubectl get rolebinding -n homeassistant`
- Check logs for restart attempts

### First run initializes empty repository
- **WARNING**: Make sure your Git repository has your config files before first run
- The add-on will clone the repository, potentially overwriting local config
- Always backup your config before first run

## Security Considerations

- SSH keys are stored as Kubernetes secrets (encrypted at rest)
- The service account has minimal permissions (only restart HA deployment)
- Consider using read-only Git access tokens/keys
- Review RBAC permissions in `service-account.yaml`

## Differences from Official Add-on

- Runs as a Kubernetes pod instead of a Home Assistant add-on
- Uses `kubectl rollout restart` instead of Home Assistant API restart
- Config validation is skipped (can be enhanced to use HA API)
- No web UI - configure via YAML manifests

## License

This is based on the Home Assistant Git Pull add-on functionality. Use at your own risk.

## Support

For issues with this Kubernetes implementation, check:
- Pod logs: `kubectl logs -n homeassistant -l app=git-pull`
- Job status: `kubectl get jobs -n homeassistant`
- CronJob status: `kubectl get cronjob -n homeassistant`

For Home Assistant Git Pull add-on issues, see:
- [Home Assistant Community Forum](https://community.home-assistant.io)
- [GitHub Issues](https://github.com/home-assistant/addons/issues)

