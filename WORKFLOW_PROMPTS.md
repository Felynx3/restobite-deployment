# Prompts OpenSpec para publicar imágenes

Ejecute cada bloque desde el repositorio indicado. Los cambios que generen estos prompts publican imágenes; no deben conectarse a la VPS ni ejecutar `docker compose` remoto.

## `restobite-backend`

```text
$openspec-propose

Crea un cambio para publicar la imagen de producción de restobite-backend en el paquete privado GHCR `ghcr.io/felynx3/restobite-backend` mediante GitHub Actions.

El cambio debe crear un workflow versionado en `.github/workflows/publish-container.yml` con estas decisiones cerradas:
- Se activa en tags `v*`; no realiza despliegues por SSH ni modifica la VPS.
- Declara exactamente los permisos mínimos `contents: read` y `packages: write`.
- Usa `actions/checkout`, `docker/setup-buildx-action`, `docker/login-action` contra `ghcr.io` con `${{ github.actor }}` y `${{ secrets.GITHUB_TOKEN }}`, `docker/metadata-action` y `docker/build-push-action`.
- Construye contexto `.` con el Dockerfile de producción del repositorio (`docker/node/DockerfileProd`), publica el paquete privado y usa caché `type=gha` para lectura y escritura.
- En todo push publicado genera `sha-<SHA-completo>` mediante `type=sha,format=long,prefix=sha-`. En un tag `vX.Y.Z` publica además ese tag exacto. No publica `latest`.
- Conserva labels OCI de metadata-action y habilita `push: true` solo para los eventos de publicación definidos.
- No pasa `DATABASE_URL`, secretos HMAC, credenciales AWS/GCP, JWT ni otros secretos server-side como build args, variables de build o capas de imagen. Esos valores se suministran sólo al ejecutar Compose en la VPS.
- Tras el primer publish, documenta que el paquete debe permanecer privado y asociado a este repositorio para que `GITHUB_TOKEN` tenga acceso.

Incluye pruebas o validaciones proporcionales para revisar la sintaxis del workflow y verificar que sus tags, permisos, Dockerfile y límites de secretos cumplen el contrato.
```

Cuando los artefactos estén revisados, aplique el cambio en ese repositorio con `$openspec-apply-change`.

## `restobite-public`

```text
$openspec-propose

Crea un cambio para publicar la imagen de producción de restobite-public en el paquete privado GHCR `ghcr.io/felynx3/restobite-public` mediante GitHub Actions.

El cambio debe crear un workflow versionado en `.github/workflows/publish-container.yml` con estas decisiones cerradas:
- Se activa en tags `v*`; no realiza despliegues por SSH ni modifica la VPS.
- Declara exactamente los permisos mínimos `contents: read` y `packages: write`.
- Usa `actions/checkout`, `docker/setup-buildx-action`, `docker/login-action` contra `ghcr.io` con `${{ github.actor }}` y `${{ secrets.GITHUB_TOKEN }}`, `docker/metadata-action` y `docker/build-push-action`.
- Construye contexto `.` con el Dockerfile de producción que el repositorio mantenga como contrato de release, publica el paquete privado y usa caché `type=gha` para lectura y escritura.
- En todo push publicado genera `sha-<SHA-completo>` mediante `type=sha,format=long,prefix=sha-`. En un tag `vX.Y.Z` publica además ese tag exacto. No publica `latest`.
- Conserva labels OCI de metadata-action y habilita `push: true` solo para los eventos de publicación definidos.
- Audita las variables de Next.js: sólo variables explícitamente públicas (`NEXT_PUBLIC_*`) pueden ser build args y deben provenir de GitHub Actions Variables. `RESTOBITE_BACKEND_API_SECRET`, `RESTOBITE_BACKEND_API_CREDENTIAL_ID`, URL/credenciales de base de datos y cualquier credencial cloud son exclusivamente runtime y no pueden entrar en build args, variables de build ni capas de imagen.
- Tras el primer publish, documenta que el paquete debe permanecer privado y asociado a este repositorio para que `GITHUB_TOKEN` tenga acceso.

Incluye pruebas o validaciones proporcionales para revisar la sintaxis del workflow y verificar que sus tags, permisos, Dockerfile y límites de secretos cumplen el contrato.
```

Cuando los artefactos estén revisados, aplique el cambio en ese repositorio con `$openspec-apply-change`.
