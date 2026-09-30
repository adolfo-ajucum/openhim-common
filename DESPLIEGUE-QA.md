# Registro de despliegue OpenHIM QA — HRO

Última actualización: 2026-09-30

## Estado actual: ✅ funcionando

Verificado desde internet el 2026-09-30:
- `https://openhim-qa-api.hro.gob.gt/heartbeat` → 200
- `https://openhim-qa.hro.gob.gt` → 200, login OK
- CORS: la API acepta el origen `https://openhim-qa.hro.gob.gt`
- Contraseña por defecto de `root@openhim.org` ya cambiada (el login con `openhim-password` devuelve 401)

## Arquitectura

```
Navegador / sistemas
   │  HTTPS
Cloudflare (proxy, nube naranja)  ── DNS de hro.gob.gt
   │  HTTPS
Nginx Proxy Manager  192.168.1.78  (otro equipo, misma LAN; certs Let's Encrypt)
   │
OpenHIM  192.168.1.18  (esta PC, Windows 11 + Docker Desktop, interfaz Ethernet)
   ├─ console  :9000  (nginx, sirve la SPA)
   ├─ core API :8080  (HTTPS, cert autofirmado)
   ├─ router   :5001 HTTP / :5000 HTTPS
   └─ mongo    (sin puerto publicado, solo en la red interna de docker)
```

| Subdominio | Proxy Host NPM (Details) | Uso |
|---|---|---|
| `openhim-qa.hro.gob.gt` | `http://192.168.1.18:9000` | Consola web |
| `openhim-qa-api.hro.gob.gt` | **`https`**`://192.168.1.18:8080` | API core (la llama el navegador) |
| `openhim-qa-router.hro.gob.gt` | `http://192.168.1.18:5001` | Canales para sistemas cliente |

Los tres usan SSL Let's Encrypt, Force SSL, HTTP/2 y HSTS. Access: Public (sin Access List).

## Configuración en el repo

- `docker-compose.yml`: versiones fijadas (core `v8.5.0`, consola `v1.18.2`, `mongo:4.4`), `restart: unless-stopped`, límite de logs, Mongo sin puerto publicado, `api_sessionKey` desde `.env`, `api_trustProxy=true`.
- `default.qa.json`: config de la consola apuntando a `https://openhim-qa-api.hro.gob.gt:443`.
- `default.json`: config local (localhost), sin cambios.
- `.env` (**no versionado**, solo en esta PC): `OPENHIM_SESSION_KEY` (aleatoria), `CONSOLE_CONFIG=./default.qa.json`, hosts QA. La plantilla es `.env.example`.

Comandos:
```
docker compose up -d          # levantar / aplicar cambios
docker compose ps
docker compose logs -f core
```

## Problemas encontrados y cómo se resolvieron

1. **"An error occurred while attempting to connect to the OpenHIM Core on https://openhim-qa-api.hro.gob.gt:443"**
   - El registro DNS de `openhim-qa-api` aún no propagaba; se resolvió solo.
   - El Proxy Host de la API en NPM tenía Scheme `http`, lo que daba **502**. El core solo atiende HTTPS en el 8080, así que se cambió a `https`.
2. El error `getaddrinfo EAI_AGAIN mxr.mozilla.org` en los logs del core es inofensivo (el paquete ssl-root-cas intenta descargar CAs).

## Pendientes

- [ ] **Firewall de Windows**: las reglas "Docker Desktop Backend" permiten cualquier puerto desde cualquier IP, incluido el perfil Public (el Wi‑Fi `192.168.7.122` está en Public). Falta:
  - poner `BIND_IP=192.168.1.18` en `.env` y correr `docker compose up -d`;
  - crear una regla de **bloqueo** para 9000, 8080 y 5001 desde toda la red excepto `192.168.1.78` (PowerShell como Administrador; las reglas de bloqueo ganan sobre las de permitir).
- [ ] **Restringir el acceso a la consola y la API**: Access List por IP. Como NPM ve las IPs de Cloudflare, conviene hacerlo en Cloudflare (regla de WAF o Cloudflare Access).
- [ ] **Canales de OpenHIM**: configurar autenticación por cliente (básica o por IP). No dejar canales públicos sin autenticación.
- [ ] **Commit** de los cambios del repo (el `.env` queda fuera por `.gitignore`).
- [ ] Opcional: configurar el correo (`email_*`) para alertas y reportes; ahora trae valores de ejemplo.
- [ ] Opcional: respaldos del volumen `openhim-common_mongo-data` (por ejemplo, `mongodump` programado).
