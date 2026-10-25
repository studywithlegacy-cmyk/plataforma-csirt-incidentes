# Manual técnico · Plataforma de gestión de incidentes CSIRT/SOC

![Node.js](https://img.shields.io/badge/Node.js-22-339933?logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-4-000000?logo=express&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-6-2D3748?logo=prisma&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.4-4479A1?logo=mysql&logoColor=white)
![Licencia](https://img.shields.io/badge/licencia-ISC-blue)

API REST y aplicación web para registrar, clasificar y dar seguimiento a incidentes de ciberseguridad, con
autenticación JWT, control de acceso por roles, validación en el servidor y persistencia en MySQL. Este
documento está dirigido a quien desarrolle, despliegue u opere la plataforma. Para el uso diario consulta el
[Manual de usuario](MANUAL_USUARIO.md); para la visión general del expediente, el [README](README.md).

| Dato                  | Valor                                                                         |
| :-------------------- | :---------------------------------------------------------------------------- |
| URL de producción     | https://csirt-incidentes-javier.onrender.com                                  |
| Repositorio           | https://github.com/studywithlegacy-cmyk/plataforma-csirt-incidentes           |
| Carpeta desplegable   | `PROYECTO_FINAL_PRODUCCION/`                                                  |
| Versión desplegada    | commit `644634e` (merge del PR #2, rama `persistencia-mysql`)                 |

## Tabla de contenidos

- [1. Arquitectura](#1-arquitectura)
- [2. Estructura del repositorio](#2-estructura-del-repositorio)
- [3. Modelo de datos](#3-modelo-de-datos)
- [4. Requisitos previos](#4-requisitos-previos)
- [5. Instalación y ejecución local](#5-instalación-y-ejecución-local)
- [6. Variables de entorno](#6-variables-de-entorno)
- [7. Referencia de la API](#7-referencia-de-la-api)
- [8. Autenticación y autorización](#8-autenticación-y-autorización)
- [9. Validación y reglas de negocio](#9-validación-y-reglas-de-negocio)
- [10. Seguridad](#10-seguridad)
- [11. Observabilidad](#11-observabilidad)
- [12. Pruebas](#12-pruebas)
- [13. Despliegue](#13-despliegue)
- [14. Operación y reversión](#14-operación-y-reversión)
- [15. Solución de problemas](#15-solución-de-problemas)
- [16. Limitaciones conocidas](#16-limitaciones-conocidas)
- [17. Contribución](#17-contribución)

## 1. Arquitectura

```text
┌──────────────────────────────┐
│ Navegador                    │  SPA en HTML, CSS y JavaScript (módulos ES)
│ public/index.html + js/*.js  │  Token JWT en sessionStorage (por pestaña)
└──────────────┬───────────────┘
               │ HTTPS (certificado gestionado por Render)
               ▼
┌──────────────────────────────────────────────────────────────────┐
│ Render · Web Service (plan Free) · Node.js 22 · Express 4        │
│                                                                  │
│  Redirección HTTP→HTTPS · Helmet · CORS · límite de tasa         │
│  Auditoría (CORRELATION_ID) · express.json (20 kb)               │
│         │                                                        │
│  routes/ ─► middlewares/ (JWT, roles, Zod, ID) ─► controllers/   │
│                                                    │             │
│                                    services/ ◄─────┘             │
│                                    utils/estado.js (transiciones)│
│                                        │ Prisma ORM              │
└────────────────────────────────────────┼─────────────────────────┘
                                         │ TLS verificado
                                         │ (sslcert + sslaccept=strict)
                                         ▼
                        ┌────────────────────────────────┐
                        │ Aiven for MySQL 8.4.11         │
                        │ plan Free-1 · base administrada│
                        └────────────────────────────────┘
```

El mismo proceso de Express sirve el frontend estático (`public/`) y la API (`/api/*`). Cliente y API comparten
origen, por lo que el frontend llama a la API con rutas relativas (`/api/...`).

| Capa            | Carpeta              | Responsabilidad                                                        |
| :-------------- | :------------------- | :--------------------------------------------------------------------- |
| Configuración   | `src/config/`        | Lee y valida variables de entorno; crea el cliente de Prisma.          |
| Rutas           | `src/routes/`        | Declara endpoints y encadena middlewares por ruta.                     |
| Middlewares     | `src/middlewares/`   | Autenticación, autorización, validación, auditoría y manejo de errores.|
| Esquemas        | `src/schemas/`       | Esquemas Zod de entrada (cuerpo y consulta) con sanitización.          |
| Controladores   | `src/controllers/`   | Traducen la petición HTTP en llamadas al servicio y forman la respuesta.|
| Servicios       | `src/services/`      | Lógica de negocio y acceso a datos con Prisma.                         |
| Utilidades      | `src/utils/`         | Máquina de estados de incidentes y firma y verificación de tokens.     |
| Persistencia    | `prisma/`            | Esquema, migraciones SQL y datos iniciales (seed).                     |

`src/services/index.js` selecciona la implementación según `PERSISTENCE`: `incidente.service.js` (Prisma y
MySQL) o `incidente.memory.service.js` (arreglo en memoria, usado por las pruebas automatizadas).

## 2. Estructura del repositorio

```text
PROYECTO_FINAL_PRODUCCION/
├── package.json              # Dependencias y scripts npm
├── .env.example              # Plantilla de variables (sin valores reales)
├── prisma/
│   ├── schema.prisma         # Modelos Usuario e Incidente, enumeraciones
│   ├── migrations/           # 20261019000000_init (creación de tablas)
│   └── seed.js               # Usuarios iniciales (upsert) e incidentes base si la tabla está vacía
├── public/                   # Frontend servido por Express
│   ├── index.html            # Vistas de inicio de sesión y panel; plantilla de fila
│   ├── css/styles.css        # Estilos y diseño adaptable (móvil, tableta, escritorio)
│   ├── js/api.js             # Cliente HTTP: añade el token y gestiona el 401
│   ├── js/auth.js            # Inicio y cierre de sesión; sessionStorage
│   ├── js/incidentes.js      # Llamadas CRUD a /api/incidentes
│   └── js/main.js            # Lógica de la interfaz y permisos visuales
├── src/
│   ├── server.js             # Arranque HTTP y apagado controlado
│   ├── app.js                # Configuración de Express y orden de middlewares
│   ├── config/               # env.js (validación fail-fast) y prisma.js
│   ├── routes/               # auth.routes.js, incidente.routes.js
│   ├── middlewares/          # auth, validate, validar-id, audit, error
│   ├── schemas/              # auth.schema.js, incidente.schema.js
│   ├── controllers/          # auth.controller.js, incidente.controller.js
│   ├── services/             # auth.service.js, incidente.service.js, incidente.memory.service.js
│   └── utils/                # estado.js, token.js
└── tests/
    ├── estado.test.js        # Pruebas unitarias de la máquina de estados
    └── integracion.test.js   # Pruebas de integración de la API
```

> [!NOTE]
> `src/middlewares/sanitize.middleware.js` existe pero no está conectado: la sanitización se aplica dentro de
> los esquemas Zod. `Dockerfile`, `docker-compose.yml` y `nginx/` corresponden a una orquestación alternativa
> con contenedores que **no** forma parte del despliegue en Render.

## 3. Modelo de datos

### Tabla `usuarios`

| Columna        | Tipo                                   | Restricciones                     |
| :------------- | :------------------------------------- | :-------------------------------- |
| `id`           | `INT`                                  | PK, `AUTO_INCREMENT`              |
| `nombre`       | `VARCHAR(100)`                         | `NOT NULL`                        |
| `email`        | `VARCHAR(120)`                         | `NOT NULL`, `UNIQUE`              |
| `passwordHash` | `VARCHAR(255)`                         | `NOT NULL` (bcrypt, coste 12)     |
| `rol`          | `ENUM('ADMIN','SUPERVISOR','ANALISTA')`| `NOT NULL`, por defecto `ANALISTA`|
| `createdAt`    | `DATETIME(3)`                          | por defecto `CURRENT_TIMESTAMP(3)`|
| `updatedAt`    | `DATETIME(3)`                          | actualizado por Prisma            |

### Tabla `incidentes`

| Columna       | Tipo                                                  | Restricciones                      |
| :------------ | :---------------------------------------------------- | :--------------------------------- |
| `id`          | `INT`                                                 | PK, `AUTO_INCREMENT`               |
| `titulo`      | `VARCHAR(120)`                                        | `NOT NULL`                         |
| `descripcion` | `TEXT`                                                | `NOT NULL`                         |
| `resolucion`  | `TEXT`                                                | opcional                           |
| `prioridad`   | `ENUM('BAJA','MEDIA','ALTA','CRITICA')`               | `NOT NULL`, por defecto `MEDIA`    |
| `estado`      | `ENUM('ABIERTO','EN_PROCESO','RESUELTO','CERRADO')`   | `NOT NULL`, por defecto `ABIERTO`  |
| `createdAt`   | `DATETIME(3)`                                         | por defecto `CURRENT_TIMESTAMP(3)` |
| `updatedAt`   | `DATETIME(3)`                                         | actualizado por Prisma             |

Ambas tablas usan `utf8mb4` con `utf8mb4_unicode_ci`. Prisma mantiene además `_prisma_migrations` con el
historial de migraciones aplicadas.

### Máquina de estados de un incidente

```text
              ┌────────────┐
  (creación)─►│  ABIERTO   │◄──────────────┐
              └─────┬──────┘               │
                    │                      │  Cualquier transición entre
                    ▼                      │  ABIERTO, EN_PROCESO y RESUELTO
              ┌────────────┐               │  está permitida.
              │ EN_PROCESO │◄──────────────┤
              └─────┬──────┘               │
                    ▼                      │
              ┌────────────┐               │
              │  RESUELTO  │───────────────┘
              └─────┬──────┘
                    │ requiere resolución (≥ 5 caracteres)
                    ▼
              ┌────────────┐
              │  CERRADO   │  estado final: no admite transiciones
              └────────────┘
```

Se puede pasar a `CERRADO` desde cualquier estado no cerrado si se aporta la resolución. Las reglas están en
`src/utils/estado.js` (`puedeCambiarEstado`).

## 4. Requisitos previos

- Node.js 22 y npm.
- Git.
- Una base MySQL 8.4 accesible (por ejemplo, Aiven), **o** ejecutar en modo memoria sin base de datos.
- Para MySQL administrado con TLS: el certificado CA del proveedor.
- Opcional: un cliente HTTP (Thunder Client, Postman o `curl`) para probar la API.

## 5. Instalación y ejecución local

1. Clona el repositorio y entra en la carpeta desplegable:

   ```bash
   git clone https://github.com/studywithlegacy-cmyk/plataforma-csirt-incidentes.git
   cd plataforma-csirt-incidentes/PROYECTO_FINAL_PRODUCCION
   ```

2. Instala las dependencias:

   ```bash
   npm ci
   ```

3. Crea el archivo de entorno a partir de la plantilla y complétalo (ver [sección 6](#6-variables-de-entorno)):

   ```bash
   cp .env.example .env
   openssl rand -base64 48   # genera un valor para JWT_SECRET
   ```

4. Elige el modo de persistencia.

   **Modo memoria** (sin base de datos; los datos se pierden al reiniciar): define `PERSISTENCE=memory` en
   `.env`.

   **Modo MySQL**: define `PERSISTENCE=prisma` y `DATABASE_URL`, y aplica migraciones y datos iniciales:

   ```bash
   npx prisma migrate deploy
   npm run prisma:seed
   ```

5. Inicia el servidor:

   ```bash
   npm start
   ```

6. Abre `http://localhost:3000` (o el puerto definido en `PORT`) y comprueba la salud del servicio:

   ```bash
   curl http://localhost:3000/api/health
   ```

> [!IMPORTANT]
> `CLIENT_ORIGIN` debe incluir el origen exacto desde el que abres la aplicación (por ejemplo,
> `http://localhost:3000`). Si no coincide, el navegador recibe `403` por CORS al iniciar sesión.

**Scripts npm disponibles**

| Script                    | Comando                                                              |
| :------------------------ | :------------------------------------------------------------------- |
| `npm start`               | `node src/server.js`                                                 |
| `npm run start:render`    | `prisma migrate deploy && node prisma/seed.js && node src/server.js` |
| `npm run build`           | `prisma generate`                                                    |
| `npm run dev`             | `nodemon src/server.js`                                              |
| `npm test`                | `node --test tests/*.test.js`                                        |
| `npm run prisma:generate` | `prisma generate`                                                    |
| `npm run prisma:migrate`  | `prisma migrate deploy`                                              |
| `npm run prisma:seed`     | `node prisma/seed.js`                                                |
| `npm run prisma:studio`   | `prisma studio`                                                      |

## 6. Variables de entorno

| Variable              | Obligatoria             | Propósito                                                        | Ejemplo                       |
| :-------------------- | :---------------------- | :--------------------------------------------------------------- | :---------------------------- |
| `NODE_ENV`            | Recomendada             | `production` activa la redirección a HTTPS y escucha en `0.0.0.0`| `production`                  |
| `PORT`                | No (por defecto `3000`) | Puerto HTTP; Render lo asigna automáticamente                    | `3000`                        |
| `HOST`                | No                      | Interfaz de escucha; por defecto `127.0.0.1` (o `0.0.0.0` en producción) | `127.0.0.1`           |
| `PERSISTENCE`         | No                      | `prisma` (MySQL) o `memory`; por defecto `prisma` si hay `DATABASE_URL` | `prisma`               |
| `DATABASE_URL`        | Sí, con `prisma`        | Cadena de conexión MySQL con parámetros TLS                      | `mysql://<USUARIO>:********@<HOST>:<PUERTO>/defaultdb?sslcert=<RUTA_CA>&sslaccept=strict` |
| `JWT_SECRET`          | Sí                      | Clave HS256 de firma de tokens (mínimo 32 bytes)                 | `<REEMPLAZAR>`                |
| `JWT_EXPIRES_IN`      | No (por defecto `30m`)  | Vigencia del token                                               | `30m`                         |
| `CLIENT_ORIGIN`       | Sí                      | Orígenes permitidos por CORS, separados por comas                | `https://<SERVICIO>.onrender.com` |
| `ADMIN_PASSWORD`      | Sí                      | Contraseña inicial de la cuenta ADMIN (mínimo 12 bytes)          | `********`                    |
| `SUPERVISOR_PASSWORD` | Sí                      | Contraseña inicial de la cuenta SUPERVISOR (mínimo 12 bytes)     | `********`                    |
| `ANALISTA_PASSWORD`   | Sí                      | Contraseña inicial de la cuenta ANALISTA (mínimo 12 bytes)       | `********`                    |
| `NODE_VERSION`        | Solo en Render          | Versión de Node.js que usa Render para construir                 | `22`                          |

Además, en Render se define el **Secret File** `aiven-ca.pem` (certificado CA del proyecto Aiven), disponible en
`/etc/secrets/aiven-ca.pem`.

> [!WARNING]
> Aiven entrega la URI con `?ssl-mode=REQUIRED`. **Prisma no reconoce ese parámetro y se conecta sin cifrar.**
> Sustitúyelo por `?sslcert=<RUTA_CA>&sslaccept=strict`.

### Arranque fail-fast

`src/config/env.js` valida la configuración al cargar el módulo y lanza un error antes de aceptar tráfico si:

- falta `JWT_SECRET`, `CLIENT_ORIGIN`, `ADMIN_PASSWORD`, `SUPERVISOR_PASSWORD` o `ANALISTA_PASSWORD`;
- `PERSISTENCE` es `prisma` y falta `DATABASE_URL`;
- `JWT_SECRET` tiene menos de 32 bytes;
- alguna de las tres contraseñas iniciales tiene menos de 12 bytes.

El mensaje indica qué variable falla, por ejemplo:
`[CRÍTICO] Faltan las siguientes variables de entorno: JWT_SECRET`.

## 7. Referencia de la API

URL base de producción: `https://csirt-incidentes-javier.onrender.com/api`. Todas las respuestas son JSON. Las
rutas de incidentes exigen la cabecera `Authorization: Bearer <TOKEN>`.

### Resumen de endpoints

| Método  | Ruta                   | Acceso                 | Respuestas posibles                    |
| :------ | :--------------------- | :--------------------- | :------------------------------------- |
| `POST`  | `/api/auth/login`      | Pública                | 200 · 400 · 401 · 429                  |
| `GET`   | `/api/health`          | Pública                | 200                                    |
| `GET`   | `/api/ready`           | Pública                | 200 · 503                              |
| `GET`   | `/api/incidentes`      | Autenticado            | 200 · 400 · 401                        |
| `GET`   | `/api/incidentes/:id`  | Autenticado            | 200 · 400 · 401 · 404                  |
| `POST`  | `/api/incidentes`      | ANALISTA, SUPERVISOR, ADMIN | 201 · 400 · 401 · 403             |
| `PUT`   | `/api/incidentes/:id`  | SUPERVISOR, ADMIN      | 200 · 400 · 401 · 403 · 404            |
| `PATCH` | `/api/incidentes/:id`  | SUPERVISOR, ADMIN      | Igual que `PUT`                        |
| `DELETE`| `/api/incidentes/:id`  | ADMIN                  | 200 · 400 · 401 · 403 · 404            |

Además, en cualquier ruta: `403` si el `Origin` no está permitido por CORS, `429` si se supera el límite de
peticiones y `500` ante un error interno.

### `POST /api/auth/login`

Inicia sesión y devuelve un token JWT. Cuerpo: `email` (formato válido, máximo 120) y `password` (8 a 72).

```bash
curl -X POST https://csirt-incidentes-javier.onrender.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"analista@incidentes.com","password":"<REEMPLAZAR>"}'
```

```json
{
  "success": true,
  "message": "Autenticación exitosa",
  "data": {
    "token": "eyJ...<TOKEN>",
    "usuario": { "id": 3, "nombre": "Laura Analista Nivel 1", "email": "analista@incidentes.com", "rol": "ANALISTA" }
  }
}
```

| Código | Cuerpo                                                                                  |
| :----: | :-------------------------------------------------------------------------------------- |
| 400    | `{"error":"Datos de entrada inválidos","detalles":{"email":["..."]}}`                   |
| 401    | `{"error":"Credenciales incorrectas","codigo":"INVALID_CREDENTIALS"}`                   |
| 429    | `{"error":"Demasiados intentos de autenticación. Bloqueado temporalmente."}`            |

### `GET /api/health`

Comprobación de vida del proceso. No consulta la base de datos. También responde en `/health`.

```bash
curl https://csirt-incidentes-javier.onrender.com/api/health
```

```json
{
  "status": "UP",
  "estado": "ok",
  "timestamp": "2026-10-24T21:31:32.474Z",
  "version": "1.0.0",
  "environment": "production",
  "services": { "api": "healthy" },
  "uptime": 7.24,
  "persistence": "prisma"
}
```

### `GET /api/ready`

Comprobación de disponibilidad: ejecuta `SELECT 1` contra MySQL. También responde en `/ready`.

```bash
curl https://csirt-incidentes-javier.onrender.com/api/ready
```

```json
{
  "status": "READY",
  "timestamp": "2026-10-24T21:31:32.223Z",
  "services": { "api": "healthy", "database": "connected" },
  "uptime": 6.99,
  "persistence": "prisma"
}
```

| Código | Situación                                                                                        |
| :----: | :----------------------------------------------------------------------------------------------- |
| 200    | `database: "connected"` con MySQL, o `database: "not_configured"` en modo memoria                |
| 503    | `{"status":"UNAVAILABLE","services":{"api":"healthy","database":"disconnected"},...}`            |

### `GET /api/incidentes`

Lista los incidentes ordenados por `createdAt` descendente. Filtros opcionales: `estado` y `prioridad`.

```bash
curl "https://csirt-incidentes-javier.onrender.com/api/incidentes?prioridad=ALTA&estado=ABIERTO" \
  -H "Authorization: Bearer eyJ...<TOKEN>"
```

```json
{
  "success": true,
  "total": 1,
  "data": [
    {
      "id": 7,
      "titulo": "Escaneo de puertos en DMZ",
      "descripcion": "Nmap SYN scan desde 203.0.113.50 contra 10.0.1.0/24",
      "resolucion": null,
      "prioridad": "ALTA",
      "estado": "ABIERTO",
      "createdAt": "<FECHA_ISO_8601>",
      "updatedAt": "<FECHA_ISO_8601>"
    }
  ]
}
```

| Código | Cuerpo                                                                                  |
| :----: | :-------------------------------------------------------------------------------------- |
| 400    | `{"error":"Parámetros de consulta inválidos","detalles":{"prioridad":["..."]}}`         |
| 401    | `{"error":"Token de acceso requerido"}` o `{"error":"Token inválido o expirado"}`       |

La respuesta incluye `Cache-Control: private, no-store`.

### `GET /api/incidentes/:id`

```bash
curl https://csirt-incidentes-javier.onrender.com/api/incidentes/7 \
  -H "Authorization: Bearer eyJ...<TOKEN>"
```

```json
{ "success": true, "data": { "id": 7, "titulo": "Escaneo de puertos en DMZ", "estado": "ABIERTO", "...": "..." } }
```

| Código | Cuerpo                                                                                                       |
| :----: | :----------------------------------------------------------------------------------------------------------- |
| 400    | `{"error":"El identificador del incidente debe ser un número entero positivo","codigo":"INVALID_ID"}`        |
| 401    | Token ausente, inválido o expirado                                                                           |
| 404    | `{"error":"Recurso no encontrado","codigo":"INCIDENTE_NO_ENCONTRADO"}`                                       |

### `POST /api/incidentes`

Cuerpo: `titulo` y `descripcion` obligatorios; `prioridad` (por defecto `MEDIA`), `estado` (por defecto
`ABIERTO`) y `resolucion` opcionales.

```bash
curl -X POST https://csirt-incidentes-javier.onrender.com/api/incidentes \
  -H "Authorization: Bearer eyJ...<TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Escaneo de puertos en DMZ","descripcion":"Nmap SYN scan desde 203.0.113.50","prioridad":"ALTA"}'
```

```json
{ "success": true, "message": "Incidente creado exitosamente", "data": { "id": 8, "estado": "ABIERTO", "...": "..." } }
```

| Código | Cuerpo                                                                                                  |
| :----: | :------------------------------------------------------------------------------------------------------ |
| 400    | `{"error":"Datos de entrada inválidos","detalles":{"titulo":["El título debe tener al menos 5 caracteres"]}}` |
| 401    | Token ausente, inválido o expirado                                                                      |
| 403    | `{"error":"Permiso insuficiente para realizar esta acción"}`                                            |

### `PUT /api/incidentes/:id` y `PATCH /api/incidentes/:id`

Actualiza uno o varios campos (`titulo`, `descripcion`, `prioridad`, `estado`, `resolucion`). Exige al menos
uno. Si cambia el estado, se aplica la máquina de estados.

```bash
curl -X PUT https://csirt-incidentes-javier.onrender.com/api/incidentes/7 \
  -H "Authorization: Bearer eyJ...<TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"estado":"CERRADO","resolucion":"IP bloqueada en firewall perimetral"}'
```

```json
{ "success": true, "message": "Incidente actualizado exitosamente", "data": { "id": 7, "estado": "CERRADO", "...": "..." } }
```

| Código | Cuerpo                                                                                                                         |
| :----: | :----------------------------------------------------------------------------------------------------------------------------- |
| 400    | `{"error":"Datos de entrada inválidos","detalles":{...}}` (también si el cuerpo llega vacío)                                   |
| 400    | `{"error":"Para cerrar un incidente es obligatorio ingresar una resolución de al menos 5 caracteres","codigo":"INVALID_STATE_TRANSITION"}` |
| 400    | `{"error":"Un incidente cerrado no puede reactivarse directamente","codigo":"INVALID_STATE_TRANSITION"}`                       |
| 401    | Token ausente, inválido o expirado                                                                                             |
| 403    | `{"error":"Permiso insuficiente para realizar esta acción"}`                                                                   |
| 404    | `{"error":"Recurso no encontrado","codigo":"INCIDENTE_NO_ENCONTRADO"}`                                                         |

### `DELETE /api/incidentes/:id`

```bash
curl -X DELETE https://csirt-incidentes-javier.onrender.com/api/incidentes/7 \
  -H "Authorization: Bearer eyJ...<TOKEN>"
```

```json
{ "success": true, "message": "Incidente eliminado exitosamente", "data": { "id": 7 } }
```

| Código | Cuerpo                                                                                 |
| :----: | :------------------------------------------------------------------------------------- |
| 400    | ID no entero positivo (`INVALID_ID`)                                                   |
| 401    | Token ausente, inválido o expirado                                                     |
| 403    | `{"error":"Permiso insuficiente para realizar esta acción"}`                           |
| 404    | `{"error":"Recurso no encontrado","codigo":"INCIDENTE_NO_ENCONTRADO"}`                 |

### Errores generales

| Código | Cuerpo                                                                                       |
| :----: | :------------------------------------------------------------------------------------------- |
| 403    | `{"error":"Origen no permitido por CORS: <ORIGEN>","codigo":"CORS_NOT_ALLOWED"}`             |
| 404    | `{"error":"Recurso no encontrado"}` (ruta de la API inexistente)                             |
| 429    | `{"error":"Demasiadas peticiones. Intente nuevamente en un minuto."}`                        |
| 500    | `{"error":"Ocurrió un error interno en el servidor"}` (sin detalles internos en producción)  |

## 8. Autenticación y autorización

### Flujo del token

1. El cliente envía `email` y `password` a `POST /api/auth/login`.
2. `auth.service.js` busca el usuario y valida la contraseña. Si el usuario no existe, calcula un hash bcrypt
   ficticio para igualar el tiempo de respuesta y devuelve el mismo mensaje genérico.
3. Si es válida, `utils/token.js` firma un JWT **HS256** con `JWT_SECRET` y vigencia `JWT_EXPIRES_IN` (30 min).
4. El frontend guarda el token en `sessionStorage` y lo envía en `Authorization: Bearer <TOKEN>`.
5. `autenticar` verifica firma y expiración **forzando el algoritmo HS256** (`algorithms: ['HS256']`), lo que
   impide ataques de confusión de algoritmo (`alg: none`).
6. `permitir(...roles)` compara el rol del token con los roles autorizados para la ruta.
7. Ante un `401`, el frontend borra la sesión y muestra «Su sesión ha caducado. Inicie sesión nuevamente.».

**Contenido del payload**

```json
{
  "sub": 3,
  "id": 3,
  "email": "analista@incidentes.com",
  "nombre": "Laura Analista Nivel 1",
  "rol": "ANALISTA",
  "iat": 1792877490,
  "exp": 1792879290
}
```

El payload está firmado, no cifrado: no debe contener datos sensibles.

### Matriz de roles

Aplicada en el servidor en `src/routes/incidente.routes.js` (`router.use(autenticar)` y `permitir(...)`).

| Operación                     | ANALISTA | SUPERVISOR | ADMIN |
| :---------------------------- | :------: | :--------: | :---: |
| Listar y consultar por ID     |    Sí    |     Sí     |  Sí   |
| Crear                         |    Sí    |     Sí     |  Sí   |
| Actualizar (`PUT`/`PATCH`)    |   403    |     Sí     |  Sí   |
| Eliminar                      |   403    |    403     |  Sí   |

> [!NOTE]
> La API permite que SUPERVISOR cree incidentes, pero la interfaz solo muestra el formulario de registro a
> ANALISTA y ADMIN (`data-roles="ANALISTA,ADMIN"` en `index.html`). La interfaz oculta acciones; la API es la
> que las autoriza o las rechaza.

### 401 frente a 403

| Código | Significado                                       | Ejemplo                                              |
| :----: | :------------------------------------------------ | :--------------------------------------------------- |
| 401    | No autenticado: no se sabe quién hace la petición | Sin token, token alterado o expirado                 |
| 403    | Autenticado sin permiso para la acción            | ANALISTA envía `DELETE /api/incidentes/1` → rechazado |

## 9. Validación y reglas de negocio

La validación se aplica en el servidor con Zod (`src/schemas/`) antes de llegar al controlador. El cuerpo
validado sustituye a `req.body`.

| Campo         | Regla                                                                       |
| :------------ | :-------------------------------------------------------------------------- |
| `email`       | Formato de correo válido, máximo 120 caracteres                             |
| `password`    | Entre 8 y 72 caracteres                                                     |
| `titulo`      | Obligatorio al crear; 5 a 120 caracteres tras recortar espacios; sanitizado |
| `descripcion` | Obligatoria al crear; 10 a 2000 caracteres; sanitizada                      |
| `prioridad`   | `BAJA`, `MEDIA`, `ALTA` o `CRITICA`; por defecto `MEDIA`                    |
| `estado`      | `ABIERTO`, `EN_PROCESO`, `RESUELTO` o `CERRADO`; por defecto `ABIERTO`      |
| `resolucion`  | Opcional; máximo 1000 caracteres; sanitizada                                |
| `:id`         | Entero estrictamente positivo                                               |
| Actualización | Al menos un campo                                                           |
| Cuerpo        | Máximo 20 kb (`express.json({ limit: '20kb' })`)                            |

**Transiciones de estado** (`src/utils/estado.js`):

| Desde                              | Hacia                              | Condición                                |
| :--------------------------------- | :--------------------------------- | :--------------------------------------- |
| `ABIERTO`, `EN_PROCESO`, `RESUELTO`| `ABIERTO`, `EN_PROCESO`, `RESUELTO`| Permitida                                |
| `ABIERTO`, `EN_PROCESO`, `RESUELTO`| `CERRADO`                          | Requiere `resolucion` de 5 o más caracteres (nueva o ya guardada) |
| `CERRADO`                          | Cualquiera                         | Rechazada (`INVALID_STATE_TRANSITION`)   |

## 10. Seguridad

| Control                     | Implementación                                                                                         |
| :-------------------------- | :----------------------------------------------------------------------------------------------------- |
| Contraseñas                 | bcrypt con coste 12 (`SALT_ROUNDS = 12`)                                                                |
| Tokens                      | JWT HS256, vigencia 30 min, algoritmo forzado al verificar                                             |
| Enumeración de usuarios     | Mensaje único «Credenciales incorrectas» y hash ficticio para igualar tiempos                          |
| Cabeceras HTTP (Helmet)     | CSP (`default-src 'self'`, `frame-ancestors 'none'`, `object-src 'none'`), HSTS, `nosniff`, `X-Frame-Options: SAMEORIGIN` |
| HSTS                        | `max-age=31536000; includeSubDomains; preload`                                                         |
| Tecnología oculta           | `x-powered-by` deshabilitado                                                                           |
| HTTPS                       | En producción, petición con `X-Forwarded-Proto: http` → `301` a `https://`                             |
| CORS                        | Solo orígenes de `CLIENT_ORIGIN` (normalizados); otro origen → `403 CORS_NOT_ALLOWED`                  |
| Límite general              | 100 peticiones por minuto e IP en `/api`                                                               |
| Límite de inicio de sesión  | 10 intentos cada 15 minutos por IP en `/api/auth`                                                      |
| XSS                         | `sanitize-html` sin etiquetas ni atributos permitidos, aplicado en los esquemas Zod                    |
| Caché de datos sensibles    | `Cache-Control: private, no-store` en `/api/incidentes`                                                |
| Errores                     | `500` genérico sin traza en producción                                                                 |

### TLS en ambos tramos

- **Navegador → Render:** HTTPS con certificado gestionado por Render, más redirección forzada y HSTS.
- **API → MySQL:** Prisma con `sslcert=<CA de Aiven>&sslaccept=strict`: cifra la conexión y verifica el
  certificado y el nombre del servidor. La CA se carga en Render como Secret File (`/etc/secrets/aiven-ca.pem`).

### Gestión de secretos

- Los valores reales viven solo en el panel de Render y en archivos locales ignorados por Git
  (`.env`, `.env.*`, `certs/`). El repositorio solo contiene `.env.example`, sin valores.
- Las contraseñas iniciales llegan por variables de entorno; `seed.js` las guarda como hash bcrypt.
- Ante una exposición accidental, rota el secreto en el proveedor y vuelve a desplegar.

## 11. Observabilidad

### Log de auditoría

`src/middlewares/audit.middleware.js` registra una línea por cada petición a `/api`:

```text
[AUDIT_LOG] <timestamp> | CORRELATION_ID: <uuid> | IP: <ip> | USER: <email> [ROL] | ACTION: <acción> | <MÉTODO> <ruta> | STATUS: <código> | DURATION: <ms>
```

Ejemplo:

```text
[AUDIT_LOG] 2026-10-24T17:55:19.139Z | CORRELATION_ID: c09d4a23-29b2-44e3-a267-4c47e7300b18 | IP: ::1 | USER: analista@incidentes.com [ANALISTA] | ACTION: CREATE_INCIDENT | POST /api/incidentes | STATUS: 201 | DURATION: 726ms
```

| Acción registrada                | Cuándo                                         |
| :------------------------------- | :--------------------------------------------- |
| `LOGIN_SUCCESS` / `LOGIN_FAILED` | Inicio de sesión correcto o fallido            |
| `LIST_INCIDENTS`, `READ_INCIDENT_DETAIL` | Consultas                              |
| `CREATE_INCIDENT`, `UPDATE_INCIDENT_STATE`, `DELETE_INCIDENT` | Operaciones de escritura |
| `UNAUTHORIZED_ACCESS_ATTEMPT`    | Respuesta `401` fuera del inicio de sesión     |
| `FORBIDDEN_PRIVILEGE_VIOLATION`  | Respuesta `403`                                |

- **No se registran** tokens, contraseñas ni cuerpos de petición.
- El `CORRELATION_ID` se toma de la cabecera `X-Correlation-ID` (o `X-Request-ID`) si llega; si no, se genera
  un UUID. Se devuelve en la cabecera de respuesta `X-Correlation-ID`.
- Los errores no controlados se registran con el prefijo `[ERROR]`.

### Health check y readiness

| Endpoint      | Tipo       | Uso                                                                  |
| :------------ | :--------- | :------------------------------------------------------------------- |
| `/api/health` | Liveness   | El proceso responde                                                  |
| `/api/ready`  | Readiness  | La base responde a `SELECT 1`; Render lo usa como Health Check Path  |

### Apagado controlado

Al recibir `SIGTERM` o `SIGINT`, `server.js` deja de aceptar conexiones, cierra el cliente de Prisma y termina
con código 0:

```text
[INFO] Señal SIGTERM recibida. Procediendo con el apagado controlado...
[INFO] Conexiones finalizadas exitosamente. Proceso terminado.
```

## 12. Pruebas

```bash
cd PROYECTO_FINAL_PRODUCCION
npm test
```

Las pruebas usan el runner nativo `node:test`, se ejecutan en modo memoria y no necesitan base de datos.

| Archivo                     | Pruebas | Cobertura                                                                                     |
| :-------------------------- | :-----: | :-------------------------------------------------------------------------------------------- |
| `tests/estado.test.js`      | 5       | Cierre sin resolución, cierre válido, no reactivar un CERRADO, estados desconocidos, transiciones regulares |
| `tests/integracion.test.js` | 13      | Health y readiness, 401 sin token y con credenciales erróneas, login HS256 por rol, creación con sanitización XSS, `Cache-Control`, ID inválido, 403 por rol, regla de cierre, actualización por SUPERVISOR, eliminación por ADMIN, cabeceras Helmet |

Resultado esperado:

```text
# tests 18
# pass 18
# fail 0
```

## 13. Despliegue

### Base de datos (Aiven)

1. Crea un servicio **MySQL** con plan **Free-1**.
2. En **Overview → Connection information**, copia el **Service URI** y descarga el **CA certificate**.
3. Activa **Termination protection** para evitar eliminaciones accidentales.

### Servicio web (Render)

1. **New → Web Service** → conecta el repositorio.
2. Configura:

   | Campo             | Valor                                  |
   | :---------------- | :------------------------------------- |
   | Language          | Node                                   |
   | Branch            | `main`                                 |
   | Root Directory    | `PROYECTO_FINAL_PRODUCCION`            |
   | Build Command     | `npm ci && npm run build`              |
   | Start Command     | `npm run start:render`                 |
   | Instance Type     | Free                                   |
   | Health Check Path | `/api/ready`                           |

3. **Environment Variables:** define las variables de la [sección 6](#6-variables-de-entorno). En
   `DATABASE_URL`, sustituye `?ssl-mode=REQUIRED` por `?sslcert=/etc/secrets/aiven-ca.pem&sslaccept=strict`.
4. **Secret Files:** añade `aiven-ca.pem` con el contenido completo del certificado CA.
5. Pulsa **Deploy Web Service** y espera el estado **Live**.

### Orden del comando de arranque

`npm run start:render` ejecuta, en orden:

1. `prisma migrate deploy`: aplica las migraciones pendientes.
2. `node prisma/seed.js`: actualiza las cuentas iniciales (upsert con las contraseñas de entorno) e inserta los
   incidentes base solo si la tabla está vacía.
3. `node src/server.js`: inicia el servidor.

Si un paso falla, los siguientes no se ejecutan y el servicio no recibe tráfico.

> [!NOTE]
> `prisma` está en `dependencies` (no en `devDependencies`): con `NODE_ENV=production`, el build omite las
> dependencias de desarrollo y `prisma migrate deploy` no estaría disponible.

### Verificación tras el despliegue

```bash
curl https://csirt-incidentes-javier.onrender.com/api/ready
# Esperado: "database":"connected" y "persistence":"prisma"
```

## 14. Operación y reversión

### Revertir un despliegue

1. En Render → **Deploys**, identifica el último despliegue estable.
2. Usa **Rollback** sobre ese despliegue, o revierte el commit en Git y vuelve a desplegar:

   ```bash
   git revert <COMMIT>
   git push origin main
   ```

3. Comprueba `/api/ready` y los logs durante los minutos siguientes.

> [!IMPORTANT]
> Revertir el código no revierte los datos. Hasta ahora solo existe la migración inicial; cualquier cambio de
> esquema futuro debe ser compatible con la versión anterior y contar con un respaldo previo.

### Consultar logs por `CORRELATION_ID`

1. Obtén el identificador de la cabecera `X-Correlation-ID` de la respuesta (DevTools → Network → Headers).
2. En Render → **Logs**, escribe el identificador en el buscador.
3. La línea `[AUDIT_LOG]` muestra usuario, acción, ruta, código y duración de esa petición.

### Respaldos (propuesto, no implementado)

El plan Free-1 de Aiven no incluye respaldos automáticos. Se propone un volcado periódico con TLS verificado:

```bash
mysqldump --ssl-mode=VERIFY_IDENTITY --ssl-ca=aiven-ca.pem \
  -h <HOST> -P <PUERTO> -u <USUARIO> -p defaultdb > respaldo_$(date +%F).sql
```

Guarda los respaldos fuera del repositorio y comprueba su restauración en una base de prueba.

## 15. Solución de problemas

| Síntoma                                                 | Causa probable                                          | Acción                                                                    |
| :------------------------------------------------------ | :------------------------------------------------------ | :------------------------------------------------------------------------ |
| El servicio no arranca: `[CRÍTICO] Faltan las siguientes variables...` | Variable obligatoria ausente                | Defínela en Render → Environment y redespliega                             |
| `[CRÍTICO] JWT_SECRET debe tener al menos 32 bytes`     | Secreto demasiado corto                                 | Genera uno con `openssl rand -base64 48`                                  |
| `/api/ready` responde `503`                             | MySQL inaccesible o credenciales incorrectas            | Revisa `DATABASE_URL`, el estado del servicio en Aiven y los logs `[DATABASE_READINESS_ERROR]` |
| `Error opening a TLS connection: The certificate was not trusted` en Render | Falta el Secret File o la ruta no coincide | Comprueba que `aiven-ca.pem` existe y que `sslcert=/etc/secrets/aiven-ca.pem` |
| El mismo error en macOS                                 | El certificado de Aiven no incluye EKU serverAuth       | En desarrollo local usa `sslaccept=accept_invalid_certs`                  |
| `/api/ready` dice `"persistence":"memory"`              | `PERSISTENCE=memory` o falta `DATABASE_URL`             | Define `PERSISTENCE=prisma` y `DATABASE_URL`                              |
| El inicio de sesión desde el navegador devuelve `403`   | `CLIENT_ORIGIN` no coincide con la URL del servicio     | Ajusta `CLIENT_ORIGIN` a la URL exacta, sin barra final                   |
| `401 Token inválido o expirado`                         | Pasaron 30 minutos o el token está alterado             | Inicia sesión de nuevo                                                    |
| `429` en el inicio de sesión                            | Más de 10 intentos en 15 minutos desde la misma IP      | Espera a que termine la ventana                                           |
| `prisma: command not found` durante el arranque         | `prisma` instalado como dependencia de desarrollo       | Mantén `prisma` en `dependencies`                                         |
| La primera petición tarda unos 50 s                     | La instancia Free estaba suspendida                     | Comportamiento esperado del plan gratuito                                 |
| Los datos desaparecen tras reiniciar                    | Ejecución en modo memoria                               | Verifica `"persistence":"prisma"` en `/api/health`                        |

## 16. Limitaciones conocidas

- **Render Free** suspende la instancia tras unos 15 minutos sin tráfico; la primera petición tarda unos 50 s.
- **Aiven Free-1** no ofrece respaldos automáticos ni alta disponibilidad.
- **macOS en desarrollo:** Prisma rechaza el certificado de Aiven por la falta de EKU serverAuth; en local se
  usa `sslaccept=accept_invalid_certs` (cifrado sin verificación) y en producción `strict`.
- **Gestión de usuarios:** las cuentas se crean con el seed; no hay altas, bajas ni cambios de contraseña
  desde la interfaz. Cada despliegue reaplica las contraseñas definidas en el entorno.
- **Validación de credenciales:** `auth.service.js` compara primero la contraseña recibida con las variables
  `*_PASSWORD` y usa bcrypt como segunda verificación. Conviene eliminar esa comparación directa y verificar
  solo contra el hash almacenado.
- **Mensajes de validación en la interfaz:** el servidor devuelve `detalles` por campo, pero la interfaz solo
  muestra «Datos de entrada inválidos».
- **Monitoreo:** no hay monitoreo externo con alertas; la retención de logs depende de Render.
- **Auditoría:** la IP registrada se toma del primer valor de `X-Forwarded-For`.

## 17. Contribución

1. Haz un **fork** del repositorio en tu cuenta de GitHub.
2. Clónalo y añade el repositorio original como `upstream`:

   ```bash
   git clone https://github.com/<TU_USUARIO>/plataforma-csirt-incidentes.git
   cd plataforma-csirt-incidentes
   git remote add upstream https://github.com/studywithlegacy-cmyk/plataforma-csirt-incidentes.git
   ```

3. Crea una rama por cambio a partir de `main` actualizado:

   ```bash
   git fetch upstream
   git checkout -b mi-cambio upstream/main
   ```

4. Haz commits descriptivos (`feat:`, `fix:`, `docs:`, `style:`) y ejecuta `npm test` antes de subir.
5. Sube la rama a tu fork y abre un **Pull Request** hacia `main` del repositorio original:

   ```bash
   git push -u origin mi-cambio
   ```

6. Tras la revisión y el merge, sincroniza tu fork:

   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   ```

---

**Versión del documento:** 1.0 · **Versión de la plataforma:** commit `644634e` · **Fecha:** 25 de octubre de 2026

**Mantenedores:** [@studywithlegacy-cmyk](https://github.com/studywithlegacy-cmyk) ·
[@CescJavier7](https://github.com/CescJavier7)
