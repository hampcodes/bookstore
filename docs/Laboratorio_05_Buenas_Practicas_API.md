# LABORATORIO 05

# PagoYa

**Buenas Practicas para el Diseno de API REST**

Curso: Ingenieria de Software
Docente: Henry Antonio Mendoza Puerta
Escuela de Ciencias de la Computacion - UPC

---

## 1. Enunciado

Este laboratorio es la continuacion directa del Laboratorio 04 (Diseno de Software). Se parte del proyecto PagoYa ya implementado por Bounded Contexts (auth, customer, account, transfer) y se aplican las buenas practicas que toda API REST profesional debe ofrecer: separacion clara entre el modelo de dominio y los contratos expuestos al cliente, validacion automatica de datos de entrada, manejo centralizado de errores, paginacion en consultas que devuelven colecciones y emision de reportes para el negocio.

Al finalizar, el API de PagoYa expondra un contrato uniforme: las peticiones llegan como Request DTO validados, las respuestas viajan como Response DTO sin filtrar entidades JPA, los errores devuelven una estructura comun con codigo y mensaje, las consultas masivas soportan paginacion con metadatos y los endpoints de reportes devuelven datos agregados listos para alimentar graficos y dashboards en el frontend.

### Buenas practicas que se incorporan

- Uso de DTO (Data Transfer Objects) implementados como Java record
- Validacion declarativa con Jakarta Bean Validation
- Manejo centralizado de excepciones con `@RestControllerAdvice`
- Paginacion estandar con su propio DTO de respuesta
- Mapeo automatico Entity <-> DTO con MapStruct
- Reportes analiticos: queries con proyeccion directa a DTO para alimentar graficos

---

## 2. Objetivos del laboratorio

- Desacoplar las entidades de persistencia de las clases que viajan por HTTP.
- Implementar Request DTO validados con Bean Validation y Response DTO especificos.
- Centralizar el manejo de excepciones y devolver un ErrorResponse uniforme.
- Implementar paginacion en endpoints de listado con un PageResponse generico.
- Generar reportes analiticos JSON (transferencias por moneda, por dia y por estado; cuentas agrupadas por tipo y estado) listos para alimentar graficos y dashboards del frontend.

---

## 3. Dependencias adicionales en pom.xml

El proyecto del Laboratorio 04 ya incluye Spring Web, Spring Data JPA, PostgreSQL, Lombok y DevTools. Para los reportes analiticos no se necesita ninguna libreria nueva: se resuelven con funciones plpgsql en PostgreSQL invocadas via `@Query nativeQuery`. Las unicas dependencias adicionales son para validacion y mapeo automatico:

```xml
<!-- Validacion de DTOs con Jakarta Bean Validation -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- MapStruct: mapeo Entity <-> DTO -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.6.3</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.6.3</version>
    <scope>provided</scope>
</dependency>
```

En el bloque `maven-compiler-plugin` se debe agregar el annotation processor de MapStruct junto al de Lombok:

```xml
<annotationProcessorPaths>
    <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </path>
    <path>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>1.6.3</version>
    </path>
</annotationProcessorPaths>
```

---

## 4. Estructura del proyecto

La estructura modular del Laboratorio 04 se mantiene. En este laboratorio se completan las carpetas `dto/` y `mapper/` en cada bounded context, y se centraliza el manejo de errores en `shared/exception/`. Los DTOs de reporte viven en la carpeta `dto/` del mismo bounded context al que pertenecen:

```
pagoya/
|-- auth/
|   |-- controller/
|   |-- service/
|   |-- repository/
|   |-- model/
|   |-- dto/              <- RegisterUserRequest, UserResponse
|   |-- mapper/           <- MapStruct
|   `-- exception/
|-- customer/
|   |-- controller/
|   |-- service/
|   |-- repository/
|   |-- model/
|   |-- dto/
|   |-- mapper/
|   `-- exception/
|-- account/
|   |-- controller/       <- AccountController + AccountReportController
|   |-- service/
|   |-- repository/
|   |-- model/
|   |-- dto/              <- DTOs + AccountSummaryReport
|   |-- mapper/
|   `-- exception/
|-- transfer/
|   |-- controller/       <- TransferController + TransferReportController
|   |-- service/
|   |-- repository/
|   |-- model/
|   |-- dto/              <- DTOs + reportes (TransferByCurrencyReport, etc)
|   |-- mapper/
|   `-- exception/
`-- shared/
    |-- config/
    |-- exception/        <- ErrorResponse + GlobalExceptionHandler
    |-- pagination/       <- PageResponse generico
    `-- response/
```

---

## 5. Capa DTO (Data Transfer Objects)

Un DTO es un objeto plano cuyo unico proposito es transportar datos entre la capa de presentacion (controlador) y el cliente del API. Devolver entidades JPA directamente expone columnas internas, ataja sobre lazy-loading y crea acoplamiento entre la base de datos y el contrato HTTP. Para evitarlo se separan dos tipos de DTO:

| DTO | Proposito | Ejemplo |
|-----|-----------|---------|
| Request DTO | Recibe los datos enviados por el cliente. Lleva las anotaciones de validacion (`@NotBlank`, `@Email`, `@Size`, `@DecimalMin`, etc). | `RegisterUserRequest`, `CreateAccountRequest` |
| Response DTO | Devuelve la informacion al cliente. Solo expone los campos relevantes para el caso de uso, nunca contrasenas ni IDs internos sensibles. | `UserResponse`, `AccountResponse` |
| Pagination DTO | Envuelve listados paginados con metadatos (page, size, total). | `PageResponse<T>` |

### 5.1 Por que se usa Java record para los DTOs

