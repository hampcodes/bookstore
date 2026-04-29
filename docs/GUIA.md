# Guia rapida: Bounded Context `beneficiary`

Contactos favoritos del cliente (como guardar un contacto en Yape/Plin). Sirve para practicar **DTO + MapStruct + validacion + paginacion** sobre la base ya implementada en el laboratorio.

---

## 0. Contexto del feature

### 0.1 Enunciado

En PagoYa los clientes hacen transferencias frecuentes a las mismas personas (familiares, amigos, roommates, proveedores). Hoy tienen que escribir el numero de cuenta cada vez, lo cual es tedioso y propenso a errores. Se necesita una **lista de contactos favoritos** (beneficiarios): el cliente guarda una vez la cuenta destino con un alias y un nombre de titular, y luego puede consultarla rapido para iniciar una transferencia. La lista es **privada por cliente** (solo el dueno la ve y la administra) y debe poder consultarse paginada para clientes con muchos contactos.

### 0.2 User Stories

---

#### US-B01 — Guardar un contacto favorito

**Como** cliente de PagoYa
**quiero** registrar la cuenta de alguien a quien le transfiero seguido, con un alias propio
**para** no tener que escribir el numero de cuenta en cada transferencia.

**Criterios de aceptacion**

- **CA-01.1**
  **Dado** que soy un cliente registrado en PagoYa
  **cuando** guardo un nuevo contacto con alias, numero de cuenta y nombre del titular validos
  **entonces** el contacto queda agregado a mi lista de favoritos y veo la confirmacion del registro.

- **CA-01.2**
  **Dado** que estoy registrando un nuevo contacto
  **cuando** ingreso un numero de cuenta con un formato que no corresponde al sistema
  **entonces** la operacion se rechaza y el sistema me indica que el numero de cuenta no es valido.

- **CA-01.3**
  **Dado** que estoy registrando un nuevo contacto
  **cuando** dejo el alias vacio o ingreso un nombre de titular demasiado largo
  **entonces** la operacion se rechaza y el sistema me indica cuales campos son invalidos.

- **CA-01.4**
  **Dado** que el cliente al que se intenta asociar el contacto no existe
  **cuando** intento guardar el contacto a su nombre
  **entonces** la operacion se rechaza y el sistema me informa que el cliente no fue encontrado.

- **CA-01.5**
  **Dado** que guardo un contacto correctamente
  **cuando** se confirma la operacion
  **entonces** queda registrada automaticamente la fecha y hora en que se creo (yo no las informo).

---

#### US-B02 — Listar mis contactos favoritos

**Como** cliente
**quiero** ver mis contactos guardados, paginados
**para** elegir rapido al destinatario de una nueva transferencia.

**Criterios de aceptacion**

- **CA-02.1**
  **Dado** que tengo varios contactos guardados
  **cuando** consulto mi lista de favoritos
  **entonces** veo mis contactos paginados, con cuantos hay en total y cuantas paginas existen.

- **CA-02.2**
  **Dado** que aun no he guardado ningun contacto
  **cuando** consulto mi lista de favoritos
  **entonces** veo una lista vacia (no recibo un error de no encontrado).

- **CA-02.3**
  **Dado** que existen varios clientes con contactos cada uno
  **cuando** consulto mi lista de favoritos
  **entonces** solo veo los mios y nunca los de otros clientes.

- **CA-02.4**
  **Dado** que consulto mi lista de favoritos
  **cuando** reviso cada contacto en la respuesta
  **entonces** veo el alias, el numero de cuenta y el nombre del titular, sin datos internos del sistema.

---

#### US-B03 — Eliminar un contacto que ya no uso

**Como** cliente
**quiero** eliminar un contacto que ya no necesito
**para** mantener mi lista corta y relevante.

**Criterios de aceptacion**

- **CA-03.1**
  **Dado** que tengo un contacto guardado en mi lista
  **cuando** solicito eliminarlo
  **entonces** el contacto desaparece de mi lista y la operacion se confirma sin contenido adicional.

- **CA-03.2**
  **Dado** que el contacto que intento eliminar no existe
  **cuando** envio la solicitud de eliminacion
  **entonces** el sistema me informa que el contacto no fue encontrado.

### 0.3 Reglas de negocio

| Codigo     | Regla                                                                                                               |
| ---------- | ------------------------------------------------------------------------------------------------------------------- |
| **RN-B01** | Un cliente no puede registrar dos veces el mismo numero de cuenta como contacto.                                    |
| **RN-B02** | Un cliente no puede registrarse a si mismo como contacto (la cuenta destino no puede ser una de sus propias cuentas). |
| **RN-B03** | Solo se pueden registrar contactos a nombre de clientes que existen en PagoYa.                                       |
| **RN-B04** | El numero de cuenta del contacto debe respetar el formato oficial de cuentas de PagoYa.                              |
| **RN-B05** | El alias del contacto es obligatorio y no puede superar los 30 caracteres.                                           |
| **RN-B06** | El nombre del titular del contacto es obligatorio y no puede superar los 100 caracteres.                             |
| **RN-B07** | La fecha y hora de creacion del contacto la registra automaticamente PagoYa; el cliente no la informa.               |

