# DigitalOcean Setup for SkyPilot

To use DigitalOcean with SkyPilot, follow these steps:

## 1. Create a DigitalOcean API Token
1. Log in to your [DigitalOcean Control Panel](https://cloud.digitalocean.com/).
2. Go to **API** in the left sidebar.
3. Click **Generate New Token**.
4. Give it a name (e.g., `skypilot`) and ensure it has **Write** access.
5. Copy the token.

## 2. Configure SkyPilot
Create a file at `~/.digitalocean/api_key` and paste your token there:

```bash
mkdir -p ~/.digitalocean
echo "YOUR_DIGITALOCEAN_API_TOKEN" > ~/.digitalocean/api_key
chmod 600 ~/.digitalocean/api_key
```

## 3. Verify
Run the following command to check if DigitalOcean is enabled:
```bash
sky check
```

## 4. Launch a Test Instance
```bash
sky launch digitalocean-setup/skypilot/do-test.yaml
```

## 5. Auto-shutdown using Flags
The most common way to ensure instances are deleted after a task is to use flags with the `sky launch` command.

### Method A: Auto-down (Terminates cluster immediately after task)
Recommended for one-off tasks where you don't need to keep the cluster for debugging:
```bash
sky launch digitalocean-setup/skypilot/do-test.yaml --down
```

### Method B: Autostop (Stops instance after idle time)
Best for development. It stops the instance if it remains idle for 10 minutes (prevents billing while idle, but keeps the disk):
```bash
sky launch digitalocean-setup/skypilot/do-test.yaml -i 10
```

### Method C: Managed Jobs (Advanced)
Automatically terminates the instance once the job finishes:
```bash
sky jobs launch digitalocean-setup/skypilot/do-ephemeral.yaml
```
