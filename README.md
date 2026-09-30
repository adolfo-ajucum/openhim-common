# OpenHIM Common

## Local

```
docker compose up -d
```

Consola en http://localhost:9000 (usa `default.json`). Requiere `.env` con `OPENHIM_SESSION_KEY` (ver `.env.example`).

## Pruebas (QA) detrás de Nginx Proxy Manager

| Subdominio | Uso | Destino en NPM |
|---|---|---|
| `openhim-qa.hro.gob.gt` | Consola web | `http://<IP_OPENHIM>:9000` |
| `openhim-qa-api.hro.gob.gt` | API core (la consola la llama desde el navegador) | `https://<IP_OPENHIM>:8080` |
| `openhim-qa-router.hro.gob.gt` | Router para clientes/sistemas | `http://<IP_OPENHIM>:5001` |

1. `cp .env.example .env`, generar `OPENHIM_SESSION_KEY` y dejar `CONSOLE_CONFIG=./default.qa.json`.
2. `docker compose up -d`
3. Cambiar la contraseña de `root@openhim.org` (default `openhim-password`) **antes** de publicar en NPM.
4. DNS: registros A de los 3 subdominios hacia la IP pública del NPM.
5. En NPM, un Proxy Host por subdominio: SSL Let's Encrypt, Force SSL, HTTP/2, HSTS, Block Common Exploits.
   El de la API usa esquema **https** (el core usa cert autofirmado; NPM no lo valida).
6. Firewall del equipo de OpenHIM: permitir 9000, 8080, 5001 solo desde la IP del NPM.
