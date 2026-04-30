# Guia rapida: Bounded Context `saving-goal`

Metas de ahorro personales del cliente. Sirve para practicar **DTO + MapStruct + validacion + paginacion** sobre la base ya implementada en el laboratorio.

---

## 0. Contexto del feature

### 0.1 Enunciado

En PagoYa cada cliente puede registrar **metas de ahorro** (un viaje, una laptop, un regalo) con un nombre, un monto objetivo y una fecha limite. Las metas son privadas por cliente y se consultan paginadas.

### 0.2 User Stories

---

#### US-S01 — Crear una meta de ahorro

**Como** cliente de PagoYa
**quiero** registrar una meta con nombre, monto objetivo y fecha limite
**para** organizar el dinero que quiero juntar.

**Criterios de aceptacion**

- **CA-01.1**
  **Dado** que soy un cliente registrado
  **cuando** registro una meta con datos validos
  **entonces** la meta queda guardada y veo la confirmacion del registro.

- **CA-01.2**
  **Dado** que estoy registrando una meta
  **cuando** el monto objetivo es 0, negativo o supera 1 000 000
  **entonces** la operacion se rechaza con un mensaje claro.

- **CA-01.3**
  **Dado** que estoy registrando una meta
  **cuando** la fecha limite es hoy o anterior
  **entonces** la operacion se rechaza indicando que la fecha debe ser futura.

- **CA-01.4**
  **Dado** que el cliente al que se asocia la meta no existe
  **cuando** intento crearla
  **entonces** el sistema informa que el cliente no fue encontrado.

- **CA-01.5**
  **Dado** que se crea una meta correctamente
  **cuando** se confirma la operacion
  **entonces** queda registrada automaticamente la fecha y hora de creacion (yo no la informo).

---

#### US-S02 — Listar mis metas

**Como** cliente
**quiero** ver mis metas paginadas
**para** revisar mi progreso de ahorro.

**Criterios de aceptacion**

- **CA-02.1**
  **Dado** que tengo varias metas
  **cuando** consulto mi lista
  **entonces** veo mis metas paginadas con total de elementos y total de paginas.

- **CA-02.2**
  **Dado** que aun no he creado ninguna meta
  **cuando** consulto mi lista
  **entonces** veo una lista vacia (no un error).

- **CA-02.3**
  **Dado** que existen metas de varios clientes
  **cuando** consulto mi lista
  **entonces** solo veo las mias.

---

#### US-S03 — Eliminar una meta

**Como** cliente
**quiero** eliminar una meta que ya no necesito
**para** mantener mi lista relevante.

**Criterios de aceptacion**

- **CA-03.1**
  **Dado** que tengo una meta guardada
  **cuando** solicito eliminarla
  **entonces** la meta desaparece y la operacion se confirma sin contenido adicional.

- **CA-03.2**
  **Dado** que la meta no existe
  **cuando** envio la solicitud de eliminacion
  **entonces** el sistema informa que la meta no fue encontrada.

### 0.3 Reglas de negocio

| Codigo     | Regla                                                                                              |
| ---------- | -------------------------------------------------------------------------------------------------- |
| **RN-S01** | Un cliente no puede tener dos metas con el mismo nombre.                                            |
| **RN-S02** | El nombre es obligatorio y no puede superar los 50 caracteres.                                      |
| **RN-S03** | El monto objetivo debe ser mayor a 0 y no superar 1 000 000.                                        |
| **RN-S04** | La fecha limite debe ser estrictamente futura (posterior al dia de creacion).                       |
| **RN-S05** | Solo se pueden registrar metas a nombre de clientes que existen en PagoYa.                          |
| **RN-S06** | La fecha y hora de creacion la registra automaticamente PagoYa; el cliente no la informa.           |

### 0.4 Modelo del dominio (resumen)

```
Customer 1 ───< * SavingGoal
                 - id
                 - name            (max 50, unico por cliente)
                 - targetAmount    (> 0, max 1 000 000)
                 - deadline        (fecha futura)
                 - createdAt
```

> Un `Customer` puede tener muchas `SavingGoal`. Cada `SavingGoal` pertenece a **exactamente un** `Customer` (`@ManyToOne`).

---

## 1. Crear rama

```bash
git checkout develop
git checkout -b feature/saving-goal-context
```

