# Guía de Laboratorio 03 - A
## Persistencia con Spring Data JPA
**1ACC0236 Ingeniería de Software | Escuela CC | Ciclo 2026-1**
Elaborado por: Henry Antonio Mendoza Puerta

---

## 1. Persistencia con JPA, Hibernate y Spring Data JPA

Para que los datos de una aplicación Java sobrevivan al cierre del programa, deben guardarse en una base de datos. JPA define las reglas de cómo hacerlo. Hibernate las ejecuta. Spring Data JPA simplifica el acceso.

```
Tu código Java
   └─ llama a  BookRepository.save(book)          ← Spring Data JPA
         └─ delega en  EntityManager.persist()    ← JPA (especificación)
               └─ ejecuta  INSERT INTO books ...  ← Hibernate (implementación)
                     └─ escribe en  PostgreSQL    ← Base de datos
```

### 1.1 Anotaciones JPA — Referencia

| Anotación | Propósito |
|---|---|
| `@Entity` | Marca la clase como entidad gestionada por Hibernate. |
| `@Table(name=)` | Define el nombre exacto de la tabla en la base de datos. |
| `@Id` | Declara el campo como clave primaria. |
| `@GeneratedValue(IDENTITY)` | Delega el auto-incremento al SERIAL de PostgreSQL. |
| `@Column` | Configura la columna: nullable, unique, length, precision. |
| `@ManyToOne` | Muchos registros apuntan a uno. La FK queda en la tabla actual. |
| `@OneToMany(mappedBy)` | Un registro tiene muchos hijos. mappedBy indica el campo dueño. |
| `@JoinColumn(name=)` | Nombre de la columna de clave foránea en la tabla. |
| `cascade = CascadeType.ALL` | Las operaciones (persist, remove) se propagan a los hijos. |
| `orphanRemoval = true` | Elimina automáticamente los hijos que quedan sin padre. |
| `FetchType.LAZY` | Carga la relación solo cuando se accede. |
| `@PrePersist` | Ejecuta un método justo antes de insertar el registro en la BD. |

---

## 2. Objetivo

Al finalizar el laboratorio, el estudiante implementa entidades JPA con asociaciones sobre el dominio BookStore, define repositorios con los tres tipos de consulta (Query Method, JPQL y SQL nativo) y verifica la persistencia ejecutando un test desde el método main.

---

## 3. User Stories

### 3.1 Épicas

| Epic ID | Título | Descripción |
|---|---|---|
| EP01 | Gestión de Catálogo | Búsqueda, registro y administración del catálogo de libros y autores. |
| EP02 | Proceso de Compra | Registro de compras, seguimiento de pedidos y consulta de historial. |

### 3.2 Historias de Usuario

#### US-01 · EP01 — Registrar autor en el catálogo

COMO propietario de librería QUIERO registrar los datos de un autor PARA asociar sus libros al catálogo de BookStore.

**Escenario exitoso:** Dado que el propietario completa nombre, apellido y nacionalidad, Cuando presiona Guardar, Entonces el autor queda registrado y disponible para asociarse a libros.

**Escenario error:** Dado que el propietario omite el apellido, Cuando intenta guardar, Entonces el sistema muestra "El apellido del autor es obligatorio".

---

#### US-02 · EP01 — Registrar libro en el catálogo

COMO propietario de librería QUIERO registrar un libro con su autor, género y precio PARA que los lectores lo encuentren en el catálogo.

**Escenario exitoso:** Dado que el propietario completa todos los datos y selecciona un autor registrado, Cuando presiona Guardar, Entonces el libro aparece en el catálogo con estado activo.

**Escenario error:** Dado que el propietario ingresa un ISBN ya registrado, Cuando intenta guardar, Entonces el sistema muestra "El ISBN ya se encuentra registrado en el catálogo".

---

#### US-03 · EP01 — Buscar libros por género y precio máximo

COMO lector QUIERO buscar libros filtrando por género y precio máximo PARA encontrar opciones dentro de mi presupuesto.

**Escenario exitoso:** Dado que el lector selecciona género FICTION y precio máximo S/50, Cuando ejecuta la búsqueda, Entonces el sistema retorna únicamente libros activos con precio menor o igual a S/50.

**Escenario alternativo:** Dado que no existen libros en ese filtro, el sistema retorna lista vacía.

---

#### US-04 · EP02 — Registrar una compra

COMO lector QUIERO confirmar la compra de los libros seleccionados PARA generar un pedido con el detalle de los títulos adquiridos.

**Escenario exitoso:** Dado que el lector tiene al menos un libro con stock, Cuando confirma la compra, Entonces el sistema genera un Purchase con estado PENDING y sus PurchaseItems.