En este laboratorio todos los DTOs se declaran como `record`, una caracteristica del lenguaje introducida en Java 16 y disponible de forma estable desde Java 17. Un record es una clase inmutable y final que el compilador construye automaticamente a partir de una unica linea de declaracion: el desarrollador solo lista los campos que componen el objeto y Java se encarga del resto.

Cuando se escribe:

```java
public record UserResponse(Long id, String email, String role) {}
```

el compilador genera de forma automatica:

- Un constructor publico con todos los campos en el mismo orden.
- Un metodo accesor por cada campo: `id()`, `email()`, `role()` (sin el prefijo `get`).
- Las implementaciones correctas de `equals()`, `hashCode()` y `toString()` basadas en los campos.
- Los campos como `private final`: el objeto es inmutable luego de su construccion.

El equivalente con una clase tradicional ocuparia entre 30 y 40 lineas con Lombok o getters manuales. Con record cabe en una sola linea.

#### Por que record encaja perfecto en un DTO

| Caracteristica del DTO | Por que record la cumple |
|------------------------|--------------------------|
| Solo transporta datos, no tiene logica | Un record esta pensado exactamente para eso: ser un contenedor plano de datos. |
| Debe ser inmutable mientras viaja por la red | Los campos de un record son `final` y no tienen setters; no pueden modificarse despues de creados. |
| Debe ser facil de serializar y deserializar a JSON | Jackson (el serializador por defecto de Spring) soporta record nativamente desde 2.12. |
| Debe quedar claro al leerlo que es un objeto de transferencia | La palabra `record` en la firma indica de inmediato la intencion: estructura de datos sin comportamiento. |
| No deberia repetir codigo boilerplate (getters, equals, hashCode) | El compilador los genera. Cero codigo manual, cero anotaciones de Lombok. |

#### Cuando NO usar record

Las entidades JPA (las clases anotadas con `@Entity`) NO se declaran como record. JPA exige un constructor sin argumentos, setters y la posibilidad de mutar campos (por ejemplo `balance` en `Account`). Por eso las entidades siguen siendo clases tradicionales con Lombok, mientras que los DTOs son record.

#### Acceso a los campos

En un record el acceso a un campo se hace llamando al metodo con el mismo nombre, sin prefijo `get`. Esto sera comun en los servicios:

```java
// con clase tradicional
request.getEmail()

// con record
request.email()
```

### 5.2 Mapeo Entity <-> DTO con MapStruct

MapStruct genera en tiempo de compilacion el codigo de conversion entre entidades y DTOs. Esto elimina mappers manuales repetitivos y deja el servicio enfocado en la logica de negocio. Cada bounded context tiene su propio mapper anotado con `@Mapper(componentModel = "spring")` para que Spring lo inyecte como bean.

---

## 6. Manejo centralizado de excepciones

En el Laboratorio 04 los servicios lanzan `RuntimeException` con mensajes en espanol. Esa estrategia funciona pero produce una respuesta 500 generica y filtra el stacktrace al cliente. En este laboratorio se introduce una jerarquia de excepciones de dominio y un `GlobalExceptionHandler` que las traduce a respuestas HTTP coherentes.

### 6.1 ErrorResponse uniforme

Todas las respuestas de error del API tienen la misma forma. Se ubica en `shared/exception/ErrorResponse.java`:

```java
package com.hampcode.pagoya.shared.exception;

import lombok.Builder;
import java.time.LocalDateTime;
import java.util.List;

@Builder
public record ErrorResponse(
    LocalDateTime timestamp,
    int status,
    String error,
    String message,
    String path,
    List<String> details
) {}
```

### 6.2 Excepciones base de dominio

`shared/exception/ResourceNotFoundException.java`

```java
package com.hampcode.pagoya.shared.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

`shared/exception/BusinessRuleException.java`

```java
package com.hampcode.pagoya.shared.exception;