### 0.4 Modelo del dominio (resumen)

```
Customer 1 ───< * Beneficiary
                 - id
                 - alias            (max 30)
                 - accountNumber    (6-12 chars)
                 - ownerName        (max 100)
                 - createdAt
```

> Un `Customer` puede tener muchos `Beneficiary`. Cada `Beneficiary` pertenece a **exactamente un** `Customer` (`@ManyToOne`).

---

## 1. Crear rama

```bash
git checkout develop
git checkout -b feature/beneficiary-context
```

---

## 2. Estructura a crear

Dentro de `src/main/java/com/hampcode/pagoya/` se crea el paquete `beneficiary/` con la misma estructura que los otros bounded contexts:

```
beneficiary/
|-- controller/   <- BeneficiaryController
|-- service/      <- IBeneficiaryService, BeneficiaryService
|-- repository/   <- BeneficiaryRepository
|-- model/        <- Beneficiary
|-- dto/          <- CreateBeneficiaryRequest, BeneficiaryResponse
|-- mapper/       <- BeneficiaryMapper
`-- exception/    <- DuplicateBeneficiaryException, SelfBeneficiaryException
```

---

## 3. Archivos (con su ubicacion exacta)

### 3.1 `model/Beneficiary.java`
```java
package com.hampcode.pagoya.beneficiary.model;

import com.hampcode.pagoya.customer.model.Customer;
import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "beneficiaries",
       uniqueConstraints = @UniqueConstraint(columnNames = {"customer_id","account_number"}))
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Beneficiary {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @Column(nullable = false, length = 30)
    private String alias;

    @Column(name = "account_number", nullable = false, length = 12)
    private String accountNumber;

    @Column(name = "owner_name", nullable = false, length = 100)
    private String ownerName;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
}
```
> El cliente se referencia con `@ManyToOne` a la entidad `Customer` (no con `Long`). JPA crea la columna `customer_id` como FK real.

### 3.2 `repository/BeneficiaryRepository.java`
```java
package com.hampcode.pagoya.beneficiary.repository;

import com.hampcode.pagoya.beneficiary.model.Beneficiary;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface BeneficiaryRepository extends JpaRepository<Beneficiary, Long> {
    boolean existsByCustomer_IdAndAccountNumber(Long customerId, String accountNumber);
    Page<Beneficiary> findByCustomer_Id(Long customerId, Pageable pageable);
}
```
> La sintaxis `Customer_Id` le dice a Spring Data JPA que navegue de `beneficiary.customer.id` (es lo mismo que escribir `WHERE b.customer.id = ?`).

### 3.3 DTOs — `dto/CreateBeneficiaryRequest.java`
```java
package com.hampcode.pagoya.beneficiary.dto;

import jakarta.validation.constraints.*;

public record CreateBeneficiaryRequest(
    @NotNull(message = "el customerId es obligatorio")
    Long customerId,

    @NotBlank(message = "el alias es obligatorio")
    @Size(max = 30, message = "el alias no puede exceder 30 caracteres")
    String alias,

    @NotBlank(message = "la cuenta es obligatoria")
    @Pattern(regexp = "[A-Z0-9-]{6,12}",
             message = "la cuenta debe tener entre 6 y 12 caracteres (A-Z, 0-9, -)")
    String accountNumber,

    @NotBlank(message = "el nombre del titular es obligatorio")
    @Size(max = 100)
    String ownerName
) {}
```

### 3.4 DTOs — `dto/BeneficiaryResponse.java`
```java
package com.hampcode.pagoya.beneficiary.dto;

public record BeneficiaryResponse(
    Long id,
    String alias,
    String accountNumber,
    String ownerName
) {}
```
> Notar: **no expone `customerId` ni `createdAt`** porque el cliente no los necesita ver. Esto es justo el sentido de tener Response DTO.

### 3.5 `mapper/BeneficiaryMapper.java`
```java
package com.hampcode.pagoya.beneficiary.mapper;