**Escenario error:** Dado que un libro tiene stock igual a cero, el sistema muestra "El libro [título] no tiene stock disponible".

---

#### US-05 · EP02 — Consultar historial de compras por rango de fechas

COMO lector QUIERO consultar mis compras filtrando por fecha de inicio y fin PARA revisar mis pedidos en un periodo específico.

**Escenario exitoso:** El sistema retorna las compras del rango indicado ordenadas de más reciente a más antigua.

**Escenario alternativo:** Si no hay compras en el periodo, retorna lista vacía.

---

## 4. Diagrama de Clases del Dominio

| Clase | Relación UML | Con | Tipo | Por qué |
|---|---|---|---|---|
| `Book` | Asociación `-->` | `Author` | `@ManyToOne` | Un libro referencia a un autor. Author existe independiente. |
| `Purchase` | Asociación `-->` | `Customer` | `@ManyToOne` | Una compra pertenece a un cliente. Customer existe independiente. |
| `Author` | Agregación `o-->` | `Book` | `@OneToMany` | Author agrupa Books pero ambos existen por separado. |
| `Purchase` | Composición `*-->` | `PurchaseItem` | `@OneToMany` | PurchaseItem no tiene sentido sin Purchase. |
| `PurchaseItem` | Asociación `-->` | `Book` | `@ManyToOne` | Un ítem referencia un libro. El libro existe independiente. |

---

## 5. Prerrequisitos

- Proyecto `bookstore_api` del Lab 02 abierto en IntelliJ IDEA
- Docker Desktop instalado y corriendo
- Archivo `archivos.zip` descargado del aula virtual

---

## 6. Levantar los Contenedores Docker

```bash
docker compose up -d
docker ps
```

Acceder a pgAdmin en http://localhost:5050 con `admin@admin.com` / `admin`.
Registrar servidor: host `postgres`, port `5432`, database `bookstore_db`, username `postgres`, password `adminadmin`.

---

## 7. Lombok — Anotaciones de Reducción de Código

| Anotación | Qué genera |
|---|---|
| `@Data` | Getters, setters, equals, hashCode y toString. |
| `@Builder` | Patrón Builder para construir objetos con sintaxis fluida. |
| `@NoArgsConstructor` | Constructor sin argumentos. JPA lo requiere obligatoriamente. |
| `@AllArgsConstructor` | Constructor con todos los campos como parámetros. |

---

## 8. Crear el Enum PurchaseStatus

```java
package com.bookstore.model.enums;

public enum PurchaseStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}
```

---

## 9. Implementar las Entidades JPA

### 9.1 Customer.java

```java
package com.bookstore.model;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "customers")
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String firstName;

    @Column(nullable = false, length = 100)
    private String lastName;

    @Column(length = 20)
    private String phone;

    @Column(length = 300)
    private String address;

    @Column(updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() { this.createdAt = LocalDateTime.now(); }
}
```

### 9.2 Author.java

```java
package com.bookstore.model;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "authors")
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String firstName;

    @Column(nullable = false, length = 100)
    private String lastName;

    @Column(length = 80)
    private String nationality;
}
```

### 9.3 Book.java

```java
package com.bookstore.model;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "books")
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(nullable = false, unique = true, length = 13)
    private String isbn;

    @Column(length = 100)
    private String genre;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock;

    @Column(nullable = false)
    private Integer minStock;

    @Column(length = 2000)
    private String description;

    @Column(nullable = false, unique = true, length = 300)
    private String slug;

    @Column(length = 500)
    private String imageUrl;

    @Column(length = 500)
    private String fileUrl;

    @Column(nullable = false)
    private boolean isActive;

    @Column(updatable = false)
    private LocalDateTime createdAt;

    @ManyToOne(fetch = FetchType.LAZY)                     // <-- cambia de String a entidad Author
    @JoinColumn(name = "author_id", nullable = false)
    private Author author;

    @PrePersist
    protected void onCreate() { this.createdAt = LocalDateTime.now(); }
}
```

### 9.4 Purchase.java

```java
package com.bookstore.model;

import com.bookstore.model.enums.PurchaseStatus;
import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "purchases")
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class Purchase {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(updatable = false)
    private LocalDateTime purchaseDate;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private PurchaseStatus status;

    @Column(length = 300)
    private String shippingAddress;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @OneToMany(mappedBy = "purchase",
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<PurchaseItem> items;

    @PrePersist
    protected void onCreate() { this.purchaseDate = LocalDateTime.now(); }
}
```

### 9.5 PurchaseItem.java

