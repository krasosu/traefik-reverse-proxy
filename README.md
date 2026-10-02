# traefik-reverse-proxy

Ansible-managed Traefik reverse proxy with automatic Let's Encrypt TLS and
Docker label-based routing.

## Usage

```bash
cp inventory.example.ini inventory.ini
cp group_vars/all.yml.example group_vars/all.yml
# edit both with your server details

ansible-playbook playbook.yml
```

## Routing a container

Join the `proxy` network and add labels:

```yaml
networks:
  - proxy
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`example.com`)"
  - "traefik.http.routers.myapp.entrypoints=https"
  - "traefik.http.routers.myapp.tls.certresolver=http"
  - "traefik.http.services.myapp.loadbalancer.server.port=8080"
```

`inventory.ini` and `group_vars/all.yml` are gitignored — no secrets live in
this repo.
