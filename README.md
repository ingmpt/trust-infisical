# trust-infisical

Instancia self-hosted de [Infisical](https://infisical.com) en Docker, para gestión centralizada de secretos, descubierta automáticamente por Traefik (local y VPS) y consumida vía IP pública.

## Arquitectura

- **backend**: imagen oficial `infisical/infisical:latest-postgres`, conectada a la red externa `traefik-public` (para ser descubierta por Traefik) y a una red interna `infisical-internal` (comunicación con `db`/`redis`, no expuesta).
- **db**: PostgreSQL 14, solo accesible dentro de `infisical-internal`.
- **redis**: Redis 7, solo accesible dentro de `infisical-internal`.
- **Traefik**: no se despliega aquí (ya existe en el servidor Local y VPS). Este proyecto solo se conecta a la red `traefik-public` ya existente y expone labels para que Traefik descubra el servicio `backend` y enrute `https://${DOMAIN}`.

## Requisitos previos

- Docker Engine 20.10+ y Docker Compose 2.0+.
- Red externa `traefik-public` ya creada donde Traefik esté escuchando (`docker network create traefik-public` si aún no existe).
- Traefik configurado con el entrypoint `websecure` y un `certresolver` llamado `letsencrypt` (ajustar labels en [docker-compose.yml](docker-compose.yml) si tu Traefik usa otros nombres).

## Configuración

1. Copiar la plantilla de variables:
   ```powershell
   Copy-Item .env.example .env
   ```
2. Completar `.env` con:
   - `DOMAIN`: no hay dominio propio, se usa el patrón `sslip.io` (resuelve al IP embebido en el subdominio):
     - Local: `infisical.127.0.0.1.sslip.io`
     - VPS (prod, IP pública `62.238.26.202`): `infisical.62-238-26-202.sslip.io`
   - `ENCRYPTION_KEY` (`openssl rand -hex 32`) y `AUTH_SECRET` (`openssl rand -base64 32`): claves propias de bootstrap de Infisical (no se gestionan vía Infisical porque este *es* el propio Infisical).
   - `POSTGRES_PASSWORD`: contraseña fuerte para la base de datos.

   > Ya se generó un `.env` local con claves aleatorias seguras para pruebas; está excluido del control de versiones (`.gitignore`). Rotar estas claves antes de ir a producción.

## Levantar el servicio

```powershell
docker network create traefik-public  # solo si no existe aún
docker compose up -d
```

## Consumo

- **Backend/Postman (consumo de la API de secretos)**: `https://<DOMAIN>/api/*`, enrutado públicamente por Traefik (router `trust-infisical-api`, sin restricción de IP) usando la IP pública del VPS (`62.238.26.202`) vía sslip.io. Traefik del VPS expone `websecure` (443) con certificado Let's Encrypt (HTTP challenge, resolver `letsencrypt`, email `ingmpt@gmail.com`).
- **Administración (UI: login, dashboard, `/admin/*`)**: mismo dominio público, pero **restringida por IP allowlist** (router `trust-infisical-admin`, middleware `ipallowlist`). Solo las IPs listadas en `ADMIN_ALLOWED_IPS` (formato CIDR, separadas por coma) pueden acceder; cualquier otra IP recibe rechazo a nivel de Traefik antes de llegar al backend.
- Ninguna aplicación externa debe apuntar directamente a los puertos de `db`/`redis`; estos permanecen únicamente en la red interna `infisical-internal`.

## Pendiente / a confirmar con el humano

- Si tu IP pública cambia (conexión dinámica), actualizar `ADMIN_ALLOWED_IPS` en el `.env` del VPS y ejecutar `docker compose up -d --force-recreate backend` para aplicarlo.
- El puerto de respaldo `127.0.0.1:${ADMIN_HOST_PORT}` sigue disponible como acceso alterno vía túnel SSH directo al contenedor, pero ya no es necesario para el flujo normal de administración (el allowlist de IP en el dominio público lo reemplaza).

## Despliegue en el VPS (flujo git)

1. En local: commitear y `git push origin main`.
2. En el VPS (dentro del clon del repo):
   ```bash
   git pull origin main
   ```
3. El `.env` **no viaja por git** (está en `.gitignore`). La primera vez, crearlo manualmente en el VPS a partir de [.env.example](.env.example) con los valores reales de producción (mismo `DOMAIN=infisical.62-238-26-202.sslip.io`, `ENCRYPTION_KEY`, `AUTH_SECRET`, `POSTGRES_PASSWORD`). En actualizaciones posteriores, el `.env` ya existente en el VPS se mantiene y no requiere tocarse salvo cambios de configuración.
4. Confirmar que la red externa `traefik-public` ya existe en el VPS (`docker network ls`).
5. Levantar/actualizar el stack:
   ```bash
   docker compose up -d
   ```
6. Validar:
   ```bash
   docker compose ps
   curl -k https://infisical.62-238-26-202.sslip.io/api/status
   ```