---

## 2. Estructura a crear

Dentro de `src/main/java/com/hampcode/pagoya/` se crea el paquete `savinggoal/` con la misma estructura que los otros bounded contexts:

```
savinggoal/
|-- controller/   <- SavingGoalController
|-- service/      <- ISavingGoalService, SavingGoalService
|-- repository/   <- SavingGoalRepository
|-- model/        <- SavingGoal
|-- dto/          <- CreateSavingGoalRequest, SavingGoalResponse
|-- mapper/       <- SavingGoalMapper
`-- exception/    <- DuplicateGoalNameException
```

---

## 3. Archivos (con su ubicacion exacta)

### 3.1 `model/SavingGoal.java`
```java
package com.hampcode.pagoya.savinggoal.model;

import com.hampcode.pagoya.customer.model.Customer;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "saving_goals",
       uniqueConstraints = @UniqueConstraint(columnNames = {"customer_id","name"}))
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class SavingGoal {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @Column(nullable = false, length = 50)
    private String name;

    @Column(name = "target_amount", nullable = false, precision = 12, scale = 2)
    private BigDecimal targetAmount;

    @Column(nullable = false)
    private LocalDate deadline;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
}
```
> El cliente se referencia con `@ManyToOne` a la entidad `Customer` (no con `Long`). JPA crea la columna `customer_id` como FK real. La restriccion `unique(customer_id, name)` apoya la RN-S01 a nivel de base de datos.

### 3.2 `repository/SavingGoalRepository.java`
```java
package com.hampcode.pagoya.savinggoal.repository;

import com.hampcode.pagoya.savinggoal.model.SavingGoal;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface SavingGoalRepository extends JpaRepository<SavingGoal, Long> {
    boolean existsByCustomer_IdAndName(Long customerId, String name);
    Page<SavingGoal> findByCustomer_Id(Long customerId, Pageable pageable);
}
```
> La sintaxis `Customer_Id` le dice a Spring Data JPA que navegue de `savingGoal.customer.id` (es lo mismo que escribir `WHERE g.customer.id = ?`).

### 3.3 DTOs — `dto/CreateSavingGoalRequest.java`
```java
package com.hampcode.pagoya.savinggoal.dto;

import jakarta.validation.constraints.*;

import java.math.BigDecimal;
import java.time.LocalDate;

public record CreateSavingGoalRequest(
    @NotNull(message = "el customerId es obligatorio")
    Long customerId,

    @NotBlank(message = "el nombre es obligatorio")
    @Size(max = 50, message = "el nombre no puede exceder 50 caracteres")
    String name,

    @NotNull(message = "el monto objetivo es obligatorio")
    @DecimalMin(value = "0.01", message = "el monto objetivo debe ser mayor a 0")
    @DecimalMax(value = "1000000.00", message = "el monto objetivo no puede superar 1 000 000")
    BigDecimal targetAmount,

    @NotNull(message = "la fecha limite es obligatoria")
    @Future(message = "la fecha limite debe ser futura")
    LocalDate deadline
) {}
```
> Aqui practicamos validaciones variadas: texto (`@NotBlank`, `@Size`), numero (`@DecimalMin`, `@DecimalMax`) y fecha (`@Future`). El framework las valida automaticamente cuando el controller usa `@Valid`.

### 3.4 DTOs — `dto/SavingGoalResponse.java`
```java
package com.hampcode.pagoya.savinggoal.dto;

import java.math.BigDecimal;
import java.time.LocalDate;

public record SavingGoalResponse(
    Long id,
    String name,
    BigDecimal targetAmount,
    LocalDate deadline
) {}
```
> Notar: **no expone `customerId` ni `createdAt`** porque el cliente no los necesita ver. Esto es justo el sentido de tener Response DTO.

### 3.5 `mapper/SavingGoalMapper.java`
```java
package com.hampcode.pagoya.savinggoal.mapper;

