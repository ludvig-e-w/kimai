# Self-hosting Kimai (E&W)

Deploy config for running Kimai with Docker Compose: the published Apache/PHP image + MySQL 8.4. This is deployment only — it does not build from the source in this repo.

## Stack

| Service | Image | Notes |
|---|---|---|
| `kimai` | `kimai/kimai2:stable` | Apache variant, listens on container port 8001 |
| `db` | `mysql:8.4` | data in named volume `db_data` |

Volumes: `db_data` (database), `kimai_var` (uploads, cache, invoices), `kimai_plugins`.

## Run

```bash
cp .env.example .env   # then fill in every value
docker compose up -d
```

Open http://localhost:8001

## Environment (.env)

- `APP_SECRET` — random 32-byte hex. Generate: `openssl rand -hex 32`
- `TRUSTED_HOSTS` — **regex, pipe-separated**, not comma-separated. `localhost|127.0.0.1`, in production `time.example.com`
- `TRUSTED_PROXIES` — set to the reverse-proxy subnet when behind Caddy/nginx/Traefik, e.g. `172.16.0.0/12`
- `ADMIN_MAIL` / `ADMIN_PASSWORD` — super admin created on first boot only. Changing them later does nothing; use the console instead.
- `MAILER_URL` — `null://null` disables mail. SMTP example: `smtp://user:pass@smtp.example.com:587`

## Common commands

```bash
docker compose logs -f kimai
docker compose ps
docker exec kimai /opt/kimai/bin/console kimai:user:list
docker exec kimai /opt/kimai/bin/console kimai:user:password admin
docker exec kimai /opt/kimai/bin/console kimai:user:create name mail@example.com ROLE_ADMIN
```

## Update

```bash
docker compose pull
docker compose down
docker compose up -d
```

Database migrations run automatically on container start.

## Backup

```bash
docker exec kimai-db mysqldump -u root -p"$DB_ROOT_PASSWORD" kimai > backup.sql
docker run --rm -v ew-time-tracking_kimai_var:/data -v "$PWD":/out alpine tar czf /out/kimai_var.tgz -C /data .
```

## Production checklist

1. Put a reverse proxy with TLS in front (Caddy is simplest — auto HTTPS).
2. Set `TRUSTED_HOSTS` to the real domain and `TRUSTED_PROXIES` to the proxy subnet.
3. Remove the `ports:` mapping from the `kimai` service and put it on the proxy network instead, so Kimai is not exposed directly.
4. Pin `KIMAI_VERSION` to an explicit version (e.g. `2.64.0`) instead of `stable` for reproducible deploys.
5. Schedule the backup commands above.

### Caddy example

```
time.example.com {
    reverse_proxy kimai:8001
}
```

## Gotchas hit during setup

- There is no `kimai/kimai2:apache-stable` tag. `apache` and `stable` are separate tags, and `stable` is already the Apache image.
- `TRUSTED_HOSTS` is a regex. A comma-separated list produces `BadRequestHttpException: Untrusted Host`. Use `localhost|127.0.0.1`.

## Status

Verified 2026-08-11 on Docker 28.3.2 / Compose v2.39.1:

- both containers report `healthy`
- `GET /en/login` returns 200
- login as `admin` returns 302 to `/` (authentication works)
- super admin present with `ROLE_SUPER_ADMIN`