public class BusinessRuleException extends RuntimeException {
    public BusinessRuleException(String message) { super(message); }
}
```

### 6.3 GlobalExceptionHandler

Se ubica en `shared/exception/GlobalExceptionHandler.java`. Atrapa las excepciones de dominio y las de validacion de Spring (`@Valid`):

```java
package com.hampcode.pagoya.shared.exception;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.List;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest req) {
        return build(HttpStatus.NOT_FOUND, ex.getMessage(), req, null);
    }

    @ExceptionHandler(BusinessRuleException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(
            BusinessRuleException ex, HttpServletRequest req) {
        return build(HttpStatus.BAD_REQUEST, ex.getMessage(), req, null);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest req) {
        List<String> details = ex.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .toList();
        return build(HttpStatus.BAD_REQUEST, "Datos invalidos", req, details);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(
            Exception ex, HttpServletRequest req) {
        return build(HttpStatus.INTERNAL_SERVER_ERROR,
                "Error interno del servidor", req, null);
    }

    private ResponseEntity<ErrorResponse> build(HttpStatus status, String message,
                                                HttpServletRequest req,
                                                List<String> details) {
        ErrorResponse body = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(status.value())
                .error(status.getReasonPhrase())
                .message(message)
                .path(req.getRequestURI())
                .details(details)
                .build();
        return ResponseEntity.status(status).body(body);
    }
}
```

Con esta clase configurada, los servicios dejan de lanzar `RuntimeException` y lanzan `ResourceNotFoundException` o `BusinessRuleException` segun corresponda.

---

## 7. Paginacion

Los endpoints que devuelven colecciones (transferencias por cuenta, cuentas por cliente, etc.) deben aceptar parametros `page` y `size` y devolver no solo el contenido sino tambien los metadatos del paginado. Se reutiliza `Pageable` y `Page` de Spring Data y se envuelve la respuesta en un DTO generico.

### 7.1 PageResponse generico

`shared/pagination/PageResponse.java`

```java
package com.hampcode.pagoya.shared.pagination;

import org.springframework.data.domain.Page;
import java.util.List;

public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {
    public static <T> PageResponse<T> from(Page<T> p) {
        return new PageResponse<>(
            p.getContent(),
            p.getNumber(),
            p.getSize(),
            p.getTotalElements(),
            p.getTotalPages(),
            p.isFirst(),
            p.isLast()
        );
    }
}
```

### 7.2 Uso en repositorio, servicio y controlador

Repositorio: el metodo de listado retorna `Page<T>` y recibe un `Pageable`.

```java
Page<Transfer> findBySourceAccountNumber(String accountNumber, Pageable pageable);
```

Servicio: se firma el metodo con `Pageable` y se devuelve `Page<DTO>`.

```java
public Page<TransferResponse> findByAccountNumber(String accountNumber, Pageable pageable) {
    return transferRepository.findBySourceAccountNumber(accountNumber, pageable)
            .map(transferMapper::toResponse);
}
```

Controlador: usa `@PageableDefault` para fijar valores por defecto y envuelve la respuesta en `PageResponse.from()`:

```java
@GetMapping("/account/{accountNumber}")
public ResponseEntity<PageResponse<TransferResponse>> findByAccountNumber(
        @PathVariable String accountNumber,
        @PageableDefault(size = 10, sort = "createdAt") Pageable pageable) {
    return ResponseEntity.ok(PageResponse.from(
            transferService.findByAccountNumber(accountNumber, pageable)));
}
```

---

## 8. Refactorizacion del Bounded Context: auth

Crear la rama `feature/auth-dto` desde `develop`:

```bash
git checkout develop
git checkout -b feature/auth-dto
```

### 8.1 Request DTO

`auth/dto/RegisterUserRequest.java`

```java
package com.hampcode.pagoya.auth.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record RegisterUserRequest(
    @NotBlank(message = "el email es obligatorio")
    @Email(message = "el formato del email no es valido")
    String email,

    @NotBlank(message = "la contrasena es obligatoria")
    @Size(min = 8, message = "la contrasena debe tener al menos 8 caracteres")
    String password
) {}
```

### 8.2 Response DTO

`auth/dto/UserResponse.java`

```java
package com.hampcode.pagoya.auth.dto;

public record UserResponse(
    Long id,
    String email,
    Boolean verified,
    String role
) {}
```

### 8.3 Mapper

`auth/mapper/UserMapper.java`

```java
package com.hampcode.pagoya.auth.mapper;

import com.hampcode.pagoya.auth.dto.RegisterUserRequest;
import com.hampcode.pagoya.auth.dto.UserResponse;
import com.hampcode.pagoya.auth.model.User;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface UserMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "verified", constant = "false")
    @Mapping(target = "role", ignore = true)
    User toEntity(RegisterUserRequest request);

    @Mapping(target = "role", source = "role.name")
    UserResponse toResponse(User user);
}
```

### 8.4 Excepciones especificas

`auth/exception/EmailAlreadyExistsException.java`

```java
package com.hampcode.pagoya.auth.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class EmailAlreadyExistsException extends BusinessRuleException {
    public EmailAlreadyExistsException(String email) {
        super("el email " + email + " ya esta registrado");
    }
}
```

### 8.5 Servicio refactorizado

`auth/service/IUserService.java`

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.dto.RegisterUserRequest;
import com.hampcode.pagoya.auth.dto.UserResponse;

public interface IUserService {
    UserResponse register(RegisterUserRequest request);
}
```

`auth/service/UserService.java`

```java
package com.hampcode.pagoya.auth.service;

import com.hampcode.pagoya.auth.dto.RegisterUserRequest;
import com.hampcode.pagoya.auth.dto.UserResponse;
import com.hampcode.pagoya.auth.exception.EmailAlreadyExistsException;
import com.hampcode.pagoya.auth.mapper.UserMapper;
import com.hampcode.pagoya.auth.model.Role;
import com.hampcode.pagoya.auth.model.User;
import com.hampcode.pagoya.auth.repository.RoleRepository;
import com.hampcode.pagoya.auth.repository.UserRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class UserService implements IUserService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final UserMapper userMapper;

    @Override
    @Transactional
    public UserResponse register(RegisterUserRequest request) {
        // RN-01: email unico
        if (userRepository.existsByEmail(request.email())) {
            throw new EmailAlreadyExistsException(request.email());
        }
        // RN-02: rol CUSTOMER obligatorio
        Role role = roleRepository.findByName("CUSTOMER")
                .orElseThrow(() -> new ResourceNotFoundException(
                        "rol CUSTOMER no esta configurado"));

        User user = userMapper.toEntity(request);
        user.setRole(role);
        return userMapper.toResponse(userRepository.save(user));
    }
}
```

### 8.6 Controlador refactorizado

`auth/controller/AuthController.java`

```java
package com.hampcode.pagoya.auth.controller;

import com.hampcode.pagoya.auth.dto.RegisterUserRequest;
import com.hampcode.pagoya.auth.dto.UserResponse;
import com.hampcode.pagoya.auth.service.IUserService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final IUserService userService;

    @PostMapping("/register")
    public ResponseEntity<UserResponse> register(
            @Valid @RequestBody RegisterUserRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(userService.register(request));
    }
}
```

Pull Request — `feature/auth-dto` -> `develop`

```bash
git add .
git commit -m "refactor(auth): apply DTO, validation and exception handling"
git push origin feature/auth-dto
```

---

## 9. Refactorizacion del Bounded Context: customer

```bash
git checkout develop && git checkout -b feature/customer-dto
```

### 9.1 Request y Response DTO

`customer/dto/CreateCustomerRequest.java`

