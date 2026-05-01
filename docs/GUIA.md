# Actividad: Bounded Context `billing`

**Elaborado por:** Henry Antonio Mendoza Puerta

## Objetivo

Implementar el bounded context `billing/` del proyecto PagoYa API: un catálogo de proveedores y el registro de pagos del cliente. Al terminar tendrás dos agregados funcionando, la API documentada con Swagger y un Pull Request abierto contra `develop`.

Lo que vas a practicar:

- **Package by feature**: agrupar el código por funcionalidad.
- **DDD**: separar el catálogo (compartido) de los pagos (por cliente).
- **Buenas prácticas**: validar la entrada, controlar los errores y cuidar la base de datos.
- **Documentación**: dejar la API lista para que otros la prueben.
- **GitFlow**: trabajar en una rama y entregar con Pull Request.

## Material previo

> **Importante:** esta actividad es la **continuación** del proyecto PagoYa API que se construye en los laboratorios previos. Si no revisas primero estos materiales vas a estar perdido, porque aquí damos por hecho el código y la estructura que se desarrollan ahí.

Antes de empezar, revisa de manera obligatoria los siguientes materiales donde se construye PagoYa API desde cero:

- `Laboratorio_04_Diseno_de_Software_Implementacion_API` (Semana 4)
- <a href="https://github.com/hampcodes/bookstore/blob/main/docs/Teoria05-Buenas%20practicas%20api%20rest.md" target="_blank" rel="noopener noreferrer"><code>Teoria 5 — Buenas prácticas de diseño de API REST</code></a> (Semana 5)
- <a href="https://github.com/hampcodes/bookstore/blob/main/docs/Laboratorio_05_Buenas_Practicas_API.md" target="_blank" rel="noopener noreferrer"><code>Laboratorio_05_Buenas_Practicas_API</code></a> (Semana 5)

## Indice