import com.hampcode.pagoya.beneficiary.dto.BeneficiaryResponse;
import com.hampcode.pagoya.beneficiary.dto.CreateBeneficiaryRequest;
import com.hampcode.pagoya.beneficiary.model.Beneficiary;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface BeneficiaryMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "customer", ignore = true)   // lo carga el service desde CustomerRepository
    @Mapping(target = "createdAt", ignore = true)
    Beneficiary toEntity(CreateBeneficiaryRequest request);

    BeneficiaryResponse toResponse(Beneficiary b);
}
```
> El mapper es un **bean Spring** (`@Mapper(componentModel = "spring")`). MapStruct genera la implementacion en `target/generated-sources/`. Como `Beneficiary.customer` es una entidad (`Customer`), el mapper la **ignora**: el service hace `customerRepository.findById(...)` y la asigna antes de guardar.

### 3.6 Excepciones — `exception/DuplicateBeneficiaryException.java`
```java
package com.hampcode.pagoya.beneficiary.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class DuplicateBeneficiaryException extends BusinessRuleException {
    public DuplicateBeneficiaryException(String accountNumber) {
        super("la cuenta " + accountNumber + " ya esta en tus contactos");
    }
}
```

### 3.7 Excepciones — `exception/SelfBeneficiaryException.java`
```java
package com.hampcode.pagoya.beneficiary.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class SelfBeneficiaryException extends BusinessRuleException {
    public SelfBeneficiaryException() {
        super("no puedes guardarte a ti mismo como beneficiario");
    }
}
```

### 3.8 `service/IBeneficiaryService.java`
```java
package com.hampcode.pagoya.beneficiary.service;

import com.hampcode.pagoya.beneficiary.dto.BeneficiaryResponse;
import com.hampcode.pagoya.beneficiary.dto.CreateBeneficiaryRequest;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface IBeneficiaryService {
    BeneficiaryResponse create(CreateBeneficiaryRequest request);
    Page<BeneficiaryResponse> findByCustomer(Long customerId, Pageable pageable);
    void delete(Long id);
}
```

### 3.9 `service/BeneficiaryService.java`
```java
package com.hampcode.pagoya.beneficiary.service;

import com.hampcode.pagoya.account.repository.AccountRepository;
import com.hampcode.pagoya.account.repository.AccountRepository;
import com.hampcode.pagoya.beneficiary.dto.BeneficiaryResponse;
import com.hampcode.pagoya.beneficiary.dto.CreateBeneficiaryRequest;
import com.hampcode.pagoya.beneficiary.exception.DuplicateBeneficiaryException;
import com.hampcode.pagoya.beneficiary.exception.SelfBeneficiaryException;
import com.hampcode.pagoya.beneficiary.mapper.BeneficiaryMapper;
import com.hampcode.pagoya.beneficiary.model.Beneficiary;
import com.hampcode.pagoya.beneficiary.repository.BeneficiaryRepository;
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
public class BeneficiaryService implements IBeneficiaryService {

    private final BeneficiaryRepository beneficiaryRepository;
    private final CustomerRepository customerRepository;
    private final AccountRepository accountRepository;
    private final BeneficiaryMapper beneficiaryMapper;

    @Override
    @Transactional
    public BeneficiaryResponse create(CreateBeneficiaryRequest request) {
        // 1) cargar el Customer (existe?) para asociarlo a la entidad
        Customer customer = customerRepository.findById(request.customerId())
            .orElseThrow(() -> new ResourceNotFoundException(
                "cliente con id " + request.customerId() + " no encontrado"));

        // RN-B02: no puede guardarse a si mismo
        accountRepository.findByAccountNumber(request.accountNumber())
            .filter(a -> a.getCustomerId().equals(customer.getId()))
            .ifPresent(a -> { throw new SelfBeneficiaryException(); });

        // RN-B01: cuenta unica por cliente
        if (beneficiaryRepository.existsByCustomer_IdAndAccountNumber(
                customer.getId(), request.accountNumber()))
            throw new DuplicateBeneficiaryException(request.accountNumber());

        Beneficiary entity = beneficiaryMapper.toEntity(request);
        entity.setCustomer(customer);                  // ← asocia la entidad real
        entity.setCreatedAt(LocalDateTime.now());
        return beneficiaryMapper.toResponse(beneficiaryRepository.save(entity));
    }

    @Override
    public Page<BeneficiaryResponse> findByCustomer(Long customerId, Pageable pageable) {
        return beneficiaryRepository.findByCustomer_Id(customerId, pageable)
            .map(beneficiaryMapper::toResponse);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!beneficiaryRepository.existsById(id))
            throw new ResourceNotFoundException("beneficiario " + id + " no encontrado");
        beneficiaryRepository.deleteById(id);
    }
}
```
> **Nota didactica**: el `BeneficiaryMapper` *no* sabe convertir un `Long customerId` (DTO) a un `Customer` (entidad), porque no tiene acceso al `CustomerRepository`. Por eso el service:
> 1. Hace `customerRepository.findById(...)` y valida que exista.
> 2. Llama al mapper con `toEntity(request)` (sin el customer).
> 3. Asigna `entity.setCustomer(customer)` antes de `save`.
>
> Asi se respeta la separacion: **el mapper solo transforma estructura, el service decide reglas de negocio**.

### 3.10 `controller/BeneficiaryController.java`
```java
package com.hampcode.pagoya.beneficiary.controller;

