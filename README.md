# Deployment (Traefik)

Zentraler Reverse Proxy für den vServer. Läuft selbst als Docker-Container
(Traefik v3) und übernimmt TLS (Let's Encrypt) sowie das Routing für alle
anderen Container auf dem Server — per Docker-Label-Discovery, kein Backend
muss dafür selbst einen Port am Host öffentlich binden.

## Domains → Container

| Domain                        | Entrypoint  | Ziel-Container                              |
|--------------------------------|-------------|----------------------------------------------|
| `krasosu.de`, `www.krasosu.de` | websecure (443) | Landingpage (`/home/omega/workspace/landingpage`) |
| `krasosu.de:3001`              | lcars (3001)     | LCARS / claude-web                        |
| `cloud.krasosu.de`             | websecure (443) | ownCloud                                  |
| `docker-image-downloader.de`   | websecure (443) | docker-image-downloader                   |

LCARS bekommt bewusst einen eigenen Traefik-Entrypoint auf Port 3001, statt
auf eine Subdomain zu wechseln — damit der bereits überall verlinkte Link
`http://krasosu.de:3001/` unverändert weiter funktioniert.

## Voraussetzungen

1. DNS: A-Records für `krasosu.de`, `www.krasosu.de`, `cloud.krasosu.de` und
   `docker-image-downloader.de` zeigen alle auf die IP dieses vServers.
2. E-Mail für Let's-Encrypt-Benachrichtigungen in
   [`traefik/traefik.yml`](traefik/traefik.yml) prüfen/anpassen (aktuell
   `admin@krasosu.de`).

## Einmalige Einrichtung auf dem vServer

```bash
# gemeinsames Netzwerk, das Traefik + alle Backend-Container teilen
docker network create proxy

# ACME-Speicher mit korrekten Rechten anlegen (Traefik verweigert sonst den Dienst)
touch traefik/acme/acme.json
chmod 600 traefik/acme/acme.json
```

## Starten

```bash
docker compose up -d
```

Traefik selbst braucht nicht neu gebaut zu werden (offizielles Image), nur
`docker compose pull && docker compose up -d` bei Updates.

## Andere Container anbinden

Jeder Backend-Container muss:
1. dem externen Netzwerk `proxy` beitreten,
2. seinen Port **nicht mehr direkt am Host** veröffentlichen (das übernimmt
   jetzt Traefik — sonst wäre der Dienst doppelt erreichbar, einmal
   ungeschützt am alten Port),
3. Traefik-Labels bekommen, die Domain und internen Port festlegen.

### Landingpage

Bereits erledigt — [`landingpage/docker-compose.yml`](../landingpage/docker-compose.yml)
ist entsprechend angepasst (Netzwerk `proxy`, Labels für `krasosu.de`).

### LCARS / claude-web

Nicht automatisch geändert — das Projekt liegt in einem eigenen Repo
(`/home/omega/workspace/claude-web`) und dessen `docker-compose.yml` enthält
den echten Anthropic-API-Key im Klartext, daher hier nur die Anleitung zum
manuellen Nachtragen. In dessen `docker-compose.yml`:

```yaml
services:
  web:
    # ... bestehende Konfiguration (image, environment) unverändert ...
    # "ports: - 3000:3000" entfernen — Traefik übernimmt die Veröffentlichung
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.lcars.rule=Host(`krasosu.de`)"
      - "traefik.http.routers.lcars.entrypoints=lcars"
      - "traefik.http.services.lcars.loadbalancer.server.port=3000"

networks:
  proxy:
    external: true
```

`loadbalancer.server.port=3000` ist der **interne** Container-Port (siehe
`PORT` in dessen `.env`/`environment:`), unabhängig vom externen Port 3001.

### ownCloud (Cloud)

In dessen `docker-compose.yml` (Port je nach verwendetem Image prüfen, beim
offiziellen `owncloud/server`-Image i.d.R. 8080; beim Nextcloud-Image 80):

```yaml
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.owncloud.rule=Host(`cloud.krasosu.de`)"
      - "traefik.http.routers.owncloud.entrypoints=websecure"
      - "traefik.http.routers.owncloud.tls.certresolver=letsencrypt"
      - "traefik.http.services.owncloud.loadbalancer.server.port=<INTERNER_PORT>"

networks:
  proxy:
    external: true
```

### docker-image-downloader

Eigene Domain, aber technisch derselbe vServer — genauso über Traefik
geroutet wie alles andere. In dessen `docker-compose.yml`:

```yaml
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.docker-image-downloader.rule=Host(`docker-image-downloader.de`)"
      - "traefik.http.routers.docker-image-downloader.entrypoints=websecure"
      - "traefik.http.routers.docker-image-downloader.tls.certresolver=letsencrypt"
      - "traefik.http.services.docker-image-downloader.loadbalancer.server.port=<INTERNER_PORT>"

networks:
  proxy:
    external: true
```

`<INTERNER_PORT>` durch den Port ersetzen, auf dem die Anwendung **im**
Container tatsächlich lauscht (nicht der alte, veröffentlichte Host-Port).

## Dashboard

Aktuell deaktiviert (`api.dashboard: false` in `traefik.yml`) — bewusst,
damit nicht versehentlich ein ungeschütztes Admin-Interface im Netz hängt.
Bei Bedarf später mit eigener Subdomain + Basic-Auth-Middleware aktivierbar.
