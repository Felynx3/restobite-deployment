## Why

El despliegue actual de la VPS está distribuido entre contenedores iniciados por scripts y un Jenkins no versionado. Centralizarlo en este repositorio permite reproducir la plataforma, retirar la dependencia de Jenkins y consumir imágenes de aplicación publicadas desde GitHub Actions.

## What Changes

- Añadir un Docker Compose de producción que administre Nginx, backend, frontend público, Loki, Promtail y Grafana.
- Consumir las imágenes privadas de backend y frontend público desde GitHub Container Registry mediante tags explícitos.
- Reemplazar las configuraciones y montajes de los lanzadores Docker actuales por configuración versionada y secretos externos.
- Retirar el proxy y la dependencia operativa de Jenkins del stack administrado.
- Documentar los prompts OpenSpec para que ambos proyectos de aplicación creen workflows que publiquen imágenes inmutables en GHCR.

## Capabilities

### New Capabilities

- `vps-compose-deployment`: Despliegue reproducible de los servicios Docker de la VPS, sin base de datos ni Jenkins.
- `application-image-publication`: Contrato de publicación y selección de imágenes privadas de backend y frontend público en GHCR.

### Modified Capabilities

- Ninguna.

## Impact

- Nuevo Docker Compose, configuración de Nginx/observabilidad, ejemplos de entorno y guía operativa en este repositorio.
- La VPS requerirá una sesión de lectura en GHCR, certificados Let’s Encrypt existentes y migración de los datos persistentes de Loki/Grafana.
- Los repositorios `restobite-backend` y `restobite-public` recibirán sus propios cambios OpenSpec/workflows en una tarea posterior guiada por este repositorio.
