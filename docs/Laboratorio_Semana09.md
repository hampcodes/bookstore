# Integracion Frontend Backend

## Integración Angular con `pagoya-api`

Elaborado por: Henry Antonio Mendoza Puerta

---

## Tabla de contenidos

- [Sección 01: ¿Qué es Angular?](#sección-01-qué-es-angular)
- [Sección 02: Preparar el backend (`pagoya-api`)](#sección-02-preparar-el-backend-pagoya-api)
- [Sección 03: Crear el proyecto y la estructura](#sección-03-crear-el-proyecto-y-la-estructura)
- [Sección 04: Generar los componentes de las interfaces de usuario](#sección-04-generar-los-componentes-de-las-interfaces-de-usuario)
- [Sección 05: Configurar las rutas](#sección-05-configurar-las-rutas)
- [Sección 06: Modelos](#sección-06-modelos)
- [Sección 07: Variables de entorno](#sección-07-variables-de-entorno)
- [Sección 08: Service de transferencias](#sección-08-service-de-transferencias)
- [Sección 09: Listado de transferencias](#sección-09-listado-de-transferencias)
- [Sección 10: Formulario de transferencias](#sección-10-formulario-de-transferencias)
- [Sección 11: Cuentas](#sección-11-cuentas)
- [Sección 12: Resumen](#sección-12-resumen)

---

## Objetivo

Construir una app Angular conectada al backend `pagoya-api` por HTTP, con estructura profesional, rutas hijas y dos interfaces de usuario completas: transferencias y cuentas.

---

## Sección 01: ¿Qué es Angular?

**Angular** es un framework de Google para SPA, escrito en TypeScript. Usamos **Angular 21** ([angular.dev](https://angular.dev/)).

| Característica          | Descripción                                                          |
|-------------------------|----------------------------------------------------------------------|
| Standalone components   | Componentes sin `NgModule`; cada uno declara sus imports.            |
| Signals                 | Valores reactivos del cliente; notifican a la vista al cambiar.      |
| Reactive Forms          | Formularios tipados con validaciones declarativas.                   |
| Zoneless                | Sin `zone.js`; detección de cambios basada en signals.               |
| Control de flujo nativo | `@if`, `@for`, `@switch` reemplazan `*ngIf` y `*ngFor`.              |
| Vitest                  | Test runner por defecto en proyectos nuevos.                         |

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 02: Preparar el backend (`pagoya-api`)

### 2.1 Arrancar el backend

Desde `pagoya-api`:

```bash
mvn spring-boot:run
```

Queda en `http://localhost:8080`. Swagger en `http://localhost:8080/swagger-ui.html`.

### 2.2 CORS y abrir `/api/transfers/**` y `/api/accounts/**`

> **¿Por qué CORS?** El navegador bloquea por *same-origin* las peticiones de `http://localhost:4200` a `http://localhost:8080`. El backend debe autorizarlo con `Access-Control-Allow-Origin`.

Crear **`pagoya-api/src/main/java/com/hampcode/pagoya/shared/config/CorsConfig.java`**:

```java
package com.hampcode.pagoya.shared.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("http://localhost:4200"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

Reemplazar **`pagoya-api/src/main/java/com/hampcode/pagoya/shared/config/SecurityConfig.java`**:

```java
package com.hampcode.pagoya.shared.config;

import com.hampcode.pagoya.auth.security.JwtAuthenticationFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.Customizer;
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
            .cors(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
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
                .requestMatchers("/api/transfers/**").permitAll()
                .requestMatchers("/api/accounts/**").permitAll()
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

Cambios:

- `.cors(Customizer.withDefaults())` → enchufa el bean `corsConfigurationSource` de `CorsConfig`.
- `.requestMatchers("/api/transfers/**").permitAll()` y `.requestMatchers("/api/accounts/**").permitAll()` → abren los endpoints sin token mientras no hay login.

Reiniciar el backend.

### 2.3 Probar en Swagger

Abrir `http://localhost:8080/swagger-ui.html` y ejecutar:

- `GET /api/transfers/account/191-001` → `200`.
- `GET /api/accounts/customer/1` → `200`.

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 03: Crear el proyecto y la estructura

### 3.1 Crear el proyecto

```bash
ng new pagoya-web --style=css --ssr=false
cd pagoya-web
ng serve
```

`http://localhost:4200`.

### 3.2 Estructura objetivo

```text
src/
├── environments/
│   ├── environment.ts
│   └── environment.development.ts
└── app/
    ├── core/
    ├── layouts/
    │   ├── main-layout/
    │   └── auth-layout/
    ├── pages/
    │   ├── transfers/
    │   │   ├── list/
    │   │   ├── form/
    │   │   └── transfers.routes.ts
    │   └── accounts/
    │       ├── list/
    │       └── accounts.routes.ts
    ├── services/
    │   ├── transfer.service.ts
    │   └── account.service.ts
    ├── models/
    │   ├── page-response.ts
    │   ├── currency.ts
    │   ├── transfer-status.ts
    │   ├── transfer-request.ts
    │   ├── transfer-response.ts
    │   ├── account-status.ts
    │   ├── account-type.ts
    │   └── account-response.ts
    ├── app.ts
    ├── app.html
    ├── app.config.ts
    └── app.routes.ts
```

| Carpeta          | Para qué sirve                                                          |
|------------------|-------------------------------------------------------------------------|
| `environments/`  | Variables por entorno (URLs del API, valores por defecto).              |
| `core/`          | Servicios singleton (guards, interceptors).                             |
| `layouts/`       | Envolturas visuales (privada, pública).                                 |
| `pages/`         | Una carpeta por interfaz de usuario con subpáginas y archivo de rutas.  |
| `services/`      | Llamadas al API, uno por dominio.                                       |
| `models/`        | Interfaces y enums que replican los DTOs del backend.                   |

### 3.3 Crear carpetas y los layouts

```bash
mkdir -p src/app/core src/app/models src/app/services
ng g c layouts/main-layout --skip-tests
ng g c layouts/auth-layout --skip-tests
```

> `--skip-tests` evita el archivo `.spec.ts`.
> En Angular 21 todos los componentes son **standalone por defecto** (ya no hace falta `standalone: true`). Cada componente declara sus dependencias en `imports: [...]` y ya no se usan `NgModule`.

### 3.4 `main-layout` (barra superior + menú)

**`src/app/layouts/main-layout/main-layout.ts`**:

```ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-main-layout',
  imports: [RouterLink, RouterLinkActive, RouterOutlet],
  templateUrl: './main-layout.html',
  styleUrl: './main-layout.css',
})
export class MainLayout {}
```

**`src/app/layouts/main-layout/main-layout.html`**:

```html
<nav class="topbar">
  <a routerLink="/transfers" class="brand">Pagoya</a>
  <a routerLink="/transfers" routerLinkActive="active">Transferencias</a>
  <a routerLink="/accounts" routerLinkActive="active">Cuentas</a>
</nav>
<main class="content">
  <router-outlet />
</main>
```

**`src/app/layouts/main-layout/main-layout.css`**:

```css
.topbar {
  display: flex;
  gap: 24px;
  background: #263238;
  color: #fff;
  padding: 12px 24px;
}
.topbar a { color: #cfd8dc; text-decoration: none; }
.topbar a.active, .brand { color: #fff; }
.brand { font-weight: bold; }
.content { padding: 24px; }
```

### 3.5 `auth-layout` (queda listo para login/registro)

Tarjeta centrada en la ventana. No se conecta a rutas todavía.

**`src/app/layouts/auth-layout/auth-layout.ts`**:

```ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-auth-layout',
  imports: [RouterOutlet],
  templateUrl: './auth-layout.html',
  styleUrl: './auth-layout.css',
})
export class AuthLayout {}
```

**`src/app/layouts/auth-layout/auth-layout.html`**:

```html
<main class="auth-shell">
  <section class="auth-card">
    <h1>Pagoya</h1>
    <router-outlet />
  </section>
</main>
```

**`src/app/layouts/auth-layout/auth-layout.css`**:

```css
.auth-shell {
  min-height: 100vh;
  display: grid;
  place-items: center;
  background: #263238;
}
.auth-card {
  background: #fff;
  padding: 32px;
  border-radius: 8px;
  min-width: 320px;
}
.auth-card h1 { margin: 0 0 16px; text-align: center; color: #d32f2f; }
```

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 04: Generar los componentes de las interfaces de usuario

Antes de tocar las rutas, generamos los componentes vacíos de cada interfaz de usuario. En este paso solo los **creamos**; el contenido (consumo del service, formulario, etc.) se completa en los pasos 9, 10 y 11.

```bash
ng g c pages/transfers/list --skip-tests
ng g c pages/transfers/form --skip-tests
ng g c pages/accounts/list --skip-tests
```

Esto genera:

- `src/app/pages/transfers/list/` con `list.ts`, `list.html`, `list.css`.
- `src/app/pages/transfers/form/` con `form.ts`, `form.html`, `form.css`.
- `src/app/pages/accounts/list/` con `list.ts`, `list.html`, `list.css`.

Las carpetas `pages/transfers/` y `pages/accounts/` las crea automáticamente el CLI al generar el primer componente de cada interfaz.

> Los componentes quedan con el HTML por defecto que pone el CLI (algo como `<p>list works!</p>`). Es lo esperado: en el paso 5 las rutas los apuntan, y en los pasos 9–11 los llenamos con la lógica real.

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 05: Configurar las rutas

Configuramos el router con `MainLayout` envolviendo a todas las interfaces de usuario y **rutas hijas con lazy load** por área.

### 5.1 Archivo de rutas de cada interfaz

Los archivos `*.routes.ts` **no los genera el CLI**: los creamos a mano dentro de cada carpeta de área (las carpetas ya existen del paso 4). Cada archivo declara qué componente responde a cada path.

Crear **`src/app/pages/transfers/transfers.routes.ts`**:

```ts
import { Routes } from '@angular/router';
import { List } from './list/list';
import { Form } from './form/form';

export const TRANSFERS_ROUTES: Routes = [
  { path: '',    component: List },
  { path: 'new', component: Form },
];
```

Crear **`src/app/pages/accounts/accounts.routes.ts`**:

```ts
import { Routes } from '@angular/router';
import { List } from './list/list';

export const ACCOUNTS_ROUTES: Routes = [
  { path: '', component: List },
];
```

### 5.2 `app.routes.ts` con `MainLayout` y `loadChildren`

Reemplazar **`src/app/app.routes.ts`**:

```ts
import { Routes } from '@angular/router';
import { MainLayout } from './layouts/main-layout/main-layout';

export const routes: Routes = [
  {
    path: '',
    component: MainLayout,
    children: [
      { path: '', redirectTo: 'transfers', pathMatch: 'full' },
      {
        path: 'transfers',
        loadChildren: () =>
          import('./pages/transfers/transfers.routes').then(m => m.TRANSFERS_ROUTES),
      },
      {
        path: 'accounts',
        loadChildren: () =>
          import('./pages/accounts/accounts.routes').then(m => m.ACCOUNTS_ROUTES),
      },
    ],
  },
];
```

Reemplazar **`src/app/app.html`**:

```html
<router-outlet />
```

**Probar en el navegador:** `http://localhost:4200` redirige a `/transfers`. Se ve la barra superior con menú **Transferencias / Cuentas** y, debajo, el HTML por defecto del componente `List` de transferencias. Clic en **Cuentas** → URL cambia a `/accounts` y aparece el HTML por defecto del `List` de cuentas. No hay errores en la consola.

| Concepto | Significado |
|---|---|
| `component: MainLayout` | Envoltura común a todas las páginas hijas. |
| `children: [...]` | Rutas que renderizan dentro del `router-outlet` del padre. |
| `loadChildren: () => ...` | Carga un archivo `*.routes.ts` solo al navegar (lazy). |
| `<router-outlet />` / `RouterOutlet` | Etiqueta del template que marca el lugar donde Angular renderiza el componente de la ruta activa. Para usarla, hay que importar `RouterOutlet` en el `imports` del componente. |
| `routerLink="..."` / `RouterLink` | Atributo que convierte un `<a>` en un link del router: navega a la ruta indicada sin recargar la página. Para usarlo, hay que importar `RouterLink`. |
| `routerLinkActive="x"` / `RouterLinkActive` | Atributo que aplica la clase CSS `x` al `<a>` cuando su `routerLink` coincide con la URL actual (típicamente se usa para resaltar el ítem activo del menú). Para usarlo, hay que importar `RouterLinkActive`. |

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 06: Modelos

Interfaces y enums que replican los DTOs del backend. Los enums son **string enums** para que el JSON coincida (`Currency.PEN` → `"PEN"`).

Generar todos con la CLI:

```bash
ng g interface models/page-response
ng g enum models/currency
ng g enum models/transfer-status
ng g interface models/transfer-request
ng g interface models/transfer-response
ng g enum models/account-status
ng g enum models/account-type
ng g interface models/account-response
```

**`src/app/models/page-response.ts`**:

```ts
export interface PageResponse<T> {
  content: T[];
  page: number;
  size: number;
  totalElements: number;
  totalPages: number;
  first: boolean;
  last: boolean;
}
```

**`src/app/models/currency.ts`**:

```ts
export enum Currency {
  PEN = 'PEN',
  USD = 'USD',
  EUR = 'EUR',
  GBP = 'GBP',
}
```

**`src/app/models/transfer-status.ts`**:

```ts
export enum TransferStatus {
  PENDING = 'PENDING',
  COMPLETED = 'COMPLETED',
  FAILED = 'FAILED',
}
```

**`src/app/models/transfer-request.ts`**:

```ts
import { Currency } from './currency';

export interface TransferRequest {
  sourceAccountNumber: string;
  targetAccountNumber: string;
  amount: number;
  currency: Currency;
}
```

**`src/app/models/transfer-response.ts`**:

```ts
import { Currency } from './currency';
import { TransferStatus } from './transfer-status';

export interface TransferResponse {
  id: number;
  sourceAccountNumber: string;
  targetAccountNumber: string;
  amount: number;
  currency: Currency;
  exchangeRate: number;
  status: TransferStatus;
  createdAt: string;
}
```

**`src/app/models/account-status.ts`**:

```ts
export enum AccountStatus {
  ACTIVE = 'ACTIVE',
  SUSPENDED = 'SUSPENDED',
  CLOSED = 'CLOSED',
}
```

**`src/app/models/account-type.ts`**:

```ts
export enum AccountType {
  SAVINGS = 'SAVINGS',
  CHECKING = 'CHECKING',
}
```

**`src/app/models/account-response.ts`**:

```ts
import { AccountStatus } from './account-status';
import { AccountType } from './account-type';

export interface AccountResponse {
  id: number;
  accountNumber: string;
  balance: number;
  status: AccountStatus;
  type: AccountType;
  customerId: number;
}
```

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 07: Variables de entorno

La URL base del backend y otros valores que cambian entre entornos (dev, qa, prod) no se hardcodean en el código. Van en archivos de **environment**: el build los reemplaza automáticamente según la configuración.

### 7.1 Generar los archivos de entorno

```bash
ng generate environments
```

Crea:

- `src/environments/environment.ts` — usado en **producción**.
- `src/environments/environment.development.ts` — usado en **desarrollo** (`ng serve` lo aplica por defecto).

### 7.2 Llenar `environment.development.ts`

Reemplazar **`src/environments/environment.development.ts`**:

```ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api',
  defaultAccount: '191-001',
  defaultCustomerId: 1,
};
```

`defaultAccount` y `defaultCustomerId` son valores de demo para esta sesión (no hay login todavía). Cuando se agregue autenticación, esos datos vendrán del usuario logueado.

### 7.3 Llenar `environment.ts`

Reemplazar **`src/environments/environment.ts`** (mismo shape, valores de producción cuando exista ese entorno; por ahora idéntico):

```ts
export const environment = {
  production: true,
  apiUrl: 'http://localhost:8080/api',
  defaultAccount: '191-001',
  defaultCustomerId: 1,
};
```

Los services y componentes importan siempre desde `'../../environments/environment'`. El build elige cuál usar según la configuración (`development` vs producción).

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 08: Service de transferencias

`TransferService` es la única capa que se comunica con el backend por HTTP. Devuelve `Observable<T>`; el componente se encarga de suscribirse. Antes activamos `HttpClient` a nivel de aplicación.

### 8.1 Habilitar `HttpClient`

Reemplazar **`src/app/app.config.ts`**:

```ts
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
    provideRouter(routes),
    provideHttpClient(),
  ],
};
```

> **¿Qué hace `provideZonelessChangeDetection()`?** Habilita la detección de cambios **sin `zone.js`**. Antes, Angular dependía de `zone.js` para interceptar APIs del navegador (`setTimeout`, eventos, `Promise`) y disparar el ciclo de detección automáticamente. En Angular 21 las apps nuevas vienen **zoneless por defecto**: la detección la disparan los signals, los eventos del template y el router. Resultado: bundles más chicos, menos trabajo de change detection, y un modelo más predecible.

### 8.2 Observable (lo que devuelve `HttpClient`)

Un **Observable** representa una operación **asíncrona** que va a emitir un valor en el futuro. `HttpClient.get` y `HttpClient.post` siempre devuelven `Observable<T>`. La petición HTTP **no se dispara** hasta que alguien llama `.subscribe(...)` — esa es la naturaleza *lazy* del Observable.

```ts
// Devuelve Observable<T> — todavía no salió ninguna request.
const respuesta$ = this.http.get<TransferResponse>(url);

// Al suscribirse, se dispara la request y llega el callback con la data.
respuesta$.subscribe(value => console.log(value));
```

Operadores útiles encadenados con `.pipe(...)`:

| Operador     | Para qué                                                          |
|--------------|-------------------------------------------------------------------|
| `map(fn)`    | Transformar el valor emitido (ej. extraer un campo).              |
| `tap(fn)`    | Ejecutar un side effect (logging, debug) sin alterar el stream.   |
| `catchError` | Capturar un error y devolver un valor alternativo o re-lanzarlo.  |

En este lab el service usa `pipe(map(page => page.content))` para que el componente reciba directamente el array, no el objeto paginado completo. El `.subscribe(...)` va **en el componente** (sección 9).

### 8.3 Generar el service

```bash
ng g s services/transfer --skip-tests
```

### 8.4 Llenar el service

**`src/app/services/transfer.service.ts`**:

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, map } from 'rxjs';
import { environment } from '../../environments/environment';
import { PageResponse } from '../models/page-response';
import { TransferRequest } from '../models/transfer-request';
import { TransferResponse } from '../models/transfer-response';

@Injectable({ providedIn: 'root' })
export class TransferService {
  private http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/transfers`;

  listByAccount(accountNumber: string): Observable<TransferResponse[]> {
    return this.http
      .get<PageResponse<TransferResponse>>(`${this.baseUrl}/account/${accountNumber}`)
      .pipe(map(page => page.content));
  }

  create(request: TransferRequest): Observable<TransferResponse> {
    return this.http.post<TransferResponse>(this.baseUrl, request);
  }
}
```

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 09: Listado de transferencias

El componente `List` ya existe (paso 4). Acá lo llenamos para que consuma el service y aparezcan los datos del backend. Aparecen dos conceptos nuevos: **Signals** (el estado vive en el componente) y **control de flujo nativo** del template.

### 9.1 Signal (estado local del componente)

Un **Signal** es un valor reactivo del cliente. Siempre tiene un valor presente y notifica a la vista cuando cambia. Se lee como función.

```ts
const counter = signal(0);
counter();                  // lee → 0
counter.set(5);             // reemplaza → 5
counter.update(n => n + 1); // actualiza → 6
```

Variantes:

| Forma                  | Para qué                                                              |
|------------------------|-----------------------------------------------------------------------|
| `signal<T>(initial)`   | Signal mutable.                                                       |
| `.asReadonly()`        | Versión de solo lectura para exponer al exterior sin permitir mutación. |
| `computed(() => ...)`  | Valor derivado que recalcula automáticamente al cambiar el signal del que depende. |

En este componente guardamos la lista que llega del service en un signal privado, exponemos una versión `asReadonly()` y derivamos el total en PEN con `computed`.

### 9.2 Control de flujo nativo

Desde Angular 17 los templates usan bloques integrados.

| Antes                       | Ahora                                       |
|-----------------------------|---------------------------------------------|
| `*ngIf="cond"`              | `@if (cond) { ... } @else { ... }`          |
| `*ngFor="let x of items"`   | `@for (x of items; track x.id) { ... }`     |
| `*ngFor` sin vacío integrado | `@for (...) { ... } @empty { ... }`        |
| `*ngSwitch`                 | `@switch (v) { @case (1) { ... } ... }`     |

`track` es obligatorio en `@for`: da una clave estable para re-render eficiente.

### 9.3 Llenar `list.ts`

**`src/app/pages/transfers/list/list.ts`**:

```ts
import { Component, OnInit, computed, inject, signal } from '@angular/core';
import { RouterLink } from '@angular/router';
import { environment } from '../../../../environments/environment';
import { Currency } from '../../../models/currency';
import { TransferResponse } from '../../../models/transfer-response';
import { TransferService } from '../../../services/transfer.service';

@Component({
  selector: 'app-transfers-list',
  imports: [RouterLink],
  templateUrl: './list.html',
  styleUrl: './list.css',
})
export class List implements OnInit {
  private service = inject(TransferService);

  title = 'Transferencias';
  private _transfers = signal<TransferResponse[]>([]);
  transfers = this._transfers.asReadonly();
  totalPEN = computed(() =>
    this._transfers()
      .filter(t => t.currency === Currency.PEN)
      .reduce((acc, t) => acc + t.amount, 0)
  );

  ngOnInit(): void {
    this.service
      .listByAccount(environment.defaultAccount)
      .subscribe(transfers => this._transfers.set(transfers));
  }
}
```

### 9.4 Llenar `list.html`

**`src/app/pages/transfers/list/list.html`**:

```html
<h1>{{ title }}</h1>
<a routerLink="/transfers/new" class="btn">Nueva transferencia</a>

<p>Total en PEN: S/. {{ totalPEN() }}</p>

@for (t of transfers(); track t.id) {
  <div class="item">
    <span>{{ t.sourceAccountNumber }} → {{ t.targetAccountNumber }}</span>
    <span>{{ t.amount }} {{ t.currency }}</span>
    <span [class.completed]="t.status === 'COMPLETED'"
          [class.pending]="t.status === 'PENDING'">{{ t.status }}</span>
  </div>
} @empty {
  <p>Aún no hay transferencias.</p>
}
```

### 9.5 Llenar `list.css`

**`src/app/pages/transfers/list/list.css`**:

```css
.btn { display: inline-block; padding: 8px 16px; background: #d32f2f; color: #fff; text-decoration: none; border-radius: 4px; }
.item { display: flex; justify-content: space-between; gap: 12px; padding: 12px; border: 1px solid #ccc; border-radius: 4px; margin-bottom: 8px; }
.completed { color: #2e7d32; font-weight: bold; }
.pending { color: #ef6c00; font-weight: bold; }
```

**Probar en el navegador:** `/transfers` muestra la lista del backend (o "Aún no hay transferencias" si está vacía). En DevTools → Network ver `GET /api/transfers/account/191-001` con `200`.

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 10: Formulario de transferencias

El componente `Form` ya existe (paso 4). Lo llenamos con un Reactive Form que valida los campos y hace `POST` al backend a través del service.

### 10.1 Reactive Forms (no template-driven)

| Aspecto         | Template-driven           | Reactive Forms ✅                                |
|-----------------|---------------------------|--------------------------------------------------|
| Lógica          | En el HTML con `ngModel`  | En la clase TS con `FormGroup`/`FormControl`     |
| Tipado          | Limitado                  | Fuerte (`FormBuilder.nonNullable.group({...})`)  |
| Validaciones    | Atributos del template    | `Validators` en la clase                         |

### 10.2 Validators más usados

| Validator              | Uso                          |
|------------------------|------------------------------|
| `Validators.required`  | Campo obligatorio.           |
| `Validators.min(n)`    | Valor numérico mínimo.       |
| `Validators.max(n)`    | Valor numérico máximo.       |
| `Validators.minLength` | Longitud mínima de string.   |
| `Validators.email`     | Formato de email.            |
| `Validators.pattern`   | Regex.                       |

Estados del control / form: `valid` / `invalid` / `touched` / `dirty` / `pending`.

**Patrón usado:** el botón **Transferir** se deshabilita mientras `form.invalid` sea `true`.

### 10.3 Validaciones aplicadas en este formulario

| Campo                  | Validación             | Por qué                                       |
|------------------------|------------------------|-----------------------------------------------|
| `sourceAccountNumber`  | `required`             | El backend rechaza el POST si está vacío.     |
| `targetAccountNumber`  | `required`             | Idem.                                         |
| `amount`               | `required`, `min(1)`   | El backend exige monto ≥ 1.                   |
| `currency`             | `required`             | El backend acepta solo `PEN/USD/EUR/GBP`.     |

### 10.4 Llenar `form.ts`

**`src/app/pages/transfers/form/form.ts`**:

```ts
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { Router, RouterLink } from '@angular/router';
import { Currency } from '../../../models/currency';
import { TransferRequest } from '../../../models/transfer-request';
import { TransferService } from '../../../services/transfer.service';

@Component({
  selector: 'app-transfer-form',
  imports: [ReactiveFormsModule, RouterLink],
  templateUrl: './form.html',
  styleUrl: './form.css',
})
export class Form {
  private fb = inject(FormBuilder);
  private router = inject(Router);
  private service = inject(TransferService);

  currencies = Object.values(Currency);

  form = this.fb.nonNullable.group({
    sourceAccountNumber: ['', [Validators.required]],
    targetAccountNumber: ['', [Validators.required]],
    amount:              [0,  [Validators.required, Validators.min(1)]],
    currency:            [Currency.PEN, [Validators.required]],
  });

  submit() {
    if (this.form.invalid) return;
    const request: TransferRequest = this.form.getRawValue();
    this.service.create(request).subscribe(() => this.router.navigate(['/transfers']));
  }
}
```

### 10.5 Llenar `form.html`

**`src/app/pages/transfers/form/form.html`**:

```html
<h1>Nueva transferencia</h1>

<form [formGroup]="form" (ngSubmit)="submit()">
  <label>Cuenta origen
    <input type="text" formControlName="sourceAccountNumber" />
  </label>

  <label>Cuenta destino
    <input type="text" formControlName="targetAccountNumber" />
  </label>

  <label>Monto
    <input type="number" formControlName="amount" />
  </label>

  <label>Moneda
    <select formControlName="currency">
      @for (c of currencies; track c) {
        <option [value]="c">{{ c }}</option>
      }
    </select>
  </label>

  <button type="submit" [disabled]="form.invalid">Transferir</button>
  <a routerLink="/transfers">Cancelar</a>
</form>
```

### 10.6 Llenar `form.css`

**`src/app/pages/transfers/form/form.css`**:

```css
form { display: flex; flex-direction: column; gap: 12px; max-width: 400px; }
label { display: flex; flex-direction: column; }
input, select { padding: 6px; }
button { padding: 8px; background: #d32f2f; color: #fff; border: none; border-radius: 4px; }
button:disabled { background: #ccc; }
```

**Probar en el navegador:**

1. Desde `/transfers`, clic en **Nueva transferencia**.
2. Completar (origen `191-001`, destino `191-002`, monto, moneda) y enviar.
3. `POST /api/transfers` → `201`. Vuelve a `/transfers` y la nueva transferencia aparece en la lista.

> Si hay error CORS: revisar `SecurityConfig` y `CorsConfig`. Reiniciar el backend.

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 11: Cuentas

Segunda interfaz de usuario. Mismo patrón: generamos el service y completamos el componente que ya existe (paso 4). Sin formulario.

### 11.1 Generar el service

```bash
ng g s services/account --skip-tests
```

### 11.2 Llenar `account.service.ts`

**`src/app/services/account.service.ts`**:

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, map } from 'rxjs';
import { environment } from '../../environments/environment';
import { AccountResponse } from '../models/account-response';
import { PageResponse } from '../models/page-response';

@Injectable({ providedIn: 'root' })
export class AccountService {
  private http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/accounts`;

  listByCustomer(customerId: number): Observable<AccountResponse[]> {
    return this.http
      .get<PageResponse<AccountResponse>>(`${this.baseUrl}/customer/${customerId}`)
      .pipe(map(page => page.content));
  }
}
```

### 11.3 Llenar `list.ts`

**`src/app/pages/accounts/list/list.ts`**:

```ts
import { Component, OnInit, inject, signal } from '@angular/core';
import { environment } from '../../../../environments/environment';
import { AccountResponse } from '../../../models/account-response';
import { AccountService } from '../../../services/account.service';

@Component({
  selector: 'app-accounts-list',
  imports: [],
  templateUrl: './list.html',
  styleUrl: './list.css',
})
export class List implements OnInit {
  private service = inject(AccountService);

  title = 'Cuentas';
  private _accounts = signal<AccountResponse[]>([]);
  accounts = this._accounts.asReadonly();

  ngOnInit(): void {
    this.service
      .listByCustomer(environment.defaultCustomerId)
      .subscribe(accounts => this._accounts.set(accounts));
  }
}
```

### 11.4 Llenar `list.html`

**`src/app/pages/accounts/list/list.html`**:

```html
<h1>{{ title }}</h1>

@for (a of accounts(); track a.id) {
  <div class="item">
    <span>{{ a.accountNumber }}</span>
    <span>S/. {{ a.balance }}</span>
    <span>{{ a.type }}</span>
    <span [class.active]="a.status === 'ACTIVE'"
          [class.suspended]="a.status === 'SUSPENDED'"
          [class.closed]="a.status === 'CLOSED'">{{ a.status }}</span>
  </div>
} @empty {
  <p>El cliente no tiene cuentas.</p>
}
```

### 11.5 Llenar `list.css`

**`src/app/pages/accounts/list/list.css`**:

```css
.item { display: flex; justify-content: space-between; gap: 12px; padding: 12px; border: 1px solid #ccc; border-radius: 4px; margin-bottom: 8px; }
.active { color: #2e7d32; font-weight: bold; }
.suspended { color: #ef6c00; font-weight: bold; }
.closed { color: #c62828; font-weight: bold; }
```

**Probar en el navegador:** clic en **Cuentas** del menú → URL `/accounts`. Lista las cuentas del cliente `1`. En DevTools → Network ver `GET /api/accounts/customer/1` con `200`.

[↑ Volver al índice](#tabla-de-contenidos)

---

## Sección 12: Resumen

| Sección | Lo que aprendiste                                                          |
|---------|----------------------------------------------------------------------------|
| 02      | Backend: `CorsConfig` y abrir transferencias y cuentas sin token.          |
| 03      | Estructura del proyecto, `main-layout` con menú, `auth-layout` listo.      |
| 04      | Generar los componentes vacíos con la CLI.                                 |
| 05      | Rutas hijas con `loadChildren` y `MainLayout` envolviendo todo.            |
| 06      | Modelos por CLI: `enum`, `interface`, `PageResponse` compartido.           |
| 07      | Variables por entorno (`environment.development.ts` / `environment.ts`).   |
| 08      | Service que devuelve `Observable<T>` + `HttpClient` activo.                |
| 09      | Componente con `inject`, `ngOnInit`, `subscribe`, Signal y control de flujo. |
| 10      | Reactive Forms tipados + `Validators` + botón controlado por `form.invalid`. |
| 11      | Cuentas: mismo patrón sin tocar el router padre.                           |

### Regla clave

> **Observable (HttpClient)** → conectividad con el API, en el service.
> **Signal** → estado local del cliente, en el componente.

### Para agregar una interfaz de usuario nueva

1. Generar componentes: `ng g c pages/<area>/<componente> --skip-tests`.
2. Crear `pages/<area>/<area>.routes.ts` con los `path` del área.
3. Agregar un `loadChildren` en `app.routes.ts` apuntando a ese archivo.
4. Generar el service: `ng g s services/<area> --skip-tests`. Devolver `Observable<T>` usando `environment.apiUrl`.
5. Llenar componentes con `inject`, `ngOnInit`, `subscribe` y signals.
6. Agregar el enlace en el menú de `main-layout`.

[↑ Volver al índice](#tabla-de-contenidos)
