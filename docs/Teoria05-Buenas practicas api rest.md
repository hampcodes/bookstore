# Teoría 05: Buenas practicas en la implementacion de API REST
## API REST, Excepciones, Paginación, DTO, Swagger / OpenAPI

**Elaborado por:** Henry Antonio Mendoza Puerta

## Objetivo

Consolidar los conceptos teóricos de la semana 5 que sustentan el diseño de una API REST profesional con Spring Boot. Al terminar esta lectura tendrás claros los criterios para nombrar rutas, usar verbos y status codes correctamente, manejar excepciones de forma centralizada, paginar resultados, separar la entidad del DTO y documentar la API con Swagger / OpenAPI.

Lo que vas a estudiar:

- **API REST**: verbos HTTP, status codes y nombres de rutas.
- **Excepciones**: `@RestControllerAdvice`, `@ExceptionHandler` y `ErrorResponse`.
- **Paginación**: `LIMIT/OFFSET`, `Pageable`, `Page<T>` y `PageResponse<T>`.
- **DTO**: `Request`, `Response`, mapper con MapStruct y DTOs para reportes.
- **Documentación**: Swagger UI, OpenAPI y su relación con Postman.

## Indice

- [1. Diseño correcto de un API REST](#1-diseño-correcto-de-un-api-rest)
- [2. Manejo de Excepciones y Paginación](#2-manejo-de-excepciones-y-paginación)
- [3. DTO (Data Transfer Object)](#3-dto-data-transfer-object)
- [4. Documentación API: Swagger y OpenAPI](#4-documentación-api-swagger-y-openapi)

---

## 1. Diseño correcto de un API REST

Verbos HTTP, status codes y nombres de rutas: las tres decisiones que definen si una API REST queda bien diseñada.

### 1.1 Los 5 verbos HTTP que usarás

| Verbo | Acción | Descripción |
|---|---|---|
| `GET` | Leer | Obtener uno o varios recursos. NO modifica nada. Es idempotente. |
| `POST` | Crear | Crea un recurso nuevo. El servidor asigna el `id`. |
| `PUT` | Reemplazar completo | Reemplaza TODOS los campos del recurso. Mandas el objeto entero. |
| `PATCH` | Actualizar parcial | Actualiza solo algunos campos. Mandas solo lo que cambia. |
| `DELETE` | Borrar | Elimina el recurso. Suele devolver `204` (sin body). |

### 1.2 Tabla CRUD: verbo + ruta + status code

Para el recurso `/api/customers`:

| Verbo | Ruta | Status | Descripción |
|---|---|---|---|
| `GET` | `/api/customers` | `200 OK` | Lista todos (paginado). |
| `GET` | `/api/customers/{id}` | `200 OK` / `404` | Devuelve uno; `404` si no existe. |
| `POST` | `/api/customers` | `201 Created` | Crea uno nuevo. Devuelve el recurso creado. |
| `PUT` | `/api/customers/{id}` | `200 OK` / `404` | Reemplaza COMPLETO el recurso existente. |
| `PATCH` | `/api/customers/{id}` | `200 OK` / `404` | Actualiza SOLO los campos enviados. |
| `DELETE` | `/api/customers/{id}` | `204` / `404` | Elimina. `204` sin body si exito. |

### 1.3 Status codes de éxito (2xx)

| Código | Significado | Cuándo usarlo |
|---|---|---|
| `200 OK` | Operación exitosa con body de respuesta. | `GET`, `PUT`, `PATCH`. |
| `201 Created` | Recurso creado. | `POST`. Buena práctica: incluir header `Location`. |
| `204 No Content` | Éxito sin body. | `DELETE` típicamente. |

### 1.4 Status codes de error (4xx y 5xx)

| Código | Significado | Cuándo se devuelve |
|---|---|---|
| `400 Bad Request` | Validación falló. | Campos inválidos, JSON malformado. |
| `401 Unauthorized` | No estás autenticado. | No hay token o token inválido. |
| `403 Forbidden` | Estás autenticado pero NO tienes permiso. | — |
| `404 Not Found` | El recurso pedido no existe. | — |
| `409 Conflict` | Regla de negocio violada. | Duplicado, estado inválido. |
| `500 Server Error` | Bug del servidor. | NPE, BD caída. NO debe llegar al cliente con stacktrace. |

### 1.5 Reglas para nombrar rutas

Cuatro reglas simples que evitan la mayoría de errores en el diseño de URLs.

**1. Usa plurales** (la URL apunta a una colección).

| Bien | Mal |
|---|---|
| `/api/customers` | `/api/customer` |

**2. Une palabras con guion** (`-`, kebab-case).

| Bien | Mal |
|---|---|
| `/api/saving-goals` | `/api/savingGoals` |
| | `/api/saving_goals` |

**3. Sin verbos en la URL** (el verbo lo da HTTP).

| Bien | Mal |
|---|---|
| `GET /api/customers/5` | `GET /api/getCustomer/5` |

**4. El `id` va en la URL entre llaves `{id}`.**

| Bien | Mal |
|---|---|
| `/api/customers/{id}` | `/api/customers?id=5` |

### 1.6 Path params vs Query params

**Path param** `/{id}` identifica al recurso. Es PARTE de la URL — forma parte de la 'identidad' de lo que pides.

```java
// Spring lo lee con @PathVariable
@GetMapping("/customers/{id}")
public CustomerDTO get(@PathVariable Long id) { ... }

// URL:
GET /api/customers/42
```

**Query param** `?key=value` modifica CÓMO traer el recurso: filtros, paginación, ordenamiento. Opcionales casi siempre; tienen valor por defecto.

```java
// Spring lo lee con @RequestParam
@GetMapping("/customers")
public Page<...> list(
    @RequestParam(defaultValue="ACTIVE") String status,
    Pageable pageable) { ... }

// URL:
GET /api/customers?status=ACTIVE&page=0&size=10
```

Combinados: `GET /api/customers/{id}/saving-goals?page=0&size=20` — path identifica, query filtra.

### 1.7 Anti-patrones típicos

Errores comunes y su corrección.

| Mal | Bien | Por qué |
|---|---|---|
| `GET /getCustomer/5` | `GET /api/customers/5` | El verbo lo da HTTP. La URL solo identifica el recurso. |
| `POST /api/customers/create` | `POST /api/customers` | `POST` ya significa 'crear'. La acción en la URL sobra. |
| `POST /api/customers/5/delete` | `DELETE /api/customers/5` | Usar el verbo `DELETE`: la URL describe el recurso, no la acción. |
| `GET /api/customer/5` | `GET /api/customers/5` | Plurales: el endpoint expone una COLECCIÓN (aunque pidas uno). |
| `HTTP 200 + {"error": "..."}` | `HTTP 404 + ErrorResponse` | Usa el status REAL. El frontend chequea el status, no el body. |

### 1.8 Ejemplo aplicado: PagoYa API

Así se ven las rutas correctas para los recursos del proyecto.

```text
# Customers
GET    /api/customers                       → 200  (listar paginado)
GET    /api/customers/{id}                  → 200 / 404
POST   /api/customers                       → 201  (crear)
PUT    /api/customers/{id}                  → 200 / 404 (reemplazar)
DELETE /api/customers/{id}                  → 204 / 404

# Saving Goals (sub-recurso del customer)
GET    /api/customers/{id}/saving-goals     → 200  (listar metas del cliente)
POST   /api/saving-goals                    → 201
DELETE /api/saving-goals/{id}               → 204 / 404

# Transfers (con filtros)
GET    /api/transfers?page=0&size=10&sort=createdAt,desc   → 200
GET    /api/customers/{id}/transfers?status=COMPLETED      → 200
POST   /api/transfers                                       → 201 / 400 / 409
```

### 1.9 Para recordar

| Concepto | Resumen |
|---|---|
| Verbos | `GET` (leer), `POST` (crear), `PUT` (reemplazar), `PATCH` (parcial), `DELETE` (borrar). |
| Status 2xx | `200 OK` · `201 Created` · `204 No Content` (sin body). |
| Status 4xx | `400` (validación) · `401` (no auth) · `403` (no permiso) · `404` (no existe) · `409` (regla negocio). |
| Rutas | Sustantivos plurales · kebab-case · sin verbos · jerarquía para sub-recursos. |
| Path vs Query | Path = identidad del recurso (`/{id}`). Query = filtros/paginación (`?page=0&size=10`). |
| Coherencia | Si `POST /customers` crea, `POST /transfers` también crea. Igual para los demás verbos. |

[↑ Volver al indice](#indice)

---

## 2. Manejo de Excepciones y Paginación

Dos pilares que separan una API que se rompe en producción de una que aguanta tráfico real: manejar errores de forma centralizada y traer datos en lotes manejables.

### 2.1 ¿Qué es una excepción?

Una excepción es un EVENTO que rompe el flujo normal del programa porque algo salió mal: pedir un cliente que no existe, dividir entre cero, una BD caída, etc.

- En Java son OBJETOS que representan errores: `NullPointerException`, `IllegalArgumentException`, etc.
- Cuando algo sale mal alguien LANZA la excepción (`throw new ...`).
- Si nadie la CAPTURA (`catch` o handler), el programa se detiene y al cliente le llega un HTTP 500 feo.
- Pueden ser de Java o tuyas (custom): `ResourceNotFoundException`, `BusinessRuleException`, etc.

Ejemplo: lanzar una excepción cuando un cliente no existe.

```java
Customer c = customerRepo.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("cliente " + id + " no existe"));
// la excepcion 'sube' hasta @RestControllerAdvice → 404 limpio al cliente
```

### 2.2 ¿Por qué manejar excepciones?

**Sin manejo (mal):**

- El cliente recibe HTTP 500 con un stacktrace de Java.
- Se expone información interna (paquetes, versiones, queries SQL).
- Cada controller maneja errores a su manera (inconsistente).
- El cliente no sabe si fue su culpa o del servidor.

```http
HTTP/1.1 500 Internal Server Error
{
  "trace": "java.lang.NullPointer
    at com.hampcode.....",
  "timestamp": "..."
}
```

**Con manejo (bien):**

- Respuestas con el HTTP STATUS correcto (`404`, `400`, `409`, etc).
- Mensaje claro y seguro (sin filtrar internals).
- Formato UNIFORME (mismo `ErrorResponse` en todo el API).
- Manejo CENTRALIZADO en un solo lugar.

```http
HTTP/1.1 404 Not Found
{
  "status": 404,
  "error": "Not Found",
  "message": "cliente 99 no existe",
  "path": "/api/customers/99"
}
```

### 2.3 La pieza clave: `@RestControllerAdvice`

Es la anotación que CAPTURA excepciones lanzadas por TODOS los `@RestController`. Define métodos con `@ExceptionHandler(MiExcepcion.class)` para indicar cómo responder ante cada excepción. Cada handler devuelve un `ResponseEntity<ErrorResponse>` con el status correcto. Una sola clase concentra todo el manejo de errores del API.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest req) {
        ErrorResponse body = new ErrorResponse(
            LocalDateTime.now(), 404, "Not Found",
            ex.getMessage(), req.getRequestURI());
        return ResponseEntity.status(404).body(body);
    }
}
```

### 2.4 Excepciones personalizadas

Cada tipo de error → su propia excepción → su propio HTTP status.

| Excepción | Status | Cuándo se usa |
|---|---|---|
| `ResourceNotFoundException` | `404 Not Found` | El recurso pedido no existe (cliente, meta, transferencia). |
| `BusinessRuleException` | `400 / 409 Conflict` | Una regla de negocio se viola (cuenta duplicada, saldo insuficiente). |
| `MethodArgumentNotValidException` | `400 Bad Request` | La validación `@Valid` del RequestDTO falló (Spring la lanza sola). |
| `AccessDeniedException` | `403 Forbidden` | El usuario está autenticado pero no tiene permisos. |

Crear una excepción propia es trivial:

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

### 2.5 `ErrorResponse`: formato uniforme de error

Un solo DTO para TODOS los errores del API. El cliente sabe qué esperar.

El record:

```java
public record ErrorResponse(
    LocalDateTime timestamp,
    int status,
    String error,
    String message,
    String path,
    List<String> details   // opcional, para validaciones
) {}
```

Lo que llega al cliente (JSON):

```json
{
  "timestamp": "2026-04-30T10:30:00",
  "status": 404,
  "error": "Not Found",
  "message": "cliente 99 no existe",
  "path": "/api/customers/99",
  "details": null
}
```

Mismo formato para `400`, `404`, `409`, `500`. El frontend muestra siempre `message` al usuario.

### 2.6 Validación con `@Valid` → 400 Bad Request

- Pones `@Valid` en el `@RequestBody`: Spring valida el RequestDTO automáticamente.
- Si una regla falla (`@NotBlank`, `@Future`, `@Size`, etc) Spring lanza `MethodArgumentNotValidException`.
- Tu `@ExceptionHandler` la captura y devuelve `400` con el detalle de cada campo inválido.

```java
// 1) Controller usa @Valid
@PostMapping
public ResponseEntity<...> create(@Valid @RequestBody CreateSavingGoalRequest req) { ... }

// 2) Handler global captura el error y arma el ErrorResponse con detalles
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidation(
        MethodArgumentNotValidException ex, HttpServletRequest req) {
    List<String> details = ex.getBindingResult().getFieldErrors().stream()
        .map(f -> f.getField() + ": " + f.getDefaultMessage())
        .toList();
    return ResponseEntity.badRequest().body(new ErrorResponse(
        LocalDateTime.now(), 400, "Bad Request",
        "validacion fallida", req.getRequestURI(), details));
}
```

### 2.7 ¿Por qué paginar?

**Sin paginación:**

```text
GET /api/transfers
→ trae 1 000 000 de filas
```

- `OutOfMemoryError` en el servidor.
- Tarda 30+ segundos en responder.
- El cliente recibe un JSON gigante que no puede mostrar de golpe.
- Si la BD crece, todo se rompe.

**Con paginación:**

```text
GET /api/transfers?page=0&size=10
→ trae 10 filas (la primera pagina)
```

- Memoria controlada.
- Respuesta rápida (~50ms).
- El cliente carga lo que necesita y pide la página siguiente cuando hace falta.
- Escala aunque la BD tenga millones de filas.

### 2.8 LIMIT y OFFSET

| Cláusula | Significado |
|---|---|
| `LIMIT` | Cuántas devolver. `LIMIT 10` te entrega máximo 10 filas. |
| `OFFSET` | Cuántas saltar. `OFFSET 10` ignora las primeras 10 filas y empieza desde la 11. |

Imagina una tabla con 30 filas. Así le pides a la BD cada página:

| Página | SQL | Resultado | Cálculo `OFFSET` |
|---|---|---|---|
| `page 0` | `LIMIT 10 OFFSET 0` | `1, 2, 3, 4, 5, 6, 7, 8, 9, 10` | `0 × 10 = 0` |
| `page 1` | `LIMIT 10 OFFSET 10` | `11, 12, 13, 14, 15, 16, 17, 18, 19, 20` | `1 × 10 = 10` |
| `page 2` | `LIMIT 10 OFFSET 20` | `21, 22, 23, 24, 25, 26, 27, 28, 29, 30` | `2 × 10 = 20` |

Fórmula: `OFFSET = page × size`. Spring `Pageable` hace esta traducción por ti.

### 2.9 `Pageable` y `Page<T>` en el Repository

- Spring Data JPA reconoce `Pageable` como parámetro: lo traduce a `LIMIT/OFFSET` en SQL.
- El método del Repository devuelve `Page<T>` en lugar de `List<T>`.
- `Page<T>` incluye el contenido + metadata (`totalElements`, `totalPages`, `isFirst`, `isLast`).

```java
public interface TransferRepository extends JpaRepository<Transfer, Long> {
    Page<Transfer> findBySender_Id(Long senderId, Pageable pageable);
}
```

SQL que Spring genera (aproximado):

```sql
SELECT * FROM transfers WHERE sender_id = ?
ORDER BY id ASC LIMIT 10 OFFSET 0;     -- pagina 0, tamano 10
```

### 2.10 Controller con paginación: `@PageableDefault`

- Spring inyecta el `Pageable` leyendo los query params: `?page=0&size=10&sort=createdAt,desc`.
- `@PageableDefault` define valores por defecto si el cliente no los manda.
- El service llama al repository con el `Pageable` y lo convierte en `PageResponse<DTO>`.

```java
@RestController
@RequestMapping("/api/transfers")
public class TransferController {

    @GetMapping("/customer/{id}")
    public ResponseEntity<PageResponse<TransferDTO>> list(
            @PathVariable Long id,
            @PageableDefault(size = 10, sort = "createdAt") Pageable pageable) {
        return ResponseEntity.ok(service.list(id, pageable));
    }
}
```

URLs válidas:

```text
GET /api/transfers/customer/1?page=0&size=20&sort=amount,desc
```

### 2.11 `PageResponse`: el wrapper que SÍ quieres exponer

`Page<T>` de Spring tiene muchos campos verbosos. Un `PageResponse` propio es más limpio.

El record:

```java
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
            p.getContent(), p.getNumber(),
            p.getSize(), p.getTotalElements(),
            p.getTotalPages(),
            p.isFirst(), p.isLast());
    }
}
```

JSON que ve el cliente:

```json
{
  "content": [
    { "id": 1, "monto": 250.00 },
    { "id": 2, "monto": 100.00 }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 1543,
  "totalPages": 155,
  "first": true,
  "last": false
}
```

### 2.12 Para recordar

| Concepto | Resumen |
|---|---|
| `@RestControllerAdvice` | Captura excepciones de TODOS los controllers en un solo lugar. |
| `@ExceptionHandler` | Define qué hacer ante cada tipo de excepción (status + `ErrorResponse`). |
| `ErrorResponse` | Formato uniforme: `timestamp`, `status`, `error`, `message`, `path`, `details`. |
| `@Valid` | Activa validación del RequestDTO. Si falla → `MethodArgumentNotValidException` → `400`. |
| `LIMIT / OFFSET` | Cláusulas SQL: cuántas filas traer y cuántas saltarse. `OFFSET = page × size`. |
| `Pageable + Page<T>` | Spring lo lee de query params y lo traduce a `LIMIT/OFFSET`. Devuelve contenido + metadata. |
| `PageResponse<T>` | Wrapper limpio: `content + page + size + totalElements + totalPages + first/last`. |

[↑ Volver al indice](#indice)

---

## 3. DTO (Data Transfer Object)

Patrón que separa la representación interna (Entity) de la representación pública (lo que viaja por la red). Decide qué campos se exponen, oculta lo sensible y permite que la API y la BD evolucionen independientemente.

### 3.1 ¿Qué es un DTO?

Un DTO es un objeto plano que transporta datos entre capas. Decide qué campos se exponen.

- Solo datos, SIN lógica de negocio.
- NO es la Entity (la entidad representa la fila de la tabla; el DTO representa la vista).
- Típico en Spring: `RequestDTO` (entrada) y `ResponseDTO` (salida).

Ejemplo:

```java
// Entity (la tabla, todo lo interno)
class Customer { Long id; String name; String email; String password; String dni; ... }

// DTO (lo que se expone publicamente)
record CustomerDTO(Long id, String name) {}   // <-- ojo: oculta email, password, dni
```

### 3.2 ¿Entre qué capas vive?

```text
Cliente  ──JSON──▶  Controller  ──DTO──▶  Service  ──Entity──▶  Repository
```

- El DTO vive entre Cliente, Controller y Service.
- El Repository NO usa DTOs. Solo trabaja con Entity.

### 3.3 Request DTO vs Response DTO

**Request DTO (lo que ENTRA):** el cliente ENVÍA estos datos al servidor.

```java
public record CreateSavingGoalRequest(
    Long customerId,
    String name,
    BigDecimal targetAmount,
    LocalDate deadline
) {}
// validado con @Valid
// (@NotBlank, @Future, ...)
```

**Response DTO (lo que SALE):** el servidor DEVUELVE estos datos al cliente.

```java
public record SavingGoalResponse(
    Long id,
    String name,
    BigDecimal targetAmount,
    LocalDate deadline
) {}
// oculta customer_id y created_at
```

Ambos son distintos a la Entity (la tabla). Cada uno tiene su propósito.

### 3.4 Ejemplo: crear una meta de ahorro

**1) JSON del cliente:**

```http
POST /api/saving-goals
{
  "customerId": 1,
  "name": "Viaje a Cusco",
  "targetAmount": 2500,
  "deadline": "2026-12-31"
}
```

**2) Request DTO:**

```java
public record CreateSavingGoalRequest(
    Long customerId,
    String name,
    BigDecimal targetAmount,
    LocalDate deadline
) {}
```

**3) Entity → tabla:**

```java
@Entity
class SavingGoal {
    Long id;
    Customer customer;
    String name;
    BigDecimal targetAmount;
    LocalDate deadline;
    LocalDateTime createdAt;
}
```

Jackson convierte JSON → DTO. El Mapper convierte DTO → Entity. JPA guarda la fila.

### 3.5 Mapper: convierte DTO ↔ Entity

- El Mapper traduce entre los dos mundos: DTO (capa API) y Entity (capa BD).
- MapStruct es una librería que GENERA el código del mapper en compile-time. Tú solo declaras la interface.
- Beneficio concreto: si la entity renombra un campo, MapStruct te avisa al compilar. A mano: bug silencioso en runtime.

```java
@Mapper(componentModel = "spring")
public interface SavingGoalMapper {

    @Mapping(target = "id",        ignore = true)
    @Mapping(target = "customer",  ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    SavingGoal toEntity(CreateSavingGoalRequest req);

    SavingGoalResponse toResponse(SavingGoal g);
}
```

### 3.6 DTOs en reportes: una tabla, varias vistas

Cada reporte tiene su propio stored procedure y su propio DTO. Veamos 2 ejemplos.

Tabla `transfers` (en PostgreSQL):

```sql
CREATE TABLE transfers (
    id          BIGSERIAL PRIMARY KEY,
    sender_id   BIGINT REFERENCES customers(id),
    receiver_id BIGINT REFERENCES customers(id),
    amount      DECIMAL(12,2) NOT NULL,
    created_at  TIMESTAMP DEFAULT now(),
    reference   VARCHAR(200),
    ip_address  VARCHAR(40)        -- sensible, no se expone
);
```

Sobre esta tabla armamos 2 reportes distintos:

| Reporte | Stored procedure | DTO |
|---|---|---|
| Lista de transferencias enviadas | `sp_transfers_by_sender` | `TransferListItem` |
| Estadísticas del cliente | `sp_customer_stats` | `CustomerStatsDTO` |

### 3.7 Reporte 1: Lista de transferencias

**1) PostgreSQL: el stored procedure.**

```sql
CREATE OR REPLACE FUNCTION sp_transfers_by_sender(p_sender_id BIGINT)
RETURNS TABLE (transfer_id BIGINT, fecha TIMESTAMP, monto DECIMAL, destinatario VARCHAR)
AS $$
BEGIN
    RETURN QUERY
    SELECT t.id, t.created_at, t.amount, c.full_name
    FROM transfers t
    JOIN customers c ON c.id = t.receiver_id
    WHERE t.sender_id = p_sender_id
    ORDER BY t.created_at DESC;
END;
$$ LANGUAGE plpgsql;
```

**2) Repository: lo invocas con `@Query (nativeQuery = true)`.**

```java
public interface TransferRepository extends JpaRepository<Transfer, Long> {
    @Query(value = "SELECT * FROM sp_transfers_by_sender(:senderId)", nativeQuery = true)
    List<Object[]> listBySender(@Param("senderId") Long senderId);
}
```

**3) DTO (lo que devuelve el reporte):**

```java
public record TransferListItem(
    Long id,
    LocalDateTime fecha,
    BigDecimal monto,
    String destinatario
) {}
```

**4) Service (arma el DTO con cada fila):**

```java
public List<TransferListItem> list(Long id) {
    return repo.listBySender(id).stream()
        .map(r -> new TransferListItem(
            (Long) r[0], (LocalDateTime) r[1],
            (BigDecimal) r[2], (String) r[3]))
        .toList();
}
```

### 3.8 Reporte 2: Estadísticas del cliente

**1) PostgreSQL: stored procedure con agregaciones (`SUM`, `COUNT`).**

```sql
CREATE OR REPLACE FUNCTION sp_customer_stats(p_customer_id BIGINT)
RETURNS TABLE (total_enviado DECIMAL, total_recibido DECIMAL, cantidad BIGINT)
AS $$
BEGIN
    RETURN QUERY
    SELECT
      COALESCE(SUM(CASE WHEN sender_id   = p_customer_id THEN amount END), 0),
      COALESCE(SUM(CASE WHEN receiver_id = p_customer_id THEN amount END), 0),
      COUNT(*)::BIGINT
    FROM transfers
    WHERE sender_id = p_customer_id OR receiver_id = p_customer_id;
END; $$ LANGUAGE plpgsql;
```

**2) Repository: misma sintaxis `@Query nativeQuery = true`.**

```java
public interface TransferRepository extends JpaRepository<Transfer, Long> {
    @Query(value = "SELECT * FROM sp_customer_stats(:customerId)", nativeQuery = true)
    Object[] getStatsFor(@Param("customerId") Long customerId);
}
```

**3) DTO (formato del reporte):**

```java
public record CustomerStatsDTO(
    BigDecimal totalEnviado,
    BigDecimal totalRecibido,
    long cantidad
) {}
```

**4) Service (1 fila → 1 DTO):**

```java
public CustomerStatsDTO statsFor(Long id) {
    Object[] r = repo.getStatsFor(id);
    return new CustomerStatsDTO(
        (BigDecimal) r[0],
        (BigDecimal) r[1],
        ((Number) r[2]).longValue());
}
```

### 3.9 Beneficios y retos del DTO

**Beneficios:**

- La API queda estable: si renombras `targetAmount` a `goalAmount` en la tabla, los clientes no notan nada (solo actualizas el mapper).
- Datos sensibles fuera del JSON: la entity `Customer` tiene `password`, `dni`, `email`; el DTO solo expone `id` y `name`.
- Validación en un solo lugar: las reglas (`@NotBlank`, `@Future`, `@Size`) viven en el Request DTO y corren con `@Valid`.
- JSON más livianos: si la entity tiene 20 campos pero el reporte usa 4, solo viajan 4 — menos bytes en la red.

**Retos:**

- Más archivos por feature: cada caso de uso suma 1-2 DTOs + mapper. En proyectos chicos se siente como overhead.
- Decidir cuándo crear un DTO nuevo vs reutilizar: ¿el Request de crear sirve para actualizar? Caso por caso.
- Sincronizar: si agregas un campo en la entity, decides si va al DTO. MapStruct ayuda con warnings al compilar.
- Curva de MapStruct: configurar el `pom.xml` con Lombok, aprender `@Mapping`, `@Named`, `@AfterMapping` toma su tiempo.

### 3.10 Para recordar

| Concepto | Resumen |
|---|---|
| DTO | Objeto plano que transporta datos. Decide qué campos se exponen. |
| Capas | Vive entre Cliente, Controller y Service. El Repository NO usa DTOs. |
| Request vs Response | Request = lo que entra (con `@Valid`). Response = lo que sale (oculta sensibles). |
| Mapper | Convierte DTO ↔ Entity. MapStruct lo genera por ti en compile-time. |
| Reportes | Una Entity → varios DTOs. Un DTO por vista. El SERVICE los construye. |

[↑ Volver al indice](#indice)

---

## 4. Documentación API: Swagger y OpenAPI

Una API sin documentación viva es una caja negra. Swagger / OpenAPI resuelve eso generando documentación automática a partir del código y exponiendo una UI interactiva para explorarla.

### 4.1 ¿Qué son?

**OpenAPI** es el estándar para describir APIs REST. Es un archivo JSON / YAML que lista todos los endpoints, sus parámetros y sus respuestas.

**Swagger** son las herramientas que CONSUMEN la spec OpenAPI. La más usada: Swagger UI, una página web interactiva para ver y probar el API.

OpenAPI = el QUÉ (la spec). Swagger = el CÓMO (las herramientas).

### 4.2 ¿En qué etapa del desarrollo se usa?

Acompaña TODO el ciclo del API.

| Etapa | Para qué sirve |
|---|---|
| Desarrollo | El backend prueba sus endpoints sin abrir Postman. |
| QA / Testing | Explora la API para diseñar pruebas. |
| Integración | Otros equipos o terceros leen la spec para consumirla. |
| Documentación | Es la doc viva — siempre refleja el código real. |

### 4.3 ¿El frontend lo usa?

Sí, pero NO como herramienta diaria.

**Swagger** lo abre cuando duda del contrato:

- Onboarding del proyecto.
- Cuando algo se rompe (¿el contrato cambió?).
- Para confirmar qué body o response esperar.

Es la 'fuente de verdad' del API.

**Postman** lo usa todos los días para:

- Probar el endpoint que está integrando.
- Guardar requests con sus auth tokens.
- Compartir colecciones con el equipo.
- Automatizar tests.

### 4.4 Swagger y Postman: no son rivales

Cumplen propósitos distintos. Tienes los dos en tu proyecto.

| Aspecto | Swagger | Postman |
|---|---|---|
| Origen | Generado del código | Escrito a mano |
| Actualización | Siempre al día | Manual |
| Para qué sirve | Documentar / explorar | Probar / automatizar |
| Acceso | URL pública del server | Colección personal |

Lo que Swagger te da y Postman no:

- Se actualiza solo (no hay que editar nada cuando cambia un endpoint).
- Es público vía URL — terceros, QA y frontend ven TODO sin que les mandes la colección.
- Postman puede leer la spec y armarse la colección completa automáticamente (siguiente sección).

### 4.5 Bonus: Postman puede leer tu spec OpenAPI

En vez de armar cada request a mano, Postman las genera por ti desde la spec.

| Paso | Acción |
|---|---|
| 1 | En Postman click `Import` (arriba a la izquierda). |
| 2 | Pega la URL de la spec: `http://localhost:8080/v3/api-docs` |
| 3 | Postman lee la spec automáticamente. |
| 4 | Genera una colección con TODOS los endpoints — folders por `@Tag`, cada request con su método, URL, parámetros y body schema. |

Resultado: no creas requests a mano. Tu spec OpenAPI alimenta Postman.

[↑ Volver al indice](#indice)
