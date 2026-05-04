# Actividad: Despliegue de PagoYa API en Render

**Elaborado por:** Henry Antonio Mendoza Puerta

## Objetivo

Llevar la API de **PagoYa** desde tu máquina local hasta una URL pública en internet, usando **Render** como plataforma de hosting y **PostgreSQL** como base de datos administrada. Al terminar, cualquier persona con la URL podrá hacer `register`, `login`, consultar Swagger y operar el sistema desde cualquier red.

Lo que vas a practicar:

- **Render**: crear un Web Service y una base de datos administrada, configurar variables de entorno y revisar logs.
- **Dockerfile multi-stage**: construir el JAR dentro del contenedor para no depender de tu máquina.
- **Perfiles de Spring Boot**: separar configuración de `local` y `prod`.
- **Variables de entorno en producción**: secretos (JWT, password de BD) fuera del repo.
- **Puertos dinámicos**: que la app escuche en el `PORT` que Render asigna en runtime.

## ⚠️ Prerrequisito obligatorio

> **Esta guía es la continuación directa de la guía de seguridad.**
> Vas a tomar el código tal como quedó después de aplicar la **`GUIA-SECURITY-JWT.md`** (con login, JWT, refresh tokens, registro atómico) y lo vas a desplegar en Render.
> **Si no completaste la guía anterior, no continúes**: aquí asumimos que ya tienes el `pom.xml` con `spring-boot-starter-security` + `jjwt`, los endpoints `/api/auth/{register,login,refresh,logout}` funcionando en local y el `application-prod.yml` ya creado.

Antes de empezar, **revisa de manera obligatoria** estos materiales (en este orden):

1. <a href="https://github.com/hampcodes/bookstore/blob/main/docs/GUIA-SECURITY-JWT.md" target="_blank" rel="noopener noreferrer"><code>GUIA-SECURITY-JWT.md</code></a> — la guía previa que cierra los endpoints con Spring Security + JWT. Es la base de lo que vas a desplegar.
2. **Repo en GitHub**: tu proyecto **debe** estar en un repositorio en GitHub (público o privado, ambos funcionan en Render). Render despliega leyendo de Git, no de tu disco local.

Si saltaste alguno, **no continúes**. Vuelve, complétalos y luego retoma esta guía.

## Indice