```java
package com.hampcode.pagoya.customer.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;

public record CreateCustomerRequest(
    @NotBlank(message = "el nombre completo es obligatorio")
    @Size(max = 100) String fullName,

    @NotBlank(message = "el DNI es obligatorio")
    @Pattern(regexp = "\\d{8}", message = "el DNI debe tener 8 digitos")
    String dni,

    @Pattern(regexp = "\\d{9}", message = "el telefono debe tener 9 digitos")
    String phone,

    @NotNull(message = "el userId es obligatorio")
    Long userId
) {}
```

`customer/dto/CustomerResponse.java`

```java
package com.hampcode.pagoya.customer.dto;

public record CustomerResponse(
    Long id,
    String fullName,
    String dni,
    String phone,
    Long userId
) {}
```

### 9.2 Mapper y excepciones

`customer/mapper/CustomerMapper.java`

```java
package com.hampcode.pagoya.customer.mapper;

import com.hampcode.pagoya.customer.dto.CreateCustomerRequest;
import com.hampcode.pagoya.customer.dto.CustomerResponse;
import com.hampcode.pagoya.customer.model.Customer;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface CustomerMapper {
    @Mapping(target = "id", ignore = true)
    Customer toEntity(CreateCustomerRequest request);
    CustomerResponse toResponse(Customer customer);
}
```

`customer/exception/DniAlreadyExistsException.java`

```java
package com.hampcode.pagoya.customer.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class DniAlreadyExistsException extends BusinessRuleException {
    public DniAlreadyExistsException(String dni) {
        super("el DNI " + dni + " ya esta registrado");
    }
}
```

`customer/exception/CustomerProfileAlreadyExistsException.java`

```java
package com.hampcode.pagoya.customer.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class CustomerProfileAlreadyExistsException extends BusinessRuleException {
    public CustomerProfileAlreadyExistsException() {
        super("el usuario ya tiene un perfil de cliente");
    }
}
```

### 9.3 Servicio y controlador

`customer/service/CustomerService.java`

```java
package com.hampcode.pagoya.customer.service;

import com.hampcode.pagoya.customer.dto.CreateCustomerRequest;
import com.hampcode.pagoya.customer.dto.CustomerResponse;
import com.hampcode.pagoya.customer.exception.CustomerProfileAlreadyExistsException;
import com.hampcode.pagoya.customer.exception.DniAlreadyExistsException;
import com.hampcode.pagoya.customer.mapper.CustomerMapper;
import com.hampcode.pagoya.customer.model.Customer;
import com.hampcode.pagoya.customer.repository.CustomerRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import com.hampcode.pagoya.shared.pagination.PageResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class CustomerService implements ICustomerService {

    private final CustomerRepository customerRepository;
    private final CustomerMapper customerMapper;

    @Override
    @Transactional
    public CustomerResponse create(CreateCustomerRequest request) {
        if (customerRepository.existsByDni(request.dni()))
            throw new DniAlreadyExistsException(request.dni());
        if (customerRepository.existsByUserId(request.userId()))
            throw new CustomerProfileAlreadyExistsException();
        Customer entity = customerMapper.toEntity(request);
        return customerMapper.toResponse(customerRepository.save(entity));
    }

    @Override
    public CustomerResponse findById(Long id) {
        return customerRepository.findById(id)
                .map(customerMapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException(
                        "cliente con id " + id + " no encontrado"));
    }

    @Override
    public Page<CustomerResponse> findAll(Pageable pageable) {
        return customerRepository.findAll(pageable)
                .map(customerMapper::toResponse);
    }
}
```

`customer/controller/CustomerController.java`

```java
package com.hampcode.pagoya.customer.controller;

import com.hampcode.pagoya.customer.dto.CreateCustomerRequest;
import com.hampcode.pagoya.customer.dto.CustomerResponse;
import com.hampcode.pagoya.customer.service.ICustomerService;
import com.hampcode.pagoya.shared.pagination.PageResponse;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/customers")
@RequiredArgsConstructor
public class CustomerController {

    private final ICustomerService customerService;

    @PostMapping
    public ResponseEntity<CustomerResponse> create(
            @Valid @RequestBody CreateCustomerRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(customerService.create(request));
    }

    @GetMapping("/{id}")
    public ResponseEntity<CustomerResponse> findById(@PathVariable Long id) {
        return ResponseEntity.ok(customerService.findById(id));
    }

    @GetMapping
    public ResponseEntity<PageResponse<CustomerResponse>> findAll(
            @PageableDefault(size = 10, sort = "id") Pageable pageable) {
        return ResponseEntity.ok(
                PageResponse.from(customerService.findAll(pageable)));
    }
}
```

```bash
git add .
git commit -m "refactor(customer): apply DTO, pagination and exception handling"
git push origin feature/customer-dto
```

---

## 10. Refactorizacion del Bounded Context: account

```bash
git checkout develop && git checkout -b feature/account-dto
```

### 10.1 DTOs

`account/dto/CreateAccountRequest.java`

```java
package com.hampcode.pagoya.account.dto;

import com.hampcode.pagoya.account.model.AccountType;
import jakarta.validation.constraints.NotNull;

public record CreateAccountRequest(
    @NotNull(message = "el tipo de cuenta es obligatorio")
    AccountType type,
    @NotNull(message = "el customerId es obligatorio")
    Long customerId
) {}
```

`account/dto/AccountResponse.java`

```java
package com.hampcode.pagoya.account.dto;

import com.hampcode.pagoya.account.model.AccountStatus;
import com.hampcode.pagoya.account.model.AccountType;
import java.math.BigDecimal;

public record AccountResponse(
    Long id,
    String accountNumber,
    BigDecimal balance,
    AccountStatus status,
    AccountType type,
    Long customerId
) {}
```

