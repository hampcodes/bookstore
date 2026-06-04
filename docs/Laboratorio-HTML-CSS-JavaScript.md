# Guía de Laboratorio: HTML + CSS + JavaScript

**Objetivo:** construir un **CRUD de empleados** (registrar, listar, filtrar, editar,
eliminar) para ver cómo se integran las 3 tecnologías:

- **HTML** → estructura
- **CSS** → apariencia + responsive
- **JavaScript** → comportamiento (con `localStorage`)

---

## 1. Responsive design

Es lograr que la página **se vea bien en cualquier pantalla** (celular, tablet,
computadora) sin tener que hacer una versión aparte para cada una. Se consigue con:

- **Viewport:** una etiqueta en el `<head>`. Sin ella, el celular "encoge" la página
  y todo se ve diminuto. Es obligatoria.
  ```html
  <meta name="viewport" content="width=device-width, initial-scale=1">
  ```
- **Anchos flexibles:** usar `%` y `max-width` en vez de medidas fijas en píxeles. Así
  el contenido se estira o encoge según el espacio disponible.
- **Media queries:** reglas de CSS que se aplican **solo** cuando se cumple una
  condición (por ejemplo, pantalla angosta). Sirven para los ajustes en móvil.
  ```css
  @media (max-width: 700px) { .contenedor { flex-direction: column; } }
  ```
- **Flexbox:** con `display: flex` acomodamos elementos en fila o columna. Aquí lo
  usamos para poner el formulario y la tabla **uno al lado del otro**, y apilarlos en
  móvil.

## 2. JavaScript

JavaScript le da **comportamiento** a la página: reacciona a lo que hace el usuario y
cambia el contenido sin recargar. Conceptos que usaremos:

- **Variables (`const` / `let`):** guardan datos. `const` no cambia; `let` sí puede.
  ```js
  const nombre = "Ana";
  let contador = 0;
  ```
- **Funciones flecha (arrow functions):** la forma corta y moderna de escribir funciones.
  ```js
  const sumar = (a, b) => a + b;
  ```
- **Seleccionar elementos del HTML:** `getElementById` busca un elemento por su `id`
  para leerlo o cambiarlo desde JS.
  ```js
  const caja = document.getElementById("nombre");
  ```
- **Eventos:** ejecutar código cuando pasa algo (clic, enviar un formulario, escribir).
  ```js
  boton.addEventListener("click", () => { /* ... */ });
  ```
- **Arreglos (listas) y sus métodos:** la base para manejar muchos datos.
  `.push()` agrega, `.map()` transforma, `.filter()` filtra, `.find()` busca uno.
- **localStorage:** la "memoria" del navegador. Solo guarda **texto**, por eso usamos
  `JSON.stringify` para guardar y `JSON.parse` para leer.
- **Template literals:** texto con variables usando comillas invertidas y `${ }`. Ideal
  para armar HTML: `` `<td>${emp.nombre}</td>` ``.
**Validaciones:** antes de guardar, revisamos que los datos estén bien (campos no
vacíos, correo con formato, salario mayor a 0). Si algo falla, mostramos un aviso y
**no** guardamos.

### Módulos: dividir el código (`export` / `import`)

Un **módulo** es simplemente un archivo `.js`. En vez de escribir todo en uno solo,
separamos el código por responsabilidad (las validaciones en un archivo, el
localStorage en otro, la lógica de la página en otro) y los conectamos entre sí.

- **`export`** marca lo que un archivo deja disponible para los demás:
  ```js
  // archivo: shared/validation.js
  export const noVacio = (texto) => texto.trim() !== "";
  ```
- **`import`** trae eso en otro archivo, indicando la ruta del archivo:
  ```js
  // archivo: empleado/empleado.js
  import { noVacio } from "../shared/validation.js";
  ```
- En el HTML, el archivo principal se carga con **`type="module"`** (eso habilita los
  `import`):
  ```html
  <script type="module" src="js/empleado/empleado.js"></script>
  ```

**¿Para qué sirve?** Cada archivo queda corto y con una sola tarea, y reutilizas
funciones sin copiarlas y pegarlas.
**Ojo:** al usar módulos, la página debe abrirse con un **servidor** (`http://`), no
con doble clic (el navegador bloquea los `import` por seguridad si es `file://`).

---

## 3. Ejercicio

Crea una página que administre empleados (**nombre, cargo, correo, salario**) y permita:
**registrar, listar, filtrar (por nombre o cargo), editar y eliminar**. Los datos se
guardan en `localStorage` y la página es responsive.