import com.hampcode.pagoya.beneficiary.dto.BeneficiaryResponse;
import com.hampcode.pagoya.beneficiary.dto.CreateBeneficiaryRequest;
import com.hampcode.pagoya.beneficiary.service.IBeneficiaryService;
import com.hampcode.pagoya.shared.pagination.PageResponse;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/beneficiaries")
@RequiredArgsConstructor
public class BeneficiaryController {

    private final IBeneficiaryService beneficiaryService;

    @PostMapping
    public ResponseEntity<BeneficiaryResponse> create(
            @Valid @RequestBody CreateBeneficiaryRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(beneficiaryService.create(request));
    }

    @GetMapping("/customer/{customerId}")
    public ResponseEntity<PageResponse<BeneficiaryResponse>> findByCustomer(
            @PathVariable Long customerId,
            @PageableDefault(size = 10, sort = "id") Pageable pageable) {
        return ResponseEntity.ok(
            PageResponse.from(beneficiaryService.findByCustomer(customerId, pageable)));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        beneficiaryService.delete(id);
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
| `account_number` | `ACC-001` *(cuenta del propio cliente, para probar RN-B02)* |
| `target_account_number` | `ACC-002` *(cuenta de otra persona, valida)* |
| `beneficiary_id` | `1` |

### 4.2 Requests

#### POST — crear beneficiario (camino feliz)
```
POST {{base_url}}/api/beneficiaries
Content-Type: application/json
```
```json
{
  "customerId": {{customer_id}},
  "alias": "Mama",
  "accountNumber": "{{target_account_number}}",
  "ownerName": "Maria Torres"
}
```
Esperado: `201 Created` + `BeneficiaryResponse` (id, alias, accountNumber, ownerName).

#### POST — validacion fallida (RN-B04, RN-B05)
```
POST {{base_url}}/api/beneficiaries
```
```json
{
  "customerId": {{customer_id}},
  "alias": "",
  "accountNumber": "abc",
  "ownerName": ""
}
```
Esperado: `400 Bad Request` con `ErrorResponse` y `details` listando cada campo invalido.

#### POST — cuenta duplicada (RN-B01)
Repetir el primer POST tal cual. Esperado: `400` con
`message: "la cuenta ACC-002 ya esta en tus contactos"`.

#### POST — auto-beneficiario (RN-B02)
```json
{
  "customerId": {{customer_id}},
  "alias": "Yo mismo",
  "accountNumber": "{{account_number}}",
  "ownerName": "Yo"
}
```
Esperado: `400` con `message: "no puedes guardarte a ti mismo como beneficiario"`.

#### POST — cliente inexistente (RN-B03)
```json
{
  "customerId": 9999,
  "alias": "Test",
  "accountNumber": "ACC-999",
  "ownerName": "Nadie"
}
```
Esperado: `404 Not Found` con `message: "cliente con id 9999 no encontrado"`.

#### GET — listar paginado los contactos del cliente
```
GET {{base_url}}/api/beneficiaries/customer/{{customer_id}}?page=0&size=10
```
Esperado: `200 OK` con `PageResponse<BeneficiaryResponse>` (`content`, `page`, `size`, `totalElements`, `totalPages`, `first`, `last`).

#### DELETE — eliminar un beneficiario
```
DELETE {{base_url}}/api/beneficiaries/{{beneficiary_id}}
```
Esperado: `204 No Content` (sin body).

#### DELETE — beneficiario inexistente
```
DELETE {{base_url}}/api/beneficiaries/9999
```
Esperado: `404 Not Found` con `message: "beneficiario 9999 no encontrado"`.

---

## 5. Compilar y arrancar

```bash
# 1) compilar
mvn clean compile

# 2) arrancar la app cargando las variables del .env
export $(grep -v '^#' .env | xargs) && mvn spring-boot:run
```

Hibernate creara la tabla `beneficiaries` automaticamente al arrancar (`ddl-auto: update`).

---

## 6. Commit y Pull Request

```bash
git add src/main/java/com/hampcode/pagoya/beneficiary
git commit -m "feat(beneficiary): contactos favoritos con DTO, mapper y paginacion"
git push origin feature/beneficiary-context
```

Abrir el PR en GitHub: **`feature/beneficiary-context` → `develop`**.

Titulo sugerido:
> `feat(beneficiary): contactos favoritos del cliente`

Descripcion sugerida (breve):
- Nuevo bounded context `beneficiary`.
- Endpoints: `POST /api/beneficiaries`, `GET /api/beneficiaries/customer/{id}`, `DELETE /api/beneficiaries/{id}`.
- Aplica DTO + MapStruct + Bean Validation + paginacion.
- Reglas: cuenta unica por cliente (RN-B01), no auto-beneficiario (RN-B02).
