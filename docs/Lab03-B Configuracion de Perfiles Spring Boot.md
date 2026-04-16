# Lab 03-B: Configuracion de Perfiles Spring Boot (Local con Docker y Produccion con Supabase)

## Objetivo

Configurar la aplicacion Bookstore API para trabajar con dos entornos de base de datos PostgreSQL mediante perfiles de Spring Boot:

- **local** — PostgreSQL ejecutandose en un contenedor Docker (ya existente).
- **prod** — PostgreSQL gestionado por Supabase en la nube.

---

## Prerequisitos

- Java 21 instalado
- Contenedor Docker de PostgreSQL ya corriendo en el puerto 5434
- Cuenta en [Supabase](https://supabase.com)
- Maven Wrapper incluido en el proyecto (`./mvnw`)

---

## Paso 1: Crear el proyecto en Supabase

### 1.1 — Crear un nuevo proyecto

Ingresar a [supabase.com](https://supabase.com), iniciar sesion y hacer clic en **New Project**. Completar el formulario con los siguientes datos:

| Campo | Valor |
|-------|-------|
| **Organization** | `hampcode` (o la organizacion que tengas configurada) |
| **Project name** | `bookstore` |
| **Database password** | Ingresar una contraseña segura. **Guardarla**, se usara en el archivo `.env` |
| **Region** | `Americas` (West US - Oregon) para menor latencia |
| **Security** | Dejar marcado `Enable Data API`. No marcar `Enable automatic RLS` |

Hacer clic en **Create new project** y esperar a que Supabase termine de aprovisionar la base de datos.

<img src="img/01createproject.png" alt="Crear proyecto en Supabase" width="500">

### 1.2 — Verificar que el proyecto este activo

Una vez creado, se mostrara el dashboard del proyecto. Verificar que el **Status** sea **Healthy** y que la base de datos primaria este en la region seleccionada (West US - Oregon).

<img src="img/02dashboard.png" alt="Dashboard del proyecto" width="500">

### 1.3 — Obtener las credenciales de conexion

En el dashboard, hacer clic en el boton **Connect**. Se abrira el panel de conexion:

1. Seleccionar la opcion **Direct** (Connection string).
2. En **Connection Method**, se muestran tres opciones disponibles:

<img src="img/05tipoconect.png" alt="Tipos de conexion disponibles" width="500">

3. Seleccionar **Session pooler** como se muestra a continuacion:

<img src="img/03conect.png" alt="Session pooler seleccionado" width="500">

4. En **Type**, seleccionar **URI**.

### 1.4 — Tipos de conexion en Supabase

Supabase ofrece tres metodos de conexion. Es importante entender las diferencias para elegir el correcto:

**Direct connection**
- Conexion directa al servidor PostgreSQL sin intermediarios.
- Ideal para aplicaciones con conexiones persistentes y de larga duracion (maquinas virtuales, contenedores).
- Usa el host `db.xxx.supabase.co` en el puerto `5432`.
- Problema: puede usar **IPv6**, lo que no funciona en todas las redes. Si tu red es solo IPv4, la conexion dara timeout.

**Transaction pooler**
- Usa un proxy (PgBouncer) que reutiliza conexiones a nivel de transaccion.
- Ideal para aplicaciones serverless (AWS Lambda, Vercel Functions) donde cada interaccion con la base de datos es breve y aislada.
- Usa el host `pooler.supabase.com` en el puerto `6543`.
- Es IPv4 compatible, pero **no soporta sentencias DDL** (`CREATE TABLE`, `ALTER TABLE`), por lo que `ddl-auto: update` de Hibernate falla.

**Session pooler**
- Usa el mismo proxy (PgBouncer) pero mantiene la sesion completa del cliente.
- Es la alternativa a Direct Connection cuando se conecta via una red IPv4.
- Usa el host `pooler.supabase.com` en el puerto `5432`.
- Es **IPv4 compatible** y **soporta DDL**, por lo que funciona correctamente con `ddl-auto: update` de Hibernate.
- **Este es el metodo que usamos en este proyecto.**

| Metodo | Host | Puerto | IPv4 | Soporta DDL | Usar cuando |
|--------|------|--------|------|-------------|-------------|
| Direct connection | `db.xxx.supabase.co` | 5432 | No | Si | Tu red soporta IPv6 |
| Transaction pooler | `pooler.supabase.com` | 6543 | Si | No | Apps serverless (Lambda, Vercel) |
| **Session pooler** | `pooler.supabase.com` | **5432** | **Si** | **Si** | **Spring Boot + Hibernate (este proyecto)** |

### 1.5 — Copiar los datos de conexion

Una vez seleccionado **Session pooler**, se muestran los datos de conexion:

<img src="img/04conect.png" alt="Datos de conexion" width="500">

Los datos que aparecen son:

| Campo | Valor |
|-------|-------|
| **Connection string** | `postgresql://postgres.lflbtyecztaabchkxjhs:[YOUR-PASSWORD]@aws-1-us-west-2.pooler.supabase.com:5432/postgres` |
| **host** | `aws-1-us-west-2.pooler.supabase.com` |
| **port** | `5432` |
| **database** | `postgres` |
| **user** | `postgres.lflbtyecztaabchkxjhs` |

Estos valores se usaran para construir la URL JDBC en el archivo `.env`.

> **Importante:** Agregar siempre `?sslmode=require` al final de la URL, ya que Supabase requiere conexiones SSL.

---

## Paso 2: Crear el archivo `.env`

En la raiz del proyecto, crear el archivo `.env` con las credenciales obtenidas de Supabase. Convertir la URL de PostgreSQL a formato JDBC:

```env
SUPABASE_DB_URL=jdbc:postgresql://aws-1-us-west-2.pooler.supabase.com:5432/postgres?sslmode=require
SUPABASE_DB_USER=postgres.lflbtyecztaabchkxjhs
SUPABASE_DB_PASSWORD=tu-password-de-supabase
```

> Reemplazar `tu-password-de-supabase` con la contraseña real que ingresaste al crear el proyecto.

---

## Paso 3: Agregar `.env` y `.DS_Store` al `.gitignore`

Verificar que el archivo `.gitignore` contenga estas entradas para no subir credenciales ni archivos del sistema al repositorio:

```
.env
.DS_Store
```

---

## Paso 4: Configurar `application.yaml` (propiedades compartidas)

El archivo `src/main/resources/application.yaml` contiene **solo las propiedades comunes** a todos los perfiles. No incluye datasource ni configuracion de JPA, ya que cada perfil define los suyos de forma independiente:

```yaml
spring:
  application:
    name: bookstore-api
  profiles:
    active: local

server:
  port: 8080
```

> `spring.profiles.active: local` indica que, si no se especifica otro perfil, se usara el perfil **local**. Cada perfil (`application-local.yaml` y `application-prod.yaml`) define su propia conexion a base de datos de forma completa e independiente, evitando redundancias y haciendo explicita la configuracion de cada entorno.

---

## Paso 5: Crear `application-local.yaml` (perfil local con Docker)

Crear el archivo `src/main/resources/application-local.yaml` con la configuracion completa de conexion al PostgreSQL local en Docker:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5434/bookstore_db
    username: postgres
    password: adminadmin
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

| Propiedad | Valor | Motivo |
|-----------|-------|--------|
| `ddl-auto` | `create-drop` | Recrea las tablas en cada reinicio, ideal para desarrollo |
| `show-sql` | `true` | Muestra las consultas SQL en consola para depuracion |
| `port` | `5434` | Puerto del contenedor Docker ya existente |

---

## Paso 6: Crear `application-prod.yaml` (perfil produccion con Supabase)

Crear el archivo `src/main/resources/application-prod.yaml` con la configuracion completa de conexion a Supabase:

```yaml
spring:
  datasource:
    url: ${SUPABASE_DB_URL}
    username: ${SUPABASE_DB_USER}
    password: ${SUPABASE_DB_PASSWORD}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        format_sql: false
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

| Propiedad | Valor | Motivo |
|-----------|-------|--------|
| `ddl-auto` | `update` | Solo aplica cambios incrementales, no destruye datos existentes |
| `show-sql` | `false` | No mostrar SQL en produccion |
| Variables `${}` | Leidas desde `.env` | Las credenciales no se hardcodean en el codigo |

---

## Paso 7: Ejecutar la aplicacion y cambiar entre perfiles

### Opcion A: Via terminal (linea de comandos)

**Perfil local** (se usa por defecto):

```bash
./mvnw spring-boot:run
```

**Perfil produccion** (Supabase):

```bash
# Cargar las variables de entorno desde .env
export $(cat .env | xargs)

# Ejecutar con perfil prod
./mvnw spring-boot:run -Dspring-boot.run.profiles=prod
```

### Opcion B: Via IntelliJ IDEA

Para cambiar de perfil sin usar la terminal:

1. Ir a **Run** > **Edit Configurations...**
2. Seleccionar la configuracion de ejecucion de `BookstoreApiApplication` (o crear una nueva de tipo **Spring Boot**).
3. En el campo **Active profiles**, escribir el perfil deseado:
   - `local` para Docker
   - `prod` para Supabase
4. Para el perfil **prod**, tambien hay que configurar las variables de entorno:
   - En el campo **Environment variables**, hacer clic en el icono de carpeta.
   - Agregar las tres variables manualmente:
     - `SUPABASE_DB_URL` = `jdbc:postgresql://aws-1-us-west-2.pooler.supabase.com:5432/postgres?sslmode=require`
     - `SUPABASE_DB_USER` = `postgres.lflbtyecztaabchkxjhs`
     - `SUPABASE_DB_PASSWORD` = tu contraseña
   - O usar la opcion de importar desde archivo `.env` si esta disponible.
5. Hacer clic en **Apply** y luego **OK**.
6. Ejecutar la aplicacion con el boton **Run** (play verde).

> **Tip:** Puedes crear dos configuraciones de ejecucion separadas en IntelliJ (por ejemplo, `Bookstore - Local` y `Bookstore - Prod`) para cambiar entre perfiles con un solo clic, sin modificar nada cada vez.

---

## Paso 8: Verificar la conexion

Una vez que la aplicacion este levantada, probar con la coleccion de Postman incluida (`bookstore-api.postman_collection.json`) o directamente con:

```bash
curl http://localhost:8080/api/books
```

Si la respuesta devuelve un JSON con la lista de libros, la conexion a la base de datos es exitosa.

---

## Resumen de diferencias entre perfiles

| Caracteristica | Perfil `local` | Perfil `prod` |
|----------------|----------------|---------------|
| Base de datos | Docker PostgreSQL | Supabase (nube) |
| Host | `localhost:5434` | `aws-1-us-west-2.pooler.supabase.com:5432` |
| Metodo de conexion | Directa | Session pooler |
| SSL | No | Si (`sslmode=require`) |
| `ddl-auto` | `create-drop` | `update` |
| `show-sql` | `true` | `false` |
| Credenciales | Hardcodeadas en YAML | Variables de entorno (`.env`) |

---

## Estructura final de archivos

```
bookstore_lab03-B/
├── .env                                         # Credenciales de Supabase (no se sube a git)
├── .gitignore                                   # Incluye .env y .DS_Store
└── src/main/resources/
    ├── application.yaml                         # Config compartida (perfil activo: local)
    ├── application-local.yaml                   # Conexion completa a Docker PostgreSQL
    ├── application-prod.yaml                    # Conexion completa a Supabase
    └── data.sql                                 # Datos semilla
```
