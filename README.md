# Mailpit — shared SMTP sink (homelab)

[Version française](README.fr.md)

A single [Mailpit](https://mailpit.axllent.org/) instance acting as the **one SMTP endpoint of the whole homelab**. Nothing is ever delivered to the internet: every mail sent by a service (password reset, invitation, notification) is captured and read in Mailpit's web UI, reachable through Tailscale only.

## Overview

| Service | Image | Host port | Role |
|---|---|---|---|
| `mailpit` | `axllent/mailpit`, pinned by digest | `1025` (SMTP) and `8025` (web UI), bound to localhost and to `TAILSCALE_IP` only | SMTP sink + mail viewer |

- **Persistence**: named volume `mailpit-data` (SQLite file), capped by `MP_MAX_MESSAGES` (default 500, oldest purged first).
- **No authentication on SMTP**: any node of the tailnet can post a mail. Acceptable for a private tailnet; the UI can be protected with `MP_UI_AUTH_FILE` (see the commented line in `docker-compose.yml`).
- **Image** pinned by digest, pinning date in a comment in the compose file.

## The contract: one address for everyone

Every client uses the same endpoint, whether it runs in a container on the host or on another machine of the tailnet:

| Setting | Value |
|---|---|
| SMTP host | `<TAILSCALE_IP of the host>` |
| SMTP port | `1025` |
| TLS / authentication | none |
| Sender | `<service>@caesura.local` (lets you filter by sender in the UI) |

Rule: **any service offering a "forgot password" flow must send through this Mailpit.**

| Client | Where it is configured | Variables |
|---|---|---|
| Infisical | `Infisical/docker-compose.yml` | `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM_ADDRESS` |
| Gitea | `woodpecker-ci/docker-compose.yml` | `GITEA__mailer__*` (host from `SMTP_HOST` in `.env`) |
| n8n (local mode) | `n8n/docker-compose.local.yml` | `N8N_EMAIL_MODE`, `N8N_SMTP_*` (host from `SMTP_HOST` in `.env`) |

For n8n's *Send Email* node, create the SMTP credential once in the UI with the same host/port, no authentication and SSL/TLS off.

## Deployment

```bash
cp .env.example .env     # set TAILSCALE_IP (tailscale ip -4)
docker compose up -d
docker compose ps        # STATUS should become "healthy"
```

### Firewall

Containers reach the host through its own Tailscale address, which goes through the host firewall. Allow the Docker subnets that need to send mail (check each one with `docker network inspect <network>`):

```bash
sudo ufw allow from <DOCKER_SUBNET> to <TAILSCALE_IP> port 1025 proto tcp
```

### Migrating from the Mailpit embedded in Infisical

The old container is named `mailpit` and owns ports `8025`: remove it before starting this one. Mails stored in the old instance are lost (it had no volume) — expected.

1. Deploy the updated `Infisical/docker-compose.yml` **without** starting it yet, then remove the old container: `docker rm -f mailpit`.
2. Start this project: `docker compose up -d`.
3. Start Infisical: `docker compose up -d` (it no longer depends on Mailpit).

## Verification

```bash
# Send a test mail to the shared Mailpit, from any machine of the tailnet
curl -s --url "smtp://<TAILSCALE_IP>:1025" \
  --mail-from "test@caesura.local" --mail-rcpt "you@caesura.local" \
  -T - <<'MAIL'
From: test@caesura.local
To: you@caesura.local
Subject: Mailpit test

It works.
MAIL
```

Then open `http://<TAILSCALE_IP>:8025`: the mail must appear.

## Security notes

- The UI shows reset links for **every** service: keep it reachable through Tailscale only (never publish it, never route it through Cloudflare).
- Do not forward real credentials in test mails.

## License

MIT — see [LICENSE](LICENSE).