**Estructura:**
```
ejemplo-empleados/
├── index.html
├── css/style.css
└── js/
    ├── shared/
    │   ├── validation.js   (validaciones, con export)
    │   └── storage.js      (localStorage, con export)
    └── empleado/
        └── empleado.js     (import + logica de la pagina)
```

---

## 4. Paso a paso

### Paso 1 — HTML (`index.html`)
Formulario a la izquierda, lista a la derecha. Cada input con su `id`; el `<tbody>`
lo llena JavaScript. El `<script type="module">` permite usar `import`.
```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <!-- responsive design -->
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Gestion de empleados</title>
  <link rel="stylesheet" href="css/style.css">
</head>
<body>

  <h1>Gestion de empleados</h1>

  <div class="contenedor">

    <!-- Formulario (izquierda) -->
    <form id="form-empleado" class="tarjeta">
      <h2>Registrar</h2>
      <input type="text" id="nombre" placeholder="Nombre" required>
      <input type="text" id="cargo" placeholder="Cargo" required>
      <input type="email" id="correo" placeholder="Correo" required>
      <input type="number" id="salario" placeholder="Salario" required>
      <p id="mensaje" class="mensaje"></p>
      <button type="submit">Guardar</button>
    </form>

    <!-- Lista (derecha) -->
    <div class="lista tarjeta">
      <input type="search" id="filtro" placeholder="Buscar por nombre o cargo...">
      <table>
        <thead>
          <tr>
            <th>Nombre</th><th>Cargo</th><th>Correo</th><th>Salario</th><th>Acciones</th>
          </tr>
        </thead>
        <tbody id="tabla-empleados"></tbody>
      </table>
    </div>

  </div>

  <script type="module" src="js/empleado/empleado.js"></script>
</body>
</html>
```

### Paso 2 — CSS (`css/style.css`)
Definimos los colores una sola vez en `:root` y los reusamos con `var(...)`. El
formulario y la tabla van uno al costado del otro con `flex`, y se apilan en móvil.
```css
/* Variables de color reutilizables */
:root {
  --rojo: #c0392b;
  --rojo-oscuro: #96281b;
  --borde: #ddd;
  --fondo: #fff;
  --fondo-app: #f4f4f4;
  --texto: #222;
}

* {
  box-sizing: border-box;
}

body {
  font-family: Arial, sans-serif;
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
  background: var(--fondo-app);
  color: var(--texto);
}

h1 {
  text-align: center;
}

/* Formulario y lista, uno al costado del otro */
.contenedor {
  display: flex;
  gap: 16px;
  align-items: flex-start;
}

.contenedor form {
  flex: 1;
}

.contenedor .lista {
  flex: 2;
}

/* Caja blanca */
.tarjeta {
  background: var(--fondo);
  border: 1px solid var(--borde);
  border-radius: 8px;
  padding: 16px;
}

input,
button {
  width: 100%;
  padding: 10px;
  margin: 6px 0;
  border: 1px solid var(--borde);
  border-radius: 6px;
  font-size: 1rem;
}

button {
  background: var(--rojo);
  color: var(--fondo);
  border: none;
  cursor: pointer;
}

button:hover {
  background: var(--rojo-oscuro);
}

/* Mensaje de error de las validaciones */
.mensaje {
  color: var(--rojo);
  font-size: 0.9rem;
  margin: 4px 0;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th,
td {
  border-bottom: 1px solid var(--borde);
  padding: 8px;
  text-align: left;
}

/* Botones de la tabla mas chicos */
td button {
  width: auto;
  margin: 0 4px 0 0;
  padding: 5px 8px;
  font-size: 0.8rem;
}

/* responsive design: en pantallas chicas se apilan (form arriba, lista abajo) */
@media (max-width: 700px) {
  .contenedor {
    flex-direction: column;
  }
}
```

### Paso 3 — JavaScript compartido (`js/shared/`)
Archivos pequeños y reutilizables. Todo lo que va a usar otro archivo lleva `export`.

**`shared/validation.js`** — validaciones (devuelven `true`/`false`):
```js
export const esCorreoValido = (correo) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(correo);
export const noVacio = (texto) => texto.trim() !== "";
export const esSalarioValido = (salario) => Number(salario) > 0;
```

**`shared/storage.js`** — guardar/leer en localStorage:
```js
export const obtener = (clave) => JSON.parse(localStorage.getItem(clave)) || [];
export const guardar = (clave, lista) => localStorage.setItem(clave, JSON.stringify(lista));
```

