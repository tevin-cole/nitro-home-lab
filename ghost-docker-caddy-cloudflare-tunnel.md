# Ghost CMS Docker + Caddy + Cloudflare Tunnel: Complete Troubleshooting Guide

A practical guide for self-hosters running **Ghost CMS in Docker** behind **Caddy** as a reverse proxy, exposed via a **Cloudflare Zero Trust Tunnel** — without opening any ports on your firewall.

This guide documents the real-world issues encountered during this setup and their fixes, written for homelabbers who are hitting the same walls.

---

## Stack Overview

| Layer | Technology |
|---|---|
| Blog CMS | Ghost (official Docker image, Caddy variant) |
| Reverse Proxy | Caddy 2 (bundled with Ghost's official Docker stack) |
| Tunnel / Exposure | Cloudflare Zero Trust Tunnel (cloudflared) |
| Container Runtime | Docker + Docker Compose |
| DNS / CDN | Cloudflare |

**Traffic flow:**
```
Browser → Cloudflare Edge (TLS termination) → Cloudflare Tunnel → Caddy :80 → Ghost :2368
```

---

## Prerequisites

- A domain managed by Cloudflare
- Docker and Docker Compose installed on your host
- A Cloudflare Zero Trust tunnel created and connector running
- Ghost's official Docker Compose stack cloned from [ghost-docker](https://github.com/TryGhost/ghost-docker)

---

## Common Issues & Fixes

### Issue 1: `ERR_TOO_MANY_REDIRECTS` / "The page isn't redirecting properly"

**Symptoms:**
- Firefox: `The page isn't redirecting properly`
- Chrome: `ERR_TOO_MANY_REDIRECTS`
- Ghost container logs show **no incoming requests at all** — the loop is happening before Ghost is ever reached

**What's happening:**

This is an infinite redirect loop caused by a chain-of-trust mismatch across three layers. Cloudflare terminates TLS at the edge and forwards traffic to your server over plain HTTP. Ghost, seeing an HTTP request, checks its configured `url` (which is `https://...`) and issues a `301 redirect` to HTTPS. That redirect goes back through Cloudflare, which again sends HTTP to Caddy — and the cycle never ends.

A secondary cause: Caddy exposes both port 80 and port 443 in the Docker Compose file. Even with `auto_https` not explicitly configured, Caddy will attempt to redirect HTTP→HTTPS when it sees port 443 is bound, which compounds the loop before Ghost even sees the request.

**The fix:**

**Step 1 — Explicitly declare the site block as HTTP and disable Caddy's automatic HTTPS.**

Add a global options block to the top of your `caddy/Caddyfile`, and explicitly use `http://` in the site block:

```caddyfile
{
    auto_https off
}

http://{$DOMAIN} {
    handle {
        reverse_proxy ghost:2368 {
            # Tell Ghost the original request came in over HTTPS
            # Without this, Ghost will keep trying to redirect to HTTPS → infinite loop
            header_up X-Forwarded-Proto https
            header_up Host {host}
        }
    }
}
```

> **Note:** The `http://` prefix in the site block is required here. Using a bare `{$DOMAIN}` without a protocol may seem cleaner but in practice — at least with this stack — Caddy does not reliably stop redirect-looping without the explicit `http://` declaration. The `auto_https off` global option is still needed alongside it.

The `header_up X-Forwarded-Proto https` header is the critical piece. It tells Ghost: "the original client request came in over HTTPS — don't redirect." Without this header, Ghost sees a plain HTTP connection, assumes it's insecure, and issues the redirect that causes the loop.

**Step 2 — Verify your Cloudflare Tunnel public hostname.**

In your Cloudflare Zero Trust dashboard under **Networks → Tunnels → [your tunnel] → Public Hostnames**, ensure the service is configured as:

```
Type: HTTP
URL: <your-host-ip>:80
```

Not HTTPS. Cloudflare handles TLS — the connection from the tunnel to Caddy should be plain HTTP on port 80.

**Step 3 — Force-recreate your containers.**

Caddy does not hot-reload config changes. You must force-recreate:

```bash
docker compose up -d --force-recreate ghost caddy
```

---

### Issue 2: `.env` file — should `DOMAIN` include `https://`?

**Short answer: No.**

In the official Ghost Docker Compose stack, the `compose.yml` already prepends the protocol:

```yaml
environment:
  url: https://${DOMAIN}
```

So your `.env` file should contain only the bare domain:

```env
DOMAIN=blog.yourdomain.com
```

Adding `https://` to `DOMAIN` in `.env` would result in `url: https://https://blog.yourdomain.com` — a broken double-protocol value. The compose file handles the protocol for you.

---

### Issue 3: `SSL_ERROR_INTERNAL_ERROR_ALERT` (Firefox)

**Symptoms:**
- Firefox throws `SSL_ERROR_INTERNAL_ERROR_ALERT`
- Ports 80 and 443 appear **closed** when you nmap the host IP

**What's happening:**

This usually means **Caddy failed to start entirely** — so nothing is listening on those ports. The most common causes in a Ghost Docker setup:

1. **Caddy is trying to provision a Let's Encrypt certificate but can't** — either because your domain's DNS doesn't point to your public IP, or port 80/443 aren't reachable from the internet (which is expected if you're using a Cloudflare Tunnel and haven't opened ports). With `auto_https off` in your Caddyfile, this is resolved.

2. **The `Caddyfile` got created as a directory instead of a file** — this is a common Docker gotcha. If the `caddy/Caddyfile` path doesn't exist on the host when Docker Compose runs, Docker creates a *directory* at that path instead of mounting a file. Caddy then crashes because it finds a directory where it expects its config file.

**Fix for the directory issue:**

```bash
# Stop everything
docker compose down

# Check if Caddyfile is a directory
ls -la caddy/

# If it's a directory, remove it and create the actual file
rm -rf caddy/Caddyfile
touch caddy/Caddyfile
# Then populate it with your config (see Issue 1 fix above)

docker compose up -d
docker ps  # Confirm caddy, ghost, and db are all "Up"
```

---

### Issue 4: Ghost admin panel 403 — "Authorization failed, cookies not being passed through"

**Symptoms:**
- The main blog loads fine at `yourdomain.com`
- Navigating to `yourdomain.com/ghost` (admin panel) returns:
  ```
  403 Authorization failed
  "Unable to determine the authenticated user or integration. 
  Check that cookies are being passed through if using session authentication."
  ```
- Docker logs show: `Expected 'http://yourdomain.com' received 'https://yourdomain.com'`

**What's happening:**

Ghost's admin session cookie is set with the `Secure` flag because your configured `url` is `https://`. The browser only sends `Secure` cookies over HTTPS connections. However, if there's any part of the stack that causes Ghost to see the request as HTTP — for example, hardcoding `http://{$DOMAIN}` in your Caddyfile site block — Ghost's session validation fails because the protocol it *expected* doesn't match what the browser presented.

This is also the same error you'll see if the `X-Forwarded-Proto https` header is missing from your Caddy config, because Ghost uses that header to determine the protocol of the original request.

**Fixes to try in order:**

1. **Ensure `X-Forwarded-Proto https` is set** in your Caddyfile reverse_proxy block (see Issue 1 fix). This is the most common cause.

2. **Use `http://{$DOMAIN}` in your Caddyfile site block with `auto_https off` in the global block.** The explicit `http://` prefix is what actually stops the redirect loop. Using a bare `{$DOMAIN}` without a protocol was not sufficient in practice.

3. **Clear browser cookies** for your domain before retesting. Stale session cookies from previous broken states will keep producing 403s even after the config is fixed.

4. **Check Cloudflare for cookie-modifying rules** — under Rules → Transform Rules. Cloudflare can sometimes strip or alter cookies depending on your WAF or cache rules.

5. **Try an incognito window** to rule out browser state as a factor.

**Known limitation:**

If you are running `cloudflared` as an external process pointing at the Docker host IP (rather than as a container inside the same Docker Compose network), there can be subtle session/cookie issues that are harder to resolve. The cleanest architecture for this full stack is to run `cloudflared` as a container inside the same Docker network:

```yaml
# In your docker-compose.yml
services:
  tunnel:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=your_tunnel_token_here
    networks:
      - ghost_network
```

Then set the tunnel's public hostname service URL to `http://caddy:80` (using the Docker service name, not an IP address).

---

### Issue 5: Caddy container not starting / missing from `docker ps`

**Symptoms:**
- `docker ps` only shows `ghost-ghost-1` and `ghost-db-1`
- No caddy container visible

**Diagnosis:**

```bash
# Show all containers including stopped/crashed ones
docker ps -a

# Check why caddy crashed
docker logs <caddy_container_id>
```

Common log errors and what they mean:

| Log Error | Cause |
|---|---|
| `reading /etc/caddy/Caddyfile: is a directory` | Caddyfile was created as a directory by Docker (see Issue 3) |
| `bind: address already in use` | Another process (nginx, apache) is already on port 80/443 |
| `syntax error` | Typo or missing brace in your Caddyfile |
| `could not get certificate` | Caddy is trying Let's Encrypt without `auto_https off` |

---

## Final Working Configuration

### `caddy/Caddyfile`

```caddyfile
{
    auto_https off
}

http://{$DOMAIN} {
    handle {
        reverse_proxy ghost:2368 {
            header_up X-Forwarded-Proto https
            header_up Host {host}
        }
    }
}
```

> The `http://` prefix on the site block is required. A bare `{$DOMAIN}` without a protocol did not reliably prevent the redirect loop in testing with this stack, even with `auto_https off` set.

### `.env` (relevant fields)

```env
DOMAIN=blog.yourdomain.com
# Note: no https:// prefix — compose.yml adds it automatically
```

### Cloudflare Tunnel Public Hostname

```
Subdomain: blog
Domain: yourdomain.com
Service Type: HTTP
Service URL: <docker-host-ip>:80
```

Or, if cloudflared runs inside the same Docker Compose network:

```
Service Type: HTTP
Service URL: caddy:80
```

---

## Why This Setup Exists (Architecture Notes)

In a standard Ghost Docker setup without a tunnel, Caddy handles everything: it obtains Let's Encrypt certificates, terminates TLS on port 443, and proxies to Ghost. This works great but requires ports 80 and 443 to be open and forwarded from your router.

With a **Cloudflare Tunnel**, TLS termination moves entirely to Cloudflare's edge. The tunnel creates an outbound-only connection from your server to Cloudflare — no open inbound ports required. This means:

- Caddy no longer needs to handle certificates
- Caddy no longer needs to listen on 443
- `auto_https off` is essential, otherwise Caddy fights with the tunnel
- Ghost must still be told the requests are HTTPS (via `X-Forwarded-Proto`) or it will redirect-loop forever

The `X-Forwarded-Proto https` header is the glue that makes the whole thing work. It preserves the protocol context across the Cloudflare edge → tunnel → Caddy → Ghost chain.

---

## Useful Commands

```bash
# Force-recreate caddy and ghost (required after Caddyfile changes)
docker compose up -d --force-recreate ghost caddy

# Tail Ghost logs
docker compose logs -f ghost

# Tail Caddy logs
docker compose logs -f caddy

# Check all containers including stopped ones
docker ps -a

# Restart everything from scratch
docker compose down && docker compose up -d
```

---

## References

- [Ghost Docker official docs](https://ghost.org/docs/install/docker/)
- [Caddy reverse_proxy documentation](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [Cloudflare Tunnel documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Ghost Forum — Cloudflare Tunnel with reverse proxy](https://forum.ghost.org/)

---

## Contributing

If you've hit a related issue not covered here, PRs and issues are welcome. This doc is specifically focused on the **Ghost official Docker stack + Caddy + Cloudflare Tunnel** combination — configurations using Nginx Proxy Manager, Traefik, or direct port forwarding may differ.
