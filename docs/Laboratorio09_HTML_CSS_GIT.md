# Guia PagoYa — HTML, CSS y Git

---

## Estructura del proyecto

Organiza tu proyecto asi antes de escribir codigo:

```
pagoya/
├── index.html
├── css/
│   └── style.css
├── js/
│   ├── account/
│   │   └── account.js
│   └── transfer/
│       └── transfer.js
├── pages/
│   ├── account/
│   │   ├── list.html
│   │   └── form.html
│   └── transfer/
│       ├── list.html
│       └── form.html
└── img/
```

Cada carpeta tiene un proposito claro:

- `css/` — todos los estilos del proyecto en un solo archivo
- `js/` — logica separada por modulo, cada uno en su carpeta
- `pages/` — paginas internas organizadas por modulo
- `img/` — imagenes del proyecto

---

## Seccion 1 — HTML

### Paso 1 — Esqueleto

Crea `index.html` y abrelo en el navegador:

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>PagoYa</title>
</head>
<body>

  <header>
    <span>PagoYa</span>
  </header>

  <main>

    <div class="cards">
      <div class="card">
        <h3>Cuenta</h3>
        <p>Gestiona tus cuentas.</p>
        <a href="pages/account/list.html">Ver cuentas</a>
      </div>
      <div class="card">
        <h3>Transferencia</h3>
        <p>Realiza transferencias.</p>
        <a href="pages/transfer/list.html">Ver transferencias</a>
      </div>
    </div>

    <form>
      <h2>Iniciar sesion</h2>

      <label for="email">Correo:</label>
      <input type="email" id="email" placeholder="tu@correo.com" required>

      <label for="password">Contrasena:</label>
      <input type="password" id="password" placeholder="••••••••" required>

      <button type="submit">Entrar</button>
    </form>

  </main>

  <footer>
    <p>© 2026 PagoYa</p>
  </footer>

</body>
</html>
```

**Lo que ves:** todo apilado verticalmente, sin colores ni estilo. Asi es HTML puro.

---

### Paso 2 — Vincular el CSS

Agrega esta linea dentro del `<head>`:

```html
<link rel="stylesheet" href="css/style.css">
```

Crea `css/style.css` y agrega:

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: Arial, sans-serif;
  background-color: #ffffff;
  color: #222222;
}
```

Recarga el navegador.
**Lo que ves:** el espacio blanco alrededor desaparece. El CSS ya esta conectado.

---

## Seccion 2 — CSS

### Paso 3 — Variables CSS

Antes de escribir cualquier estilo, define los colores y valores que vas a usar en todo el proyecto. Agrega esto al inicio de `style.css`, encima del reset:

```css
:root {
  --rojo:        #c0392b;   /* color principal */
  --rojo-oscuro: #96281b;   /* hover de botones */
  --texto:       #222222;   /* texto principal */
  --gris:        #666666;   /* texto secundario */
  --fondo:       #ffffff;   /* fondo de la pagina */
  --sombra:      0 2px 6px rgba(0, 0, 0, 0.08);
  --radio:       8px;       /* redondeo de esquinas */
}
```

Ahora en lugar de escribir `#c0392b` cada vez, usas `var(--rojo)`.
Si decides cambiar el color principal, solo lo cambias en un lugar.

**Que significa cada parte de --sombra:**

```
0 2px 6px rgba(0, 0, 0, 0.08)
|  |   |   |
|  |   |   color negro con 8% de opacidad (muy suave)
|  |   difuminado: cuanto se expande la sombra
|  cae 2px hacia abajo
sin desplazamiento horizontal
```

---

### Paso 4 — Header

Agrega al final de `style.css`:

```css
header {
  background-color: var(--fondo);
  border-bottom: 3px solid var(--rojo);
  padding: 16px 24px;
}

header span {
  font-size: 1.3rem;
  font-weight: bold;
  color: var(--rojo);
}
```

**Lo que ves:** barra blanca con el nombre PagoYa en rojo y una linea roja abajo.

---

### Paso 5 — Cards y formulario en fila

Agrega al final de `style.css`:

```css
main {
  max-width: 900px;
  margin: 40px auto;   /* 40px arriba/abajo, auto izq/der = centrado */
  padding: 0 20px;
  display: flex;       /* pone los hijos en fila */
  gap: 24px;           /* espacio entre cards y formulario */
  align-items: flex-start;
}
```

**Lo que ves:** las cards y el formulario ahora estan lado a lado.

**Como funciona el flex aqui:**

Los hijos directos de `main` son `.cards` y `form`.
Ambos tienen `flex: 1` lo que significa que cada uno ocupa la mitad del espacio.
`align-items: flex-start` hace que cada uno mantenga su propia altura sin estirarse.

---

### Paso 6 — Cards

Agrega al final de `style.css`:

```css
/* PADRE — organiza las cards en columna */
.cards {
  display: flex;
  flex-direction: column;
  gap: 20px;
  flex: 1;             /* ocupa la mitad del main */
}

/* HIJO — cada card individual */
.card {
  border-left: 4px solid var(--rojo);
  border-radius: var(--radio);
  padding: 20px;
  box-shadow: var(--sombra);
}

.card h3 {
  color: var(--rojo);
  margin-bottom: 8px;
}

.card p {
  color: var(--gris);
  font-size: 0.9rem;
  margin-bottom: 12px;
}

.card a {
  color: var(--rojo);
  font-size: 0.9rem;
  font-weight: bold;
  text-decoration: none;
}
```

**Lo que ves:** dos cards apiladas a la izquierda, cada una con borde rojo lateral y sombra suave.

---

### Paso 7 — Formulario

Agrega al final de `style.css`:

```css
form {
  border-radius: var(--radio);
  padding: 28px;
  flex: 1;             /* ocupa la mitad del main */
  box-shadow: var(--sombra);
}

form h2 {
  font-size: 1.1rem;
  margin-bottom: 20px;
}

label {
  display: block;
  margin-top: 14px;
  margin-bottom: 5px;
  font-weight: bold;
  font-size: 0.9rem;
}

input {
  width: 100%;
  padding: 10px;
  border: 1px solid #dddddd;
  border-radius: var(--radio);
  font-size: 14px;
}

input:focus {
  outline: none;
  border-color: var(--rojo);
}

button {
  margin-top: 20px;
  width: 100%;
  padding: 12px;
  background-color: var(--rojo);
  color: var(--fondo);
  border: none;
  border-radius: var(--radio);
  cursor: pointer;
  font-size: 15px;
  font-weight: bold;
}

button:hover {
  background-color: var(--rojo-oscuro);
}
```

**Lo que ves:** formulario a la derecha con campos limpios, boton rojo y borde rojo al hacer clic en un campo.

---

### Paso 8 — Footer

Agrega al final de `style.css`:

```css
footer {
  text-align: center;
  padding: 16px;
  margin-top: 40px;
  background-color: var(--texto);
  color: var(--fondo);
  font-size: 0.9rem;
}
```

**Lo que ves:** barra oscura al fondo con texto blanco centrado. Pagina completa.

---

## Seccion 3 — Git y GitHub

### Configuracion inicial

```bash
git config --global user.name  "Tu Nombre"
git config --global user.email "tu@correo.com"
```

### Iniciar proyecto

```bash
git init
git add .
git commit -m "chore: estructura inicial PagoYa"

git remote add origin https://github.com/tu-usuario/pagoya.git
git branch -M main
git push -u origin main

git checkout -b develop
git push -u origin develop
```

### Flujo por modulo

```bash
git checkout develop
git checkout -b feature/account

git add .
git commit -m "feat: list y form de account"
git push -u origin feature/account
```

### Pull Request

```
Base:    develop
Compare: feature/account
Titulo:  feat: modulo account — lista y formulario

## Cambios realizados
- Se agrego `pages/account/list.html` con tabla de cuentas.
- Se agrego `pages/account/form.html` con formulario de nueva cuenta.
- Se agrego `js/account/account.js` para la logica del modulo.
- Se configuraron validaciones basicas con el atributo `required`.

## Como probar
1. Abrir `pages/account/list.html` en el navegador.
2. Hacer clic en el boton de nueva cuenta.
3. Intentar enviar el formulario sin completar los campos.
4. Verificar que el navegador muestre las validaciones obligatorias.

Revisado por: @companero
```

