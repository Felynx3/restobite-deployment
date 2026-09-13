# RestoBite deployment

Este repositorio define el stack Docker de producción para la VPS de RestoBite. Consume imágenes ya publicadas: no clona, construye ni ejecuta código fuente de las aplicaciones en la VPS.

## Componentes

| Componente actual | Estado en este stack | Función |
| --- | --- | --- |
| Nginx | Incluido | Termina TLS y enruta los orígenes de API, sitio público y Grafana. |
| `restobite-backend` | Incluido como `backend` | API NestJS en el puerto privado 4000. |
| `restobite-public` | Incluido como `public` | Frontend Next.js en el puerto privado 3000. |
| Loki | Incluido | Almacena logs durante siete días. |
| Promtail | Incluido | Envía los logs de Nginx a Loki. |
| Grafana | Incluido | Consulta los logs mediante `logs.restobite.com`. |
| Jenkins | Excluido | Se retira del despliegue; no hay servicio ni vhost de CI. |
| PostgreSQL / Cloud SQL | Excluido | Sigue siendo un servicio externo configurado mediante `DATABASE_URL`. |
| Frontend administrativo | Excluido | Sigue desplegado estáticamente en S3 y CloudFront. |

Solo Nginx publica `80` y `443`. Los demás servicios se comunican mediante la red privada `restobite` creada por Compose.

## Configuración inicial de la VPS

1. Copie cada archivo de `env/*.env.example` a su equivalente sin `.example` y asigne valores de producción. No suba esos archivos a Git.
2. Instale la credencial de la cuenta de servicio GCP en `secrets/gcp-service-account.json`; el backend la recibe como `/run/secrets/gcp_service_account`.
3. Verifique que `LETSENCRYPT_DIR` apunte a la ruta absoluta que contiene `live/origin-api.restobite.com`, `live/origin-www.restobite.com` y `live/logs.restobite.com`.
4. Use tags `sha-<commit>` publicados por los workflows de aplicación en `env/compose.env`.
5. Inicie sesión antes de descargar imágenes privadas. El token debe pertenecer a `Felynx3` y tener únicamente `read:packages`:

```sh
docker login ghcr.io -u Felynx3
docker compose --env-file env/compose.env config
docker compose --env-file env/compose.env pull
```

Los valores de `API_ORIGIN_SECRET` y `PUBLIC_ORIGIN_SECRET` deben ser cadenas opacas URL-safe sin comillas ni saltos de línea. Nginx las inyecta al iniciar; nunca se escriben en una configuración versionada.

## Migración desde los contenedores legacy

Planifique una ventana de mantenimiento: el Nginx legacy y el nuevo stack compiten por los puertos 80/443.

1. Respalde el volumen legacy `restobite-logs-loki` y el directorio `grafana/data/grafana` del checkout legacy `restobite-logs`.
2. Cree o copie esos respaldos en los volúmenes `restobite_loki_data` y `restobite_grafana_data` antes del corte. Conserve los respaldos hasta validar el stack nuevo.
3. Valide configuración y certificados con `docker compose --env-file env/compose.env config` y, una vez iniciado el contenedor, `docker compose --env-file env/compose.env exec nginx nginx -t`.
4. Detenga los contenedores legacy `restobite-nginx`, `restobite-backend`, `restobite-public`, `restobite-logs-loki`, `restobite-logs-promtail`, `restobite-logs-grafana` y `jenkins`.
5. Ejecute `docker compose --env-file env/compose.env up -d`.
6. Compruebe las rutas API y públicas mediante CloudFront, el login de Grafana y las dos fuentes de logs: aplicación → Loki y Nginx → Promtail → Loki.

Para revertir, detenga el stack Compose, restaure los datos si fuese necesario y reinicie los contenedores legacy en `restobite-network`.

La eliminación del registro Route 53 para `ci.restobite.com`, la renovación de certificados y la gestión de Cloud SQL no se automatizan aquí y deben coordinarse fuera de este repositorio.

## Operación diaria

Para actualizar una aplicación, cambie solo su tag SHA en `env/compose.env`, descargue la imagen y recree ese servicio:

```sh
docker compose --env-file env/compose.env pull backend public
docker compose --env-file env/compose.env up -d backend public
```

Revise `WORKFLOW_PROMPTS.md` para crear los workflows de publicación de imágenes en los repositorios de backend y frontend público.
