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

## 5. Auto-shutdown Examples
To save costs, you should ensure instances are stopped or deleted after work.

### Method A: Managed Jobs (Recommended for auto-termination)
SkyPilot's job system automatically terminates the instance once the task finishes:
```bash
sky jobs launch -n my-job digitalocean-setup/skypilot/do-ephemeral.yaml
```

### Method B: Autostop (Stops instance after idle time)
This stops the instance if it's idle for 10 minutes:
```bash
sky launch -c my-cluster digitalocean-setup/skypilot/do-test.yaml --idle-minutes-to-autostop 10
```

### Method C: Auto-down (Terminates cluster after task)
```bash
sky launch -c my-ephemeral-cluster digitalocean-setup/skypilot/do-test.yaml --down
```