```java
package com.bookstore.model;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;

@Entity
@Table(name = "purchase_items")
@Data @Builder @NoArgsConstructor @AllArgsConstructor
public class PurchaseItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Integer quantity;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal unitPrice;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal subtotal;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "purchase_id", nullable = false)
    private Purchase purchase;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id", nullable = false)
    private Book book;
}
```

---

## 10. Repositorios Spring Data JPA

### 10.1 Los tres tipos de consulta

| Tipo | Cómo funciona | Cuándo usar |
|---|---|---|
| Query Method | Spring genera el SQL a partir del nombre del método | Búsquedas simples por uno o dos atributos |
| JPQL (`@Query`) | Consulta sobre clases Java, no sobre tablas SQL | Lógica de negocio y condiciones compuestas |
| SQL nativo | SQL puro de PostgreSQL (`nativeQuery = true`) | Funciones de BD, agrupaciones, optimización |

### 10.2 CustomerRepository.java

```java
package com.bookstore.repository;

import com.bookstore.model.Customer;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CustomerRepository extends JpaRepository<Customer, Long> { }
```

### 10.3 AuthorRepository.java

```java
package com.bookstore.repository;

import com.bookstore.model.Author;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface AuthorRepository extends JpaRepository<Author, Long> {

    // Verifica si ya existe un autor con ese apellido — evita duplicados (US-01)
    boolean existsByLastName(String lastName);

    // Retorna autor por apellido exacto — búsqueda en CRUD (US-01)
    Optional<Author> findByLastName(String lastName);
}
```

### 10.4 BookRepository.java

```java
package com.bookstore.repository;

import com.bookstore.model.Book;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

public interface BookRepository extends JpaRepository<Book, Long> {

    // ── Lab 02 — métodos existentes, se mantienen ─────────────────────────────
    List<Book> findByTitle(String title);
    Optional<Book> findBySlug(String slug);
    boolean existsBySlug(String slug);

    @Query("SELECT b FROM Book b WHERE b.stock < b.minStock")
    List<Book> findLowStockBooks();

    @Query(value = "SELECT * FROM books WHERE price > 50", nativeQuery = true)
    List<Book> findExpensiveBooks();

    // ── Lab 03 — métodos nuevos ───────────────────────────────────────────────

    // Retorna libro por ISBN — valida unicidad al registrar (US-02)
    Optional<Book> findByIsbn(String isbn);
    boolean existsByIsbn(String isbn);

    // Verifica que el libro tenga stock antes de permitir la compra (US-04)
    Optional<Book> findByIdAndStockGreaterThan(Long id, int stock);

    // Filtra libros activos por género — primer nivel de búsqueda (US-03)
    List<Book> findByGenreAndIsActiveTrue(String genre);

    // Filtra libros activos por género y precio máximo — búsqueda combinada (US-03)
    @Query("""
           SELECT b FROM Book b
           WHERE b.genre = :genre
             AND b.price <= :maxPrice
             AND b.isActive = true
           """)
    List<Book> findByGenreAndMaxPrice(@Param("genre") String genre,
                                      @Param("maxPrice") BigDecimal maxPrice);
}
```

### 10.5 PurchaseRepository.java

```java
package com.bookstore.repository;

import com.bookstore.model.Purchase;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

public interface PurchaseRepository extends JpaRepository<Purchase, Long> {

    // Retorna todas las compras de un cliente — historial general (US-04)
    List<Purchase> findByCustomerId(Long customerId);

    // Filtra compras de un cliente entre dos fechas ordenadas descendente (US-05)
    @Query("""
           SELECT p FROM Purchase p
           WHERE p.customer.id = :customerId
             AND p.purchaseDate BETWEEN :from AND :to
           ORDER BY p.purchaseDate DESC
           """)
    List<Purchase> findByCustomerAndDateRange(@Param("customerId") Long customerId,
                                              @Param("from") LocalDateTime from,
                                              @Param("to") LocalDateTime to);

    // Suma el total gastado por el cliente en compras con estado DELIVERED (US-05)
    @Query(value = """
           SELECT COALESCE(SUM(total_amount), 0)
           FROM purchases
           WHERE customer_id = :customerId
             AND status = 'DELIVERED'
           """, nativeQuery = true)
    BigDecimal getTotalSpentByCustomer(@Param("customerId") Long customerId);
}
```

---

## 11. Test Rápido desde el Método Main

