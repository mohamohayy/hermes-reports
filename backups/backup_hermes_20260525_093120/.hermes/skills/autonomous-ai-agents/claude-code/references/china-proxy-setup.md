# Claude Code Behind the Great Firewall

api.anthropic.com is blocked in mainland China. Claude Code needs a proxy or reverse proxy to function.

## Symptom

```
Unable to connect to Anthropic services
Failed to connect to api.anthropic.com: ERR_BAD_REQUEST (or ETIMEDOUT)
```

## Option 1: Airport + Clash Verge (Most Common)

1. Subscribe to an "airport" service (机场, monthly fee ~¥5-30) — provides a Clash subscription URL
2. Install **Clash Verge** on Windows (GUI client, one-click system proxy)
3. WSL inherits Windows system proxy automatically; if not:
   ```bash
   export HTTPS_PROXY=http://<Windows-IP>:<clash-port>
   export HTTP_PROXY=http://<Windows-IP>:<clash-port>
   ```
4. Verify: `curl -x $HTTPS_PROXY https://api.anthropic.com` should return a non-timeout response

## Option 2: API Reverse Proxy

Some services provide api.anthropic.com reverse proxies for Chinese users. If Claude Code ever supports a custom `ANTHROPIC_BASE_URL` env var, this becomes trivial. As of v2.1.138, it does not expose this.

## Option 3: Self-hosted on VPS

Deploy Shadowsocks/V2Ray/WireGuard on a VPS outside China (AWS, Vultr, etc.). More stable but costs more.

## WSL-Specific Notes

- WSL 2 has its own virtual network; the Windows host IP is in `/etc/resolv.conf` (`nameserver` line)
- Windows system proxy settings (set by Clash Verge) do NOT always propagate to WSL automatically
- If auto-propagation fails, manually set `HTTPS_PROXY` in `~/.bashrc`

## Verification

```bash
# With proxy set:
HTTPS_PROXY=http://<host>:<port> claude -p "say hello" --max-turns 1

# Should return a normal response, not "Unable to connect"
```
