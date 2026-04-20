# Vaultwarden behind Nginx (lab)

A home lab exercise: Vaultwarden running in Docker, with Nginx in front as a reverse proxy handling TLS. Running on an Ubuntu VM in VirtualBox.

This is one of the labs from a self-directed course I've been working through as part of studying for Security+ and building hands-on skills for an IT career change.

## What it does

Browser hits Nginx on `:443` over HTTPS → Nginx terminates TLS → forwards the request as plain HTTP to the Vaultwarden container bound to `127.0.0.1:8080`.

The point of the exercise is to practice the reverse proxy + TLS termination pattern that sits behind most real web apps.

## It running

![Vaultwarden login served over HTTPS](screenshots/Screenshot%202026-04-19%20184626.png)
![Ports bound correctly — Nginx on 443, Vaultwarden on loopback only](screenshots/ss-tlnp.png)

## Files

- `nginx/vaultwarden.conf` — the Nginx server block
- `docker-compose.yml` — Vaultwarden container
- `docs/SETUP.md` — the steps I followed to build it

## Things I got stuck on

**Docker port binding.** First time around I had the port mapped as `8080:80`, which exposed the container on all interfaces. Changed it to `127.0.0.1:8080:80` so only Nginx on the same host can reach it. Confirmed with `ss -tlnp`.

**Nginx proxy_pass.** Took a few tries to get the right set of `proxy_set_header` directives. Without them the backend only sees traffic from `127.0.0.1`, which breaks anything that cares about the real client IP. Also needed the `Upgrade` / `Connection` headers for Vaultwarden's WebSocket sync.

## What I took away from it

- Why the backend binds to loopback instead of `0.0.0.0` — the firewall is a second line of defense, not the first.
- How to read `/var/log/nginx/error.log` to figure out whether a 502 is Nginx's fault or the backend being down.
- Self-signed certs are fine for a lab but browsers (correctly) scream about them. Let's Encrypt would be the next step for anything real.

## Stack

Ubuntu 22.04 · Nginx · Docker · Vaultwarden · OpenSSL · UFW
