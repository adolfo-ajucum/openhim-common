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
2. **El servidor FHIR (`..\hapi-fhirs-hro`, contenedor `hapi-fhir-facade`, puerto 8082) devolvía URLs internas** (`http://192.168.1.18:8082/fhir/...`) en `fullUrl` y `link` al pasar por OpenHIM.
   OpenHIM no requiere configuración: HAPI arma las URLs con el `Host` que recibe. Se agregó la variable `FHIR_SERVER_ADDRESS` (en el `.env` de ese repo) con `HardcodedServerAddressStrategy` en `ExampleRestfulServlet`. Valor actual: `https://openhim-qa-router.hro.gob.gt/fhir`. Si cambia el path del canal en OpenHIM, hay que ajustarla.
3. El error `getaddrinfo EAI_AGAIN mxr.mozilla.org` en los logs del core es inofensivo (el paquete ssl-root-cas intenta descargar CAs).

## Documentación de la API FHIR (en `..\hapi-fhirs-hro`)

- **Swagger UI**: `https://openhim-qa-router.hro.gob.gt/fhir/swagger-ui/`. La especificación está en `/fhir/api-docs` (YAML). Se genera con `OpenApiInterceptor` y usa `FHIR_SERVER_ADDRESS` como URL del servidor.
- **Guía de implementación FHIR** (SUSHI + IG Publisher) en `hapi-fhirs-hro/ig/`:
  - perfil `HROPatient`, extensiones `nombre-madre`/`nombre-padre`, NamingSystems de CUI e historia clínica, CapabilityStatement y ejemplos;
  - páginas: acceso, consultas y errores.
  - Build: `cd ig`, luego `_updatePublisher.bat` (solo la primera vez) y `_genonce.bat -tx n/a`. Abrir `ig/output/index.html`. La primera ejecución tarda unos 10 minutos.

## Pendientes

- [ ] **Firewall de Windows**: las reglas "Docker Desktop Backend" permiten cualquier puerto desde cualquier IP, incluido el perfil Public (el Wi‑Fi `192.168.7.122` está en Public). Falta:
  - poner `BIND_IP=192.168.1.18` en `.env` y correr `docker compose up -d`;
  - crear una regla de **bloqueo** para 9000, 8080 y 5001 desde toda la red excepto `192.168.1.78` (PowerShell como Administrador; las reglas de bloqueo ganan sobre las de permitir).
- [ ] **Restringir el acceso a la consola y la API**: Access List por IP. Como NPM ve las IPs de Cloudflare, conviene hacerlo en Cloudflare (regla de WAF o Cloudflare Access).
- [ ] **Canales de OpenHIM**: configurar autenticación por cliente (básica o por IP). No dejar canales públicos sin autenticación.
- [x] Commit de los cambios de este repo (rama `deploy-qa`, commit `6ed962f`; falta merge a master).
- [ ] Commit en `hapi-fhirs-hro` del cambio de `FHIR_SERVER_ADDRESS` (`ExampleRestfulServlet.java`, `.env.example`).
- [ ] **El FHIR en el puerto 8082 es accesible directo desde la LAN, sin pasar por OpenHIM.** Debería aceptar solo conexiones de OpenHIM: ponerlo en la red docker de OpenHIM sin publicar el puerto, o limitarlo con el firewall.
- [ ] Publicar `ig/output/` como sitio estático (por ejemplo, nginx + Proxy Host `fhir-ig-qa.hro.gob.gt`).
- [ ] Búsqueda de Patient sin parámetros responde 500 en vez de 400: el `catch (Exception)` de `searchPatients` atrapa el `InvalidRequestException`. La guía documenta el comportamiento actual.
- [ ] Probar `/fhir/swagger-ui/` y `/fhir/api-docs` por la URL pública con credenciales de cliente. Sin credenciales, OpenHIM responde 401 (2026-10-01), así que el canal ya exige autenticación.
- [ ] Opcional: configurar el correo (`email_*`) para alertas y reportes; ahora trae valores de ejemplo.
- [ ] Opcional: respaldos del volumen `openhim-common_mongo-data` (por ejemplo, `mongodump` programado).
