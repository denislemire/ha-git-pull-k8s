# Quick Start Guide

## Step 1: Set Up Git Repository

Make sure your Home Assistant configuration is in a Git repository (GitHub, GitLab, etc.).

## Step 2: Choose Authentication Method

### Option A: SSH (Recommended)

1. Generate SSH key:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/git-pull-key -N ""
   ```

2. Add public key to your Git provider:
   ```bash
   cat ~/.ssh/git-pull-key.pub
   # Copy and add to GitHub/GitLab SSH keys
   ```

3. Create Kubernetes secret:
   ```bash
   # For GitHub
   ssh-keyscan github.com > /tmp/known_hosts
   kubectl create secret generic git-pull-ssh-key \
     --from-file=id_ed25519=~/.ssh/git-pull-key \
     --from-file=known_hosts=/tmp/known_hosts \
     -n homeassistant
   ```

4. Use `k8s-git-pull-ssh.yaml` and set repository to SSH format:
   ```yaml
   - name: REPOSITORY
     value: "git@github.com:yourusername/your-repo.git"
   ```

### Option B: HTTPS with Token

1. Create a personal access token on GitHub/GitLab

2. Use `k8s-git-pull.yaml` and set repository with token:
   ```yaml
   - name: REPOSITORY
     value: "https://your-token@github.com/yourusername/your-repo.git"
   ```

## Step 3: Configure

Edit `k8s-git-pull.yaml` (or `k8s-git-pull-ssh.yaml` for SSH):

1. Set your repository URL
2. Adjust branch if needed (default: `master`)
3. Set `AUTO_RESTART: "true"` if you want auto-restart on changes
4. Adjust schedule (default: every 5 minutes)

## Step 4: Deploy

```bash
# Apply service account and RBAC
kubectl apply -f service-account.yaml

# Apply the git pull CronJob
kubectl apply -f k8s-git-pull.yaml
# OR for SSH:
# kubectl apply -f k8s-git-pull-ssh.yaml
```

## Step 5: Verify

```bash
# Check CronJob
kubectl get cronjob -n homeassistant

# Check recent jobs
kubectl get jobs -n homeassistant

# View logs
kubectl logs -n homeassistant -l app=git-pull --tail=50
```

## Troubleshooting

- **Can't connect to Git**: Check authentication (SSH key or token)
- **Permission denied**: Verify service account has correct permissions
- **Config not updating**: Check logs for errors
- **HA not restarting**: Set `AUTO_RESTART: "true"` and check RBAC

For more details, see [README.md](README.md).

