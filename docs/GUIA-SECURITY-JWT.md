# Actividad: Spring Security + JWT en PagoYa

**Elaborado por:** Henry Antonio Mendoza Puerta

## Objetivo

Asegurar la API de PagoYa con **Spring Security**, autenticación stateless basada en **JWT** y **refresh tokens** para soportar logout real. Al terminar, todos los endpoints estarán protegidos, los usuarios podrán hacer `login` (recibe access + refresh), `refresh` (renovar el access sin reingresar credenciales) y `logout` (revoca el refresh en BD).

Lo que vas a practicar:

- **Spring Security**: configurar la cadena de filtros, endpoints públicos y protegidos.
- **JWT**: emitir, firmar y validar tokens (`HS256`) con la librería `jjwt`.
- **Refresh tokens**: invalidación real al cerrar sesión, sin Redis ni blacklists.
- **BCrypt**: hashear contraseñas en lugar de guardarlas en texto plano.
- **Autorización por roles**: distinguir entre `CUSTOMER`, `ADMIN` y `MERCHANT`.
- **Stateless API**: sin sesiones del lado del servidor.
- **GitFlow**: rama `feature/security-jwt` y Pull Request a `develop`.

## ⚠️ Prerrequisito obligatorio

> **Esta guía es la continuación directa del proyecto PagoYa API construido en las guías anteriores.**
> Vas a tomar el código tal como quedó en la **semana 05** y le vas a aplicar cambios encima.
> **Si no has leído ni completado las guías previas, vas a estar perdido**: aquí asumimos que ya existen los paquetes `auth/`, `customer/`, `account/`, `transfer/`, `shared/` y que la base de datos arranca con `docker compose`.

Antes de empezar, **revisa de manera obligatoria** estos materiales (en este orden):

1. <a href="https://github.com/hampcodes/bookstore/blob/main/docs/GUIA.md" target="_blank" rel="noopener noreferrer"><code>Actividad de aprendizaje (Semana 05)</code></a> — construye PagoYa API desde cero, buenas prácticas de API REST y todo el código base que aquí asumimos como existente.
2. **`Teoría 06 — Spring Security + JWT`**  — material teórico que acompaña a esta guía. Cubre los conceptos de Spring Security, anatomía del JWT, refresh tokens, configuración de duración y el flujo desde el frontend. **Léelo antes de implementar**: aquí aplicamos lo que ahí se explica.

Si saltaste alguno de esos materiales, **no continúes**. Vuelve, léelos y luego retoma esta guía.

## Indice

