# Free always-on deployment: Oracle Cloud VM + Cloudflare Tunnel

Result: Claude connects to `https://garmin.<your-domain>/<MCP_SECRET>/mcp`.

## Prerequisites
- Oracle Cloud account with an **Always Free** VM (Ampere A1 ARM or AMD micro), Ubuntu.
  Open no inbound ports other than SSH; the tunnel is outbound only.
- A domain on Cloudflare (free plan). Quick tunnels (`trycloudflare.com`) change URL
  on every restart, so they are not suitable for a Claude connector.

## 1. Set up the VM
```bash
sudo apt-get update && sudo apt-get install -y docker.io docker-compose-v2 git
sudo usermod -aG docker $USER && newgrp docker
git clone https://github.com/mrcaron/garmin_mcp && cd garmin_mcp/deploy/oracle
cp .env.example .env
# fill in .env: openssl rand -hex 32 for MCP_SECRET and GARMIN_MCP_HTTP_TOKEN
chmod 600 .env
```

## 2. Create the tunnel (on the VM, from the cloned repo)
Install `cloudflared` on the VM ([downloads](https://github.com/cloudflare/cloudflared/releases);
on Ubuntu ARM: `curl -fsSL -o cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb && sudo dpkg -i cloudflared.deb`; use `amd64` on x86), then:
```bash
cloudflared tunnel login                  # prints a URL; open it, pick your domain
cloudflared tunnel create garmin-mcp      # prints the tunnel ID, writes ~/.cloudflared/<ID>.json
cloudflared tunnel route dns garmin-mcp garmin.<your-domain>   # adds one CNAME, nothing else changes
cd ~/garmin_mcp/deploy/oracle
cp ~/.cloudflared/<ID>.json cloudflared/
cp cloudflared/config.yml.example cloudflared/config.yml   # replace <TUNNEL_ID> and the hostname
chmod 600 cloudflared/*.json
```
The credentials JSON is a secret: it is git-ignored, don't share it.

## 3. Log in to Garmin once (handles MFA)
```bash
docker compose build garmin-mcp
docker compose run --rm -it -e GARMIN_EMAIL=you@example.com -e GARMIN_PASSWORD='...' \
  --entrypoint garmin-mcp-auth garmin-mcp
```
Tokens are saved to the `garmin-tokens` volume and survive restarts. Re-run this
if Garmin ever invalidates them (roughly every few months).

## 4. Start
```bash
docker compose up -d
docker compose logs -f garmin-mcp     # look for "Serving MCP over streamable-http"
curl -s https://garmin.<your-domain>/<MCP_SECRET>/mcp -o /dev/null -w '%{http_code}\n'  # not 404
```

## 5. Add to Claude
Settings > Connectors > Add custom connector, URL
`https://garmin.<your-domain>/<MCP_SECRET>/mcp`, no OAuth.

## Notes
- The URL is the credential. Anyone with it can read and write your Garmin data; do
  not paste it anywhere public. To rotate: change `MCP_SECRET` in `.env`, then
  `docker compose up -d`, and update the connector.
- Garmin may rate-limit or block datacenter IPs. If login fails from the VM with
  429/403, that is the likely cause.
- Oracle can reclaim idle Always Free instances; light but regular use usually avoids this.