`account/dto/AccountBalanceResponse.java` — vista reducida solo con saldo:

```java
package com.hampcode.pagoya.account.dto;

import java.math.BigDecimal;

public record AccountBalanceResponse(
    String accountNumber,
    BigDecimal balance
) {}
```

### 10.2 Mapper y excepciones

`account/mapper/AccountMapper.java`

```java
package com.hampcode.pagoya.account.mapper;

import com.hampcode.pagoya.account.dto.AccountBalanceResponse;
import com.hampcode.pagoya.account.dto.AccountResponse;
import com.hampcode.pagoya.account.model.Account;
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface AccountMapper {
    AccountResponse toResponse(Account account);
    AccountBalanceResponse toBalance(Account account);
}
```

`account/exception/DuplicateAccountTypeException.java`

```java
package com.hampcode.pagoya.account.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class DuplicateAccountTypeException extends BusinessRuleException {
    public DuplicateAccountTypeException() {
        super("ya tiene una cuenta de este tipo");
    }
}
```

`account/exception/AccountNotOperativeException.java`

```java
package com.hampcode.pagoya.account.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class AccountNotOperativeException extends BusinessRuleException {
    public AccountNotOperativeException() {
        super("la cuenta origen no esta operativa");
    }
}
```

### 10.3 Servicio y controlador con paginacion

`account/service/AccountService.java`

```java
package com.hampcode.pagoya.account.service;

import com.hampcode.pagoya.account.dto.AccountBalanceResponse;
import com.hampcode.pagoya.account.dto.AccountResponse;
import com.hampcode.pagoya.account.dto.CreateAccountRequest;
import com.hampcode.pagoya.account.exception.DuplicateAccountTypeException;
import com.hampcode.pagoya.account.mapper.AccountMapper;
import com.hampcode.pagoya.account.model.*;
import com.hampcode.pagoya.account.repository.AccountRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class AccountService implements IAccountService {

    private final AccountRepository accountRepository;
    private final AccountMapper accountMapper;

    @Override
    @Transactional
    public AccountResponse create(CreateAccountRequest request) {
        if (accountRepository.existsByCustomerIdAndType(
                request.customerId(), request.type()))
            throw new DuplicateAccountTypeException();
        Account account = Account.builder()
                .accountNumber(UUID.randomUUID().toString().substring(0,12).toUpperCase())
                .balance(BigDecimal.ZERO)
                .status(AccountStatus.ACTIVE)
                .type(request.type())
                .customerId(request.customerId())
                .build();
        return accountMapper.toResponse(accountRepository.save(account));
    }

    @Override
    public AccountBalanceResponse getBalance(String accountNumber) {
        return accountRepository.findByAccountNumber(accountNumber)
                .map(accountMapper::toBalance)
                .orElseThrow(() -> new ResourceNotFoundException(
                        "cuenta " + accountNumber + " no encontrada"));
    }

    @Override
    public Page<AccountResponse> findByCustomer(Long customerId, Pageable pageable) {
        return accountRepository.findByCustomerId(customerId, pageable)
                .map(accountMapper::toResponse);
    }
}
```

`account/controller/AccountController.java`

```java
package com.hampcode.pagoya.account.controller;

import com.hampcode.pagoya.account.dto.*;
import com.hampcode.pagoya.account.service.IAccountService;
import com.hampcode.pagoya.shared.pagination.PageResponse;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/accounts")
@RequiredArgsConstructor
public class AccountController {

    private final IAccountService accountService;

    @PostMapping
    public ResponseEntity<AccountResponse> create(
            @Valid @RequestBody CreateAccountRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(accountService.create(request));
    }

    @GetMapping("/{accountNumber}/balance")
    public ResponseEntity<AccountBalanceResponse> getBalance(
            @PathVariable String accountNumber) {
        return ResponseEntity.ok(accountService.getBalance(accountNumber));
    }

    @GetMapping("/customer/{customerId}")
    public ResponseEntity<PageResponse<AccountResponse>> findByCustomer(
            @PathVariable Long customerId,
            @PageableDefault(size = 10, sort = "id") Pageable pageable) {
        return ResponseEntity.ok(
                PageResponse.from(accountService.findByCustomer(customerId, pageable)));
    }
}
```

```bash
git add .
git commit -m "refactor(account): apply DTO, pagination and exception handling"
git push origin feature/account-dto
```

---

## 11. Refactorizacion del Bounded Context: transfer

```bash
git checkout develop && git checkout -b feature/transfer-dto-report
```

### 11.1 DTOs

`transfer/dto/TransferRequest.java`

```java
package com.hampcode.pagoya.transfer.dto;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record TransferRequest(
    @NotBlank(message = "la cuenta origen es obligatoria")
    String sourceAccountNumber,

    @NotBlank(message = "la cuenta destino es obligatoria")
    String targetAccountNumber,

    @NotNull(message = "el monto es obligatorio")
    @DecimalMin(value = "1.00", message = "el monto minimo es S/. 1.00")
    BigDecimal amount,

    @NotBlank(message = "la moneda es obligatoria")
    @Pattern(regexp = "PEN|USD|EUR|GBP",
             message = "la moneda debe ser PEN, USD, EUR o GBP")
    String currency
) {}
```

`transfer/dto/TransferResponse.java`

```java
package com.hampcode.pagoya.transfer.dto;

import com.hampcode.pagoya.transfer.model.TransferStatus;
import java.math.BigDecimal;
import java.time.LocalDateTime;

public record TransferResponse(
    Long id,
    String sourceAccountNumber,
    String targetAccountNumber,
    BigDecimal amount,
    String currency,
    BigDecimal exchangeRate,
    TransferStatus status,
    LocalDateTime createdAt
) {}
```