- [1. Enunciado](#1-enunciado)
- [2. Historias de usuario](#2-historias-de-usuario)
- [3. Reglas de seguridad](#3-reglas-de-seguridad)
- [4. Modelo del dominio (lo que ya existe)](#4-modelo-del-dominio-lo-que-ya-existe)
- [5. Punto de partida — qué hay y qué falta](#5-punto-de-partida--qué-hay-y-qué-falta)
- [6. Crear la rama](#6-crear-la-rama)
- [7. Estructura final del paquete `auth/` y `customer/`](#7-estructura-final-del-paquete-auth-y-customer)
- [8. Agregar dependencias en `pom.xml`](#8-agregar-dependencias-en-pomxml)
- [9. Variables de entorno (`.env`) y `application-local.yml`](#9-variables-de-entorno-env-y-application-localyml)
- [10. Registro atómico: `User` + `Customer` + BCrypt](#10-registro-atómico-user--customer--bcrypt)
- [11. Crear `JwtService`](#11-crear-jwtservice)
- [12. Crear `UserDetailsServiceImpl`](#12-crear-userdetailsserviceimpl)
- [13. Crear `JwtAuthenticationFilter`](#13-crear-jwtauthenticationfilter)
- [14. Crear `SecurityConfig`](#14-crear-securityconfig)
- [15. Modelo de Refresh Tokens (con rotation)](#15-modelo-de-refresh-tokens)
- [16. Endpoints `login`, `refresh`, `logout` y `logout-all`](#16-endpoints-login-refresh-logout)
- [17. Endpoints de perfil `GET /me` y `PUT /me`](#17-endpoints-de-perfil-me)
- [18. Documentar JWT en Swagger](#18-documentar-jwt-en-swagger)
- [19. Manejo de errores 401 / 403](#19-manejo-de-errores-401--403)
- [20. Autorización fina con `@PreAuthorize`](#20-autorización-fina-con-preauthorize)
- [21. Probar en Postman](#21-probar-en-postman)
- [22. Commit y Pull Request](#22-commit-y-pull-request)

---

## 1. Enunciado

Los endpoints de PagoYa están abiertos: cualquiera puede crear customers, mover plata, pagar recibos. Vamos a cerrar la puerta.

### 1.1 Lo que vamos a hacer

1. **Registro atómico**: `POST /api/auth/register` crea de un solo golpe el `User` (credenciales) + `Customer` (perfil) en una sola transacción.
2. Las contraseñas dejan de guardarse en texto plano y pasan a **BCrypt**.
3. Aparece `POST /api/auth/login` que devuelve un **access token (JWT)** y un **refresh token**.
4. Aparece `POST /api/auth/refresh` para renovar el access token sin reingresar credenciales.
5. Aparecen `POST /api/auth/logout` y `POST /api/auth/logout-all` para cerrar sesión de verdad.
6. Aparecen `GET /api/customers/me` y `PUT /api/customers/me` para que el usuario consulte y actualice su propio perfil sin pasar el `id` por URL.
7. Todos los endpoints de negocio quedan **protegidos**: el cliente debe enviar `Authorization: Bearer <accessToken>`.
8. Algunos endpoints (admin, datos del propio usuario) quedan restringidos por **rol** con `@PreAuthorize`.
9. Endpoints públicos (`/api/auth/{register,login,refresh,logout}`, Swagger) siguen abiertos.

### 1.2 Por qué dos tokens (access + refresh)

Un JWT por sí solo tiene un problema serio: **es stateless por diseño**. Una vez emitido, el servidor no guarda registro de él. Mientras la firma sea válida y `exp` no haya pasado, **funciona aunque el usuario haya cerrado sesión**. Eso significa que un logout "real" — invalidar el token en el momento — no es posible con JWT puro.

La solución estándar de la industria (Auth0, AWS Cognito, OAuth2) es usar **dos tokens trabajando en equipo**:

| Token | Tipo | Vida | Dónde vive | Para qué |
|---|---|---|---|---|
| **Access token** | JWT firmado (HS256) | **15 minutos** | En el cliente, viaja en cada request | Autenticar las llamadas a la API |
| **Refresh token** | UUID opaco | **7 días** | En el servidor (tabla `refresh_tokens`) y en el cliente | Pedir un nuevo access token cuando el actual expira |

**Cómo logramos un logout real con esto:**

- El access token vive sólo 15 minutos. Si se roba, el ladrón lo puede usar máximo 15 min.
- El refresh token vive en BD con un flag `revoked`. **Cuando el usuario hace logout, marcamos `revoked = true`**. A partir de ese momento ya no se puede usar para renovar.
- Combinado: el access expira solo en menos de 15 min y el refresh ya está revocado → sesión efectivamente cerrada.

**Flujo típico del cliente:**

```
1. Login                  → recibe { accessToken, refreshToken }
2. Llama endpoints        → header Authorization: Bearer <accessToken>
3. Pasan 15 min           → API responde 401 al siguiente request
4. POST /api/auth/refresh → con el refreshToken, recibe nuevo accessToken
5. Logout                 → POST /api/auth/logout con el refreshToken
```

Esta es la estrategia que vamos a implementar. **No es opcional**: refresh tokens son parte del flujo principal de la guía.

[↑ Volver al indice](#indice)

---

## 2. Historias de usuario

| Código | Historia |
|---|---|
| **US-S01** | Como **visitante** quiero **registrarme con mi email, password, nombre, DNI y teléfono** en una sola operación, para no tener que hacer varios pasos para empezar a usar PagoYa. |
| **US-S02** | Como **usuario registrado** quiero **iniciar sesión** con mi email y password para recibir un access token y un refresh token y operar la app. |
| **US-S03** | Como **usuario logueado** quiero que **mi sesión se renueve automáticamente** cada cierto tiempo (mediante refresh token) para no tener que reingresar credenciales mientras uso la app. |
| **US-S04** | Como **usuario logueado** quiero **cerrar sesión** y que el token quede invalidado de inmediato para que nadie más pueda usar mi cuenta desde ese dispositivo. |
| **US-S05** | Como **usuario logueado** quiero **cerrar sesión en todos mis dispositivos** de un solo paso, para reaccionar rápido si sospecho que mi cuenta fue comprometida. |
| **US-S06** | Como **usuario logueado** quiero **ver mi perfil** sin tener que recordar mi propio id, para revisar mis datos personales. |
| **US-S07** | Como **usuario logueado** quiero **actualizar mi nombre y teléfono** desde un endpoint propio (`/me`), sin poder modificar campos sensibles como DNI o email. |
| **US-S08** | Como **administrador** quiero **proteger ciertos endpoints por rol** (eliminar clientes, dar de alta proveedores) para que sólo personas autorizadas los ejecuten. |

[↑ Volver al indice](#indice)

---

## 3. Reglas de seguridad

| Código | Regla |
|---|---|
| **RN-S01** | Las contraseñas se guardan hasheadas con `BCrypt` (nunca en texto plano). |
| **RN-S02** | El login emite un **access token** JWT firmado con `HS256` y secret de mínimo 256 bits. |
| **RN-S03** | El access token expira en **15 minutos** (corto a propósito). |
| **RN-S04** | El access token incluye el email (`sub`) y el rol como claim (`role`). |
| **RN-S05** | El login emite también un **refresh token** UUID que vive **7 días** y se persiste en BD. |
| **RN-S06** | Sin token o con token inválido → `401 Unauthorized`. |
| **RN-S07** | Con token válido pero rol insuficiente → `403 Forbidden`. |
| **RN-S08** | La API es **stateless** para los access tokens (no hay sesión HTTP). |
| **RN-S09** | El logout marca el refresh token como `revoked = true`. Los refresh revocados o expirados no pueden renovar nada. |
| **RN-S10** | Endpoints públicos: `/api/auth/{register,login,refresh,logout}`, `/swagger-ui/**`, `/v3/api-docs/**`. |
| **RN-S11** | El registro crea `User` + `Customer` **atómicamente**: si falla cualquiera de los dos, ninguno se persiste. |
| **RN-S12** | El `userId` jamás viene del body en endpoints autenticados: se obtiene del token (`Authentication`). |
| **RN-S13** | Los campos `email`, `dni` y `password` no se editan vía `PUT /me` — requieren flujos especiales. |

[↑ Volver al indice](#indice)

---

## 4. Modelo del dominio (lo que ya existe)

| Dominio | Modelo | Que representa |
|---|---|---|
| `auth/` | `User` | Credenciales de acceso (email, password, verified, role). |
| `auth/` | `Role` | Rol del usuario: `ADMIN`, `CUSTOMER` o `MERCHANT`. |

```
[ auth ]

   Role 1 ────<  *  User  >────── 1  Customer (en customer/)
   - id              - id
   - name            - email
                     - password (BCrypt)
                     - verified
                     - role
```

`User` y `Role` ya están creados desde la semana 04. **No los toques.** Lo que cambia es lo que viene alrededor.

[↑ Volver al indice](#indice)

---

## 5. Punto de partida — qué hay y qué falta

Antes de tocar código, ubícate. Esto es lo que **ya existe** en `pagoya-api/` heredado de las semanas anteriores:

| Archivo | Estado |
|---|---|
| `auth/model/User.java` | ✅ Listo |
| `auth/model/Role.java` | ✅ Listo |
| `auth/repository/UserRepository.java` | ✅ Tiene `findByEmail` y `existsByEmail` |
| `auth/repository/RoleRepository.java` | ✅ Listo |
| `auth/dto/RegisterUserRequest.java` | ✅ Listo |
| `auth/dto/UserResponse.java` | ✅ Listo |
| `auth/mapper/UserMapper.java` | ✅ Listo |
| `auth/service/UserService.java` | ⚠️ Existe pero **guarda password en texto plano** |
| `auth/controller/AuthController.java` | ⚠️ Sólo expone `register`, falta `login` |
| `shared/exception/GlobalExceptionHandler.java` | ✅ Listo (lo extenderemos) |
| `pom.xml` | ⚠️ **Falta** `spring-boot-starter-security` y `jjwt` |

Esto es lo que **vamos a crear**:

| Archivo nuevo | Para qué |
|---|---|
| `auth/service/JwtService.java` | Emitir, firmar y validar access tokens (JWT). |
| `auth/service/UserDetailsServiceImpl.java` | Cargar el `User` desde BD para Spring Security. |
| `auth/security/JwtAuthenticationFilter.java` | Filtro que valida el access token en cada request. |
| `auth/model/RefreshToken.java` | Entidad JPA para refresh tokens persistidos. |
| `auth/repository/RefreshTokenRepository.java` | Acceso a BD para refresh tokens. |
| `auth/service/RefreshTokenService.java` | Crear, validar y revocar refresh tokens. |
| `auth/dto/LoginRequest.java` | DTO con email + password. |
| `auth/dto/AuthResponse.java` | DTO de salida con access + refresh tokens. |
| `auth/dto/RefreshRequest.java` | DTO con el refresh token (para `/refresh` y `/logout`). |
| `auth/exception/InvalidRefreshTokenException.java` | Excepción de negocio. |
| `auth/service/IAuthService.java` y `AuthService.java` | Endpoints de login, refresh y logout. |
| `shared/config/SecurityConfig.java` | Configuración central de seguridad. |

[↑ Volver al indice](#indice)

---

## 6. Crear la rama

Toda feature parte de `develop`:

```bash
git checkout develop
git pull origin develop
git checkout -b feature/security-jwt
```

[↑ Volver al indice](#indice)

---

## 7. Estructura final del paquete `auth/` y `customer/`

Cuando termines, así debe quedar el paquete:

```
src/main/java/com/hampcode/pagoya/auth/
├── controller/
│   └── AuthController.java          (modificado: register atomico + login/refresh/logout/logout-all)
├── dto/
│   ├── RegisterUserRequest.java     (modificado: ahora incluye fullName, dni, phone)
│   ├── RegisterResponse.java        (NUEVO: devuelve datos de User + Customer)
│   ├── UserResponse.java            (existente)
│   ├── LoginRequest.java            (NUEVO)
│   ├── AuthResponse.java            (NUEVO: access + refresh)
│   └── RefreshRequest.java          (NUEVO)
├── exception/
│   ├── EmailAlreadyExistsException.java   (existente)
│   └── InvalidRefreshTokenException.java  (NUEVO)
├── mapper/
│   └── UserMapper.java              (modificado: agrega toRegisterResponse(User, Customer))
├── model/
│   ├── User.java                    (existente)
│   ├── Role.java                    (existente)
│   └── RefreshToken.java            (NUEVO)
├── repository/
│   ├── UserRepository.java          (existente)
│   ├── RoleRepository.java          (existente)
│   └── RefreshTokenRepository.java  (NUEVO)
├── security/                         (NUEVO subpaquete)
│   └── JwtAuthenticationFilter.java (NUEVO)
└── service/
    ├── IUserService.java            (modificado: register devuelve RegisterResponse)
    ├── UserService.java             (modificado: registro atomico User+Customer)
    ├── IAuthService.java            (NUEVO)
    ├── AuthService.java             (NUEVO)
    ├── JwtService.java              (NUEVO)
    ├── RefreshTokenService.java     (NUEVO)
    └── UserDetailsServiceImpl.java  (NUEVO)
```

```
src/main/java/com/hampcode/pagoya/customer/
├── controller/
│   └── CustomerController.java      (modificado: -POST, +GET /me, +PUT /me)
├── dto/
│   ├── CreateCustomerRequest.java   (deprecado/eliminable: ya no se usa publicamente)
│   ├── CustomerResponse.java        (existente)
│   └── UpdateCustomerRequest.java   (NUEVO: solo fullName y phone)
├── mapper/
│   └── CustomerMapper.java          (existente)
├── model/
│   └── Customer.java                (existente)
├── repository/
│   └── CustomerRepository.java      (modificado: + findByUserId)
└── service/
    ├── ICustomerService.java        (modificado: + findByEmail, + updateByEmail)
    └── CustomerService.java         (modificado)
```

Y en `shared/config/`:

```
src/main/java/com/hampcode/pagoya/shared/config/
├── AppConfig.java         (existente)
├── OpenApiConfig.java     (modificado: agrega security scheme JWT)
└── SecurityConfig.java    (NUEVO)
```

[↑ Volver al indice](#indice)

---

## 8. Agregar dependencias en `pom.xml`

Abre `pom.xml` y dentro de `<dependencies>` agrega tres bloques:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

**Por qué cada una:**

| Dependencia | Para qué |
|---|---|
| `spring-boot-starter-security` | Activa Spring Security: filtros, `PasswordEncoder`, `AuthenticationManager`. |
| `jjwt-api` | API pública para escribir y leer tokens JWT. |
| `jjwt-impl` | Implementación interna (sólo en runtime). |
| `jjwt-jackson` | Serializador JSON usado por `jjwt`. |

Guarda el archivo y deja que Maven descargue las dependencias (botón **Load Maven Changes** en IntelliJ, o `mvn clean compile` en terminal).

> ⚠️ Apenas agregues `spring-boot-starter-security` y arranques la app, **TODOS los endpoints exigirán autenticación básica** y verás algo como `Using generated security password: ...` en consola. Eso es esperado: lo vamos a sobreescribir con `SecurityConfig`.

[↑ Volver al indice](#indice)

---

## 9. Variables de entorno (`.env`) y `application-local.yml`

### 8.1 Agregar al `.env`

Edita el archivo `.env` (en la raíz de `pagoya-api/`) y agrega tres variables nuevas al final:

```bash
JWT_SECRET=cambia-este-secreto-por-uno-de-32-bytes-o-mas-aleatorio
JWT_EXPIRATION_MS=900000
JWT_REFRESH_EXPIRATION_MS=604800000
```

| Variable | Valor sugerido | Significa |
|---|---|---|
| `JWT_SECRET` | string aleatorio ≥ 32 caracteres | Llave para firmar el access token. |
| `JWT_EXPIRATION_MS` | `900000` | 15 minutos de vida del access token. |
| `JWT_REFRESH_EXPIRATION_MS` | `604800000` | 7 días de vida del refresh token. |

> El secret debe tener **mínimo 32 caracteres** (256 bits) para `HS256`. En producción genéralo con `openssl rand -base64 64`. **Jamás** lo commitees al repo.

### 8.2 Exponerlas en `application-local.yml`

Agrega al final de `src/main/resources/application-local.yml`:

```yaml
pagoya:
  security:
    jwt:
      secret: ${JWT_SECRET:cambia-este-secreto-por-uno-de-32-bytes-o-mas-aleatorio}
      expiration-ms: ${JWT_EXPIRATION_MS:900000}
      refresh-expiration-ms: ${JWT_REFRESH_EXPIRATION_MS:604800000}
```

Las dos sintaxis (`${VAR:default}`) significan: usa la variable de entorno si está; si no, el valor por defecto (sólo válido en local).

> 📖 **¿Cómo se calculan esos milisegundos? ¿Cómo lo cambio? ¿Cuándo invoca el frontend a `/refresh`?** Todo eso está en la **Teoría 06** (`Teoria 06-SPRING-SECURITY-JWT.html`), slides "Configuración: duración de los tokens" y "¿Cuándo se invoca `/refresh` desde el frontend?". Ahí encontrás la fórmula, ejemplos de duración por tipo de app y el patrón de interceptor de Axios.

[↑ Volver al indice](#indice)

---

## 10. Registro atómico: `User` + `Customer` + BCrypt

### 10.1 El problema con el registro actual

El proyecto base trae **dos endpoints separados**:

```
POST /api/auth/register   → crea SOLO el User (credenciales)
POST /api/customers       → crea Customer, recibe userId en el body
```

Esto tiene tres problemas serios:

1. **UX rota**: el cliente hace 2 calls. Entre el primero y el segundo, el `User` queda en limbo sin perfil.
2. **Inseguro**: `CreateCustomerRequest` tiene `userId` en el body. Un atacante podría poner el `userId` de **otro** usuario y crearle perfil ajeno.
3. **No es atómico**: si el segundo call falla, queda un `User` huérfano sin `Customer`.

Lo arreglamos haciendo que **`POST /api/auth/register` cree los dos en una transacción**, recibiendo todos los datos en un solo body.

### 10.2 Modificar `RegisterUserRequest`

Reemplaza el contenido de `auth/dto/RegisterUserRequest.java` para incluir los campos del perfil:

```java
package com.hampcode.pagoya.auth.dto;

import jakarta.validation.constraints.*;

public record RegisterUserRequest(
    @NotBlank(message = "el email es obligatorio")
    @Email(message = "el formato del email no es valido")
    String email,

    @NotBlank(message = "la contrasena es obligatoria")
    @Size(min = 8, message = "la contrasena debe tener al menos 8 caracteres")
    String password,

    @NotBlank(message = "el nombre completo es obligatorio")
    @Size(max = 100)
    String fullName,

    @NotBlank(message = "el DNI es obligatorio")
    @Pattern(regexp = "\\d{8}", message = "el DNI debe tener 8 digitos")
    String dni,

    @Pattern(regexp = "\\d{9}", message = "el telefono debe tener 9 digitos")
    String phone
) {}
```

### 10.3 Crear `RegisterResponse`

Nuevo DTO en `auth/dto/RegisterResponse.java` que devuelve datos de **ambos** dominios:

```java
package com.hampcode.pagoya.auth.dto;

public record RegisterResponse(
    Long userId,
    String email,
    String role,
    Long customerId,
    String fullName,
    String dni
) {}
```

### 10.4 Modificar `UserMapper` para combinar `User` + `Customer`

`UserMapper` cambia de mapear sólo `User → UserResponse` a mapear **dos entidades a la vez** (`User` + `Customer`) hacia el `RegisterResponse`. MapStruct soporta múltiples fuentes:

```java
package com.hampcode.pagoya.auth.mapper;

import com.hampcode.pagoya.auth.dto.RegisterResponse;
import com.hampcode.pagoya.auth.model.User;
import com.hampcode.pagoya.customer.model.Customer;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface UserMapper {

    @Mapping(target = "userId",     source = "user.id")
    @Mapping(target = "email",      source = "user.email")
    @Mapping(target = "role",       source = "user.role.name")
    @Mapping(target = "customerId", source = "customer.id")
    @Mapping(target = "fullName",   source = "customer.fullName")
    @Mapping(target = "dni",        source = "customer.dni")
    RegisterResponse toRegisterResponse(User user, Customer customer);
}
```

**Por qué dos parámetros:** el `RegisterResponse` mezcla campos de los dos dominios (`auth` y `customer`). En lugar de armarlo a mano con `new RegisterResponse(...)`, le pasamos las dos entidades al mapper y MapStruct resuelve los `source` con notación `param.field`.

### 10.5 Modificar `UserService` para registro atómico

Inyecta `PasswordEncoder`, `CustomerRepository` y `UserMapper`. Ambas inserts viajan en la misma transacción:

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.dto.RegisterResponse;
import com.hampcode.pagoya.auth.dto.RegisterUserRequest;
import com.hampcode.pagoya.auth.exception.EmailAlreadyExistsException;
import com.hampcode.pagoya.auth.mapper.UserMapper;
import com.hampcode.pagoya.auth.model.Role;
import com.hampcode.pagoya.auth.model.User;
import com.hampcode.pagoya.auth.repository.RoleRepository;
import com.hampcode.pagoya.auth.repository.UserRepository;
import com.hampcode.pagoya.customer.exception.DniAlreadyExistsException;
import com.hampcode.pagoya.customer.model.Customer;
import com.hampcode.pagoya.customer.repository.CustomerRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class UserService implements IUserService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final CustomerRepository customerRepository;
    private final PasswordEncoder passwordEncoder;
    private final UserMapper userMapper;

    @Override
    @Transactional   // atomico: si falla cualquiera de los 2 inserts, rollback total
    public RegisterResponse register(RegisterUserRequest request) {
        // RN-S11.1 — email unico
        if (userRepository.existsByEmail(request.email())) {
            throw new EmailAlreadyExistsException();
        }
        // RN-S11.2 — DNI unico
        if (customerRepository.existsByDni(request.dni())) {
            throw new DniAlreadyExistsException();
        }
        Role role = roleRepository.findByName("CUSTOMER")
            .orElseThrow(() -> new ResourceNotFoundException(
                "no se puede completar el registro en este momento"));

        // 1. Crear User con password BCrypteado
        User user = User.builder()
            .email(request.email())
            .password(passwordEncoder.encode(request.password()))
            .verified(false)
            .role(role)
            .build();
        user = userRepository.save(user);

        // 2. Crear Customer asociado (userId lo asigna el SERVIDOR, no el body)
        Customer customer = Customer.builder()
            .fullName(request.fullName())
            .dni(request.dni())
            .phone(request.phone())
            .userId(user.getId())
            .build();
        customer = customerRepository.save(customer);

        // 3. Armar la respuesta delegando en el mapper
        return userMapper.toRegisterResponse(user, customer);
    }
}
```

> 🔒 **Por qué `userId` ya no viene del body**: aunque el endpoint es público (`permitAll`), seguimos la regla **RN-S12**: nunca confiar en ids del body para autenticación/autorización. El servidor genera el `User`, obtiene su id y lo asigna internamente al `Customer`. Imposible falsificar.

### 10.6 Actualizar `IUserService`

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.dto.RegisterResponse;
import com.hampcode.pagoya.auth.dto.RegisterUserRequest;

public interface IUserService {
    RegisterResponse register(RegisterUserRequest request);
}
```

### 10.7 Actualizar `AuthController` (endpoint register)

Cambia el tipo de respuesta de `UserResponse` a `RegisterResponse`:

```java
@Operation(summary = "Registro: crea credenciales (User) y perfil (Customer) atomicamente")
@ApiResponses({
    @ApiResponse(responseCode = "201", description = "Usuario y perfil creados"),
    @ApiResponse(responseCode = "400", description = "Email ya registrado, DNI duplicado o datos invalidos")
})
@PostMapping("/register")
public ResponseEntity<RegisterResponse> register(@Valid @RequestBody RegisterUserRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.register(request));
}
```

### 10.8 Limpiar `CustomerController`

El método `create()` que recibía `userId` en el body **se elimina** (la creación pasa a ser interna del registro). Si más adelante necesitás un endpoint admin para alta manual, podés reintroducirlo bajo `@PreAuthorize("hasRole('ADMIN')")`.

```java
// Borrar este metodo de CustomerController:
// @PostMapping
// public ResponseEntity<CustomerResponse> create(@Valid @RequestBody CreateCustomerRequest request) { ... }
```

> 🔒 **Buena práctica de seguridad**: el mensaje de error del rol es **genérico** a propósito. No expone el nombre del rol (`CUSTOMER`), ni el campo que falta, ni detalles internos. Los mensajes que se devuelven al cliente NUNCA deben revelar:
> - Nombres de roles, tablas, columnas, campos internos.
> - Si un email existe o no (en login responde igual a `usuario inexistente` y `password incorrecto`: `"credenciales invalidas"`).
> - Configuración del servidor, stack traces, rutas.
>
> El detalle real va al **log del servidor**, al cliente sólo el mensaje genérico.

El `PasswordEncoder` lo registraremos como `@Bean` en `SecurityConfig` (sección 14).

[↑ Volver al indice](#indice)

---

## 11. Crear `JwtService`

Esta clase concentra toda la lógica de JWT: firmar, leer claims, validar.

### 10.1 `auth/service/JwtService.java`

```java
package com.hampcode.pagoya.auth.service;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
public class JwtService {

    @Value("${pagoya.security.jwt.secret}")
    private String secret;

    @Value("${pagoya.security.jwt.expiration-ms}")
    private long expirationMs;

    public String generateToken(UserDetails user, String role) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("role", role);
        return Jwts.builder()
            .claims(claims)
            .subject(user.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(getSigningKey())
            .compact();
    }

    public String extractEmail(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean isValid(String token, UserDetails user) {
        return extractEmail(token).equals(user.getUsername()) && !isExpired(token);
    }

    private boolean isExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    private <T> T extractClaim(String token, Function<Claims, T> resolver) {
        Claims claims = Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
        return resolver.apply(claims);
    }

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(
            java.util.Base64.getEncoder().encodeToString(secret.getBytes()));
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

**Decisiones clave:**

- **`HS256`**: algoritmo simétrico, basta con un secret compartido. Para producción multi-servicio usa `RS256` (par de llaves).
- **Claim `role`**: lo agregamos para que el filtro pueda asignar la `GrantedAuthority` sin volver a la BD.
- **`parseSignedClaims`**: lanza excepción si la firma no coincide o el token está expirado. La atrapa el filtro.

[↑ Volver al indice](#indice)

---

## 12. Crear `UserDetailsServiceImpl`

Spring Security necesita una implementación de `UserDetailsService` para cargar el usuario desde BD.

### 11.1 `auth/service/UserDetailsServiceImpl.java`

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.model.User;
import com.hampcode.pagoya.auth.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String email) {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException(
                "usuario no encontrado: " + email));

        return new org.springframework.security.core.userdetails.User(
            user.getEmail(),
            user.getPassword(),
            List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole().getName()))
        );
    }
}
```

**Por qué `ROLE_`**: Spring Security espera que las autoridades de tipo "rol" empiecen con `ROLE_`. Cuando uses `hasRole("ADMIN")` en realidad busca `ROLE_ADMIN`.

[↑ Volver al indice](#indice)

---

## 13. Crear `JwtAuthenticationFilter`

El filtro que se ejecuta una vez por request, lee el header y autentica al usuario.

### 12.1 Crear el subpaquete `auth/security/`

En `src/main/java/com/hampcode/pagoya/auth/`, crea un nuevo package llamado `security`.

### 12.2 `auth/security/JwtAuthenticationFilter.java`

```java
package com.hampcode.pagoya.auth.security;

import com.hampcode.pagoya.auth.service.JwtService;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);
        try {
            String email = jwtService.extractEmail(token);
            if (email != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails user = userDetailsService.loadUserByUsername(email);
                if (jwtService.isValid(token, user)) {
                    UsernamePasswordAuthenticationToken auth =
                        new UsernamePasswordAuthenticationToken(
                            user, null, user.getAuthorities());
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (Exception ex) {
            // token invalido o expirado: dejamos al SecurityContext vacio
            // para que SecurityConfig responda 401.
        }

        filterChain.doFilter(request, response);
    }
}
```

**Notas:**

- `OncePerRequestFilter` garantiza que el filtro se ejecuta **una sola vez** por request, incluso si Spring lo invoca varias veces internamente.
- Si el header no viene o es inválido, dejamos el `SecurityContext` vacío y la cadena sigue. Será `SecurityConfig` quien decida si el endpoint requiere auth.

[↑ Volver al indice](#indice)

---

## 14. Crear `SecurityConfig`

Configuración central: define qué es público, qué es privado, el `PasswordEncoder` y dónde se inserta el filtro JWT.

### 13.1 `shared/config/SecurityConfig.java`

```java
package com.hampcode.pagoya.shared.config;

import com.hampcode.pagoya.auth.security.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // Endpoints de auth publicos (excepto logout-all que requiere token)
                .requestMatchers(
                    "/api/auth/register",
                    "/api/auth/login",
                    "/api/auth/refresh",
                    "/api/auth/logout"
                ).permitAll()
                .requestMatchers(
                    "/v3/api-docs/**",
                    "/swagger-ui/**",
                    "/swagger-ui.html"
                ).permitAll()
                // logout-all requiere autenticacion
                .requestMatchers("/api/auth/logout-all").authenticated()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

**Decisiones clave:**

- **`csrf().disable()`**: válido sólo porque la API es stateless (sin cookies de sesión) y se consume desde clientes que envían el token explícitamente.
- **`SessionCreationPolicy.STATELESS`**: Spring no creará `HttpSession`. Cada request se autentica de cero con el token.
- **`@EnableMethodSecurity`**: habilita `@PreAuthorize` y `@PostAuthorize` en métodos de servicios y controllers.
- **`addFilterBefore(...)`**: nuestro filtro JWT corre **antes** del filtro de username/password tradicional.

[↑ Volver al indice](#indice)

---

## 15. Modelo de Refresh Tokens

Aquí entra la pieza que permite el **logout real**. Mientras el access token vive corto y stateless, el refresh token vive en BD y puede revocarse.

### 14.1 `auth/model/RefreshToken.java`

```java
package com.hampcode.pagoya.auth.model;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "refresh_tokens")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class RefreshToken {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String token;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;

    @Column(nullable = false)
    private boolean revoked;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
}
```

### 14.2 `auth/repository/RefreshTokenRepository.java`

```java
package com.hampcode.pagoya.auth.repository;

import com.hampcode.pagoya.auth.model.RefreshToken;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    Optional<RefreshToken> findByToken(String token);

    @Modifying
    @Transactional
    @Query("UPDATE RefreshToken r SET r.revoked = true WHERE r.user.id = :userId")
    void revokeAllByUserId(@Param("userId") Long userId);
}
```

### 14.3 `auth/exception/InvalidRefreshTokenException.java`

```java
package com.hampcode.pagoya.auth.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class InvalidRefreshTokenException extends BusinessRuleException {
    public InvalidRefreshTokenException() {
        super("refresh token invalido, expirado o revocado");
    }
}
```

### 14.4 `auth/service/RefreshTokenService.java`

Esta clase concentra el ciclo de vida del refresh token: crear, validar, **rotar en cada uso** y revocar. La rotación es importante: cada vez que el cliente llama a `/refresh`, **emitimos un refresh nuevo y revocamos el anterior**. Si alguien intentara reusar un refresh ya rotado, lo detectamos como señal de robo y revocamos todos los del usuario.

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.exception.InvalidRefreshTokenException;
import com.hampcode.pagoya.auth.model.RefreshToken;
import com.hampcode.pagoya.auth.model.User;
import com.hampcode.pagoya.auth.repository.RefreshTokenRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;

    @Value("${pagoya.security.jwt.refresh-expiration-ms}")
    private long refreshExpirationMs;

    @Transactional
    public RefreshToken create(User user) {
        RefreshToken rt = RefreshToken.builder()
            .token(UUID.randomUUID().toString())
            .user(user)
            .expiresAt(LocalDateTime.now().plusNanos(refreshExpirationMs * 1_000_000))
            .revoked(false)
            .createdAt(LocalDateTime.now())
            .build();
        return refreshTokenRepository.save(rt);
    }

    @Transactional
    public RefreshToken validate(String token) {
        RefreshToken rt = refreshTokenRepository.findByToken(token)
            .orElseThrow(InvalidRefreshTokenException::new);

        // Si ya fue revocado, podria ser un intento de reuso (robo).
        // Defensa: revocar todos los refresh del usuario y forzar relogin.
        if (rt.isRevoked()) {
            refreshTokenRepository.revokeAllByUserId(rt.getUser().getId());
            throw new InvalidRefreshTokenException();
        }
        if (rt.getExpiresAt().isBefore(LocalDateTime.now())) {
            throw new InvalidRefreshTokenException();
        }
        return rt;
    }

    /**
     * Rotacion: revoca el refresh actual y emite uno nuevo para el mismo usuario.
     * Se llama desde AuthService.refresh() en cada renovacion.
     */
    @Transactional
    public RefreshToken rotate(RefreshToken current) {
        current.setRevoked(true);
        refreshTokenRepository.save(current);
        return create(current.getUser());
    }

    @Transactional
    public void revoke(String token) {
        refreshTokenRepository.findByToken(token).ifPresent(rt -> {
            rt.setRevoked(true);
            refreshTokenRepository.save(rt);
        });
    }

    /**
     * Revoca TODOS los refresh tokens del usuario.
     * Util para logout-all (cerrar sesion en todos los dispositivos)
     * y como defensa cuando se detecta reuso de un token revocado.
     */
    @Transactional
    public void revokeAllByUserId(Long userId) {
        refreshTokenRepository.revokeAllByUserId(userId);
    }
}
```

**Decisiones clave:**

- **`UUID.randomUUID()`**: el refresh token NO es un JWT — es un UUID opaco que sólo el servidor puede validar (porque vive en BD).
- **`validate()`** falla con `InvalidRefreshTokenException` si: no existe, está revocado, o expiró. Si está revocado, **además revoca todos los del usuario**: es la firma típica de un token robado siendo reusado.
- **`rotate()`** implementa **refresh token rotation**: cada llamada a `/refresh` genera un token nuevo y revoca el anterior. El `GlobalExceptionHandler` ya devuelve `400` para `BusinessRuleException`.
- **`revokeAllByUserId`**: usado por el endpoint `logout-all` y como defensa contra robo.

[↑ Volver al indice](#indice)

---

## 16. Endpoints `login`, `refresh`, `logout` y `logout-all`

### 15.1 `auth/dto/LoginRequest.java`

```java
package com.hampcode.pagoya.auth.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public record LoginRequest(
    @NotBlank @Email String email,
    @NotBlank String password
) {}
```

### 15.2 `auth/dto/AuthResponse.java`

```java
package com.hampcode.pagoya.auth.dto;

public record AuthResponse(
    String accessToken,
    String refreshToken,
    String email,
    String role,
    long accessExpiresInMs
) {}
```

### 15.3 `auth/dto/RefreshRequest.java`

```java
package com.hampcode.pagoya.auth.dto;

import jakarta.validation.constraints.NotBlank;

public record RefreshRequest(
    @NotBlank String refreshToken
) {}
```

### 15.4 `auth/service/IAuthService.java`

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.dto.AuthResponse;
import com.hampcode.pagoya.auth.dto.LoginRequest;

public interface IAuthService {
    AuthResponse login(LoginRequest request);
    AuthResponse refresh(String refreshToken);
    void logout(String refreshToken);
    void logoutAll(String email);
}
```

### 15.5 `auth/service/AuthService.java`

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.dto.AuthResponse;
import com.hampcode.pagoya.auth.dto.LoginRequest;
import com.hampcode.pagoya.auth.model.RefreshToken;
import com.hampcode.pagoya.auth.model.User;
import com.hampcode.pagoya.auth.repository.UserRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class AuthService implements IAuthService {

    private final AuthenticationManager authenticationManager;
    private final UserDetailsService userDetailsService;
    private final UserRepository userRepository;
    private final JwtService jwtService;
    private final RefreshTokenService refreshTokenService;

    @Value("${pagoya.security.jwt.expiration-ms}")
    private long accessExpirationMs;

    @Override
    public AuthResponse login(LoginRequest request) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password()));

        UserDetails userDetails = userDetailsService.loadUserByUsername(request.email());
        User user = userRepository.findByEmail(request.email()).orElseThrow();
        String role = user.getRole().getName();

        String accessToken = jwtService.generateToken(userDetails, role);
        RefreshToken refresh = refreshTokenService.create(user);

        return new AuthResponse(
            accessToken, refresh.getToken(),
            user.getEmail(), role, accessExpirationMs);
    }

    /**
     * Renueva el access token aplicando rotation: el refresh recibido se
     * revoca y se emite uno nuevo, junto con el nuevo access.
     */
    @Override
    public AuthResponse refresh(String refreshToken) {
        RefreshToken current = refreshTokenService.validate(refreshToken);
        RefreshToken rotated = refreshTokenService.rotate(current);

        User user = rotated.getUser();
        UserDetails userDetails = userDetailsService.loadUserByUsername(user.getEmail());
        String role = user.getRole().getName();

        String newAccessToken = jwtService.generateToken(userDetails, role);
        return new AuthResponse(
            newAccessToken, rotated.getToken(),
            user.getEmail(), role, accessExpirationMs);
    }

    @Override
    public void logout(String refreshToken) {
        refreshTokenService.revoke(refreshToken);
    }

    /**
     * Cierra sesion en todos los dispositivos del usuario autenticado.
     * Revoca todos sus refresh tokens activos.
     */
    @Override
    public void logoutAll(String email) {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new ResourceNotFoundException("usuario no encontrado"));
        refreshTokenService.revokeAllByUserId(user.getId());
    }
}
```

**Decisiones clave:**

- **`login`** emite el access token (JWT) Y crea el refresh token en BD. Devuelve ambos.
- **`refresh`** aplica **rotation**: valida el refresh recibido, lo revoca, emite uno nuevo. La respuesta incluye **un access nuevo y un refresh nuevo**. El cliente debe reemplazar el refresh anterior en su almacenamiento.
- **`logout`** revoca un refresh específico. El access token sigue vivo hasta su `exp` (máx 15 min), pero ya no se puede renovar — sesión efectivamente cerrada.
- **`logoutAll`** revoca todos los refresh del usuario. Útil cuando cambia su contraseña o detecta actividad sospechosa.

### 15.6 Modificar `auth/controller/AuthController.java`

```java
package com.hampcode.pagoya.auth.controller;

import com.hampcode.pagoya.auth.dto.*;
import com.hampcode.pagoya.auth.service.IAuthService;
import com.hampcode.pagoya.auth.service.IUserService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
@Tag(name = "Auth", description = "Registro, login, refresh, logout y logout-all")
public class AuthController {

    private final IUserService userService;
    private final IAuthService authService;

    @Operation(summary = "Registrar un nuevo usuario")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Usuario registrado"),
        @ApiResponse(responseCode = "400", description = "Datos invalidos o email ya registrado")
    })
    @PostMapping("/register")
    public ResponseEntity<UserResponse> register(@Valid @RequestBody RegisterUserRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(userService.register(request));
    }

    @Operation(summary = "Login: emite access token (JWT) y refresh token")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Tokens emitidos"),
        @ApiResponse(responseCode = "401", description = "Credenciales invalidas")
    })
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }

    @Operation(summary = "Renovar tokens (rotation): emite nuevo access y nuevo refresh")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Tokens rotados"),
        @ApiResponse(responseCode = "400", description = "Refresh token invalido, expirado o revocado")
    })
    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@Valid @RequestBody RefreshRequest request) {
        return ResponseEntity.ok(authService.refresh(request.refreshToken()));
    }

    @Operation(summary = "Cerrar sesion en este dispositivo: revoca el refresh token")
    @ApiResponses({
        @ApiResponse(responseCode = "204", description = "Sesion cerrada")
    })
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@Valid @RequestBody RefreshRequest request) {
        authService.logout(request.refreshToken());
        return ResponseEntity.noContent().build();
    }

    @Operation(summary = "Cerrar sesion en TODOS los dispositivos del usuario autenticado")
    @ApiResponses({
        @ApiResponse(responseCode = "204", description = "Todas las sesiones cerradas"),
        @ApiResponse(responseCode = "401", description = "No autenticado")
    })
    @PostMapping("/logout-all")
    public ResponseEntity<Void> logoutAll(Authentication authentication) {
        UserDetails principal = (UserDetails) authentication.getPrincipal();
        authService.logoutAll(principal.getUsername());
        return ResponseEntity.noContent().build();
    }
}
```

> ⚠️ **Importante:** `logout-all` requiere usuario autenticado. Hay que excluirlo de los `permitAll` en `SecurityConfig` (ya viene cubierto por el `anyRequest().authenticated()` que viste antes — por eso el endpoint exige `Authentication` inyectada).

[↑ Volver al indice](#indice)

---

## 17. Endpoints de perfil `GET /me` y `PUT /me`

El usuario logueado **nunca debe pasar su propio id por URL**. El backend lo saca del JWT vía `Authentication`. Esto cubre **US-S06** y **US-S07**.

### 17.1 `customer/dto/UpdateCustomerRequest.java`

DTO con SOLO los campos editables. `email`, `dni` y `password` quedan fuera (RN-S13):

```java
package com.hampcode.pagoya.customer.dto;

import jakarta.validation.constraints.*;

public record UpdateCustomerRequest(
    @NotBlank(message = "el nombre completo es obligatorio")
    @Size(max = 100)
    String fullName,

    @Pattern(regexp = "\\d{9}", message = "el telefono debe tener 9 digitos")
    String phone
) {}
```

### 17.2 Agregar `findByUserId` al `CustomerRepository`

```java
package com.hampcode.pagoya.customer.repository;

import com.hampcode.pagoya.customer.model.Customer;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface CustomerRepository extends JpaRepository<Customer, Long> {
    boolean existsByDni(String dni);
    boolean existsByUserId(Long userId);
    Optional<Customer> findByUserId(Long userId);   // NUEVO
}
```

### 17.3 Modificar `ICustomerService` y `CustomerService`

Agrega los métodos `findByEmail` y `updateByEmail`:

```java
// ICustomerService.java
CustomerResponse findByEmail(String email);
CustomerResponse updateByEmail(String email, UpdateCustomerRequest request);
```

```java
// CustomerService.java
private final UserRepository userRepository;   // inyectar

@Override
@Transactional(readOnly = true)
public CustomerResponse findByEmail(String email) {
    User user = userRepository.findByEmail(email)
        .orElseThrow(() -> new ResourceNotFoundException("usuario no encontrado"));
    return customerRepository.findByUserId(user.getId())
        .map(customerMapper::toResponse)
        .orElseThrow(() -> new ResourceNotFoundException("perfil no encontrado"));
}

@Override
@Transactional
public CustomerResponse updateByEmail(String email, UpdateCustomerRequest request) {
    User user = userRepository.findByEmail(email)
        .orElseThrow(() -> new ResourceNotFoundException("usuario no encontrado"));
    Customer customer = customerRepository.findByUserId(user.getId())
        .orElseThrow(() -> new ResourceNotFoundException("perfil no encontrado"));

    customer.setFullName(request.fullName());
    customer.setPhone(request.phone());
    return customerMapper.toResponse(customerRepository.save(customer));
}
```

### 17.4 Agregar endpoints a `CustomerController`

```java
@Operation(summary = "Obtener mi perfil")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Perfil del usuario autenticado"),
    @ApiResponse(responseCode = "401", description = "No autenticado")
})
@GetMapping("/me")
public ResponseEntity<CustomerResponse> findMe(Authentication authentication) {
    String email = ((UserDetails) authentication.getPrincipal()).getUsername();
    return ResponseEntity.ok(customerService.findByEmail(email));
}

@Operation(summary = "Actualizar mi perfil (fullName, phone)")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Perfil actualizado"),
    @ApiResponse(responseCode = "400", description = "Datos invalidos"),
    @ApiResponse(responseCode = "401", description = "No autenticado")
})
@PutMapping("/me")
public ResponseEntity<CustomerResponse> updateMe(
        Authentication authentication,
        @Valid @RequestBody UpdateCustomerRequest request) {
    String email = ((UserDetails) authentication.getPrincipal()).getUsername();
    return ResponseEntity.ok(customerService.updateByEmail(email, request));
}
```

Imports nuevos en el controller:

```java
import com.hampcode.pagoya.customer.dto.UpdateCustomerRequest;
import jakarta.validation.Valid;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
```

### 17.5 ¿Y para cambiar email o password?

`PUT /me` **no toca** esos campos. Para esos casos creás endpoints aparte (fuera del alcance de esta guía):

| Endpoint | Por qué necesita flujo propio |
|---|---|
| `POST /api/auth/change-password` | Requiere validar la password actual antes de cambiar la nueva, y revoca todos los refresh tokens del usuario. |
| `POST /api/auth/change-email` | Requiere verificación del nuevo email vía link y reemisión de tokens (porque el `sub` cambia). |

[↑ Volver al indice](#indice)

---

## 18. Documentar JWT en Swagger

Para que Swagger UI muestre el botón **Authorize** y permita pegar un token, ajusta `OpenApiConfig`.

### 16.1 Modificar `shared/config/OpenApiConfig.java`

```java
package com.hampcode.pagoya.shared.config;

import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    private static final String SCHEME = "bearerAuth";

    @Bean
    public OpenAPI pagoyaOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("PagoYa API")
                .description("API de la billetera digital PagoYa")
                .version("v1")
                .contact(new Contact()
                    .name("Equipo PagoYa by HampCode")
                    .email("devacademyweb@gmail.com")))
            .addSecurityItem(new SecurityRequirement().addList(SCHEME))
            .components(new Components().addSecuritySchemes(SCHEME,
                new SecurityScheme()
                    .name(SCHEME)
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")));
    }
}
```

Tras esto, Swagger UI muestra el candado en cada endpoint y un botón **Authorize** arriba a la derecha.

[↑ Volver al indice](#indice)

---

## 19. Manejo de errores 401 / 403

Sin configurar nada extra, Spring devuelve `401` y `403` con un body genérico. Para que se vean uniformes con el resto de errores de la API, extiende `GlobalExceptionHandler`:

### 17.1 Modificar `shared/exception/GlobalExceptionHandler.java`

Agrega estos dos handlers (deja todo lo demás como está):

```java
@ExceptionHandler(org.springframework.security.authentication.BadCredentialsException.class)
public ResponseEntity<ErrorResponse> handleBadCredentials(
        org.springframework.security.authentication.BadCredentialsException ex,
        HttpServletRequest req) {
    return build(HttpStatus.UNAUTHORIZED, "credenciales invalidas", req, null);
}

@ExceptionHandler(org.springframework.security.access.AccessDeniedException.class)
public ResponseEntity<ErrorResponse> handleAccessDenied(
        org.springframework.security.access.AccessDeniedException ex,
        HttpServletRequest req) {
    return build(HttpStatus.FORBIDDEN, "no tienes permiso para esta operacion", req, null);
}
```

> El `401` que devuelve el filtro cuando el token está expirado lo gestiona Spring Security directamente. Si quieres que también pase por tu `ErrorResponse`, configura un `AuthenticationEntryPoint` en `SecurityConfig`. Para esta guía, lo dejamos en el comportamiento por defecto.

[↑ Volver al indice](#indice)

---

## 20. Autorización fina con `@PreAuthorize`

Hasta acá `anyRequest().authenticated()` exige que el usuario esté logueado, pero **no distingue roles**. En PagoYa hay reglas más finas: sólo un `ADMIN` puede crear proveedores; un `CUSTOMER` sólo puede ver sus propios pagos. Para eso usamos **`@PreAuthorize`** sobre métodos específicos.

`@EnableMethodSecurity` ya está activado en `SecurityConfig` (sección 13), así que las anotaciones funcionan directamente.

### 18.1 Restricción por rol simple

Caso: sólo un `ADMIN` puede dar de baja a un cliente. Edita `customer/controller/CustomerController.java`:

```java
import org.springframework.security.access.prepost.PreAuthorize;

@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    customerService.delete(id);
    return ResponseEntity.noContent().build();
}
```

Si un `CUSTOMER` intenta llamar este endpoint con su token válido → `403 Forbidden`. El `GlobalExceptionHandler` devuelve `"no tienes permiso para esta operacion"`.

### 18.2 Restricción por múltiples roles

Caso: `MERCHANT` o `ADMIN` pueden dar de alta un proveedor. Cuando crees `ServiceProviderController` (post `POST`):

```java
@PreAuthorize("hasAnyRole('MERCHANT','ADMIN')")
@PostMapping
public ResponseEntity<ServiceProviderResponse> create(...) { ... }
```

### 18.3 Restricción por **dueño del recurso** (SpEL)

Caso: un cliente sólo puede ver **sus propios** pagos. Si pasa el `customerId` por URL, hay que verificar que coincide con el usuario autenticado.

Primero, una pequeña ayuda: que `User` también guarde una referencia al `Customer` (eso ya está en el modelo del proyecto base via la relación inversa). Lo más simple es comparar por **email**: el `principal` autenticado tiene el email, y `Customer` tiene el campo `userEmail` o referencia a `User`.

Para PagoYa adaptamos el approach a la estructura existente: creamos un bean `@securityChecks` que el SpEL puede consultar.

`auth/security/SecurityChecks.java`:

```java
package com.hampcode.pagoya.auth.security;

import com.hampcode.pagoya.customer.repository.CustomerRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

@Component("securityChecks")
@RequiredArgsConstructor
public class SecurityChecks {

    private final CustomerRepository customerRepository;

    /**
     * Devuelve true si el customer con ese id pertenece al usuario autenticado.
     */
    public boolean isCustomerOwner(Long customerId, Object principal) {
        if (!(principal instanceof UserDetails ud)) return false;
        return customerRepository.findById(customerId)
            .map(c -> c.getUser().getEmail().equals(ud.getUsername()))
            .orElse(false);
    }
}
```

Luego en `BillPaymentController`:

```java
@PreAuthorize("hasRole('ADMIN') or @securityChecks.isCustomerOwner(#customerId, authentication.principal)")
@GetMapping("/customer/{customerId}")
public ResponseEntity<PageResponse<BillPaymentResponse>> findByCustomer(
        @PathVariable Long customerId,
        @PageableDefault(size = 10, sort = "id") Pageable pageable) {
    return ResponseEntity.ok(
        PageResponse.from(billPaymentService.findByCustomer(customerId, pageable)));
}
```

**Cómo se lee la expresión SpEL:**
- `hasRole('ADMIN')` → si es admin, pasa.
- `@securityChecks.isCustomerOwner(#customerId, authentication.principal)` → si no, llama al bean `securityChecks` y consulta si el customer del path le pertenece al usuario logueado.
- `#customerId` referencia el parámetro `@PathVariable`.
- `authentication.principal` es el `UserDetails` que el filtro JWT puso en el `SecurityContext`.

### 18.4 Resumen de qué proteger

| Endpoint | Restricción |
|---|---|
| `POST /api/auth/**` (excepto logout-all) | público |
| `POST /api/auth/logout-all` | autenticado |
| `POST /api/customers` | `CUSTOMER` (alta de su propio perfil) |
| `DELETE /api/customers/{id}` | `ADMIN` |
| `POST /api/service-providers` | `MERCHANT` o `ADMIN` |
| `GET /api/bill-payments/customer/{id}` | `ADMIN` o dueño del recurso |
| `GET /api/service-providers` | autenticado |
| Resto | autenticado |

[↑ Volver al indice](#indice)

---

## 21. Probar en Postman

Importa la colección `pagoya-api.postman_collection.json` (si no lo hiciste en guías anteriores). Luego **agrega un folder nuevo** llamado `Auth` y dentro estos requests:

### 21.1 Registrar usuario (registro atómico)

| | |
|---|---|
| Método | `POST` |
| URL | `http://localhost:8080/api/auth/register` |
| Body (JSON) | `{ "email": "ana@pagoya.com", "password": "supersecreta", "fullName": "Ana Lopez", "dni": "12345678", "phone": "987654321" }` |

Esperado: `201 Created` con `{ userId, email, role, customerId, fullName, dni }`.

> En BD se crearon **dos filas**: una en `users` y otra en `customers`. Si `dni` ya existía, el `User` no se persiste (rollback).

### 21.2 Login

| | |
|---|---|
| Método | `POST` |
| URL | `http://localhost:8080/api/auth/login` |
| Body (JSON) | `{ "email": "ana@pagoya.com", "password": "supersecreta" }` |

Esperado: `200 OK` con `{ accessToken, refreshToken, email, role, accessExpiresInMs }`.

En la pestaña **Tests** del request, agrega esto para guardar **ambos tokens** automáticamente:

```javascript
const data = pm.response.json();
pm.collectionVariables.set("jwt", data.accessToken);
pm.collectionVariables.set("refresh", data.refreshToken);
```

### 21.3 Refresh (con rotation)

| | |
|---|---|
| Método | `POST` |
| URL | `http://localhost:8080/api/auth/refresh` |
| Body (JSON) | `{ "refreshToken": "{{refresh}}" }` |

En la pestaña **Tests**, actualiza **ambos** tokens (porque la rotación devuelve refresh nuevo):

```javascript
const data = pm.response.json();
pm.collectionVariables.set("jwt", data.accessToken);
pm.collectionVariables.set("refresh", data.refreshToken);
```

Esperado: `200 OK` con un nuevo `accessToken` y un nuevo `refreshToken`. El refresh anterior **queda revocado**.

### 21.4 Logout

| | |
|---|---|
| Método | `POST` |
| URL | `http://localhost:8080/api/auth/logout` |
| Body (JSON) | `{ "refreshToken": "{{refresh}}" }` |

Esperado: `204 No Content`.

### 21.5 Logout en todos los dispositivos

| | |
|---|---|
| Método | `POST` |
| URL | `http://localhost:8080/api/auth/logout-all` |
| Auth | Bearer `{{jwt}}` (este endpoint requiere token) |
| Body | (vacío) |

Esperado: `204 No Content`. **Todos** los refresh del usuario quedan revocados.

### 21.6 Ver mi perfil (`GET /me`)

| | |
|---|---|
| Método | `GET` |
| URL | `http://localhost:8080/api/customers/me` |
| Auth | Bearer `{{jwt}}` |

Esperado: `200 OK` con `{ id, fullName, dni, phone, userId }`. **No tuviste que pasar tu propio id**: el backend lo saca del token.

### 21.7 Actualizar mi perfil (`PUT /me`)

| | |
|---|---|
| Método | `PUT` |
| URL | `http://localhost:8080/api/customers/me` |
| Auth | Bearer `{{jwt}}` |
| Body (JSON) | `{ "fullName": "Ana Lopez Garcia", "phone": "999888777" }` |

Esperado: `200 OK` con el perfil actualizado. Si intentás mandar `dni` o `email` en el body, simplemente se ignoran (el DTO no los expone).

### 21.8 Probar endpoint protegido

Para el resto de endpoints (`/api/bill-payments`, etc.), en la pestaña **Authorization**:

- Type: **Bearer Token**
- Token: `{{jwt}}`

### 21.9 Casos a verificar

| Caso | Esperado |
|---|---|
| Registrar con DNI duplicado | `400` y **ningún** `User` queda en BD (rollback) |
| Registrar sin `fullName` o `dni` | `400` con `"datos invalidos"` |
| Llamar `GET /me` sin token | `401 Unauthorized` |
| Llamar `GET /me` con token válido | `200 OK` con tu propio perfil |
| Llamar `PUT /me` con token válido | `200 OK` y los cambios se reflejan en BD |
| Login con password incorrecto | `401` con `"credenciales invalidas"` |
| Access token modificado a mano | `401 Unauthorized` |
| Esperar 15 min y reusar el access | `401 Unauthorized` |
| Llamar `refresh` con un refresh válido | `200 OK` con **nuevo access y nuevo refresh** |
| Llamar `refresh` dos veces con el **mismo** refresh viejo | Segunda llamada → `400` y revoca **todos** los del usuario (defensa contra robo) |
| Llamar `logout` y luego intentar `refresh` | `400` con `"refresh token invalido, expirado o revocado"` |
| Llamar `refresh` con un token inventado | `400 Bad Request` |
| Cliente `CUSTOMER` llama `DELETE /api/customers/{id}` | `403` con `"no tienes permiso para esta operacion"` |
| Cliente `CUSTOMER` consulta pagos de **otro** cliente | `403 Forbidden` |
| Llamar `logout-all` sin token | `401 Unauthorized` |

[↑ Volver al indice](#indice)

---

## 22. Commit y Pull Request

### 22.1 Commit y push

```bash
git add pom.xml \
        .env \
        src/main/resources/application-local.yml \
        src/main/java/com/hampcode/pagoya/auth \
        src/main/java/com/hampcode/pagoya/customer \
        src/main/java/com/hampcode/pagoya/shared/config/SecurityConfig.java \
        src/main/java/com/hampcode/pagoya/shared/config/OpenApiConfig.java \
        src/main/java/com/hampcode/pagoya/shared/exception/GlobalExceptionHandler.java
git commit -m "feat(security): registro atomico, JWT + refresh rotation, /me y autorizacion fina"
git push -u origin feature/security-jwt
```

> ⚠️ **Cuidado con el `.env`**: si tu `.gitignore` lo ignora (recomendado), no lo agregues al commit. En su lugar, comparte el formato del archivo en un `.env.example` sin secretos reales.

### 22.2 Abrir el PR en GitHub

Pull Request: `feature/security-jwt` → `develop`.

**Título sugerido:**

```
feat(security): registro atomico User+Customer, JWT + refresh rotation, /me y @PreAuthorize
```

**Descripción sugerida:**

```markdown
## Que entrega

Capa de seguridad completa sobre la API:
- Registro atomico: POST /api/auth/register crea User (credenciales) + Customer (perfil) en una sola transaccion.
- BCrypt para hashear contraseñas.
- Spring Security configurado en modo stateless.
- Access token JWT (HS256, 15 min) emitido en login y validado por filtro custom.
- Refresh token (UUID, 7 dias) persistido en BD con rotation en cada uso.
- Logout real (revoca el refresh) y logout-all (revoca todos los refresh del usuario).
- Endpoints /me para que el usuario consulte y actualice su propio perfil sin pasar id por URL.
- Autorizacion fina con @PreAuthorize: por rol y por dueño del recurso.
- Swagger documenta el esquema bearerAuth.

## Endpoints nuevos

- POST /api/auth/login      -> { accessToken, refreshToken, email, role, accessExpiresInMs }
- POST /api/auth/refresh    -> rotation: nuevo accessToken Y nuevo refreshToken
- POST /api/auth/logout     -> revoca el refresh recibido
- POST /api/auth/logout-all -> revoca todos los refresh del usuario autenticado
- GET  /api/customers/me    -> mi perfil (sin id en URL)
- PUT  /api/customers/me    -> actualizar mi perfil (fullName, phone)

## Endpoints afectados

- POST /api/auth/register   -> recibe email+password+fullName+dni+phone, hashea, crea ambos atomicamente.
- POST /api/customers       -> ELIMINADO (la creacion va via register).
- DELETE /api/customers/{id}-> @PreAuthorize hasRole('ADMIN').
- GET /api/bill-payments/customer/{id} -> ADMIN o dueño del recurso (SpEL).
- Resto de la API -> exigen Authorization: Bearer <accessToken>.

## Historias cubiertas

- US-S01: registro atomico en una sola operacion.
- US-S02 / S03: login + refresh automatico via interceptor frontend.
- US-S04 / S05: logout y logout-all reales (refresh revocado).
- US-S06 / S07: GET /me y PUT /me sin pasar id propio.
- US-S08: endpoints admin protegidos por rol.

## Reglas cubiertas

- RN-S01: passwords con BCrypt.
- RN-S02 / S03 / S04: access token HS256, 15min, claims sub+role.
- RN-S05: refresh token UUID 7 dias en BD con rotation.
- RN-S06 / S07: 401 sin token, 403 sin rol.
- RN-S08: stateless para los access tokens.
- RN-S09: logout revoca el refresh; reuso de refresh revocado revoca todos los del usuario (defensa).
- RN-S10: /api/auth/{register,login,refresh,logout}, swagger-ui, v3/api-docs son publicos.
- RN-S11: registro User+Customer atomico (transaccion unica).
- RN-S12: userId nunca viene del body en endpoints autenticados.
- RN-S13: PUT /me NO permite editar email, dni ni password (flujos especiales).

## Como probarlo

- docker compose up -d
- mvn spring-boot:run
- En Swagger UI o Postman: register (con todos los datos) -> login -> GET /me -> PUT /me -> refresh -> logout.
- Probar rollback: registrar con DNI duplicado -> 400, ningun User queda en BD.
- Probar 403: con un CUSTOMER intentar DELETE /api/customers/{id} o consultar pagos ajenos.
```

[↑ Volver al indice](#indice)

---


[↑ Volver al indice](#indice)
