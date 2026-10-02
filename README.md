# Deployment (Traefik)

Zentraler Reverse Proxy für den vServer. Läuft selbst als Docker-Container
(Traefik v3) und übernimmt TLS (Let's Encrypt) sowie das Routing für alle
anderen Container auf dem Server — per Docker-Label-Discovery, kein Backend
muss dafür selbst einen Port am Host öffentlich binden.

Ausgerollt wird per Ansible-Playbook, damit der komplette Server-Zustand
reproduzierbar und nachvollziehbar ist, statt manuell per SSH gepflegt zu
werden.

## Domains → Container

| Domain                        | Entrypoint       | Ziel-Container                                    |
|--------------------------------|------------------|----------------------------------------------------|
| `krasosu.de`, `www.krasosu.de` | websecure (443)  | Landingpage (`/home/omega/workspace/landingpage`) |
| `krasosu.de:3001`              | lcars (3001)     | LCARS / claude-web                                |
| `cloud.krasosu.de`             | websecure (443)  | ownCloud                                          |
| `docker-image-downloader.de`   | websecure (443)  | docker-image-downloader                           |
| `baby-physio.de`, `www.baby-physio.de` | websecure (443) | WordPress                                 |

LCARS bekommt bewusst einen eigenen Traefik-Entrypoint auf Port 3001, statt
auf eine Subdomain zu wechseln — damit der bereits überall verlinkte Link
`http://krasosu.de:3001/` unverändert weiter funktioniert.

## Voraussetzungen

1. DNS: A-Records für alle oben gelisteten Domains zeigen auf die IP des
   vServers (bei Strato im DNS-Management einzutragen).
2. SSH-Zugriff auf den vServer mit einem Benutzer, der `sudo`/root-Rechte
   hat (für die Docker-Installation und das Deployment).
3. [Ansible](https://docs.ansible.com/) lokal installiert
   (`pip install ansible-core` reicht, keine Collections nötig).

## Einrichtung

Dieses Repo enthält keine Server-Adresse, E-Mail-Adressen oder sonstigen
Umgebungs-spezifischen Werte im Klartext — die liegen ausschließlich in
zwei lokalen, gitignorten Dateien, die aus den mitgelieferten `.example`-
Vorlagen entstehen:

```bash
cp inventory.example.ini inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
```

Dann in beiden Dateien die echten Werte eintragen:

- `inventory.ini` — IP/Hostname und SSH-User des vServers.
- `group_vars/all.yml` — Let's-Encrypt-E-Mail-Adresse, Traefik-Version,
  Name des externen Docker-Netzwerks, Zielpfad auf dem Server.

Beide Dateien sind über `.gitignore` ausgeschlossen und werden nie
committet.

## Ausrollen

```bash
ansible-playbook playbook.yml
```

Das Playbook ist idempotent und kann gefahrlos mehrfach laufen:

1. Installiert Docker Engine + Compose-Plugin (überspringbar über
   `install_docker: false` in `group_vars/all.yml`, falls Docker auf dem
   Server bereits vorhanden ist).
2. Legt das externe Docker-Netzwerk `proxy` an, falls es noch nicht
   existiert.
3. Rendert `templates/traefik.yml.j2` und
   `templates/docker-compose.yml.j2` mit den Werten aus `group_vars/all.yml`
   nach `{{ deploy_path }}` auf den Server.
4. Legt `acme.json` mit den nötigen `600`-Rechten an, falls noch nicht
   vorhanden (eine bereits vorhandene Datei mit echten Zertifikaten wird
   dabei **nicht** überschrieben).
5. Startet den Stack per `docker compose up -d`.

### Migration von einem bereits laufenden Traefik

Läuft auf dem Server schon ein Traefik-Container (z. B. der aktuell
produktive, der `baby-physio.de` bedient), unbedingt vorher sicherstellen,
dass `deploy_path` in `group_vars/all.yml` auf dessen vorhandenes
Verzeichnis zeigt bzw. dessen `acme.json` vorher dorthin kopiert wird —
sonst entsteht ein zweiter Container, der um die Ports 80/443 konkurriert.
Docker verweigert in dem Fall zwar zuverlässig den Start (Containername
`traefik` ist bereits belegt), eine kurze Downtime beim Umschalten lässt
sich aber nur vermeiden, wenn der alte Container erst gestoppt wird,
nachdem WordPress' eigene Labels (siehe unten) bereits auf das neue Setup
vorbereitet sind.

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

### WordPress (baby-physio.de)

Läuft bereits auf dem Server und soll unverändert unter `baby-physio.de`
erreichbar bleiben. Damit das über dieses Traefik-Setup weiterläuft, muss
dessen `docker-compose.yml` dieselben Labels bekommen (Port beim
offiziellen `wordpress`-Image i.d.R. 80):

```yaml
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress.rule=Host(`baby-physio.de`) || Host(`www.baby-physio.de`)"
      - "traefik.http.routers.wordpress.entrypoints=websecure"
      - "traefik.http.routers.wordpress.tls.certresolver=letsencrypt"
      - "traefik.http.services.wordpress.loadbalancer.server.port=<INTERNER_PORT>"

networks:
  proxy:
    external: true
```

Vor dem Umschalten prüfen, ob `baby-physio.de` aktuell schon über Traefik
läuft oder noch direkt (z. B. eigener Port-Bind auf 80/443) — nur im
zweiten Fall muss der alte Bind entfernt werden, sonst kollidieren beide
Setups um dieselben Ports.

`<INTERNER_PORT>` durch den Port ersetzen, auf dem die Anwendung **im**
Container tatsächlich lauscht (nicht der alte, veröffentlichte Host-Port).

## Dashboard

Aktuell deaktiviert (`api.dashboard: false` in `templates/traefik.yml.j2`)
— bewusst, damit nicht versehentlich ein ungeschütztes Admin-Interface im
Netz hängt. Bei Bedarf später mit eigener Subdomain + Basic-Auth-Middleware
aktivierbar.

## Öffentliches Repo

Dieses Repo ist dafür gedacht, öffentlich auf GitHub zu liegen. Es enthält
keine Passwörter, Tokens oder echten Zertifikate — die einzigen
umgebungsspezifischen Werte (Server-IP, SSH-User, E-Mail-Adresse) liegen in
`inventory.ini` und `group_vars/all.yml`, beide per `.gitignore`
ausgeschlossen. `acme.json` (enthält den privaten Let's-Encrypt-
Account-Key) wird erst auf dem Server selbst erzeugt und existiert in
diesem Repo nicht.
