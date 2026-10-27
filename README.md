# Proxmox VZDump Backup Hook with Healthchecks Integration

This project provides a Python script that integrates Proxmox VZDump backups with [Healthchecks.io](https://healthchecks.io) or a self-hosted instance for monitoring and alerting.

## Overview

The script runs at various phases of the Proxmox backup task to monitor and report backup status to Healthchecks.io (or self-hosted). It creates check endpoints for each VM/container and for the overall backup job, allowing you to track the progress and success of your backups.

## Features

- Monitors all phases of the Proxmox backup process
- Creates and updates Healthchecks.io endpoints automatically
- Sends real-time status updates during the backup process
- Captures and forwards backup logs to Healthchecks.io
- Works with both standalone Proxmox servers and clusters
- Supports multiple backup jobs

## Prerequisites

- Proxmox VE 6.x or later
- Python 3.6 or later (included with Proxmox)
- Python `requests` library
- `jq` command-line JSON processor
- `git`
- Healthchecks.io account (self-hosted or cloud)

## Installation

The repository is cloned directly into `/etc/pve/healthchecks`. Everything (script, example configuration, and your configuration) then lives inside the git working tree, which makes upgrades a simple `git pull`. Because `/etc/pve` is synchronized across the cluster, the script and its configuration propagate to every node automatically.

### 1. Install dependencies

```bash
apt-get update
apt-get install -y python3-requests jq git
```

### 2. Clone the repository

```bash
git clone https://github.com/entrhopi/proxmox_backup_healthchecks.git /etc/pve/healthchecks
chmod +x /etc/pve/healthchecks/vzdump-hook-script.py
```

### 3. Configure environment variables

The script reads its configuration from `vzdump-hook-script.env` located next to the script (i.e. `/etc/pve/healthchecks/vzdump-hook-script.env`). Create it from the bundled example:

```bash
cp /etc/pve/healthchecks/vzdump-hook-script.env.example /etc/pve/healthchecks/vzdump-hook-script.env
```

Edit the file with your Healthchecks.io details:

```bash
nano /etc/pve/healthchecks/vzdump-hook-script.env
```

Example configuration:

```
# Healthchecks.io Configuration for Proxmox VZDump Backup Hook Script
# This file is loaded by the vzdump-hook-script.py script
# This file should be located at /etc/pve/healthchecks/vzdump-hook-script.env
# Since it's in /etc/pve, it will be synchronized across the Proxmox cluster

# Base domain for Healthchecks API
HC_BASE_DOMAIN=https://<your hosted healthchecks>

# Domain for Healthchecks ping endpoints
HC_PING_DOMAIN=https://<your hosted healthchecks>/ping

# API key with read-write permissions
HC_RW_API_KEY=<your project read-write api key>

# Ping key used for sending pings
HC_PING_KEY=<your project ping key>

# Optional: grace and timeout (in seconds) for the Healthchecks endpoints.
# Leave a line commented out to use the built-in default (shown on the right).
# HC_HOST_GRACE=7200        # host backup job endpoint
# HC_HOST_TIMEOUT=86400     # host backup job endpoint
# HC_VM_GRACE=3600          # per-VM/LXC endpoint
# HC_VM_TIMEOUT=86400       # per-VM/LXC endpoint
```

The file is listed in the repository's `.gitignore`, so your API keys never end up in git.

### 4. Add the script to your backup jobs

Run this command to automatically add the hook script to all existing VZDump backup jobs:

```bash
awk 'BEGIN{job=0;has_script=0}/^vzdump:/{if(job&&!has_script)print"        script /etc/pve/healthchecks/vzdump-hook-script.py";job=1;has_script=0;print;next}/script \/etc\/pve\/healthchecks\/vzdump-hook-script.py/{has_script=1;print;next}/^[a-zA-Z]/{if(job&&$0!~/^[[:space:]]/) {if(!has_script)print"        script /etc/pve/healthchecks/vzdump-hook-script.py";job=0;has_script=0}print;next}{print}END{if(job&&!has_script)print"        script /etc/pve/healthchecks/vzdump-hook-script.py"}' /etc/pve/jobs.cfg > /tmp/jobs.cfg.new && sudo cp /tmp/jobs.cfg.new /etc/pve/jobs.cfg
```
The idea behind this is to add the line "script /etc/pve/healthchecks/vzdump-hook-script.py" under each job in /etc/pve/jobs.cfg

### 5. Verify installation

Test the script with:

```bash
/etc/pve/healthchecks/vzdump-hook-script.py job-init
```

You should see a new check created in your Healthchecks.io dashboard (or self hosted).

## Cluster Installation

If you're running a Proxmox cluster, no extra steps are needed: `/etc/pve` is synchronized across all nodes, so cloning the repository on one node (steps 1-4 above) makes the script and its configuration available on every node. Run the awk command on one node to update all backup jobs.

If for some reason the files do not appear on the other nodes, clone the repository there as well:

```bash
git clone https://github.com/entrhopi/proxmox_backup_healthchecks.git /etc/pve/healthchecks
chmod +x /etc/pve/healthchecks/vzdump-hook-script.py
```

## How It Works

The script hooks into the following phases of the Proxmox backup process:

- `job-init`: Creates the Healthchecks endpoint for the host
- `job-start`: Sends a start ping to the host endpoint
- `backup-start`: Creates and starts the Healthchecks endpoint for the VM/container
- `pre-stop`, `pre-restart`, `post-restart`: Logs progress during VM/container snapshots
- `backup-end`: Sends a success ping when a VM/container backup completes
- `backup-abort`: Sends a failure ping when a VM/container backup is aborted
- `log-end`: Sends VM/container logs to Healthchecks
- `job-end`: Sends a success ping when the entire backup job completes, or a failure ping with the collected error log if any step reported an error
- `job-abort`: Sends a failure ping when the backup job is aborted

When issues occur, the script captures errors and reports them to Healthchecks.io with detailed logs.

## Customization

You can customize the script behavior by modifying the variables in `/etc/pve/healthchecks/vzdump-hook-script.env`. The most important options are:

- `HC_BASE_DOMAIN`: Your Healthchecks instance base URL
- `HC_PING_DOMAIN`: URL for sending pings (usually `HC_BASE_DOMAIN/ping`)
- `HC_RW_API_KEY`: API key with read-write permissions
- `HC_PING_KEY`: Key used for sending pings
- `HC_HOST_GRACE`: (optional) Grace period in seconds for the host backup job endpoint (default `7200`)
- `HC_HOST_TIMEOUT`: (optional) Maximum run time in seconds for the host backup job endpoint (default `86400`)
- `HC_VM_GRACE`: (optional) Grace period in seconds for each VM/LXC endpoint (default `3600`)
- `HC_VM_TIMEOUT`: (optional) Maximum run time in seconds for each VM/LXC endpoint (default `86400`)

The grace period is how long a check may be silent before Healthchecks marks it as failing. The timeout is the maximum time a backup is allowed to run. Any of these can be left unset to fall back to the defaults above.

## Upgrading

To upgrade to a newer version, pull the latest changes on any node in the cluster:

```bash
git -C /etc/pve/healthchecks pull
```

Because `/etc/pve` is synchronized across the cluster, the update propagates to all nodes automatically. Your `vzdump-hook-script.env` is not part of the git repository, so it is left untouched by the pull; if a release changes the configuration options, compare against the updated `vzdump-hook-script.env.example` and adjust your file accordingly.

## Troubleshooting

If you encounter issues:

1. Check the script permissions: `chmod +x /etc/pve/healthchecks/vzdump-hook-script.py`
2. Verify Python and requests are installed: `python3 -c "import requests"`
3. Check for errors in the Proxmox task logs
4. Manually run the script with different phases to test
5. Ensure your Healthchecks API keys have the correct permissions

## Author & Changelog

Author: Cédric MARCOUX  
Localisation: Aywaille, Belgium  

## License

[MIT License](LICENSE)

## Acknowledgments

- [Proxmox Team](https://www.proxmox.com/) for the VZDump backup tool
- [Healthchecks.io](https://healthchecks.io) for the monitoring service
- Original script based on [waza-ari's work](https://gist.github.com/waza-ari/8fb8375ec5770a50486abeb2a7bb9c52)