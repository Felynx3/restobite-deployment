## Why

El proxy actual depende de certificados de Let's Encrypt emitidos y renovados fuera del stack, además de un directorio de certificados de la VPS. Caddy puede conservar las mismas reglas de borde mientras obtiene, renueva y guarda automáticamente sus certificados.

## What Changes

- Sustituir por completo el servicio, configuración, variables de entorno y documentación de Nginx por Caddy.
- Conservar los tres hosts publicados, la redirección de HTTP a HTTPS, HSTS, los destinos internos y la protección por secreto de los orígenes de API y sitio público.
- Mantener los logs de acceso y error del sitio público disponibles para Promtail/Loki con semántica operativa equivalente.
- **BREAKING** Eliminar `LETSENCRYPT_DIR` y la dependencia de certificados preexistentes en la VPS; Caddy gestionará el ciclo de vida de certificados mediante ACME.
- Crear y montar `caddy/` como la ubicación versionada y persistente del estado que Caddy necesita conservar entre recreaciones, incluidos sus datos y configuración de certificados.

## Capabilities

### New Capabilities

- `edge-reverse-proxy`: Publicación TLS de los orígenes de RestoBite mediante Caddy, con enrutamiento, controles de acceso y observabilidad equivalentes a los del proxy actual.

### Modified Capabilities

- Ninguna.

## Impact

- Afecta `docker-compose.yml`, los directorios y variables de entorno del proxy, la configuración de Promtail y la guía operativa de la VPS.
- Reemplaza la imagen y los archivos de Nginx por Caddy y un directorio `caddy/` montado en el contenedor.
- Requiere que los tres nombres DNS públicos permitan la validación ACME HTTP de Caddy por los puertos 80 y 443.
