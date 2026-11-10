# evolution-api-deploy

Despliegue de [Evolution API](https://github.com/EvolutionAPI/evolution-api) **v2.3.7** para
**DCM** (cliente DevEx Consulting), con PostgreSQL 15 y Redis 7.4 propios y aislados.
Instalación nueva: no incluye datos, sesiones ni credenciales de ningún servidor anterior.

> Este repositorio es público. **No contiene secretos.** Las variables privadas se configuran
> únicamente en el panel de DCM.

## Qué despliega

| Servicio | Imagen (versión fija) | Puertos |
|---|---|---|
| `api` | `evoapicloud/evolution-api:v2.3.7` | 8080 interno (DCM lo publica solo en `127.0.0.1` y Traefik lo expone por HTTPS) |
| `postgres` | `postgres:15.14-alpine` | ninguno |
| `redis` | `redis:7.4-alpine` | ninguno |
| `commands` | `node:22.21.1-alpine` | ninguno (recibe el webhook de `api` por la red privada) |

- Sin `latest`, sin `ports`, sin `container_name`, sin redes ni volúmenes externos (DCM los rechaza).
- Volúmenes con nombre: `evolution_instances`, `postgres_data`, `redis_data`.
- Healthchecks en los tres servicios; `api` espera a que PostgreSQL y Redis estén sanos.
- `dcm.overlap: stop-blue-before-start` en `api`: nunca hay dos copias conectadas a la misma sesión de WhatsApp.
- Mensajes, contactos, chats e historial **se guardan en PostgreSQL** para verlos en el chat de `/manager`
  (y entran en los respaldos). El chat se actualiza al recargar la página: websocket apagado.
- Los chats privados `@lid` se guardan bajo el número real (trigger `unificar_lid` instalado al arrancar `api`),
  para que cada persona sea un solo chat. El identificador original queda en `key.previousRemoteJid`.
- Webhook global solo de `messages.upsert`, hacia el servicio interno `commands`. Websocket, Chatwoot,
  RabbitMQ y demás integraciones apagadas.

## Variables (se cargan en DCM, ver `.env.example`)

| Variable | Descripción |
|---|---|
| `SERVER_URL` | URL pública HTTPS del servicio, sin barra final |
| `AUTHENTICATION_API_KEY` | Clave de administración de la API (aleatoria, >= 32 caracteres) |
| `POSTGRES_PASSWORD` | Contraseña de PostgreSQL (solo letras y números) |
| `REDIS_PASSWORD` | Contraseña de Redis (solo letras y números) |

Si falta alguna, el despliegue falla antes de arrancar (`${VAR:?...}`) y el anterior no se toca.

## Despliegue en DCM

1. Iniciar sesión en el panel como el cliente **DevEx Consulting**.
2. Crear un proyecto nuevo con este repositorio (URL HTTPS pública), rama `main`.
3. En *Environment*, cargar las 4 variables. Generar los valores con `openssl rand -hex 24`.
4. Servicio web: `api`, puerto `8080`. Usar el dominio automático `<slug>.dcm.devex.pe`
   y poner esa misma URL en `SERVER_URL`.
5. Recursos del proyecto: subir el límite de memoria a **1024 MB** (el predeterminado de DCM es 512 MB).
6. Desplegar y esperar a que los tres servicios estén *healthy*.

## Vincular WhatsApp (QR)

```bash
URL=https://<slug>.dcm.devex.pe
KEY=<AUTHENTICATION_API_KEY>

# 1) crear la instancia y pedir el QR
curl -s -X POST "$URL/instance/create" \
  -H "apikey: $KEY" -H "Content-Type: application/json" \
  -d '{"instanceName":"DevEx","integration":"WHATSAPP-BAILEYS","qrcode":true}'

# 2) volver a pedir el QR si caducó
curl -s "$URL/instance/connect/DevEx" -H "apikey: $KEY"
```

El panel web está en `$URL/manager`. Escanear desde WhatsApp → *Dispositivos vinculados*.

## Probar el envío

```bash
curl -s -X POST "$URL/message/sendText/DevEx" \
  -H "apikey: $KEY" -H "Content-Type: application/json" \
  -d '{"number":"51XXXXXXXXX","text":"Prueba Evolution API"}'
```

## Comandos de WhatsApp

| Comando | Dónde | Respuesta |
|---|---|---|
| `!revealId` | Solo en grupos (exacto, respeta mayúsculas) | El id del grupo, por ejemplo `120363000000000000@g.us` |

Requiere que "Ignore Groups" siga apagado en la instancia.

## Datos y respaldos

Los volúmenes los respalda el sistema de backups de DCM (restic) junto con el resto del servidor.
No se publican puertos de PostgreSQL ni de Redis.

## Actualizaciones

Las actualizaciones automáticas se añadirán más adelante, solo para Evolution API, tras verificar
el envío de mensajes. La versión se cambia siempre editando la etiqueta de `compose.yaml`.
