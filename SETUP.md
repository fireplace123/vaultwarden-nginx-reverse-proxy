# Setup Guide

Step-by-step to rebuild this lab from a clean Ubuntu 22.04 VM.

## 1. Prerequisites

A VirtualBox (or any hypervisor) VM running Ubuntu 22.04 LTS with:

- A user with `sudo` access
- Network connectivity
- At least 2 GB RAM, 10 GB disk

## 2. Install Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
docker compose version
```

## 3. Run Vaultwarden

Create a working directory and the compose file:

```bash
mkdir -p ~/vaultwarden && cd ~/vaultwarden
# copy docker-compose.yml from this repo into the current directory
docker compose up -d
```

Verify the container is listening on loopback only:

```bash
sudo ss -tlnp | grep 8080
# Expected:  LISTEN 0 ... 127.0.0.1:8080 ...
```

If you see `0.0.0.0:8080` instead, the bind address in `docker-compose.yml` is wrong — that would expose Vaultwarden directly, bypassing the proxy.

## 4. Install Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

## 5. Generate the Self-Signed Certificate

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/lab.key \
    -out    /etc/ssl/certs/lab.crt
```

Fill in whatever you want for the prompts — for a lab cert, the Common Name just needs to not be empty.

Lock down the private key:

```bash
sudo chmod 600 /etc/ssl/private/lab.key
```

## 6. Configure Nginx

Copy `nginx/vaultwarden.conf` from this repo to the server:

```bash
sudo cp nginx/vaultwarden.conf /etc/nginx/sites-available/vaultwarden
sudo ln -s /etc/nginx/sites-available/vaultwarden /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

Test the config before reloading — `nginx -t` will catch typos and bad directives before they take down the running server:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 7. Open the Firewall

```bash
sudo ufw allow 443/tcp
sudo ufw allow OpenSSH    # don't lock yourself out
sudo ufw enable
sudo ufw status
```

Note that we do **not** open port 8080. That's intentional — Vaultwarden is bound to loopback, so there's nothing to reach there from outside anyway, but keeping 8080 closed in UFW is belt-and-suspenders defense in depth.

## 8. Test It

From your host machine (not the VM), open a browser to:

```
https://<vm-ip-address>
```

The browser will warn about the self-signed cert — that's expected. Accept the risk and continue. You should land on the Vaultwarden signup page.

## Troubleshooting

**502 Bad Gateway** — Nginx can't reach the backend.

```bash
sudo systemctl status nginx
docker ps                          # is the container running?
curl -v http://127.0.0.1:8080      # from inside the VM, does it respond?
sudo tail -f /var/log/nginx/error.log
```

**Connection refused from host browser** — firewall or wrong IP.

```bash
sudo ufw status                    # is 443 allowed?
ip a                               # confirm the VM's IP
```

Also check that your VM network adapter is set to **Bridged** (not NAT) in VirtualBox if you want to reach it from the host by IP.

**Certificate warning won't let you through** — some browsers (especially on strict HSTS) make bypassing hard. Try Firefox, which reliably allows adding an exception, or type `thisisunsafe` on the Chrome warning page.