### 11.2 Mapper

`transfer/mapper/TransferMapper.java`

```java
package com.hampcode.pagoya.transfer.mapper;

import com.hampcode.pagoya.transfer.dto.TransferResponse;
import com.hampcode.pagoya.transfer.model.Transfer;
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface TransferMapper {
    TransferResponse toResponse(Transfer transfer);
}
```

### 11.3 Excepciones especificas

```java
package com.hampcode.pagoya.transfer.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class InsufficientBalanceException extends BusinessRuleException {
    public InsufficientBalanceException() { super("saldo insuficiente"); }
}
```

```java
package com.hampcode.pagoya.transfer.exception;

import com.hampcode.pagoya.shared.exception.BusinessRuleException;

public class SameAccountTransferException extends BusinessRuleException {
    public SameAccountTransferException() {
        super("la cuenta origen y destino no pueden ser la misma");
    }
}
```

### 11.4 Servicio refactorizado con paginacion

`transfer/service/TransferService.java`

```java
package com.hampcode.pagoya.transfer.service;

import com.hampcode.pagoya.account.exception.AccountNotOperativeException;
import com.hampcode.pagoya.account.model.Account;
import com.hampcode.pagoya.account.model.AccountStatus;
import com.hampcode.pagoya.account.repository.AccountRepository;
import com.hampcode.pagoya.shared.exception.ResourceNotFoundException;
import com.hampcode.pagoya.transfer.dto.TransferRequest;
import com.hampcode.pagoya.transfer.dto.TransferResponse;
import com.hampcode.pagoya.transfer.exception.InsufficientBalanceException;
import com.hampcode.pagoya.transfer.exception.SameAccountTransferException;
import com.hampcode.pagoya.transfer.mapper.TransferMapper;
import com.hampcode.pagoya.transfer.model.Transfer;
import com.hampcode.pagoya.transfer.model.TransferStatus;
import com.hampcode.pagoya.transfer.repository.TransferRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.client.RestTemplate;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class TransferService implements ITransferService {

    private final TransferRepository transferRepository;
    private final AccountRepository accountRepository;
    private final TransferMapper transferMapper;
    private final RestTemplate restTemplate;

    @Override
    @Transactional
    public TransferResponse transfer(TransferRequest request) {
        if (request.sourceAccountNumber().equals(request.targetAccountNumber()))
            throw new SameAccountTransferException();

        Account source = accountRepository
                .findByAccountNumber(request.sourceAccountNumber())
                .orElseThrow(() -> new ResourceNotFoundException(
                        "cuenta origen no encontrada"));
        Account target = accountRepository
                .findByAccountNumber(request.targetAccountNumber())
                .orElseThrow(() -> new ResourceNotFoundException(
                        "cuenta destino no encontrada"));

        if (source.getStatus() != AccountStatus.ACTIVE)
            throw new AccountNotOperativeException();
        if (source.getBalance().compareTo(request.amount()) < 0)
            throw new InsufficientBalanceException();

        BigDecimal finalAmount = request.amount();
        BigDecimal rate = null;
        if (!"PEN".equals(request.currency())) {
            rate = getExchangeRate(request.currency());
            finalAmount = request.amount().multiply(rate);
        }
        source.setBalance(source.getBalance().subtract(request.amount()));
        target.setBalance(target.getBalance().add(finalAmount));
        accountRepository.save(source);
        accountRepository.save(target);

        Transfer transfer = Transfer.builder()
                .sourceAccountNumber(request.sourceAccountNumber())
                .targetAccountNumber(request.targetAccountNumber())
                .amount(request.amount())
                .currency(request.currency())
                .exchangeRate(rate)
                .status(TransferStatus.COMPLETED)
                .createdAt(LocalDateTime.now())
                .build();
        return transferMapper.toResponse(transferRepository.save(transfer));
    }

    @Override
    public Page<TransferResponse> findByAccountNumber(
            String accountNumber, Pageable pageable) {
        return transferRepository
                .findBySourceAccountNumber(accountNumber, pageable)
                .map(transferMapper::toResponse);
    }

    private BigDecimal getExchangeRate(String currency) {
        String url = "https://api.frankfurter.dev/v2/rates?base=PEN&quotes=" + currency;
        Map response = restTemplate.getForObject(url, Map.class);
        Map rates = (Map) response.get("rates");
        return new BigDecimal(rates.get(currency).toString());
    }
}
```

`transfer/controller/TransferController.java`

```java
@RestController
@RequestMapping("/api/transfers")
@RequiredArgsConstructor
public class TransferController {

    private final ITransferService transferService;

    @PostMapping
    public ResponseEntity<TransferResponse> transfer(
            @Valid @RequestBody TransferRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(transferService.transfer(request));
    }

    @GetMapping("/account/{accountNumber}")
    public ResponseEntity<PageResponse<TransferResponse>> findByAccountNumber(
            @PathVariable String accountNumber,
            @PageableDefault(size = 10, sort = "createdAt") Pageable pageable) {
        return ResponseEntity.ok(PageResponse.from(
                transferService.findByAccountNumber(accountNumber, pageable)));
    }
}
```

---

## 12. Reportes analiticos con funciones PostgreSQL

Los reportes no se escriben en JPQL ni dentro de `@Query` del repositorio. Se escriben como funciones SQL en PostgreSQL y el repositorio solo las invoca. El resultado es una lista de `Object[]` que el servicio mapea al record DTO correspondiente.

### 12.1 Por que funciones en la base de datos

