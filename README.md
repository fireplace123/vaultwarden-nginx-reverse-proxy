# Vaultwarden Behind an Nginx Reverse Proxy (with TLS)

A self-hosted password manager (Vaultwarden) running in Docker, fronted by an Nginx reverse proxy that handles TLS termination with a self-signed certificate. Built as a home lab to practice the exact architecture used in production web stacks.

## Architecture

```
           ┌──────────────────────────────────────────────┐
           │              Ubuntu VM (VirtualBox)          │
           │                                              │
  HTTPS    │   ┌─────────┐        HTTP         ┌───────┐  │
 ────────► │──►│  Nginx  │────────────────────►│ Vault-│  │
 port 443  │   │ (TLS    │   127.0.0.1:8080    │warden │  │
           │   │  term)  │                     │ (Docker)│ │
           │   └─────────┘                     └───────┘  │
           │                                              │
           └──────────────────────────────────────────────┘
```

The browser connects to Nginx over TLS on port 443. Nginx terminates the encrypted connection and forwards the request as plain HTTP to the Vaultwarden container bound to localhost. The backend never has to know certificates exist.

## Why This Pattern Matters

This is how real production web stacks work. Cloudflare, AWS Application Load Balancers, Kubernetes ingress controllers — they all do the same thing: a reverse proxy out front handles TLS, rate limiting, compression, and load balancing, while the application container stays focused on application logic.

Learning this locally means you understand:

- **TLS termination** — where and why encryption stops at the edge
- **Reverse proxy request flow** — how `proxy_pass`, headers, and upstream blocks fit together
- **Defense in depth** — binding the backend to `127.0.0.1` so it's unreachable except through the proxy
- **Certificate fundamentals** — generating, installing, and trusting an X.509 cert
- **Container networking** — exposing a port to the host without exposing it to the world

## Stack

| Component    | Version / Role                                    |
| ------------ | ------------------------------------------------- |
| Ubuntu       | 22.04 LTS (VirtualBox VM)                         |
| Nginx        | Reverse proxy, TLS termination on `:443`          |
| Vaultwarden  | Bitwarden-compatible password manager (Rust impl) |
| Docker       | Runs Vaultwarden, bound to `127.0.0.1:8080`       |
| OpenSSL      | Self-signed RSA 2048 cert, 365-day validity       |
| UFW          | Only `:443/tcp` exposed                           |

## Quick Look

**Nginx server block** (full file in [`nginx/vaultwarden.conf`](nginx/vaultwarden.conf)):

```nginx
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/ssl/certs/lab.crt;
    ssl_certificate_key /etc/ssl/private/lab.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Vaultwarden container** (full file in [`docker-compose.yml`](docker-compose.yml)):

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - ./vw-data:/data
    restart: unless-stopped
```

Note the `127.0.0.1:8080:80` bind — the container is only reachable from the loopback interface, which means **Nginx is the only way in**.

## Rebuild It Yourself

Full step-by-step in [`docs/SETUP.md`](docs/SETUP.md).

## What I Learned

- Why you never expose an application container directly to the internet, and how `127.0.0.1` binding enforces that at the network layer.
- How to read an Nginx error log and trace a 502 back to either a misconfigured upstream or a backend that isn't listening.
- The difference between a self-signed cert (what I used here) and a real CA-signed cert (what Let's Encrypt would provide in production), and why browsers warn on the former.
- How `X-Forwarded-For` and `X-Forwarded-Proto` propagate client info to the backend that would otherwise only see traffic from `127.0.0.1`.
- Why UFW rules alone aren't enough security — the defense starts with the bind address.

## Next Steps

- Replace the self-signed cert with Let's Encrypt using certbot + DNS-01 challenge
- Add fail2ban to block brute-force login attempts
- Put Vaultwarden behind a VPN (WireGuard) so port 443 doesn't need to be public at all
- Rebuild the whole stack with Caddy to compare — Caddy handles TLS automatically

---

Built as part of my transition into IT. Background: CompTIA A+ and Network+ certified, currently studying for Security+.