### Paso 4 — JavaScript del empleado (`js/empleado/empleado.js`)

**a) Importar lo compartido y tomar referencias:**
```js
import { esCorreoValido, noVacio, esSalarioValido } from "../shared/validation.js";
import { obtener, guardar } from "../shared/storage.js";

const form = document.getElementById("form-empleado");
const tabla = document.getElementById("tabla-empleados");
const filtro = document.getElementById("filtro");
const mensaje = document.getElementById("mensaje");

let editandoId = null;

const mostrarMensaje = (texto) => { mensaje.textContent = texto; };
```

**b) Mostrar + filtrar:**
```js
const mostrar = () => {
  const texto = filtro.value.toLowerCase();

  const empleados = obtener("empleados").filter(
    (emp) =>
      emp.nombre.toLowerCase().includes(texto) ||
      emp.cargo.toLowerCase().includes(texto)
  );

  tabla.innerHTML = empleados
    .map(
      (emp) => `
      <tr>
        <td>${emp.nombre}</td>
        <td>${emp.cargo}</td>
        <td>${emp.correo}</td>
        <td>S/ ${emp.salario}</td>
        <td>
          <button data-editar="${emp.id}">Editar</button>
          <button data-eliminar="${emp.id}">Eliminar</button>
        </td>
      </tr>`
    )
    .join("");
};
```

**c) Registrar o editar (con validaciones):**
```js
form.addEventListener("submit", (evento) => {
  evento.preventDefault();

  const nombre = document.getElementById("nombre").value;
  const cargo = document.getElementById("cargo").value;
  const correo = document.getElementById("correo").value;
  const salario = document.getElementById("salario").value;

  // Validaciones: si algo falla, avisamos y NO guardamos
  if (!noVacio(nombre) || !noVacio(cargo)) {
    mostrarMensaje("Nombre y cargo son obligatorios");
    return;
  }
  if (!esCorreoValido(correo)) {
    mostrarMensaje("El correo no es valido");
    return;
  }
  if (!esSalarioValido(salario)) {
    mostrarMensaje("El salario debe ser mayor a 0");
    return;
  }

  const empleado = { id: editandoId || Date.now(), nombre, cargo, correo, salario };
  let empleados = obtener("empleados");

  if (editandoId) {
    empleados = empleados.map((emp) => (emp.id === editandoId ? empleado : emp));
    editandoId = null;
  } else {
    empleados.push(empleado);
  }

  guardar("empleados", empleados);
  form.reset();
  mostrarMensaje("");
  mostrar();
});
```

**d) Editar y eliminar (un solo listener en la tabla):**
```js
tabla.addEventListener("click", (evento) => {
  const idEditar = evento.target.dataset.editar;
  const idEliminar = evento.target.dataset.eliminar;

  if (idEditar) {
    const emp = obtener("empleados").find((e) => e.id === Number(idEditar));
    document.getElementById("nombre").value = emp.nombre;
    document.getElementById("cargo").value = emp.cargo;
    document.getElementById("correo").value = emp.correo;
    document.getElementById("salario").value = emp.salario;
    editandoId = emp.id;
  }
  if (idEliminar && confirm("¿Eliminar este empleado?")) {
    guardar("empleados", obtener("empleados").filter((e) => e.id !== Number(idEliminar)));
    mostrar();
  }
});
```

**e) Buscar y arrancar:**
```js
filtro.addEventListener("input", mostrar);
mostrar();
```

---

## 5. Probar
Como usamos **módulos** (`import`/`export`), no funciona con doble clic: ábrelo con un
servidor. En VS Code usa **Live Server** (clic derecho en `index.html` → *Open with
Live Server*). Luego registra empleados, prueba las validaciones (correo inválido,
salario 0), busca, edita y elimina.

## 6. Actividad
Sobre el CRUD que construiste, agrega estas funcionalidades para practicar:

1. **Nuevo campo:** agrega "área" o "fecha de ingreso" al empleado (en el HTML, el
   objeto y la tabla).
2. **Total de empleados:** muestra debajo de la tabla cuántos empleados hay.
3. **Promedio de salarios:** calcula y muestra el promedio (pista: `reduce` o un bucle).
4. **Correo único:** no dejar registrar dos empleados con el mismo correo (avisa al usuario).
5. **Ordenar:** un botón que ordene la lista por nombre o por salario.
