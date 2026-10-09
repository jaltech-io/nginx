# nginx — Reverse proxy platform

Reverse proxy nginx de la plateforme (repo standalone, anciennement `platform-apps/deploy/nginx`).

Gère tous les vhosts de la VM EC2 :

| Fichier | Domaine |
|---|---|
| `conf.d/keycloak.conf` | `https://auth.profeskills.com` |
| `conf.d/00-default.conf` | nom inconnu : connexion fermée (port 80 : challenge ACME seulement) |
| `conf.d/fo-projectflow.conf` | `https://fo.projectflow.profeskills.com` (application : front + BFF) |
| `conf.d/projectflow.conf` | `https://projectflow.profeskills.com` (vitrine) |
| `conf.d/profeskills.conf` | `https://profeskills.com` (landing) |

Le réseau Docker `platform-net` est créé ici — rejoint par chaque app (keycloak, admin-api, admin-front, landing, redis).

## Structure

```
nginx.conf          # Config principale (log json, gzip, rate limiting, resolver Docker)
conf.d/             # Un vhost par app + proxy_params.inc partagé
docker/             # docker-compose.yml du conteneur nginx (image nginx:1.27-alpine)
ansible/            # Déploiement sur la VM : site.yml (deploy + SSL) / reload.yml
```

## Déploiement

Via la pipeline GitLab (`.gitlab-ci.yml`) :

- `lint:nginx` — automatique : vérifie la présence des fichiers et `nginx -t`
- `deploy:nginx` — **manuel** : sync configs vers S3 puis `ansible-playbook site.yml` sur la VM via SSM (nginx + certificat SSL Let's Encrypt)
- `deploy:nginx:reload` — **manuel** : re-sync + `ansible-playbook reload.yml` (reload de la conf sans redéploiement complet)

Les jobs de déploiement sont volontairement manuels : ils touchent la VM en service.

## Variables CI/CD requises (GitLab > Settings > CI/CD > Variables)

| Variable | Rôle |
|---|---|
| `AWS_ROLE_ARN` | ARN du rôle IAM assumé via OIDC (`assume-role-with-web-identity`) |

`AWS_REGION`, `DOMAIN_NAME`, `ADMIN_DOMAIN`, `LANDING_DOMAIN` ont des valeurs par défaut dans `.gitlab-ci.yml`, surchargeables au besoin.

## Prérequis

La VM est provisionnée par le repo `vm-aws` (VPC, EC2, Docker). Ce repo suppose une instance EC2 taguée `Project=platform` en service, joignable via SSM.