- **Separacion**: la logica analitica vive donde viven los datos.
- **Rendimiento**: el plan de ejecucion queda cacheado por PostgreSQL.
- **Reutilizable**: la misma funcion la consume el API, BI, scripts o psql.
- **Mantenible**: cambiar un reporte es un `CREATE OR REPLACE FUNCTION`; sin redeploy.
- **Versionable**: el `.sql` vive en el repo y se revisa en Pull Request.
- **Seguridad**: `GRANT EXECUTE` en la funcion y `REVOKE SELECT` en las tablas.

### 12.2 Archivo SQL con las funciones

`src/main/resources/db/reports.sql`

```sql
-- Ejecutar manualmente despues del primer arranque (cuando Hibernate ya
-- creo las tablas). Ambos archivos se cargan con psql:
--   psql -h localhost -p 55432 -U postgres -d pagoya_db -f src/main/resources/data.sql
--   psql -h localhost -p 55432 -U postgres -d pagoya_db -f src/main/resources/db/reports.sql

-- 1. Total transferido por moneda (solo COMPLETED)
CREATE OR REPLACE FUNCTION fn_transfer_report_by_currency()
RETURNS TABLE(currency VARCHAR, total_transfers BIGINT, total_amount NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT t.currency::VARCHAR, COUNT(*)::BIGINT, SUM(t.amount)::NUMERIC
    FROM transfers t
    WHERE t.status = 'COMPLETED'
    GROUP BY t.currency
    ORDER BY SUM(t.amount) DESC;
END;
$$ LANGUAGE plpgsql;

-- 2. Transferencias por dia en rango
CREATE OR REPLACE FUNCTION fn_transfer_report_by_day(p_from DATE, p_to DATE)
RETURNS TABLE(day DATE, total_transfers BIGINT, total_amount NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT DATE(t.created_at), COUNT(*)::BIGINT, SUM(t.amount)::NUMERIC
    FROM transfers t
    WHERE DATE(t.created_at) BETWEEN p_from AND p_to
    GROUP BY DATE(t.created_at)
    ORDER BY DATE(t.created_at);
END;
$$ LANGUAGE plpgsql;

-- 3. Distribucion por estado
CREATE OR REPLACE FUNCTION fn_transfer_report_by_status()
RETURNS TABLE(status VARCHAR, total BIGINT) AS $$
BEGIN
    RETURN QUERY
    SELECT t.status::VARCHAR, COUNT(*)::BIGINT
    FROM transfers t
    GROUP BY t.status;
END;
$$ LANGUAGE plpgsql;

-- 4. Cuentas por tipo y estado
CREATE OR REPLACE FUNCTION fn_account_report_summary()
RETURNS TABLE(type VARCHAR, status VARCHAR, total BIGINT, total_balance NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT a.type::VARCHAR, a.status::VARCHAR,
           COUNT(*)::BIGINT, COALESCE(SUM(a.balance), 0)::NUMERIC
    FROM accounts a
    GROUP BY a.type, a.status
    ORDER BY a.type, a.status;
END;
$$ LANGUAGE plpgsql;
```

El archivo `db/reports.sql` se ejecuta manualmente con `psql` despues de que Hibernate haya creado las tablas (primer arranque de la app). El parser de Spring no entiende los bloques `$$...$$` de plpgsql y los rompe en medio, por eso la inicializacion automatica de scripts se desactiva en `application-local.yml` con `mode: never`. Esto tambien implica que `data.sql` (semilla de roles) deja de cargarse solo, asi que ambos archivos se ejecutan manualmente desde psql:

```yaml
spring:
  sql:
    init:
      mode: never
```

### 12.3 DTOs de reporte (record)

```java
// transfer/dto/TransferByCurrencyReport.java
public record TransferByCurrencyReport(
    String currency, Long totalTransfers, BigDecimal totalAmount) {}

// transfer/dto/TransferByDayReport.java
public record TransferByDayReport(
    LocalDate day, Long totalTransfers, BigDecimal totalAmount) {}

// transfer/dto/TransferByStatusReport.java
public record TransferByStatusReport(
    String status, Long total) {}

// account/dto/AccountSummaryReport.java
public record AccountSummaryReport(
    String type, String status, Long total, BigDecimal totalBalance) {}
```

### 12.4 Repositorio: solo invoca la funcion

`transfer/repository/TransferRepository.java`

```java
@Query(value = "SELECT * FROM fn_transfer_report_by_currency()",
       nativeQuery = true)
List<Object[]> reportByCurrencyRaw();

@Query(value = "SELECT * FROM fn_transfer_report_by_day(:from, :to)",
       nativeQuery = true)
List<Object[]> reportByDayRaw(@Param("from") LocalDate from,
                              @Param("to") LocalDate to);

@Query(value = "SELECT * FROM fn_transfer_report_by_status()",
       nativeQuery = true)
List<Object[]> reportByStatusRaw();
```

`account/repository/AccountRepository.java`

```java
@Query(value = "SELECT * FROM fn_account_report_summary()",
       nativeQuery = true)
List<Object[]> reportSummaryRaw();
```

### 12.5 Servicio: mapea Object[] al record

```java
public List<TransferByCurrencyReport> reportByCurrency() {
    return transferRepository.reportByCurrencyRaw().stream()
            .map(r -> new TransferByCurrencyReport(
                    (String) r[0],
                    ((Number) r[1]).longValue(),
                    (BigDecimal) r[2]))
            .toList();
}

public List<TransferByDayReport> reportByDay(LocalDate from, LocalDate to) {
    return transferRepository.reportByDayRaw(from, to).stream()
            .map(r -> new TransferByDayReport(
                    ((java.sql.Date) r[0]).toLocalDate(),
                    ((Number) r[1]).longValue(),
                    (BigDecimal) r[2]))
            .toList();
}

public List<TransferByStatusReport> reportByStatus() {
    return transferRepository.reportByStatusRaw().stream()
            .map(r -> new TransferByStatusReport(
                    (String) r[0],
                    ((Number) r[1]).longValue()))
            .toList();
}

public List<AccountSummaryReport> reportSummary() {
    return accountRepository.reportSummaryRaw().stream()
            .map(r -> new AccountSummaryReport(
                    (String) r[0],
                    (String) r[1],
                    ((Number) r[2]).longValue(),
                    (BigDecimal) r[3]))
            .toList();
}
```

