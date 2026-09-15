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

- **Backend/Postman (consumo de la API de secretos)**: `https://<DOMAIN>`, enrutado públicamente por Traefik usando la IP pública del VPS (`62.238.26.202`) vía sslip.io. Traefik del VPS ahora expone `websecure` (443) con certificado Let's Encrypt (HTTP challenge, resolver `letsencrypt`, email `ingmpt@gmail.com`).
- **Administración (UI)**: **no** se expone por Traefik ni por el dominio público. El puerto interno del backend (`8080`) solo se publica en `127.0.0.1:9080` del host del VPS (puerto `8080` ya está tomado localmente por el dashboard de Traefik; ver `ports` en [docker-compose.yml](docker-compose.yml)). El acceso se hace exclusivamente mediante túnel SSH desde el equipo local:

  ```powershell
  plink -N -L 9090:localhost:9080 -i "D:\...\key\server-private_key.ppk" root@62.238.26.202
  ```

  Luego abrir `http://localhost:9090` en el navegador local para administrar la instancia.
- Ninguna aplicación externa debe apuntar directamente a los puertos de `db`/`redis`; estos permanecen únicamente en la red interna `infisical-internal`.

## Pendiente / a confirmar con el humano

- Confirmar que el firewall del VPS bloquea el puerto `8080` a nivel público (el binding `127.0.0.1:8080` ya evita exposición externa, pero se recomienda reforzarlo con reglas de firewall).
- Pruebas locales: pendientes hasta confirmar que las variables de entorno están listas (Fase 3).

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