### Flujo diario

```bash
git status
git add .
git commit -m "descripcion del cambio"
git push
```

### Tag de version

```bash
git checkout main
git merge develop
git push
git tag -a v1.0 -m "Release v1.0 PagoYa"
git push origin v1.0
```

---

## Actividades

Crea una rama para todas las actividades:

```bash
git checkout develop
git checkout -b feature/mejoras-index
```

Realiza cada actividad, recarga el navegador y verifica el resultado antes de pasar a la siguiente.

---

### Actividad 1 — Cambia el tema visual

Abre `css/style.css` y cambia las variables en `:root`:

```css
:root {
  --rojo:        #2980b9;   /* cambia a azul */
  --rojo-oscuro: #1a6a9a;
}
```

Recarga el navegador. El header, las cards, el boton y el focus del input deben cambiar de color.

---

### Actividad 2 — Agrega una tercera card

En `index.html`, dentro de `<div class="cards">`, agrega una card debajo de las dos existentes:

```html
<div class="card">
  <h3>Pagos</h3>
  <p>Consulta y realiza tus pagos.</p>
  <a href="#">Ver pagos</a>
</div>
```

Recarga el navegador. Deben aparecer tres cards con el mismo estilo.

---

### Actividad 3 — Agrega un campo al formulario

En `index.html`, dentro del `<form>`, agrega el campo nombre antes del correo:

```html
<label for="nombre">Nombre:</label>
<input type="text" id="nombre" placeholder="Tu nombre completo" required>
```

Recarga el navegador. El campo debe aparecer arriba del correo con el mismo estilo.
Intenta enviar el formulario sin llenarlo y verifica que muestra el mensaje de campo obligatorio.

---

### Actividad 4 — Construye list.html

Primero agrega estos estilos al final de `css/style.css`:

```css
/* Paginas internas */
.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.page-header h2 {
  font-size: 1.2rem;
}

.page-header a {
  color: var(--rojo);
  font-weight: bold;
  text-decoration: none;
}

.main-block {
  max-width: 900px;
  margin: 40px auto;
  padding: 0 20px;
}
```

Luego abre `pages/account/list.html` y construye la pagina:

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>PagoYa — Cuentas</title>
  <link rel="stylesheet" href="../../css/style.css">
</head>
<body>

  <header>
    <span>PagoYa</span>
    <a href="../../index.html">← Inicio</a>
  </header>

  <main class="main-block">
    <div class="page-header">
      <h2>Cuentas</h2>
      <a href="form.html">+ Nueva cuenta</a>
    </div>

    <div class="card">
      <h3>Cuenta Principal</h3>
      <p>Tipo: Ahorro</p>
      <p>Saldo: S/ 1,000</p>
      <a href="form.html">Ver detalle</a>
    </div>
  </main>

  <footer>
    <p>© 2026 PagoYa</p>
  </footer>

</body>
</html>
```

Desde `index.html` haz clic en Ver cuentas. La pagina debe cargar con el mismo estilo porque comparten el mismo `style.css`.

---

### Git — al terminar todas las actividades

```bash
git add .
git commit -m "feat: actividades completadas"
git push -u origin feature/mejoras-index
```

Abre un PR hacia `develop`:

```
Base:    develop
Compare: feature/mejoras-index
Titulo:  feat: actividades de practica

## Cambios realizados
- Se cambio el color principal a azul via variables CSS.
- Se agrego una tercera card de pagos en index.html.
- Se agrego el campo nombre al formulario con validacion required.
- Se construyo pages/account/list.html con header, card y footer.

## Como probar
1. Abrir index.html y verificar el nuevo color.
2. Verificar que aparecen tres cards.
3. Intentar enviar el formulario sin completar los campos.
4. Hacer clic en Ver cuentas y verificar que carga con el mismo estilo.

Revisado por: @companero
```

Cuando el PR este aprobado y mergeado a `develop`:

```bash
git checkout main
git merge develop
git push
git tag -a v1.1 -m "Release v1.1 — actividades completadas"
git push origin v1.1
```