import com.hampcode.pagoya.savinggoal.dto.CreateSavingGoalRequest;
import com.hampcode.pagoya.savinggoal.dto.SavingGoalResponse;
import com.hampcode.pagoya.savinggoal.model.SavingGoal;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface SavingGoalMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "customer", ignore = true)   // lo carga el service desde CustomerRepository
    @Mapping(target = "createdAt", ignore = true)
    SavingGoal toEntity(CreateSavingGoalRequest request);

    SavingGoalResponse toResponse(SavingGoal goal);
}
```
> El mapper es un **bean Spring** (`@Mapper(componentModel = "spring")`). MapStruct genera la implementacion en `target/generated-sources/`. Como `SavingGoal.customer` es una entidad (`Customer`), el mapper la **ignora**: el service hace `customerRepository.findById(...)` y la asigna antes de guardar.

### 3.6 Excepciones — `exception/DuplicateGoalNameException.java`
```java
package com.hampcode.pagoya.savinggoal.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class DuplicateGoalNameException extends BusinessRuleException {
    public DuplicateGoalNameException(String name) {
        super("ya tienes una meta con el nombre '" + name + "'");
    }
}
```

### 3.7 `service/ISavingGoalService.java`
```java
package com.hampcode.pagoya.savinggoal.service;

import com.hampcode.pagoya.savinggoal.dto.CreateSavingGoalRequest;
import com.hampcode.pagoya.savinggoal.dto.SavingGoalResponse;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ISavingGoalService {
    SavingGoalResponse create(CreateSavingGoalRequest request);
    Page<SavingGoalResponse> findByCustomer(Long customerId, Pageable pageable);
    void delete(Long id);
}
```

### 3.8 `service/SavingGoalService.java`
```java
package com.hampcode.pagoya.savinggoal.service;

import com.hampcode.pagoya.customer.model.Customer;
import com.hampcode.pagoya.customer.repository.CustomerRepository;
import com.hampcode.pagoya.savinggoal.dto.CreateSavingGoalRequest;
import com.hampcode.pagoya.savinggoal.dto.SavingGoalResponse;
import com.hampcode.pagoya.savinggoal.exception.DuplicateGoalNameException;
import com.hampcode.pagoya.savinggoal.mapper.SavingGoalMapper;
import com.hampcode.pagoya.savinggoal.model.SavingGoal;
import com.hampcode.pagoya.savinggoal.repository.SavingGoalRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
public class SavingGoalService implements ISavingGoalService {

    private final SavingGoalRepository savingGoalRepository;
    private final CustomerRepository customerRepository;
    private final SavingGoalMapper savingGoalMapper;

    @Override
    @Transactional
    public SavingGoalResponse create(CreateSavingGoalRequest request) {
        // 1) cargar el Customer (existe?) para asociarlo a la entidad
        Customer customer = customerRepository.findById(request.customerId())
            .orElseThrow(() -> new ResourceNotFoundException(
                "cliente con id " + request.customerId() + " no encontrado"));

        // RN-S01: nombre unico por cliente
        if (savingGoalRepository.existsByCustomer_IdAndName(customer.getId(), request.name()))
            throw new DuplicateGoalNameException(request.name());

        SavingGoal entity = savingGoalMapper.toEntity(request);
        entity.setCustomer(customer);                  // ← asocia la entidad real
        entity.setCreatedAt(LocalDateTime.now());
        return savingGoalMapper.toResponse(savingGoalRepository.save(entity));
    }

    @Override
    public Page<SavingGoalResponse> findByCustomer(Long customerId, Pageable pageable) {
        return savingGoalRepository.findByCustomer_Id(customerId, pageable)
            .map(savingGoalMapper::toResponse);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!savingGoalRepository.existsById(id))
            throw new ResourceNotFoundException("meta " + id + " no encontrada");
        savingGoalRepository.deleteById(id);
    }
}
```
> **Nota didactica**: el `SavingGoalMapper` *no* sabe convertir un `Long customerId` (DTO) a un `Customer` (entidad), porque no tiene acceso al `CustomerRepository`. Por eso el service:
> 1. Hace `customerRepository.findById(...)` y valida que exista.
> 2. Llama al mapper con `toEntity(request)` (sin el customer).
> 3. Asigna `entity.setCustomer(customer)` antes de `save`.
>
> Asi se respeta la separacion: **el mapper solo transforma estructura, el service decide reglas de negocio**.

### 3.9 `controller/SavingGoalController.java`
```java
package com.hampcode.pagoya.savinggoal.controller;

