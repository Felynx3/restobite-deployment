# RestoBite deployment

Este repositorio define el stack Docker de producción para la VPS de RestoBite. Consume imágenes ya publicadas: no clona, construye ni ejecuta código fuente de las aplicaciones en la VPS.

## Componentes

| Componente actual | Estado en este stack | Función |
| --- | --- | --- |
| Caddy | Incluido | Gestiona certificados ACME, termina TLS y enruta los orígenes de API, sitio público y Grafana. |
| `restobite-backend` | Incluido como `backend` | API NestJS en el puerto privado 4000. |
| `restobite-public` | Incluido como `public` | Frontend Next.js en el puerto privado 3000. |
| Loki | Incluido | Almacena logs durante siete días. |
| Promtail | Incluido | Envía los logs públicos de Caddy a Loki. |
| Grafana | Incluido | Consulta los logs mediante `logs.restobite.com`. |
| Jenkins | Excluido | Se retira del despliegue; no hay servicio ni vhost de CI. |
| PostgreSQL / Cloud SQL | Excluido | Sigue siendo un servicio externo configurado mediante `DATABASE_URL`. |
| Frontend administrativo | Excluido | Sigue desplegado estáticamente en S3 y CloudFront. |

Solo Caddy publica `80` y `443`. Los demás servicios se comunican mediante la red privada `restobite` creada por Compose.

## Configuración inicial de la VPS

1. Copie cada archivo de `env/*.env.example` a su equivalente sin `.example` y asigne valores de producción. No suba esos archivos a Git.
2. Instale la credencial de la cuenta de servicio GCP en `secrets/gcp-service-account.json`; el backend la recibe como `/run/secrets/gcp_service_account`.
3. Antes del primer inicio, confirme que `origin-api.restobite.com`, `origin-www.restobite.com` y `logs.restobite.com` resuelven a la VPS y que los puertos entrantes `80` y `443` son alcanzables. Caddy usará HTTP-01 para obtener y renovar los certificados ACME.
4. Cree los directorios persistentes de Caddy con permisos de la cuenta de despliegue: `mkdir -p caddy/data caddy/config caddy/logs`. No copie certificados a estos directorios ni suba su contenido generado a Git.
5. Use tags `sha-<commit>` publicados por los workflows de aplicación en `env/compose.env`.
6. Inicie sesión antes de descargar imágenes privadas. El token debe pertenecer a `Felynx3` y tener únicamente `read:packages`:

```sh
docker login ghcr.io -u Felynx3
docker compose --env-file env/compose.env config
docker compose --env-file env/compose.env pull
```

Los valores de `API_ORIGIN_SECRET` y `PUBLIC_ORIGIN_SECRET` deben ser cadenas opacas URL-safe sin comillas ni saltos de línea. Caddy las recibe desde `env/caddy.env`; nunca se escriben en una configuración versionada.

## Migración desde los contenedores legacy

Planifique una ventana de mantenimiento: el proxy legacy y el nuevo stack con Caddy compiten por los puertos 80/443.

1. Respalde el volumen legacy `restobite-logs-loki` y el directorio `grafana/data/grafana` del checkout legacy `restobite-logs`.
2. Cree o copie esos respaldos en los volúmenes `restobite_loki_data` y `restobite_grafana_data` antes del corte. Conserve los respaldos hasta validar el stack nuevo.
3. Confirme que los tres DNS y los puertos públicos cumplen los requisitos de ACME, y valide la configuración con:

```sh
docker compose --env-file env/compose.env config
docker compose --env-file env/compose.env run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

4. Detenga los contenedores legacy `restobite-nginx`, `restobite-backend`, `restobite-public`, `restobite-logs-loki`, `restobite-logs-promtail`, `restobite-logs-grafana` y `jenkins`.
5. Ejecute `docker compose --env-file env/compose.env up -d`.
6. Confirme en `docker compose --env-file env/compose.env logs caddy` que Caddy obtuvo certificados para los tres hosts. Recree únicamente Caddy y confirme que conserva el estado en `caddy/data` y `caddy/config`:

```sh
docker compose --env-file env/compose.env up -d --force-recreate caddy
docker compose --env-file env/compose.env logs caddy
```

7. Compruebe las rutas API y públicas mediante CloudFront, el login de Grafana y las dos fuentes de logs: aplicación → Loki y Caddy → Promtail → Loki. Las solicitudes API y públicas sin `X-Origin-Secret` deben responder `403` tanto en HTTP como en HTTPS; las autorizadas redirigen de HTTP a HTTPS y se proxifican por HTTPS. `logs.restobite.com` no requiere el secreto. Verifique también el encabezado `Strict-Transport-Security` en una respuesta HTTPS de cada host.

Para revertir antes de aceptar el corte, detenga el stack Compose, restaure los datos si fuese necesario y reinicie los contenedores legacy en `restobite-network`. Conserve `caddy/data`, `caddy/config` y `caddy/logs` para diagnóstico; no elimine estado ACME válido hasta terminar la validación de API, sitio público, Grafana, certificados y logs.

La eliminación del registro Route 53 para `ci.restobite.com` y la gestión de Cloud SQL no se automatizan aquí y deben coordinarse fuera de este repositorio.

## Operación diaria

Para actualizar una aplicación, cambie solo su tag SHA en `env/compose.env`, descargue la imagen y recree ese servicio:

```sh
docker compose --env-file env/compose.env pull backend public
docker compose --env-file env/compose.env up -d backend public
```

Caddy renueva automáticamente los certificados mientras conserva su estado en `caddy/data` y `caddy/config`. Revise periódicamente `docker compose --env-file env/compose.env logs caddy` y los registros persistentes `caddy/logs/restobite_public_access.json` y `caddy/logs/restobite_public_error.json`, que Promtail publica en Loki como `source=restobite-caddy` con niveles `info` y `error`, respectivamente. Respalde los directorios persistentes de Caddy con acceso restringido y nunca versione sus contenidos generados.

Revise `WORKFLOW_PROMPTS.md` para crear los workflows de publicación de imágenes en los repositorios de backend y frontend público.
