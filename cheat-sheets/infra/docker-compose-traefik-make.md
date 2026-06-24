# Docker Compose + Traefik + Makefile

Orchestration locale multi-services via Docker Compose, routage par nom de domaine via Traefik, raccourcis Makefile.

## Routing Traefik

Tous les services accessibles via `*.localhost` (port 80), pas de port-forwarding manuel.

| Type | Pattern |
|---|---|
| API backend | `<service>.api.localhost` |
| Frontend | `<service>.localhost` |
| Infra (Kafka UI, mail, dashboard Traefik) | `<outil>.localhost` |

## Live reload

Tous les services montent leur répertoire source en volume → les changements de code sont reflétés sans rebuild d'image.

## Profils de services

Démarrage par profil métier (infra seule, ou un sous-ensemble de services applicatifs) plutôt que tout démarrer.

## Commandes make essentielles

```bash
make up                    # tous les services
make up-i                  # infra seule (Mongo + Kafka)
make up-<profile>          # un sous-ensemble de services (ex: up-qn, up-ir, up-mv)
make restart-<service>     # redémarrer un service précis
make logs-<service>        # suivre les logs d'un service
make sh-<service>          # shell dans le conteneur
make mongo-sh              # mongosh
make ps / make status      # état des conteneurs
make urls                  # liste toutes les URLs de service
make clean                 # stop + suppression conteneurs
make clean-volumes         # + suppression des données persistées (destructif)
make clean-all             # full cleanup (+ images)
```

`lazydocker` recommandé pour la gestion interactive des conteneurs.

## Environnement

- `.env` — UID/GID pour permissions des volumes
- `.envrc` — tokens privés (registre npm interne, etc.)
- Les variables de `compose.yml` **surchargent** celles des `.env` par service

## Pièges courants

- Changer `compose.yml` (définition de service) → nécessite `docker compose up -d --force-recreate <service>`, pas juste un restart
- Changer un `.env` de service → `make restart-<service>` suffit
- Mongo (replica set) et les topics Kafka sont auto-initialisés au premier démarrage — pas de setup manuel