- [1. Enunciado](#1-enunciado)
- [2. Arquitectura del despliegue](#2-arquitectura-del-despliegue)
- [3. Prerrequisitos de cuentas](#3-prerrequisitos-de-cuentas)
- [4. Plan de variables de entorno en producción](#4-plan-de-variables-de-entorno-en-producción)
- [5. Crear la rama](#5-crear-la-rama)
- [6. Adaptar el `Dockerfile` (multi-stage)](#6-adaptar-el-dockerfile-multi-stage)
- [7. Crear `.dockerignore`](#7-crear-dockerignore)
- [8. Ajustar `application-prod.yml` y `application.yml`](#8-ajustar-application-prodyml-y-applicationyml)
- [9. Commit y push de los cambios](#9-commit-y-push-de-los-cambios)
- [10. Crear la base de datos PostgreSQL en Render](#10-crear-la-base-de-datos-postgresql-en-render)
- [11. Crear el Web Service en Render](#11-crear-el-web-service-en-render)
- [12. Configurar variables de entorno del Web Service](#12-configurar-variables-de-entorno-del-web-service)
- [13. Primer deploy y revisión de logs](#13-primer-deploy-y-revisión-de-logs)
- [14. Probar la API en producción](#14-probar-la-api-en-producción)
- [15. Conectar pgAdmin a la BD de Render](#15-conectar-pgadmin-a-la-bd-de-render)
- [16. Redeploys automáticos](#16-redeploys-automáticos)
- [17. Troubleshooting común](#17-troubleshooting-común)
- [18. Pull Request](#18-pull-request)

---

## 1. Enunciado

PagoYa funciona perfecto en `localhost:8080`, pero un sistema que sólo corre en tu máquina **no le sirve a nadie**. Vamos a publicar la API en internet con una URL del estilo `https://pagoya-api.onrender.com`.

### 1.1 Lo que vamos a hacer

1. Reescribir el **`Dockerfile`** para que **construya el JAR dentro del contenedor** (multi-stage), sin depender de que el JAR ya esté compilado en `target/`.
2. Hacer que la app escuche en el **puerto que Render le asigne** (`PORT`), no en `8080` fijo.
3. Completar `application-prod.yml` con el bloque de JWT que falta.
4. Subir el repo a **GitHub**.
5. Crear una **base de datos PostgreSQL administrada** en Render.
6. Crear un **Web Service** en Render apuntado al repo, que use el `Dockerfile`.
7. Configurar las variables de entorno (`DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, etc.) en el panel de Render.
8. Verificar que **Swagger UI** abre desde la URL pública y probar el flujo `register → login → /me` desde Postman.

### 1.2 Por qué un Dockerfile multi-stage

El `Dockerfile` actual del repo es así:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
COPY target/pagoya-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Este Dockerfile **asume que tú ya corriste `mvn package` localmente** y que el JAR está en `target/`. Render no tiene tu `target/` — sólo tiene el código fuente del repo. Si lo dejas así, el build en Render falla con `COPY failed: file does not exist`.

La solución es un **build multi-stage**: una primera etapa con Maven que compila el JAR, y una segunda etapa con sólo el JRE que ejecuta el JAR. Así el build es 100% reproducible desde el código fuente y la imagen final queda mucho más liviana.

[↑ Volver al indice](#indice)

---

## 2. Arquitectura del despliegue

Vas a crear **dos recursos** en Render que viven en la misma región y se ven por la red interna:

```
                           Internet (HTTPS)
                                  │
                                  ▼
                ┌─────────────────────────────────┐
                │      Render Web Service         │
                │  pagoya-api.onrender.com        │
                │  ─────────────────────────      │
                │  Imagen Docker (Spring Boot)    │
                │  Variables de entorno:          │
                │   SPRING_PROFILES_ACTIVE=prod   │
                │   DB_URL, DB_USERNAME, ...      │
                │   JWT_SECRET, ...               │
                └────────────────┬────────────────┘
                                 │ red interna de Render
                                 ▼
                ┌─────────────────────────────────┐
                │   Render PostgreSQL             │
                │   ─────────────────────         │
                │   Hostname (internal)           │
                │   db: pagoya_db                 │
                │   user / password (autogen.)    │
                └─────────────────────────────────┘
```

**Reglas a seguir:**

- El **Web Service** y la **base de datos** deben estar en la **misma región** (ej: `Oregon`).
- El Web Service usa la **Internal Database URL** (sin salir a internet). La External URL es sólo para conectarte tú con pgAdmin desde tu PC.
- **Nunca commitees** las credenciales reales: viven en el panel de Render, nunca en `application-prod.yml`.

[↑ Volver al indice](#indice)

---

## 3. Prerrequisitos de cuentas

| Cuenta | Para qué | Cómo |
|---|---|---|
| **GitHub** | Hosting del código que Render va a clonar. | https://github.com/signup |
| **Render** | Hosting del API y la BD. | https://render.com/register — recomendado registrarse con la cuenta de GitHub para autorizar el acceso al repo en un solo paso. |

**Sube tu repo a GitHub** si todavía no lo hiciste:

```bash
# desde la raíz del proyecto pagoya-api/
git remote add origin https://github.com/<tu-usuario>/pagoya-api.git
git push -u origin main
git push -u origin develop
```

> ⚠️ Asegúrate de que tu `.gitignore` incluye `.env` (la guía anterior ya lo tiene). El `.env` con secretos **jamás** debe ir al repo.

[↑ Volver al indice](#indice)

---

## 4. Plan de variables de entorno en producción

Producción va a usar **un set distinto** de variables. No mezclamos las de local. Esta es la lista completa que vas a configurar en Render:

| Variable | Valor en Render | De dónde sale |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | `prod` | Fija. Activa `application-prod.yml`. |
| `PORT` | (lo asigna Render) | **No la pongas tú**. Render la inyecta automáticamente y la app debe leerla. |
| `DB_URL` | `jdbc:postgresql://<internal-host>/<db-name>` | Construida con datos del paso [10](#10-crear-la-base-de-datos-postgresql-en-render). |
| `DB_USERNAME` | (autogenerado por Render) | Lo verás en la pantalla de la BD. |
| `DB_PASSWORD` | (autogenerado por Render) | Idem. |
| `JWT_SECRET` | string aleatorio ≥ 32 caracteres | Genéralo con `openssl rand -base64 64` y **NO** reutilices el de local. |
| `JWT_EXPIRATION_MS` | `900000` | 15 min de access token (igual que en local). |
| `JWT_REFRESH_EXPIRATION_MS` | `604800000` | 7 días de refresh token. |

> 🔐 **El `JWT_SECRET` de producción es distinto del de local**. Si usas el mismo, un access token emitido en local valdría en producción y al revés. **Genera uno nuevo**:
>
> ```bash
> openssl rand -base64 64
> ```

[↑ Volver al indice](#indice)

---

## 5. Crear la rama

Toda feature parte de `develop`:

```bash
git checkout develop
git pull origin develop
git checkout -b feature/deploy-render
```

[↑ Volver al indice](#indice)

---

## 6. Adaptar el `Dockerfile` (multi-stage)

Reemplaza **todo** el contenido de `pagoya-api/Dockerfile` por:

```dockerfile
# =========================================================
#  Stage 1: build — compila el JAR con Maven y JDK 21
# =========================================================
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace

# Cache de dependencias: copiamos solo el pom primero
COPY pom.xml .
RUN mvn -B -q dependency:go-offline

# Ahora sí, copiamos el código y empaquetamos
COPY src ./src
RUN mvn -B -q clean package -DskipTests

# =========================================================
#  Stage 2: runtime — sólo JRE 21, imagen final liviana
# =========================================================
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# ⚠️ REEMPLAZA "pagoya-0.0.1-SNAPSHOT.jar" por el nombre del JAR
# que genera TU proyecto. Ese nombre se arma con los valores
# <artifactId> y <version> de tu pom.xml:
#
#     target/<artifactId>-<version>.jar
#
# Ejemplos:
#   - artifactId=pagoya, version=0.0.1-SNAPSHOT  → pagoya-0.0.1-SNAPSHOT.jar
#   - artifactId=bookstore, version=1.0.0        → bookstore-1.0.0.jar
#
# Si no estás seguro, corre `mvn clean package` localmente y mira
# qué archivo .jar aparece dentro de la carpeta target/.
COPY --from=build /workspace/target/pagoya-0.0.1-SNAPSHOT.jar app.jar

# Render asigna el puerto en la variable PORT en runtime;
# EXPOSE es informativo, no fija el puerto real.
EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Por qué cada cosa:**

| Línea | Para qué |
|---|---|
| `FROM maven:3.9-eclipse-temurin-21 AS build` | Imagen con Maven + JDK 21 sólo para compilar. Etiquetada `build` para referenciarla luego. |
| `COPY pom.xml .` + `dependency:go-offline` | Truco clásico: si el `pom.xml` no cambia, esta capa se cachea y Maven no vuelve a bajar deps. |
| `mvn ... -DskipTests` | En el build de Render saltamos los tests. Si quieres que el build falle ante un test roto, quita `-DskipTests`. |
| `FROM eclipse-temurin:21-jre-alpine` | Imagen final con sólo el **JRE** (no el JDK). Mucho más liviana. |
| `COPY --from=build ...` | Saca el JAR de la etapa anterior y lo deja en la imagen final. Maven y el código fuente NO quedan en producción. |

[↑ Volver al indice](#indice)

---

## 7. Crear `.dockerignore`

Crea un nuevo archivo `pagoya-api/.dockerignore` con:

```
target/
.idea/
.vscode/
.DS_Store
*.iml
.env
.env.*
.git/
.gitignore
README.md
HELP.md
```

**Por qué importa**: sin `.dockerignore`, todo lo que esté en la carpeta del proyecto se copia al contenedor (incluido tu `.env` local con secretos). Esto:

1. **Filtra secretos**: tu `.env` local NUNCA llega al contenedor.
2. **Acelera el build**: no copia el `target/` viejo, ni `.git/`, ni `.idea/`.

[↑ Volver al indice](#indice)

---

## 8. Ajustar `application-prod.yml` y `application.yml`

### 8.1 Puerto dinámico en `application.yml`

Render asigna el puerto en una variable de entorno `PORT` y el contenedor **debe** escuchar ahí. Reemplaza el contenido de `src/main/resources/application.yml` por:

```yaml
server:
  port: ${PORT:8080}

spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}
  application:
    name: pagoya
```

`${PORT:8080}` significa: usá la variable de entorno `PORT` si existe; si no, usá `8080`. Así local sigue funcionando en `8080` sin cambios.

### 8.2 Completar `application-prod.yml`

El `application-prod.yml` actual sólo tiene la conexión a BD. Le falta el bloque de JWT. Reemplaza el contenido completo por:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

pagoya:
  security:
    jwt:
      secret: ${JWT_SECRET}
      expiration-ms: ${JWT_EXPIRATION_MS:900000}
      refresh-expiration-ms: ${JWT_REFRESH_EXPIRATION_MS:604800000}
```

**Diferencias clave con `application-local.yml`:**

| Punto | local | prod |
|---|---|---|
| `show-sql` | `true` | `false` (no spamear logs) |
| `JWT_SECRET` | con default visible | **sin default** — si no está, la app falla al arrancar (queremos eso) |

### 8.3 Verificar que `application-local.yml` no cambió

`application-local.yml` se queda **igual** (con su default visible para `JWT_SECRET`). Esto te permite seguir levantando la app en local con `mvn spring-boot:run` sin necesidad de exportar el secret cada vez.

[↑ Volver al indice](#indice)

---

## 9. Commit y push de los cambios

```bash
git add Dockerfile \
        .dockerignore \
        src/main/resources/application.yml \
        src/main/resources/application-prod.yml
git commit -m "chore(deploy): dockerfile multistage + perfil prod para Render"
git push -u origin feature/deploy-render
```

> Mergea esta rama a `develop` y luego a `main` cuando hayas validado el deploy. Render por defecto va a desplegar desde `main`.

[↑ Volver al indice](#indice)

---

## 10. Crear la base de datos PostgreSQL en Render

1. En el dashboard de Render: **`New +`** → **`PostgreSQL`**.
2. Llena el formulario:

   | Campo | Valor |
   |---|---|
   | **Name** | `pagoya-db` |
   | **Database** | `pagoya_db` |
   | **User** | `pagoya_user` (puedes dejar el autogenerado) |
   | **Region** | `Oregon (US West)` — *anótala, debe coincidir con la del Web Service* |
   | **PostgreSQL Version** | `16` |
   | **Plan** | `Free` |

3. Click **`Create Database`**. Espera 1–2 minutos a que el status pase a `Available`.

4. En la página de la BD, baja a la sección **`Connections`** y copia los siguientes datos a un bloc de notas temporal:

   | Campo | Lo vas a usar para |
   |---|---|
   | **Hostname** (internal) | armar el `DB_URL` |
   | **Port** | siempre `5432` |
   | **Database** | nombre de la base (`pagoya_db`) |
   | **Username** | `DB_USERNAME` |
   | **Password** | `DB_PASSWORD` |
   | **External Database URL** | sólo para conectar pgAdmin desde tu PC |

5. Construí el `DB_URL` con formato JDBC:

   ```
   jdbc:postgresql://<internal-hostname>/<database-name>
   ```

   Ejemplo:

   ```
   jdbc:postgresql://dpg-abc123-a.oregon-postgres.render.com/pagoya_db
   ```

   > ⚠️ **No uses la `Internal Database URL` tal cual**. Esa empieza con `postgresql://` (el formato libpq). Spring Boot espera el prefijo `jdbc:postgresql://` y user/password como variables separadas.

[↑ Volver al indice](#indice)

---

## 11. Crear el Web Service en Render

1. En el dashboard: **`New +`** → **`Web Service`**.
2. Conectar el repo: si te registraste con GitHub, listas tus repos directo. Si no, agrega la integración `GitHub` y autoriza el acceso al repo `pagoya-api`.
3. Selecciona el repo `pagoya-api` y click **`Connect`**.
4. Llena el formulario:

   | Campo | Valor |
   |---|---|
   | **Name** | `pagoya-api` |
   | **Region** | `Oregon (US West)` — **misma región que la BD** |
   | **Branch** | `main` |
   | **Root Directory** | `pagoya-api` *(si tu repo tiene la API en una subcarpeta. Si la API es la raíz del repo, déjalo vacío.)* |
   | **Runtime / Language** | `Docker` (lo detecta automáticamente al ver el `Dockerfile`) |
   | **Plan** | `Free` |

5. **No toques `Build Command` ni `Start Command`**: como usamos `Dockerfile`, Render ejecuta el `ENTRYPOINT` directamente.

6. **No clickees `Create Web Service` todavía**. Antes, baja a la sección **`Environment Variables`** del mismo formulario para configurarlas (paso siguiente).

[↑ Volver al indice](#indice)

---

## 12. Configurar variables de entorno del Web Service

En la sección **`Environment Variables`** del formulario, agrega **una por una** estas variables:

| Key | Value |
|---|---|
| `SPRING_PROFILES_ACTIVE` | `prod` |
| `DB_URL` | `jdbc:postgresql://<internal-host>/pagoya_db` (el que armaste en el paso 10) |
| `DB_USERNAME` | (el del paso 10) |
| `DB_PASSWORD` | (el del paso 10) |
| `JWT_SECRET` | resultado de `openssl rand -base64 64` (≥ 32 chars) |
| `JWT_EXPIRATION_MS` | `900000` |
| `JWT_REFRESH_EXPIRATION_MS` | `604800000` |

> 🔑 **`PORT` no la configures vos**. Render la inyecta automáticamente.

> 🔒 Render enmascara los valores luego de guardar (`••••••`). Si alguna vez necesitas verlos, debes rotarlos.

Ahora sí: click **`Create Web Service`**. Render arranca el primer build automáticamente.

[↑ Volver al indice](#indice)

---

## 13. Primer deploy y revisión de logs

El primer build tarda **5–10 minutos** (Maven baja todas las dependencias por primera vez). En la pestaña **`Logs`** del servicio vas a ver, secuencialmente:

1. **Build phase** (Docker)
   ```
   ==> Building image from Dockerfile
   Step 1/9 : FROM maven:3.9-eclipse-temurin-21 AS build
   ...
   Successfully tagged pagoya-api:latest
   ```

2. **Deploy phase** (arranque del contenedor)
   ```
   Starting service with command 'java -jar app.jar'
   ```

3. **Spring Boot startup**
   ```
   Tomcat initialized with port 10000   ← el PORT que asignó Render
   Started PagoyaApplication in 12.3 seconds
   ```

4. **Servicio Live**
   ```
   ==> Your service is live 🎉
   ```

Apenas veas el `🎉`, el header de la página muestra la URL pública:

```
https://pagoya-api.onrender.com
```

> Si en el log ves `caused by: java.net.ConnectException: Connection refused` al conectarse a la BD, la causa más común es que copiaste la **External** URL en lugar de armar el `jdbc:postgresql://<internal-host>/db`. Volvé al paso 10.

[↑ Volver al indice](#indice)

---

## 14. Probar la API en producción

### 14.1 Smoke test con Swagger

Abre en tu navegador:

```
https://pagoya-api.onrender.com/swagger-ui/index.html
```

Si carga el UI con todos tus endpoints, **el deploy funciona**. Probá un endpoint público (ej: `POST /api/auth/register`) directo desde Swagger con el botón **Try it out**.

### 14.2 Smoke test con `curl`

```bash
# Registro
curl -X POST https://pagoya-api.onrender.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "ana@pagoya.com",
    "password": "secret123",
    "fullName": "Ana Lopez",
    "dni": "12345678",
    "phone": "987654321"
  }'

# Login
curl -X POST https://pagoya-api.onrender.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "ana@pagoya.com", "password": "secret123" }'

# /me con el accessToken devuelto en login
curl https://pagoya-api.onrender.com/api/customers/me \
  -H "Authorization: Bearer <ACCESS_TOKEN>"
```

### 14.3 Postman

1. Abre tu colección `pagoya-api.postman_collection.json`.
2. Edita la variable `base_url` de `http://localhost:8080` a `https://pagoya-api.onrender.com`.
3. Corre el flujo `register → login → /me → refresh → logout`.

[↑ Volver al indice](#indice)

---

## 15. Conectar pgAdmin a la BD de Render

Para inspeccionar la data en producción (ver qué `users` se registraron, qué `refresh_tokens` están activos, qué `customers` existen) podés conectar tu pgAdmin **local** directo a la BD de Render. Útil para debuggear, verificar que JPA creó las tablas y correr `SELECT` manuales.

> ⚠️ Recordá: lo que ves desde pgAdmin **es la BD de producción**. Hacer `DELETE` o `UPDATE` aquí afecta a los usuarios reales del sistema. Para inspeccionar está perfecto; para modificar, mejor desde la API.

### 15.1 Conseguir los datos de conexión externa

En el dashboard de Render, abre tu BD `pagoya-db` y baja a la sección **`Connections`**. **Esta vez vas a usar los datos `External`** (la `Internal URL` sólo funciona desde dentro de Render):

| Campo en pgAdmin | De dónde sale en Render |
|---|---|
| **Host name/address** | `Hostname` (External) — termina con `.oregon-postgres.render.com` |
| **Port** | `Port` — siempre `5432` |
| **Maintenance database** | `Database` — `pagoya_db` |
| **Username** | `Username` |
| **Password** | `Password` |

> 🔑 Si copiás la `External Database URL` completa, su formato es:
>
> ```
> postgresql://<user>:<password>@<external-host>/<database>
> ```
>
> Cada parte de la URL corresponde a un campo de pgAdmin. **No** pegues esa URL completa en un solo campo: pgAdmin pide los datos por separado.

### 15.2 Registrar el servidor en pgAdmin

Esto funciona igual con el **pgAdmin Desktop** (instalado en tu PC) o con el **pgAdmin del `compose.yml`** (en `http://localhost:8082` con `admin@pagoya.com` / `admin`).

1. En el árbol de la izquierda (`Object Explorer`), click derecho en **`Servers`** → **`Register`** → **`Server...`**.

2. Pestaña **`General`**:

   | Campo | Valor |
   |---|---|
   | **Name** | `PagoYa Render` *(el label que vos quieras)* |

3. Pestaña **`Connection`**:

   | Campo | Valor |
   |---|---|
   | **Host name/address** | el `Hostname` (External) de Render |
   | **Port** | `5432` |
   | **Maintenance database** | `pagoya_db` |
   | **Username** | el `Username` de Render |
   | **Password** | el `Password` de Render |
   | **Save password?** | ✅ (marca para no tener que reingresarlo) |

4. Pestaña **`Parameters`** (en pgAdmin 4) — agregá un parámetro:

   | Name | Value |
   |---|---|
   | **SSL mode** | `Require` |

   > Render exige SSL para conexiones externas. Si dejás `Prefer` o `Disable`, la conexión falla con `FATAL: SSL connection is required`.

5. Click **`Save`**. Si los datos están bien, pgAdmin se conecta y vas a ver el nuevo server en el árbol.

### 15.3 Explorar las tablas

En el árbol:

```
Servers
└── PagoYa Render
    └── Databases
        └── pagoya_db
            └── Schemas
                └── public
                    └── Tables
                        ├── users
                        ├── customers
                        ├── roles
                        ├── refresh_tokens
                        ├── accounts
                        ├── transfers
                        └── ...
```

Si las tablas no aparecen, refrescá con click derecho → **`Refresh`**. Aparecen apenas la app arranca por primera vez (porque `ddl-auto: update` las crea al inicio).

**Ver el contenido de una tabla:**

Click derecho en la tabla → **`View/Edit Data`** → **`All Rows`**.

**Correr una consulta SQL:**

Menú **`Tools`** → **`Query Tool`**. Algunas consultas útiles:

```sql
-- Usuarios registrados (sin mostrar el hash de password)
SELECT id, email, verified, role_id
FROM users
ORDER BY id DESC;

-- Customers con su email del User
SELECT c.id, c.full_name, c.dni, c.phone, u.email
FROM customers c
JOIN users u ON u.id = c.user_id
ORDER BY c.id DESC;

-- Refresh tokens activos (no revocados ni expirados)
SELECT user_id, token, expires_at, revoked
FROM refresh_tokens
WHERE revoked = false
  AND expires_at > NOW();

-- Cuántas cuentas hay por estado
SELECT status, COUNT(*) FROM accounts GROUP BY status;
```

### 15.4 Errores comunes al conectar

| Síntoma | Causa | Solución |
|---|---|---|
| `could not translate host name` | Usaste el `Internal Hostname` (sólo funciona dentro de Render). | Copiá el **External Hostname** (con `.oregon-postgres.render.com`). |
| `FATAL: SSL connection is required` | Falta el parámetro `SSL mode = Require`. | Agregalo en la pestaña `Parameters` del server. |
| `password authentication failed` | Username o password mal copiados. | Revisá los datos en el panel de la BD en Render. |
| `connection timed out` | Algún firewall corporativo o VPN bloquea el 5432. | Probá desde otra red (datos móviles, por ejemplo). |

[↑ Volver al indice](#indice)

---

## 16. Redeploys automáticos

Por defecto, Render escucha la branch que configuraste (`main`). Cada vez que mergees un PR a `main`, Render automáticamente lanza un nuevo build con el `Dockerfile` y reemplaza el contenedor en ejecución.

**Flujo recomendado:**

```bash
# desarrollo en feature/<x> → develop → main
git checkout feature/nueva-funcionalidad
# ... commits ...
git push -u origin feature/nueva-funcionalidad
# PR a develop, code review, merge

# Cuando develop esté estable y querás liberar:
git checkout main && git pull
git merge --no-ff develop
git push origin main         # ← este push dispara el deploy en Render
```

[↑ Volver al indice](#indice)

---

## 17. Troubleshooting común

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Build failed: COPY failed: file not found` | Sigues con el `Dockerfile` viejo que copia `target/`. | Reemplaza por el multi-stage del paso 6. |
| `Build failed: ... pagoya-0.0.1-SNAPSHOT.jar: not found` | El nombre del JAR en el `COPY --from=build` no coincide con el de tu `pom.xml`. | Ajusta el nombre siguiendo el comentario del Dockerfile (`<artifactId>-<version>.jar`). |
| `WeakKeyException: The signing key's size is X bits, must be ≥256` | El `JWT_SECRET` es muy corto (< 32 chars). | Genera uno nuevo: `openssl rand -base64 64`. |
| `psql: FATAL: password authentication failed` | Copiaste user/password mal. | Vuelve al panel de la BD y copia los valores otra vez. |
| Cold start de 30–60 seg en la primera request | Plan Free duerme tras 15 min de inactividad. | Es esperado. Para evitarlo: plan Starter. |
| `Connection refused` al arrancar | El `DB_URL` apunta al hostname **externo** o no es JDBC. | Usa el internal hostname con prefijo `jdbc:postgresql://`. |
| Build OK pero el contenedor no arranca y muere en loop | App escucha en `8080` fijo, no en `PORT`. | Confirma `server.port: ${PORT:8080}` en `application.yml`. |

### Cómo leer logs en vivo

Dashboard del servicio → pestaña **`Logs`** → activa el toggle **`Tail`**. Puedes filtrar por nivel (`Error`, `Warn`, `Info`) y por palabra clave.

[↑ Volver al indice](#indice)

---

## 18. Pull Request

Pull Request: `feature/deploy-render` → `develop`.

**Título sugerido:**

```
chore(deploy): publicacion en Render con Dockerfile multistage y perfil prod
```

**Descripción sugerida:**

```markdown
## Que entrega

Despliegue de PagoYa API en Render con Postgres administrado:
- URL publica: https://pagoya-api.onrender.com
- Swagger:    https://pagoya-api.onrender.com/swagger-ui/index.html

## Cambios en el repo

- Dockerfile multi-stage (build con Maven+JDK 21, runtime con JRE 21 alpine).
- .dockerignore para excluir target/, .env, .git/ del contexto de build.
- application.yml: server.port leido de la variable PORT (que Render asigna).
- application-prod.yml: bloque pagoya.security.jwt con JWT_SECRET sin default.

## Infra creada en Render

- PostgreSQL 16, plan Free, region Oregon (US West).
- Web Service "pagoya-api", plan Free, region Oregon, runtime Docker, branch main.
- 7 variables de entorno configuradas (ver paso 12 de la guia).

## Como probarlo

- Abrir https://pagoya-api.onrender.com/swagger-ui/index.html
- POST /api/auth/register -> 200 OK con userId, customerId, role.
- POST /api/auth/login -> 200 OK con accessToken + refreshToken.
- GET /api/customers/me con Authorization: Bearer <accessToken> -> 200 OK.
- POST /api/auth/refresh con el refreshToken -> 200 OK con nuevos tokens.
- POST /api/auth/logout con el refreshToken -> 204 No Content.
- Cualquier endpoint de negocio sin Authorization -> 401 Unauthorized.

## Limitaciones conocidas

- Plan Free de Render duerme el contenedor tras 15 min sin trafico (cold start ~30-60s en la primera request despues).
```

[↑ Volver al indice](#indice)

---


[↑ Volver al indice](#indice)