```java
package com.bookstore;

import com.bookstore.model.*;
import com.bookstore.model.enums.PurchaseStatus;
import com.bookstore.repository.*;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;

@SpringBootApplication
public class BookstoreApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(BookstoreApiApplication.class, args);
    }

    @Bean
    CommandLineRunner initData(CustomerRepository customerRepo,
                               AuthorRepository authorRepo,
                               BookRepository bookRepo,
                               PurchaseRepository purchaseRepo) {
        return args -> {

            // 1. Clientes
            Customer pedro = customerRepo.save(Customer.builder()
                    .firstName("Pedro").lastName("Ramirez")
                    .phone("999111222")
                    .address("Av. Larco 123, Miraflores").build());
            Customer sofia = customerRepo.save(Customer.builder()
                    .firstName("Sofia").lastName("Torres")
                    .phone("999333444")
                    .address("Jr. Union 456, Cercado de Lima").build());

            // 2. Autor (US-01)
            Author vargas = authorRepo.save(Author.builder()
                    .firstName("Mario").lastName("Vargas Llosa")
                    .nationality("Peruana").build());

            // 3. Libros (US-02)
            Book libro1 = bookRepo.save(Book.builder()
                    .title("La ciudad y los perros")
                    .isbn("9780374529529").genre("FICTION")
                    .price(new BigDecimal("45.00"))
                    .stock(10).minStock(5).isActive(true)
                    .slug("la-ciudad-y-los-perros")
                    .author(vargas).build());

            Book libro2 = bookRepo.save(Book.builder()
                    .title("Cosmos")
                    .isbn("9780345539434").genre("SCIENCE")
                    .price(new BigDecimal("55.00"))
                    .stock(3).minStock(5).isActive(true)
                    .slug("cosmos")
                    .author(vargas).build());

            // 4. Compra con items (US-04)
            PurchaseItem item = PurchaseItem.builder()
                    .book(libro1).quantity(2)
                    .unitPrice(libro1.getPrice())
                    .subtotal(libro1.getPrice().multiply(new BigDecimal("2")))
                    .build();
            Purchase compra = Purchase.builder()
                    .customer(sofia)
                    .totalAmount(item.getSubtotal())
                    .status(PurchaseStatus.PENDING)
                    .shippingAddress("Jr. Union 456, Cercado de Lima")
                    .build();
            compra.setItems(new ArrayList<>());
            compra.getItems().add(item);
            item.setPurchase(compra);
            purchaseRepo.save(compra);

            // ── Query Method (US-03) ──────────────────────────────────────────
            System.out.println("\n=== Query Method — libros FICTION ===");
            bookRepo.findByGenreAndIsActiveTrue("FICTION")
                    .forEach(b -> System.out.println("  " + b.getTitle()));

            // ── JPQL (US-03) ──────────────────────────────────────────────────
            System.out.println("\n=== JPQL — FICTION <= S/50 ===");
            bookRepo.findByGenreAndMaxPrice("FICTION", new BigDecimal("50"))
                    .forEach(b -> System.out.println("  " + b.getTitle() + " S/" + b.getPrice()));

            // ── JPQL con fechas (US-05) ───────────────────────────────────────
            System.out.println("\n=== JPQL — compras de sofia hoy ===");
            purchaseRepo.findByCustomerAndDateRange(
                    sofia.getId(),
                    LocalDateTime.now().minusDays(1),
                    LocalDateTime.now().plusDays(1))
                    .forEach(p -> System.out.println("  Compra #" + p.getId()
                            + " S/" + p.getTotalAmount()));

            // ── SQL nativo (US-05) ────────────────────────────────────────────
            System.out.println("\n=== SQL Nativo — total gastado sofia ===");
            System.out.println("  S/" + purchaseRepo.getTotalSpentByCustomer(sofia.getId()));
            System.out.println("  (status PENDING, no DELIVERED — resultado: 0)");
        };
    }
}
```

---

## 12. Verificación

### 12.1 Salida esperada en consola

```
=== Query Method — libros FICTION ===
  La ciudad y los perros

=== JPQL — FICTION <= S/50 ===
  La ciudad y los perros  S/45.00

=== JPQL — compras de sofia hoy ===
  Compra #1  S/90.00

=== SQL Nativo — total gastado sofia ===
  S/0  (status es PENDING, no DELIVERED)
```

### 12.2 Verificar tablas en pgAdmin

1. Navegar a `bookstore_db → Schemas → public → Tables`
2. Verificar que existen: `customers`, `authors`, `books`, `purchases`, `purchase_items`
3. En `books` confirmar columna `author_id` como FK hacia `authors`
4. En `purchases` confirmar columna `customer_id` como FK hacia `customers`

---

## 13. Publicar en GitHub

```bash
git checkout -b feature/jpa-entities
git add .
git commit -m "feat: implementar entidades JPA y repositorios"
git push -u origin feature/jpa-entities
```

**PR:** `base: develop ← compare: feature/jpa-entities`