- [1. Enunciado](#1-enunciado)
- [2. Historias de usuario](#2-historias-de-usuario)
- [3. Reglas de negocio](#3-reglas-de-negocio)
- [4. Modelo del dominio](#4-modelo-del-dominio)
- [5. Setup inicial](#5-setup-inicial)
- [6. Levantar la infraestructura](#6-levantar-la-infraestructura)
- [7. Ejecutar el proyecto](#7-ejecutar-el-proyecto)
- [8. Crear la rama](#8-crear-la-rama)
- [9. Estructura del paquete](#9-estructura-del-paquete)
- [10. Implementar `ServiceProvider` (catalogo)](#10-implementar-serviceprovider-catalogo)
- [11. Implementar `BillPayment` (transaccional)](#11-implementar-billpayment-transaccional)
- [12. Implementar el reporte por categoria (US-B04)](#12-implementar-el-reporte-por-categoria-us-b04)
- [13. Configurar Swagger / OpenAPI](#13-configurar-swagger--openapi)
- [14. Probar la API en Postman](#14-probar-la-api-en-postman)
- [15. Commit y Pull Request](#15-commit-y-pull-request)
- [16. Comandos de Git: cuando usar cada uno](#16-comandos-de-git-cuando-usar-cada-uno)
- [17. Tarea propuesta](#17-tarea-propuesta)

---

## 1. Enunciado

En PagoYa cada cliente paga sus servicios (luz, agua, internet, telefono) desde la billetera. Vas a implementar el bounded context `billing/` con dos agregados: un **catalogo de proveedores** y el **registro de pagos** que cada cliente realiza.

[↑ Volver al indice](#indice)

---

## 2. Historias de usuario

| Codigo | Historia |
|---|---|
| **US-B01** | Como cliente quiero ver el catalogo de proveedores activos para elegir a quien pagarle. |
| **US-B02** | Como cliente quiero registrar el pago de un servicio para saldar mi recibo desde la app. |
| **US-B03** | Como cliente quiero ver mis pagos paginados para revisar mi historial. |
| **US-B04** | Como cliente quiero ver mis pagos agrupados por categoria para entender en que gasto mas. |

[↑ Volver al indice](#indice)

## 3. Reglas de negocio

| Codigo | Regla |
|---|---|
| **RN-B01** | Solo se puede pagar a proveedores activos. |
| **RN-B02** | El monto debe ser mayor a 0 y no superar los 5000 soles. |
| **RN-B03** | Un cliente no puede pagar dos veces el mismo recibo al mismo proveedor. |
| **RN-B04** | La fecha y hora del pago se registran automaticamente. |
| **RN-B05** | Si el pago se completa, queda en estado `PAID`. |
| **RN-B06** | El cliente debe existir en PagoYa para poder pagar. |

[↑ Volver al indice](#indice)

## 4. Modelo del dominio

| Dominio | Modelos | Que representa |
|---|---|---|
| `customer/` | `Customer` | Cliente de PagoYa. |
| `billing/` | `ServiceProvider` | Catalogo de empresas a las que se les puede pagar. |
| `billing/` | `BillPayment` | Cada pago que un cliente registra a un proveedor. |

```
[ customer ]              [ billing ]

   Customer 1 ────<  *  BillPayment  *  >────  1  ServiceProvider
                         - billCode                 - name
                         - amount                   - category
                         - status                   - active
                         - paidAt                   - createdAt
                         - createdAt
```

[↑ Volver al indice](#indice)

---

## 5. Setup inicial

Prepara tu entorno y tu repositorio.

### 5.1 Descargar el proyecto

Desde el material de la **semana 5**, descarga `pagoya-api.zip` y descomprimelo en una carpeta de tu eleccion.

### 5.2 Importar la coleccion en Postman

1. Abre Postman e **inicia sesion** (sin login se pierde la coleccion al cerrar la app).
2. Sidebar → **Import**.
3. Selecciona `pagoya-api.postman_collection.json` que viene en el proyecto.

La coleccion `PagoYa API` aparece con los endpoints existentes (auth, customer, transfer).

### 5.3 Crear tu repositorio en GitHub

1. `github.com` → **New repository**.
2. Nombre: `pagoya-api`.
3. **NO marques** *Add a README*, *.gitignore* ni *license*: el repo debe quedar VACIO.
4. **Create repository**.

### 5.4 Subir el proyecto

Desde la carpeta del proyecto:

```bash
git init
git add .
git commit -m "chore: initial commit"
git branch -M main
git remote add origin https://github.com/<TU_USUARIO>/pagoya-api.git
git push -u origin main
```

### 5.5 Crear la rama `develop`

Toda nueva feature parte de `develop`:

```bash
git checkout -b develop
git push -u origin develop
```

Tu repo queda con dos ramas: `main` (produccion) y `develop` (integracion).

[↑ Volver al indice](#indice)

---

## 6. Levantar la infraestructura

El proyecto ya tiene `compose.yml` con PostgreSQL y pgAdmin listos.

### 6.1 Levantar los contenedores

Desde la raiz del proyecto:

```bash
docker compose up -d
docker compose ps
```

Debes ver dos contenedores corriendo: `pagoya_db` y `pagoya_pgadmin`.

### 6.2 Datos de los servicios

| Servicio | URL host | Usuario | Password |
|---|---|---|---|
| PostgreSQL | `localhost:55432` (BD `pagoya_db`) | `postgres` | `postgres` |
| pgAdmin | `http://localhost:8082` | `admin@pagoya.com` | `admin` |

### 6.3 Crear el server en pgAdmin

1. Abre `http://localhost:8082` y entra con `admin@pagoya.com / admin`.
2. Click derecho en **Servers → Register → Server...**
3. **General → Name**: `pagoya-local`.
4. **Connection**, completa con estos datos:

   | Campo | Valor |
   |---|---|
   | Host name/address | `postgres` |
   | Port | `5432` |
   | Maintenance database | `pagoya_db` |
   | Username | `postgres` |
   | Password | `postgres` |

5. Click **Save**.

Importante: el host es `postgres` (nombre del servicio en `compose.yml`), NO `localhost`. pgAdmin se conecta a Postgres por la red interna de Docker.

### 6.4 Variables de entorno

El `.env` del proyecto ya tiene `DB_URL`, `DB_USERNAME` y `DB_PASSWORD`. No necesitas modificarlo.

[↑ Volver al indice](#indice)

---

## 7. Ejecutar el proyecto

```bash
mvn clean compile
export $(grep -v '^#' .env | xargs) && mvn spring-boot:run
```

Al arrancar, Hibernate crea las tablas `service_providers` y `bill_payments` en la base de datos `pagoya_db` (schema `public`).

Para verlas en pgAdmin sigue esta ruta:

```
Servers → pagoya-local → Databases → pagoya_db → Schemas → public → Tables
```

Ahí debes ver listadas `service_providers` y `bill_payments` junto al resto de tablas del proyecto. Click derecho sobre cualquiera → **View/Edit Data → All Rows** para inspeccionar su contenido.

### 7.1 Cargar datos de prueba

El proyecto trae los proveedores semilla en:

- `src/main/resources/data.sql`

Como el perfil `local` tiene `sql.init.mode: never`, este archivo NO se carga automatico. Cargalo a mano despues del primer arranque (cuando ya existen las tablas):

**Opcion A — pgAdmin**

1. `pagoya-local` → `pagoya_db` → Tools → Query Tool.
2. Icono **Open File** y selecciona `src/main/resources/data.sql`.
3. `F5` para ejecutar.

**Opcion B — psql**

```bash
psql -h localhost -p 55432 -U postgres -d pagoya_db \
     -f src/main/resources/data.sql
```

Inserta los proveedores `Sedapal`, `Luz del Sur`, `Movistar`, `Claro` y `Win`. El `ON CONFLICT (id) DO NOTHING` evita duplicados si lo corres mas de una vez.

[↑ Volver al indice](#indice)

---

## 8. Crear la rama

```bash
git checkout develop
git pull origin develop
git checkout -b feature/billing-context
```

[↑ Volver al indice](#indice)

---

## 9. Estructura del paquete

### 9.1 Crear el paquete `billing`

Dentro del proyecto, el paquete principal es `com.hampcode.pagoya`. Ahí ya existen otros bounded contexts (`auth`, `customer`, `transfer`, `shared`). Vas a crear uno nuevo al mismo nivel: `billing`.

**En IntelliJ IDEA**

1. Abre el panel **Project** (`Cmd/Alt + 1`).
2. Navega a `src/main/java/com/hampcode/pagoya`.
3. Click derecho sobre `pagoya` → **New → Package**.
4. Nombre: `billing` y `Enter`.

**En VSCode**

1. Click derecho sobre `src/main/java/com/hampcode/pagoya` → **New Folder** → `billing`.

### 9.2 Crear los subpaquetes

Dentro de `billing` crea los siete subpaquetes que agrupan el código por responsabilidad. Repite el paso anterior (click derecho → **New → Package** o **New Folder**) para cada uno:

| Subpaquete | Qué contiene |
|---|---|
| `controller` | Recibe la petición HTTP y la delega al service (listar proveedores, registrar pago, ver historial, reporte). |
| `service` | Reglas de negocio: validar proveedor activo, evitar pagos duplicados, registrar el pago y armar el reporte por categoría. |
| `repository` | Consultas a la base: proveedores activos, pagos del cliente y verificación de pago duplicado. |
| `model` | Entidades JPA y enums del dominio. |
| `dto` | Objetos de entrada y salida de los endpoints. |
| `mapper` | Conversión entre entidad y DTO con MapStruct. |
| `exception` | Excepciones propias de las reglas de negocio. |

> En IntelliJ, si escribes `billing.controller` directo en **New → Package**, te crea ambos niveles de un tirón.

### 9.3 Estructura final

Cuando termines, el árbol del paquete debe verse así:

```
src/main/java/com/hampcode/pagoya/billing/
├── controller/
│   ├── ServiceProviderController.java
│   └── BillPaymentController.java
├── service/
│   ├── IServiceProviderService.java   ServiceProviderService.java
│   └── IBillPaymentService.java       BillPaymentService.java
├── repository/
│   ├── ServiceProviderRepository.java
│   └── BillPaymentRepository.java
├── model/
│   ├── ServiceProvider.java   ProviderCategory.java
│   └── BillPayment.java       PaymentStatus.java
├── dto/
│   ├── ServiceProviderResponse.java
│   ├── CreateBillPaymentRequest.java
│   ├── BillPaymentResponse.java
│   └── PaymentByCategoryResponse.java
├── mapper/
│   ├── ServiceProviderMapper.java
│   └── BillPaymentMapper.java
└── exception/
    ├── InactiveProviderException.java
    └── DuplicateBillPaymentException.java
```

Por ahora los subpaquetes están vacíos; en las siguientes secciones vas creando las clases una por una.

[↑ Volver al indice](#indice)

---

## 10. Implementar `ServiceProvider` (catalogo)

### 10.1 `model/ProviderCategory.java`

Enum con las categorias del catalogo. Cada proveedor pertenece a una.

```java
package com.hampcode.pagoya.billing.model;

public enum ProviderCategory {
    UTILITIES, TELECOM, INTERNET, CABLE_TV, OTHER
}
```

### 10.2 `model/ServiceProvider.java`

La entidad. Representa un proveedor que aparece en el catalogo. Es compartido — todos los clientes ven el mismo catalogo.

```java
package com.hampcode.pagoya.billing.model;

import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;

@Entity
@Table(name = "service_providers")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class ServiceProvider {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private ProviderCategory category;

    @Column(nullable = false)
    private boolean active;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
}
```

### 10.3 `repository/ServiceProviderRepository.java`

El repositorio. `findByActiveTrue` filtra solo proveedores activos (RN-B01).

```java
package com.hampcode.pagoya.billing.repository;

import com.hampcode.pagoya.billing.model.ServiceProvider;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ServiceProviderRepository extends JpaRepository<ServiceProvider, Long> {
    Page<ServiceProvider> findByActiveTrue(Pageable pageable);
}
```

### 10.4 `dto/ServiceProviderResponse.java`

DTO de salida. Solo expone `id`, `name` y `category`. NO incluye `active` ni `createdAt` porque son metadatos internos.

```java
package com.hampcode.pagoya.billing.dto;

public record ServiceProviderResponse(
    Long id,
    String name,
    String category
) {}
```

### 10.5 `mapper/ServiceProviderMapper.java`

Convierte `ServiceProvider` → `ServiceProviderResponse`.

```java
package com.hampcode.pagoya.billing.mapper;

import com.hampcode.pagoya.billing.dto.ServiceProviderResponse;
import com.hampcode.pagoya.billing.model.ServiceProvider;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface ServiceProviderMapper {
    @Mapping(target = "category", expression = "java(p.getCategory().name())")
    ServiceProviderResponse toResponse(ServiceProvider p);
}
```

Por que ese `@Mapping`: el DTO tiene `category` como `String` pero la entity lo tiene como enum `ProviderCategory`. MapStruct no convierte enum → String solo, asi que con `expression = "java(...)"` le decimos como hacerlo (`p.getCategory().name()`).

### 10.6 Excepciones

En esta etapa `ServiceProvider` solo expone una operacion de lectura, asi que NO necesita excepciones propias. Si mas adelante agregas POST o PUT podrias necesitar una `DuplicateProviderException` u otra de negocio.

### 10.7 `service/IServiceProviderService.java`

```java
package com.hampcode.pagoya.billing.service;

import com.hampcode.pagoya.billing.dto.ServiceProviderResponse;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface IServiceProviderService {
    Page<ServiceProviderResponse> findAllActive(Pageable pageable);
}
```

### 10.8 `service/ServiceProviderService.java`

Implementacion. Cada metodo declara su propio `@Transactional` (no a nivel de clase).

```java
package com.hampcode.pagoya.billing.service;

import com.hampcode.pagoya.billing.dto.ServiceProviderResponse;
import com.hampcode.pagoya.billing.mapper.ServiceProviderMapper;
import com.hampcode.pagoya.billing.repository.ServiceProviderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class ServiceProviderService implements IServiceProviderService {

    private final ServiceProviderRepository providerRepository;
    private final ServiceProviderMapper providerMapper;

    @Override
    @Transactional(readOnly = true)
    public Page<ServiceProviderResponse> findAllActive(Pageable pageable) {
        return providerRepository.findByActiveTrue(pageable)
            .map(providerMapper::toResponse);
    }
}
```

### 10.9 `controller/ServiceProviderController.java`

Expone el listado paginado en `GET /api/service-providers`. Anotaciones de Swagger lo documentan.

```java
package com.hampcode.pagoya.billing.controller;

import com.hampcode.pagoya.billing.dto.ServiceProviderResponse;
import com.hampcode.pagoya.billing.service.IServiceProviderService;
import com.hampcode.pagoya.shared.pagination.PageResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/service-providers")
@RequiredArgsConstructor
@Tag(name = "Service Providers", description = "Catalogo de proveedores de servicios")
public class ServiceProviderController {

    private final IServiceProviderService providerService;

    @Operation(summary = "Listar proveedores activos")
    @GetMapping
    public ResponseEntity<PageResponse<ServiceProviderResponse>> findAll(
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        return ResponseEntity.ok(
            PageResponse.from(providerService.findAllActive(pageable)));
    }
}
```

[↑ Volver al indice](#indice)

---

## 11. Implementar `BillPayment` (transaccional)

### 11.1 `model/PaymentStatus.java`

Enum con los estados que puede tener un pago.

```java
package com.hampcode.pagoya.billing.model;

public enum PaymentStatus {
    PAID, FAILED, REFUNDED
}
```

### 11.2 `model/BillPayment.java`

La entidad transaccional. Tiene FKs a `Customer` y a `ServiceProvider`. La restriccion `unique(customer_id, provider_id, bill_code)` apoya RN-B03 a nivel de tabla.

```java
package com.hampcode.pagoya.billing.model;

import com.hampcode.pagoya.customer.model.Customer;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "bill_payments",
       uniqueConstraints = @UniqueConstraint(
           columnNames = {"customer_id", "provider_id", "bill_code"}))
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class BillPayment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "provider_id", nullable = false)
    private ServiceProvider provider;

    @Column(name = "bill_code", nullable = false, length = 50)
    private String billCode;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private PaymentStatus status;

    @Column(name = "paid_at", nullable = false)
    private LocalDateTime paidAt;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
}
```

### 11.3 `repository/BillPaymentRepository.java`

`existsBy...` valida pagos duplicados (RN-B03). `findByCustomer_Id` es la lista paginada del historial.

```java
package com.hampcode.pagoya.billing.repository;

import com.hampcode.pagoya.billing.model.BillPayment;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface BillPaymentRepository extends JpaRepository<BillPayment, Long> {
    boolean existsByCustomer_IdAndProvider_IdAndBillCode(
        Long customerId, Long providerId, String billCode);
    Page<BillPayment> findByCustomer_Id(Long customerId, Pageable pageable);
}
```

### 11.4 `dto/CreateBillPaymentRequest.java`

DTO de entrada con validaciones declarativas (Bean Validation). Cumple RN-B02 con `@DecimalMin/Max`.

```java
package com.hampcode.pagoya.billing.dto;

import jakarta.validation.constraints.*;

import java.math.BigDecimal;

public record CreateBillPaymentRequest(
    @NotNull Long customerId,
    @NotNull Long providerId,
    @NotBlank @Size(max = 50) String billCode,
    @NotNull
    @DecimalMin(value = "0.01", message = "el monto debe ser mayor a 0")
    @DecimalMax(value = "5000.00", message = "el monto no puede superar 5000")
    BigDecimal amount
) {}
```

### 11.5 `dto/BillPaymentResponse.java`

DTO de salida. Solo expone el nombre del proveedor (no su id), no expone `customerId` ni FK alguna.

```java
package com.hampcode.pagoya.billing.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record BillPaymentResponse(
    Long id,
    String providerName,
    String billCode,
    BigDecimal amount,
    String status,
    LocalDateTime paidAt
) {}
```

### 11.6 `mapper/BillPaymentMapper.java`

Convierte DTO ↔ Entity en ambos sentidos.

```java
package com.hampcode.pagoya.billing.mapper;

import com.hampcode.pagoya.billing.dto.BillPaymentResponse;
import com.hampcode.pagoya.billing.dto.CreateBillPaymentRequest;
import com.hampcode.pagoya.billing.model.BillPayment;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface BillPaymentMapper {

    @Mapping(target = "id",        ignore = true)
    @Mapping(target = "customer",  ignore = true)
    @Mapping(target = "provider",  ignore = true)
    @Mapping(target = "status",    ignore = true)
    @Mapping(target = "paidAt",    ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    BillPayment toEntity(CreateBillPaymentRequest request);

    @Mapping(target = "providerName", source = "provider.name")
    @Mapping(target = "status", expression = "java(p.getStatus().name())")
    BillPaymentResponse toResponse(BillPayment p);
}
```

Por que esos `@Mapping`:
- En `toEntity`: los seis `ignore = true` indican que MapStruct NO debe copiar esos campos del DTO. El service los completa: `id` lo asigna la BD; `customer` y `provider` los carga el service desde sus repos; `status`, `paidAt` y `createdAt` los pone el sistema.
- En `toResponse`: `source = "provider.name"` navega de la entity al campo `name` del proveedor. `expression` convierte el enum `status` a `String` (igual que en el ServiceProviderMapper).

### 11.7 Excepciones

Dos excepciones de negocio: una para cuando el proveedor no esta activo (RN-B01) y otra para pagos duplicados (RN-B03). Heredan de `BusinessRuleException`, asi el `GlobalExceptionHandler` las atrapa con HTTP 400.

`exception/InactiveProviderException.java`:

```java
package com.hampcode.pagoya.billing.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class InactiveProviderException extends BusinessRuleException {
    public InactiveProviderException() {
        super("el proveedor seleccionado no esta disponible");
    }
}
```

`exception/DuplicateBillPaymentException.java`:

```java
package com.hampcode.pagoya.billing.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class DuplicateBillPaymentException extends BusinessRuleException {
    public DuplicateBillPaymentException() {
        super("ya tienes registrado un pago para este recibo");
    }
}
```

### 11.8 `service/IBillPaymentService.java`

Interface del servicio.

```java
package com.hampcode.pagoya.billing.service;

import com.hampcode.pagoya.billing.dto.BillPaymentResponse;
import com.hampcode.pagoya.billing.dto.CreateBillPaymentRequest;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface IBillPaymentService {
    BillPaymentResponse pay(CreateBillPaymentRequest request);
    Page<BillPaymentResponse> findByCustomer(Long customerId, Pageable pageable);
}
```

### 11.9 `service/BillPaymentService.java`

Implementacion. Es el corazon de la logica: valida cliente, valida proveedor activo, valida no-duplicado, y registra el pago.

```java
package com.hampcode.pagoya.billing.service;

import com.hampcode.pagoya.billing.dto.BillPaymentResponse;
import com.hampcode.pagoya.billing.dto.CreateBillPaymentRequest;
import com.hampcode.pagoya.billing.exception.DuplicateBillPaymentException;
import com.hampcode.pagoya.billing.exception.InactiveProviderException;
import com.hampcode.pagoya.billing.mapper.BillPaymentMapper;
import com.hampcode.pagoya.billing.model.BillPayment;
import com.hampcode.pagoya.billing.model.PaymentStatus;
import com.hampcode.pagoya.billing.model.ServiceProvider;
import com.hampcode.pagoya.billing.repository.BillPaymentRepository;
import com.hampcode.pagoya.billing.repository.ServiceProviderRepository;
import com.hampcode.pagoya.customer.model.Customer;
import com.hampcode.pagoya.customer.repository.CustomerRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
public class BillPaymentService implements IBillPaymentService {

    private final BillPaymentRepository billPaymentRepository;
    private final ServiceProviderRepository providerRepository;
    private final CustomerRepository customerRepository;
    private final BillPaymentMapper billPaymentMapper;

    @Override
    @Transactional
    public BillPaymentResponse pay(CreateBillPaymentRequest request) {
        Customer customer = customerRepository.findById(request.customerId())
            .orElseThrow(() -> new ResourceNotFoundException("cliente no encontrado"));

        ServiceProvider provider = providerRepository.findById(request.providerId())
            .orElseThrow(() -> new ResourceNotFoundException("proveedor no encontrado"));

        if (!provider.isActive()) {
            throw new InactiveProviderException();
        }

        if (billPaymentRepository.existsByCustomer_IdAndProvider_IdAndBillCode(
                customer.getId(), provider.getId(), request.billCode())) {
            throw new DuplicateBillPaymentException();
        }

        BillPayment entity = billPaymentMapper.toEntity(request);
        entity.setCustomer(customer);
        entity.setProvider(provider);
        entity.setStatus(PaymentStatus.PAID);
        entity.setPaidAt(LocalDateTime.now());
        entity.setCreatedAt(LocalDateTime.now());

        return billPaymentMapper.toResponse(billPaymentRepository.save(entity));
    }

    @Override
    @Transactional(readOnly = true)
    public Page<BillPaymentResponse> findByCustomer(Long customerId, Pageable pageable) {
        if (!customerRepository.existsById(customerId)) {
            throw new ResourceNotFoundException("cliente no encontrado");
        }
        return billPaymentRepository.findByCustomer_Id(customerId, pageable)
            .map(billPaymentMapper::toResponse);
    }
}
```

### 11.10 `controller/BillPaymentController.java`

Expone los dos endpoints: POST para crear y GET paginado para listar el historial.

```java
package com.hampcode.pagoya.billing.controller;

import com.hampcode.pagoya.billing.dto.BillPaymentResponse;
import com.hampcode.pagoya.billing.dto.CreateBillPaymentRequest;
import com.hampcode.pagoya.billing.service.IBillPaymentService;
import com.hampcode.pagoya.shared.pagination.PageResponse;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/bill-payments")
@RequiredArgsConstructor
@Tag(name = "Bill Payments", description = "Pagos de servicios del cliente")
public class BillPaymentController {

    private final IBillPaymentService billPaymentService;

    @Operation(summary = "Registrar un pago")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Pago registrado"),
        @ApiResponse(responseCode = "400", description = "Datos invalidos o pago duplicado"),
        @ApiResponse(responseCode = "404", description = "Cliente o proveedor no encontrado")
    })
    @PostMapping
    public ResponseEntity<BillPaymentResponse> pay(
            @Valid @RequestBody CreateBillPaymentRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(billPaymentService.pay(request));
    }

    @Operation(summary = "Listar pagos de un cliente (paginado)")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Lista paginada"),
        @ApiResponse(responseCode = "404", description = "Cliente no encontrado")
    })
    @GetMapping("/customer/{customerId}")
    public ResponseEntity<PageResponse<BillPaymentResponse>> findByCustomer(
            @PathVariable Long customerId,
            @PageableDefault(size = 10, sort = "id") Pageable pageable) {
        return ResponseEntity.ok(
            PageResponse.from(billPaymentService.findByCustomer(customerId, pageable)));
    }
}
```

[↑ Volver al indice](#indice)

---

## 12. Implementar el reporte por categoria (US-B04)

Vas a agregar un endpoint que cruza las dos tablas via stored procedure de PostgreSQL.

### 12.1 Cargar el stored procedure

Todos los stored procedures del proyecto estan en:

- `src/main/resources/db/reports.sql`

Alli esta `sp_payments_by_category` (la que necesita este endpoint) junto con otras funciones de reporte. Cargalo despues del primer arranque, cuando Hibernate ya creo las tablas:

**Opcion A — pgAdmin**

1. Tools → Query Tool sobre `pagoya_db`.
2. **Open File** → selecciona `src/main/resources/db/reports.sql`.
3. `F5` para ejecutar. Veras varios `CREATE FUNCTION` en el output.

**Opcion B — psql**

```bash
psql -h localhost -p 55432 -U postgres -d pagoya_db \
     -f src/main/resources/db/reports.sql
```

La funcion que usa este endpoint:

```sql
CREATE OR REPLACE FUNCTION sp_payments_by_category(p_customer_id BIGINT)
RETURNS TABLE(category VARCHAR, total_count BIGINT, total_amount NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT sp.category::VARCHAR,
           COUNT(bp.id)::BIGINT,
           COALESCE(SUM(bp.amount), 0)::NUMERIC
    FROM bill_payments bp
    JOIN service_providers sp ON sp.id = bp.provider_id
    WHERE bp.customer_id = p_customer_id
    GROUP BY sp.category
    ORDER BY total_amount DESC;
END;
$$ LANGUAGE plpgsql;
```

### 12.2 `dto/PaymentByCategoryResponse.java`

DTO de salida del reporte. Una fila por categoria con su total.

```java
package com.hampcode.pagoya.billing.dto;

import java.math.BigDecimal;

public record PaymentByCategoryResponse(
    String category,
    long totalCount,
    BigDecimal totalAmount
) {}
```

### 12.3 Sumar al `BillPaymentRepository`

Agrega al repositorio una query nativa que invoca al stored procedure.

```java
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.List;

@Query(value = "SELECT * FROM sp_payments_by_category(:customerId)", nativeQuery = true)
List<Object[]> getPaymentsByCategory(@Param("customerId") Long customerId);
```

### 12.4 Sumar al `IBillPaymentService` y a su implementacion

Se agrega un metodo que valida que el cliente exista, ejecuta el stored procedure y mapea cada fila (`Object[]`) al DTO.

En la interface:

```java
List<PaymentByCategoryResponse> reportByCategory(Long customerId);
```

En la implementacion:

```java
@Override
@Transactional(readOnly = true)
public List<PaymentByCategoryResponse> reportByCategory(Long customerId) {
    if (!customerRepository.existsById(customerId)) {
        throw new ResourceNotFoundException("cliente no encontrado");
    }
    return billPaymentRepository.getPaymentsByCategory(customerId).stream()
        .map(r -> new PaymentByCategoryResponse(
            (String) r[0],
            ((Number) r[1]).longValue(),
            (BigDecimal) r[2]))
        .toList();
}
```

### 12.5 Sumar al `BillPaymentController`

Endpoint nuevo que sirve el reporte.

```java
@Operation(summary = "Reporte de pagos por categoria")
@GetMapping("/customer/{customerId}/by-category")
public ResponseEntity<List<PaymentByCategoryResponse>> reportByCategory(
        @PathVariable Long customerId) {
    return ResponseEntity.ok(billPaymentService.reportByCategory(customerId));
}
```

[↑ Volver al indice](#indice)

---

## 13. Configurar Swagger / OpenAPI

### 13.1 Verificar dependencias del proyecto

El proyecto ya trae las dependencias clave en `pom.xml`. Solo necesitas agregar la de Swagger.

| Dependencia | Para que sirve | Estado |
|---|---|---|
| `lombok` | Genera getters, setters y constructores automaticamente. | Ya en pom.xml |
| `mapstruct` + `mapstruct-processor` | Genera la implementacion del mapper en compile-time. | Ya en pom.xml |
| `springdoc-openapi-starter-webmvc-ui` | Activa Swagger UI / OpenAPI en la API. | **Hay que agregarla** |

### 13.2 Agregar la dependencia de Swagger al `pom.xml`

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

Despues de pegarla **guarda el archivo (`Cmd/Ctrl + S`)**. En IntelliJ aparecera un boton flotante **"Load Maven Changes"** (o el icono de Maven en la esquina inferior derecha). Click ahi para que descargue la nueva dependencia. En VSCode con la extension de Java: click derecho sobre `pom.xml` → **Reload Project**.

### 13.3 Crear `shared/config/OpenApiConfig.java`

```java
package com.hampcode.pagoya.shared.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI pagoyaOpenAPI() {
        return new OpenAPI().info(new Info()
            .title("PagoYa API").version("v1")
            .description("API de la billetera digital PagoYa"));
    }
}
```

### 13.4 URLs tras arrancar

| URL | Para que |
|---|---|
| `http://localhost:8080/swagger-ui/index.html` | UI para probar endpoints |
| `http://localhost:8080/v3/api-docs` | Spec OpenAPI en JSON |

[↑ Volver al indice](#indice)

---

## 14. Probar la API en Postman

En la seccion 0 ya importaste la coleccion `PagoYa API`. Ahora vas a agregarle el folder y los requests de `billing`.

1. En Postman, abre la coleccion `PagoYa API`.
2. Click derecho sobre la coleccion → **Add folder** → nombre: `Billing`.
3. Dentro del folder, agrega estas 4 requests (solo nombre, metodo y URL — el body lo armas guiandote por Swagger UI):

| Nombre | Metodo | URL |
|---|---|---|
| Listar proveedores | GET | `http://localhost:8080/api/service-providers` |
| Pagar un servicio | POST | `http://localhost:8080/api/bill-payments` |
| Listar mis pagos | GET | `http://localhost:8080/api/bill-payments/customer/{customerId}` |
| Reporte por categoria | GET | `http://localhost:8080/api/bill-payments/customer/{customerId}/by-category` |

4. Guarda los cambios con `Cmd/Ctrl + S`.

[↑ Volver al indice](#indice)

---

## 15. Commit y Pull Request

### 15.1 Commit y push

```bash
git add src/main/java/com/hampcode/pagoya/billing \
        src/main/java/com/hampcode/pagoya/shared/config/OpenApiConfig.java \
        pom.xml
git commit -m "feat(billing): pago de servicios con catalogo y reporte por categoria"
git push -u origin feature/billing-context
```

### 15.2 Abrir el PR en GitHub

Crea el Pull Request: `feature/billing-context` → `develop`.

**Titulo sugerido:**

```
feat(billing): pago de servicios (catalogo + pagos + reporte por categoria)
```

**Descripcion sugerida:**

```markdown
## Que entrega

Bounded context `billing/` con dos agregados:
- ServiceProvider: catalogo de proveedores (solo lectura).
- BillPayment: pagos del cliente (crear, listar paginado, reporte por categoria).

## Endpoints

- GET    /api/service-providers
- POST   /api/bill-payments
- GET    /api/bill-payments/customer/{id}
- GET    /api/bill-payments/customer/{id}/by-category   (stored procedure)

## Reglas de negocio cubiertas

- RN-B01: solo proveedores activos aceptan pagos.
- RN-B02: monto entre 0.01 y 5000.00 (Bean Validation).
- RN-B03: no se permiten pagos duplicados (mismo billCode al mismo proveedor).
- RN-B04 / RN-B05: paidAt y status PAID los pone el sistema.

## Como probarlo

- docker compose up -d
- mvn spring-boot:run
- Swagger UI: http://localhost:8080/swagger-ui/index.html
- Coleccion de Postman "PagoYa API → Billing".

## Documentacion

- Swagger / OpenAPI activado con springdoc-openapi 2.7.0.
```

[↑ Volver al indice](#indice)

---

## 16. Comandos de Git: cuando usar cada uno

Resumen practico — solo recomendaciones de cuando elegir cada comando.

| Comando | Cuando usarlo |
|---|---|
| `git status` | A cada paso, para saber donde estas (archivos modificados, en staging, sin trackear). |
| `git diff` | Antes de commitear, para revisar linea por linea lo que vas a subir. |
| `git fetch` | Cuando quieres ver que hay nuevo en el remoto SIN aplicarlo todavia a tu rama. Tu codigo local NO se modifica. |
| `git pull` | Cuando solo necesitas estar al dia y confias en mergear directo. Equivale a `git fetch` + `git merge`. |
| `git merge` | Para unir otra rama con la tuya. Tipico: traer `develop` a tu feature antes de abrir el PR para evitar conflictos al final. |
| `git switch` | Para cambiar de rama. Es la version moderna y mas legible que `git checkout`. |
| `git push` | Despues de tus commits, para subir tu rama al remoto y poder abrir el PR en GitHub. |
| `git log` | Para revisar el historial de commits: quien hizo que, cuando y en que rama. |
| `git stash` | Cuando necesitas cambiar de rama urgente y no quieres perder lo que estas haciendo (sin commitear). |

### 16.1 Diferencia clave entre `fetch` y `pull`

`fetch` solo TRAE informacion del remoto a tu copia local de las ramas remotas. NO toca tu rama de trabajo.

`pull` trae Y aplica los cambios. Es la suma de `fetch` + `merge`.

Usa `fetch` cuando quieras inspeccionar antes de mezclar; usa `pull` cuando solo necesites estar al dia.

### 16.2 Resolver conflictos de merge

Pasa cuando dos personas tocan la misma linea. `git merge` o `git pull` no puede decidir y deja el archivo en conflicto. Sigue estos pasos:

| Paso | Comando | Descripcion |
|---|---|---|
| 1 | `git status` | Lista los archivos en conflicto (aparecen con `UU`). |
| 2 | (editar archivo) | Abre cada archivo. Veras los marcadores `<<<<<<<`, `=======`, `>>>>>>>` delimitando los dos lados. Deja el codigo final correcto y borra los marcadores. |
| 3 | `git add <archivo>` | Marca cada archivo como resuelto. |
| 4 | `git commit` | Finaliza el merge con un mensaje automatico. |
| 5 | `git push` | Sube los cambios al remoto. |
| — | `git merge --abort` | Si te enredas, aborta el merge y vuelve al estado anterior. |

### 16.3 Pasar un feature a `main` (release)

Cuando el feature ya fue mergeado a `develop` y esta listo para produccion, un integrante crea una rama `release/*` y abre el PR a `main`.

| Paso | Comando | Descripcion |
|---|---|---|
| 1 | `git checkout develop` | Posicionarse en develop. |
| 2 | `git pull origin develop` | Traer la version mas reciente. |
| 3 | `git checkout -b release/v1.X.0` | Crear la rama de release desde develop. |
| 4 | `git push -u origin release/v1.X.0` | Subir la rama al remoto. |
| 5 | (en GitHub) | Abrir Pull Request `release/v1.X.0` → `main`. Esperar revision y aprobar. |
| 6 | `git checkout main` | Despues del merge, cambiar a main local. |
| 7 | `git pull origin main` | Traer el merge recien aplicado. |
| 8 | `git tag -a v1.X.0 -m "Release v1.X.0"` | Crear el tag de la version. |
| 9 | `git push origin v1.X.0` | Subir el tag al remoto. |

### 16.4 Arreglar una falla en `main` (hotfix)

Cuando la version en `main` tiene un bug que NO puede esperar al proximo release, se crea una rama `hotfix/*` desde main, se corrige y se merge de vuelta a `main` y a `develop`.

| Paso | Comando | Descripcion |
|---|---|---|
| 1 | `git checkout main` | Posicionarse en main (donde esta la falla). |
| 2 | `git pull origin main` | Traer la version actual. |
| 3 | `git checkout -b hotfix/descripcion-corta` | Crear la rama de hotfix DESDE main. |
| 4 | (corregir el bug + commit) | Hacer la correccion en pocos archivos. Commit con mensaje claro: `fix(...)...`. |
| 5 | `git push -u origin hotfix/descripcion-corta` | Subir la rama al remoto. |
| 6 | (en GitHub) | Abrir PR `hotfix/...` → `main`. Aprobar y mergear. |
| 7 | `git checkout main` y `git pull origin main` | Traer el hotfix mergeado. |
| 8 | `git tag -a v1.X.1 -m "Hotfix v1.X.1"` y `git push origin v1.X.1` | Taggear con version de parche. |
| 9 | `git checkout develop` y `git pull origin develop` | Cambiar a develop. |
| 10 | `git merge main` | Traer el hotfix tambien a develop (asi no se pierde en el proximo release). |
| 11 | `git push origin develop` | Subir develop actualizado. |

Resumen de las dos diferencias importantes:

| Caso | Rama base | Donde merger al final |
|---|---|---|
| Feature normal | `develop` | `develop` (luego release a `main`) |
| Hotfix | `main` | `main` Y tambien `develop` |

### 16.5 `git rebase`: mantener tu feature al dia con historial limpio

Alternativa a `git merge` cuando quieres traer cambios de `develop` a tu feature SIN crear un commit de merge. `rebase` toma tus commits y los reaplica encima de la version actual de `develop`. El historial queda lineal y mas facil de leer.

| Paso | Comando | Descripcion |
|---|---|---|
| 1 | `git fetch origin` | Trae los cambios del remoto sin aplicar. |
| 2 | `git checkout feature/billing-context` | Posicionarse en tu feature. |
| 3 | `git rebase origin/develop` | Reaplica tus commits encima de la ultima version de develop. |
| 4 | (resolver conflictos) | Si aparecen, los resuelves como en 16.2 y luego `git rebase --continue`. |
| 5 | `git rebase --abort` | Si te enredas, aborta y vuelves al estado anterior. |
| 6 | `git push --force-with-lease` | Si ya habias pusheado: rebase reescribe tus commits, `--force-with-lease` evita pisar trabajo de otros. |

**Cuando usar rebase (en lugar de `merge`)**:
- Antes de abrir el PR, para que tu feature aparezca limpio sobre `develop`.
- Cuando el equipo prefiere historial lineal.

**Cuando NO usar rebase**:
- En ramas compartidas (`develop`, `main`) — reescribe historia y rompe la rama de los demas.
- Despues de que otros ya jalaron tu rama.

Regla simple: rebase es seguro mientras la rama es **tuya y solo tuya**.

### 16.6 Que hacer si un PR fue enviado de `feature/*` directo a `main`

Pasa cuando un integrante se equivoca de rama base y abre el Pull Request contra `main` en lugar de `develop`. Tienes dos casos: que aun NO se haya mergeado, o que ya se haya mergeado por error.

**Caso A — el PR esta abierto (todavia NO se merge)**

| Paso | Comando / accion | Descripcion |
|---|---|---|
| 1 | (en GitHub) | Abrir el PR. En el dropdown `base: main` cambiar a `base: develop`. GitHub recalcula los cambios automaticamente. |
| 2 | (en GitHub) | Pedir revision, aprobar y mergear como un PR normal a `develop`. |

**Caso B — el PR ya fue mergeado a `main` por error**

El feature quedo en `main` pero NO en `develop`. Hay que sincronizar para que el proximo release no genere conflicto y el feature siga vivo en la rama de integracion.

| Paso | Comando | Descripcion |
|---|---|---|
| 1 | `git checkout develop` | Posicionarse en develop. |
| 2 | `git pull origin develop` | Asegurar que esta al dia. |
| 3 | `git merge main` | Traer los commits que se mergearon en main. |
| 4 | (resolver conflictos si aparecen) | Igual que en 16.2. |
| 5 | `git push origin develop` | Subir develop sincronizado. |

Para evitar que vuelva a pasar, configura en GitHub una **branch protection rule** sobre `main` que prohiba merges directos desde ramas que no sean `release/*` o `hotfix/*` (ver 16.8).

### 16.7 Roles tipicos en el equipo

GitFlow funciona mejor cuando los roles estan claros. Cualquier integrante puede tomar mas de uno segun el tamano del equipo.

| Rol | Responsabilidades | Permisos en GitHub |
|---|---|---|
| **Developer** | Implementa features. Abre PRs hacia `develop`. Atiende los comentarios del review. | Write |
| **Reviewer** | Revisa PRs de sus companeros, aprueba o pide cambios. Cualquier developer puede serlo. | Write |
| **Tech Lead / Maintainer** | Aprueba PRs criticos, mergea a `develop`, vela por la calidad del codigo. | Maintain |
| **Release Manager** | Crea ramas `release/*`, mergea a `main`, taggea las versiones. | Maintain o Admin |
| **Admin** | Configura branch protection, secrets, integraciones (CI, Sonar, etc.). | Admin |

### 16.8 Automatizar el rechazo de PRs no permitidos a `main`

Dos formas de bloquear automaticamente que alguien envie un PR de `feature/*` directo a `main`.

**Opcion A — Branch protection rule (sin codigo, desde la UI de GitHub)**

En el repo: **Settings → Branches → Add branch protection rule**.

| Configuracion | Para que sirve |
|---|---|
| Branch name pattern: `main` | Aplica la regla solo a main. |
| Require a pull request before merging | Prohibe push directo a main. |
| Require approvals (al menos 1) | Necesita revision antes de mergear. |
| Require status checks to pass | Los tests / build deben pasar antes de mergear. |
| Restrict who can push to matching branches | Solo el tech lead o release manager. |

Esto evita push directo a main pero NO restringe la rama origen del PR. Para esto ultimo, suma la Opcion B.

**Opcion B — GitHub Actions que valida la rama origen**

Crea `.github/workflows/validate-pr-source.yml`:

```yaml
name: Validate PR source branch
on:
  pull_request:
    branches: [main]
jobs:
  check-source:
    runs-on: ubuntu-latest
    steps:
      - name: Reject if not release/* or hotfix/*
        run: |
          BRANCH="${{ github.head_ref }}"
          if [[ ! "$BRANCH" =~ ^(release|hotfix)/ ]]; then
            echo "::error::PRs a main solo desde release/* o hotfix/*. Tu rama: $BRANCH"
            exit 1
          fi
```

Si alguien abre un PR de `feature/...` → `main`, este workflow falla y el PR queda bloqueado hasta que cambien la rama base a `develop`.

### 16.9 Automatizar el code review

Herramientas que reducen el trabajo manual de revisar PRs.

| Herramienta | Que hace | Documentacion |
|---|---|---|
| **GitHub Actions + JaCoCo** | Corre tests y mide cobertura en cada PR. Bloquea el merge si los tests fallan o la cobertura baja. | [GitHub Actions con Maven](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-java-with-maven) · [JaCoCo](https://www.jacoco.org/jacoco/) |
| **SonarCloud / SonarQube** | Analiza calidad del codigo: bugs, code smells, duplicacion, deuda tecnica. Deja comentarios en el PR. | [SonarQube Cloud](https://docs.sonarsource.com/sonarqube-cloud/) |
| **Spotless** (plugin Maven) | Formatea automaticamente codigo Java. Bloquea el PR si hay archivos sin formatear. | [Spotless plugin Maven](https://github.com/diffplug/spotless/tree/main/plugin-maven) |
| **CODEOWNERS** | Archivo que asigna reviewers automaticos segun los archivos modificados. | [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) |
| **Dependabot** | Crea PRs automaticos para actualizar dependencias vulnerables o desactualizadas. | [Dependabot docs](https://docs.github.com/en/code-security/dependabot) |
| **CodeRabbit** | IA que comenta el PR senalando posibles bugs, sugiriendo mejoras y resumiendo cambios. | [CodeRabbit](https://www.coderabbit.ai/) |
| **Copilot Code Review** | Revision automatizada de PRs con IA integrada en GitHub. | [Copilot Code Review](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review) |

Lo minimo recomendable para arrancar:
- **Required status checks** en `main` y `develop`: tests + build deben pasar.
- **CODEOWNERS** para que cada PR asigne reviewer automaticamente.
- **Dependabot** activado (es gratis en GitHub, dos clicks para configurar).

[↑ Volver al indice](#indice)

---

## 17. Tarea propuesta

Agregar al MISMO dominio `billing/` una tercera entidad: **`RecurringBillPayment`** (pago recurrente programado).

### Justificacion

Es una entidad con peso propio — no puede ser un campo:

- Tiene **estado**: `ACTIVE`, `PAUSED`, `CANCELLED`.
- Tiene **scheduling**: frecuencia (`MONTHLY`/`WEEKLY`), dia del mes/semana, `nextRunAt`.
- Tiene **lifecycle**: programar, pausar, reanudar, cancelar.
- Genera **multiples ejecuciones**: cada run real crea un `BillPayment`.

### Modelo

`RecurringBillPayment` se agrega al MISMO dominio `billing/`. El paquete pasa a tener TRES modelos: `ServiceProvider`, `BillPayment` y `RecurringBillPayment`.

```
[ customer ]                  [ billing ]                         [ billing ]

   Customer 1 ───<  *  RecurringBillPayment  *  >───  1  ServiceProvider
                        - id
                        - billCode
                        - amount
                        - frequency       (MONTHLY / WEEKLY)
                        - dayOfMonth      (1-28, si MONTHLY)
                        - dayOfWeek       (1-7, si WEEKLY)
                        - status          (ACTIVE / PAUSED / CANCELLED)
                        - nextRunAt
                        - createdAt
```

### Endpoints a implementar

| Metodo | URL | Que hace | Status |
|---|---|---|---|
| POST | `/api/recurring-bill-payments` | Programar un pago recurrente | `201` |
| GET | `/api/recurring-bill-payments/customer/{id}` | Listar los recurrentes del cliente | `200` |
| PATCH | `/api/recurring-bill-payments/{id}/pause` | Pausar | `200` |
| PATCH | `/api/recurring-bill-payments/{id}/resume` | Reanudar | `200` |
| DELETE | `/api/recurring-bill-payments/{id}` | Cancelar | `204` |

### Reglas de negocio

| Codigo | Regla |
|---|---|
| **RN-R01** | Solo se puede programar a un proveedor activo. |
| **RN-R02** | Si la frecuencia es mensual, se debe indicar el dia del mes (1 al 28). |
| **RN-R03** | Si la frecuencia es semanal, se debe indicar el dia de la semana (lunes a domingo). |
| **RN-R04** | Solo se puede pausar un pago que este activo. |
| **RN-R05** | Solo se puede reanudar un pago que este pausado. |
| **RN-R06** | No se puede modificar un pago que ya esta cancelado. |

### Lo que vas a practicar

- Agregar una **tercera clase** al mismo bounded context (justifica el nombre `billing/`).
- Modelar **enums** (`RecurringStatus`, `RecurringFrequency`).
- Validar **transiciones de estado** en el service.
- Endpoints **PATCH** para acciones (`pause`, `resume`).
- Excepciones especificas (ej: `InvalidStatusTransitionException`).

### Reto extra (opcional)

- Implementar el **scheduler real** con `@Scheduled` de Spring: cada hora el sistema busca recurrentes con `nextRunAt <= now()` y crea un `BillPayment` real.
- Calcular `nextRunAt` correctamente al pausar y reanudar.

### Entrega

Crea la rama `feature/recurring-bill-payments`, implementa la tarea, abre PR a `develop`. Pidele a un companero que lo revise.

[↑ Volver al indice](#indice)
