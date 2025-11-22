# Named Cloudflare Tunnel Setup

Using a named tunnel gives you a permanent URL that never changes between runs.

## ✅ Setup Complete!

Your tunnel is already configured:
- **Tunnel URL**: `https://local-webhooks.tryfltr.com`
- **Target**: `http://localhost:8787` (R2 Upload Proxy)
- **Config file**: `.env.tunnel` (already created, protected by .gitignore)

## Usage

Just run the script:

```bash
./start-local-dev.sh
```

The script automatically loads `.env.tunnel` if it exists and uses your named tunnel instead of creating random quick tunnels.

## Benefits of Named Tunnels

✅ **Permanent URL** - Never changes, no need to update Modal env vars
✅ **Faster startup** - No URL extraction needed
✅ **Production-like** - Same URL every time for consistent testing
✅ **Easier debugging** - Can configure custom DNS, logging, etc.

## Verify Tunnel Configuration

Make sure your Cloudflare tunnel is configured correctly:

1. Go to your Cloudflare dashboard → Zero Trust → Access → Tunnels
2. Find tunnel: `fltr-local-fastapi-webhooks`
3. Verify the published application route:
   - **Hostname**: `local-webhooks.tryfltr.com`
   - **Service**: `http://localhost:8787` ✅ (Must be port 8787, not 8000)

**Important**: The tunnel must point to port **8787** (R2 proxy), not port 8000 (FastAPI).

## Troubleshooting

### Tunnel not starting
- Verify your token in `.env.tunnel` is correct
- Check that you sourced the env file: `source .env.tunnel`
- View tunnel logs: `tail -f logs/tunnel.log`

### Files not found in Modal
- Verify tunnel points to `http://localhost:8787` (not 8000)
- Check R2 proxy is running: `curl http://localhost:8787/health`
- Ensure tunnel URL is correct: `echo $CLOUDFLARE_TUNNEL_URL`