import com.hampcode.pagoya.savinggoal.dto.CreateSavingGoalRequest;
import com.hampcode.pagoya.savinggoal.dto.SavingGoalResponse;
import com.hampcode.pagoya.savinggoal.service.ISavingGoalService;
import com.hampcode.pagoya.shared.pagination.PageResponse;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/saving-goals")
@RequiredArgsConstructor
public class SavingGoalController {

    private final ISavingGoalService savingGoalService;

    @PostMapping
    public ResponseEntity<SavingGoalResponse> create(
            @Valid @RequestBody CreateSavingGoalRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(savingGoalService.create(request));
    }

    @GetMapping("/customer/{customerId}")
    public ResponseEntity<PageResponse<SavingGoalResponse>> findByCustomer(
            @PathVariable Long customerId,
            @PageableDefault(size = 10, sort = "id") Pageable pageable) {
        return ResponseEntity.ok(
            PageResponse.from(savingGoalService.findByCustomer(customerId, pageable)));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        savingGoalService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## 4. Probar en Postman

### 4.1 Variables sugeridas
Agregalas a la coleccion `pagoya-api` (o reusa las que ya tienes):

| key | value |
|---|---|
| `base_url` | `http://localhost:8080` |
| `customer_id` | `1` |
| `goal_id` | `1` |

### 4.2 Requests

#### POST — crear meta (camino feliz)
```
POST {{base_url}}/api/saving-goals
Content-Type: application/json
```
```json
{
  "customerId": {{customer_id}},
  "name": "Viaje a Cusco",
  "targetAmount": 2500.00,
  "deadline": "2026-12-31"
}
```
Esperado: `201 Created` + `SavingGoalResponse` (id, name, targetAmount, deadline).

#### POST — validacion fallida (RN-S02, RN-S03, RN-S04)
```
POST {{base_url}}/api/saving-goals
```
```json
{
  "customerId": {{customer_id}},
  "name": "",
  "targetAmount": 0,
  "deadline": "2020-01-01"
}
```
Esperado: `400 Bad Request` con `ErrorResponse` y `details` listando cada campo invalido.

#### POST — nombre duplicado (RN-S01)
Repetir el primer POST tal cual. Esperado: `400` con
`message: "ya tienes una meta con el nombre 'Viaje a Cusco'"`.

#### POST — cliente inexistente (RN-S05)
```json
{
  "customerId": 9999,
  "name": "Laptop nueva",
  "targetAmount": 4500.00,
  "deadline": "2026-10-01"
}
```
Esperado: `404 Not Found` con `message: "cliente con id 9999 no encontrado"`.

#### GET — listar paginado las metas del cliente
```
GET {{base_url}}/api/saving-goals/customer/{{customer_id}}?page=0&size=10
```
Esperado: `200 OK` con `PageResponse<SavingGoalResponse>` (`content`, `page`, `size`, `totalElements`, `totalPages`, `first`, `last`).

#### DELETE — eliminar una meta
```
DELETE {{base_url}}/api/saving-goals/{{goal_id}}
```
Esperado: `204 No Content` (sin body).

#### DELETE — meta inexistente
```
DELETE {{base_url}}/api/saving-goals/9999
```
Esperado: `404 Not Found` con `message: "meta 9999 no encontrada"`.

---

## 5. Compilar y arrancar

```bash
# 1) compilar
mvn clean compile

# 2) arrancar la app cargando las variables del .env
export $(grep -v '^#' .env | xargs) && mvn spring-boot:run
```

Hibernate creara la tabla `saving_goals` automaticamente al arrancar (`ddl-auto: update`).

---

## 6. Commit y Pull Request

```bash
git add src/main/java/com/hampcode/pagoya/savinggoal
git commit -m "feat(saving-goal): metas de ahorro con DTO, mapper y paginacion"
git push origin feature/saving-goal-context
```

Abrir el PR en GitHub: **`feature/saving-goal-context` → `develop`**.

Titulo sugerido:
> `feat(saving-goal): metas de ahorro del cliente`

Descripcion sugerida (breve):
- Nuevo bounded context `saving-goal`.
- Endpoints: `POST /api/saving-goals`, `GET /api/saving-goals/customer/{id}`, `DELETE /api/saving-goals/{id}`.
- Aplica DTO + MapStruct + Bean Validation + paginacion.
- Reglas: nombre unico por cliente (RN-S01), monto y fecha validados (RN-S03, RN-S04).
