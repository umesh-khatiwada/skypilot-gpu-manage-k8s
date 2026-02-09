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