### 12.6 Controladores de reportes

```java
@RestController
@RequestMapping("/api/transfers/reports")
@RequiredArgsConstructor
public class TransferReportController {
    private final ITransferService transferService;

    @GetMapping("/by-currency")
    public List<TransferByCurrencyReport> byCurrency() {
        return transferService.reportByCurrency();
    }

    @GetMapping("/by-day")
    public List<TransferByDayReport> byDay(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        return transferService.reportByDay(from, to);
    }

    @GetMapping("/by-status")
    public List<TransferByStatusReport> byStatus() {
        return transferService.reportByStatus();
    }
}

@RestController
@RequestMapping("/api/accounts/reports")
@RequiredArgsConstructor
public class AccountReportController {
    private final IAccountService accountService;

    @GetMapping("/summary")
    public List<AccountSummaryReport> summary() {
        return accountService.reportSummary();
    }
}
```

### 12.7 Respuesta lista para graficos

`GET /api/transfers/reports/by-currency`

```json
[
  { "currency": "PEN", "totalTransfers": 248, "totalAmount": 152340.50 },
  { "currency": "USD", "totalTransfers":  61, "totalAmount":  18420.00 }
]
```

`GET /api/transfers/reports/by-day?from=2026-04-01&to=2026-04-25`

```json
[
  { "day": "2026-04-01", "totalTransfers": 12, "totalAmount": 3540.00 },
  { "day": "2026-04-02", "totalTransfers": 18, "totalAmount": 6720.50 }
]
```

Pull Request final del laboratorio:

```bash
git add .
git commit -m "feat(reports): analytical reports via PostgreSQL functions"
git push origin feature/transfer-dto-report
```

---

## 13. Endpoints actualizados para pruebas en Postman

### auth

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/api/auth/register` | Registrar nuevo usuario (DTO validado) |

Body — `RegisterUserRequest`

```json
{
  "email": "ana@pagoya.pe",
  "password": "12345678"
}
```

### customer

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/api/customers` | Registrar datos del cliente |
| GET | `/api/customers/{id}` | Consultar cliente por id |
| GET | `/api/customers?page=0&size=10` | Listado paginado de clientes |

```json
{
  "fullName": "Ana Torres",
  "dni": "12345678",
  "phone": "987654321",
  "userId": 1
}
```

### account

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/api/accounts` | Crear cuenta digital |
| GET | `/api/accounts/{accountNumber}/balance` | Consultar saldo |
| GET | `/api/accounts/customer/{customerId}?page=0&size=10` | Listar cuentas del cliente con paginacion |

```json
{
  "type": "SAVINGS",
  "customerId": 1
}
```

### transfer

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/api/transfers` | Ejecutar transferencia |
| GET | `/api/transfers/account/{accountNumber}?page=0&size=10` | Historial paginado de transferencias |

### Reportes analiticos

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| GET | `/api/transfers/reports/by-currency` | Total y conteo por moneda (grafico de torta) |
| GET | `/api/transfers/reports/by-day?from=2026-04-01&to=2026-04-25` | Transferencias por dia en un rango (grafico de lineas) |
| GET | `/api/transfers/reports/by-status` | Distribucion por estado PENDING/COMPLETED/FAILED |
| GET | `/api/accounts/reports/summary` | Cuentas agrupadas por tipo y estado con saldo total |

Body — Transferencia en soles

```json
{
  "sourceAccountNumber": "ACC-001",
  "targetAccountNumber": "ACC-002",
  "amount": 200.00,
  "currency": "PEN"
}
```

Body — Transferencia internacional

```json
{
  "sourceAccountNumber": "ACC-001",
  "targetAccountNumber": "ACC-002",
  "amount": 100.00,
  "currency": "USD"
}
```

---

## 14. Casos de prueba

Validar que cada buena practica se aplica correctamente:

| Caso | Como probarlo | Resultado esperado |
|------|---------------|--------------------|
| Validacion de DTO | POST `/api/auth/register` con email invalido y password de 3 caracteres | 400 con `ErrorResponse` y `details: [email: ...; password: ...]` |
| Excepcion de negocio | POST `/api/customers` con un DNI ya registrado | 400 con `message: el DNI 12345678 ya esta registrado` |
| Excepcion de recurso | GET `/api/customers/9999` | 404 con `message: cliente con id 9999 no encontrado` |
| Paginacion | GET `/api/transfers/account/ACC-001?page=0&size=5` | 200 con `content` (max 5), `page`, `size`, `totalElements`, `totalPages`, `first`, `last` |
| Reporte por moneda | GET `/api/transfers/reports/by-currency` | 200 con array de `{ currency, totalTransfers, totalAmount }` listo para grafico |
| Reporte por dia | GET `/api/transfers/reports/by-day?from=2026-04-01&to=2026-04-25` | 200 con array de `{ day, totalTransfers, totalAmount }` ordenado por fecha |
| Reporte por estado | GET `/api/transfers/reports/by-status` | 200 con array de `{ status, total }` para grafico de torta |
| RN-09 - monto minimo | POST `/api/transfers` con `amount = 0.50` | 400 con `details: [amount: el monto minimo es S/. 1.00]` |
| RN-10 - misma cuenta | POST `/api/transfers` con `sourceAccountNumber == targetAccountNumber` | 400 con `message: la cuenta origen y destino no pueden ser la misma` |
