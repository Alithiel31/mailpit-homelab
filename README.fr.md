# Mailpit — puits SMTP mutualisé (homelab)

[English version](README.md)

Une seule instance de [Mailpit](https://mailpit.axllent.org/) joue le rôle de **point d'entrée SMTP unique du homelab**. Rien n'est jamais livré sur Internet : chaque mail émis par un service (reset de mot de passe, invitation, notification) est capté et se lit dans l'interface web de Mailpit, accessible uniquement via Tailscale.

## Vue d'ensemble

| Service | Image | Port hôte | Rôle |
|---|---|---|---|
| `mailpit` | `axllent/mailpit`, épinglée par digest | `1025` (SMTP) et `8025` (interface web), liés à localhost et à `TAILSCALE_IP` uniquement | Puits SMTP + visionneuse de mails |

- **Persistance** : volume nommé `mailpit-data` (fichier SQLite), plafonné par `MP_MAX_MESSAGES` (500 par défaut, les plus anciens sont purgés).
- **Pas d'authentification SMTP** : tout nœud du tailnet peut poster un mail. Acceptable sur un tailnet privé ; l'interface peut être protégée avec `MP_UI_AUTH_FILE` (ligne commentée dans `docker-compose.yml`).
- **Image** épinglée par digest, date d'épinglage en commentaire dans le compose.

## Le contrat : une seule adresse pour tous

Tous les clients utilisent le même point d'entrée, qu'ils tournent dans un conteneur sur l'hôte ou sur une autre machine du tailnet :

| Paramètre | Valeur |
|---|---|
| Hôte SMTP | `<TAILSCALE_IP de l'hôte>` |
| Port SMTP | `1025` |
| TLS / authentification | aucun |
| Expéditeur | `<service>@caesura.local` (permet de filtrer par expéditeur dans l'interface) |

Règle : **tout service proposant un « mot de passe oublié » envoie ses mails via ce Mailpit.**

| Client | Où c'est configuré | Variables |
|---|---|---|
| Infisical | `Infisical/docker-compose.yml` | `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM_ADDRESS` |
| Gitea | `woodpecker-ci/docker-compose.yml` | `GITEA__mailer__*` (hôte lu dans `SMTP_HOST` du `.env`) |
| n8n (mode local) | `n8n/docker-compose.local.yml` | `N8N_EMAIL_MODE`, `N8N_SMTP_*` (hôte lu dans `SMTP_HOST` du `.env`) |

Pour le nœud *Send Email* de n8n, créer une fois le credential SMTP dans l'interface avec le même hôte/port, sans authentification et SSL/TLS désactivé.

## Déploiement

```bash
cp .env.example .env     # renseigner TAILSCALE_IP (tailscale ip -4)
docker compose up -d
docker compose ps        # STATUS doit passer à « healthy »
```

### Pare-feu

Les conteneurs joignent l'hôte via sa propre adresse Tailscale, ce qui passe par le pare-feu de l'hôte. Autoriser les sous-réseaux Docker qui doivent envoyer des mails (à relever avec `docker network inspect <réseau>`) :

```bash
sudo ufw allow from <SOUS_RESEAU_DOCKER> to <TAILSCALE_IP> port 1025 proto tcp
```

### Migration depuis le Mailpit embarqué dans Infisical

L'ancien conteneur s'appelle aussi `mailpit` et occupe le port `8025` : il faut le supprimer avant de démarrer celui-ci. Les mails de l'ancienne instance sont perdus (elle n'avait pas de volume) — c'est attendu.

1. Déployer le `Infisical/docker-compose.yml` mis à jour **sans le démarrer**, puis supprimer l'ancien conteneur : `docker rm -f mailpit`.
2. Démarrer ce projet : `docker compose up -d`.
3. Démarrer Infisical : `docker compose up -d` (il ne dépend plus de Mailpit).

## Vérification

```bash
# Envoyer un mail de test au Mailpit mutualisé, depuis n'importe quelle machine du tailnet
curl -s --url "smtp://<TAILSCALE_IP>:1025" \
  --mail-from "test@caesura.local" --mail-rcpt "toi@caesura.local" \
  -T - <<'MAIL'
From: test@caesura.local
To: toi@caesura.local
Subject: Test Mailpit

Ça marche.
MAIL
```

Puis ouvrir `http://<TAILSCALE_IP>:8025` : le mail doit apparaître.

## Notes de sécurité

- L'interface affiche les liens de reset de **tous** les services : la laisser accessible via Tailscale uniquement (ne jamais la publier, ne jamais la router via Cloudflare).
- Ne pas envoyer de vrais identifiants dans les mails de test.

## Licence

MIT — voir [LICENSE](LICENSE).
