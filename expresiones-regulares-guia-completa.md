# Guía Definitiva de Expresiones Regulares (RegEx): De Cero a Experto

Una **Expresión Regular** (conocida comúnmente como **RegEx** o **RegExp**) es una secuencia de caracteres que conforma un patrón de búsqueda. Es una de las herramientas más poderosas en programación para **validar datos**, **buscar texto**, **extraer información** y **reemplazar cadenas**.

---

## 📋 Tabla de Contenidos
1. [¿Qué es y cómo se lee una RegEx?](#1-qué-es-y-cómo-se-lee-una-regex)
2. [Banderas / Modificadores (Flags)](#2-banderas--modificadores-flags)
3. [Metacaracteres y Clases Predefinidas](#3-metacaracteres-y-clases-predefinidas)
4. [Conjuntos y Rangos `[...]`](#4-conjuntos-y-rangos-)
5. [Cuantificadores: ¿Cuántas veces se repite?](#5-cuantificadores-cuántas-veces-se-repite)
6. [Anclas de Posición (`^`, `$`, `\b`)](#6-anclas-de-posición---b)
7. [Grupos de Captura y Alternancia `(...)`, `|`](#7-grupos-de-captura-y-alternancia---)
8. [Aserciones Avanzadas: Lookaheads y Lookbehinds](#8-aserciones-avanzadas-lookaheads-y-lookbehinds)
9. [Recetas Listas para Usar en Proyectos Reales](#9-recetas-listas-para-usar-en-proyectos-reales)
10. [Cómo usar RegEx en JavaScript, PHP y Python](#10-cómo-usar-regex-en-javascript-php-y-python)
11. [Tabla Resumen (Cheat Sheet)](#11-tabla-resumen-cheat-sheet)

---

## 1. ¿Qué es y cómo se lee una RegEx?

Una expresión regular suele escribirse entre barras diagonales `/.../`:

```text
/hola/i
```
- `/`: Delimitador de inicio.
- `hola`: El **patrón** exacto que buscamos.
- `/`: Delimitador de fin.
- `i`: La **bandera** (modificador). En este caso, ignora mayúsculas/minúsculas.

### Caracteres especiales que deben escaparse con `\`
Si quieres buscar literalmente uno de estos caracteres especiales, debes anteponerle una barra invertida `\`:
```text
\  .  ^  $  *  +  ?  (  )  [  ]  {  }  |
```
*Ejemplo:* Para buscar un punto literal `.`, escribe `\.`. Para buscar un signo de pesos `$`, escribe `\$`.

---

## 2. Banderas / Modificadores (Flags)

Se colocan al final de la expresión y alteran el comportamiento global de la búsqueda:

| Bandera | Nombre | ¿Qué hace? |
| :---: | :--- | :--- |
| **`g`** | *Global* | No se detiene en la primera coincidencia; busca **todas** las ocurrencias en el texto. |
| **`i`** | *Case Insensitive* | Ignora diferencias entre MAYÚSCULAS y minúsculas (`/a/i` coincide con `a` y `A`). |
| **`m`** | *Multiline* | Hace que `^` y `$` coincidan con el inicio y fin de **cada línea** individual, no solo del texto completo. |
| **`s`** | *DotAll* | Permite que el punto `.` coincida también con saltos de línea (`\n`). |
| **`u`** | *Unicode* | Habilita soporte completo para caracteres Unicode y emojis. |

*Ejemplo combinado:* `/error/gim` busca la palabra "error" en mayúsculas o minúsculas en todas las líneas y coincidencias del documento.

---

## 3. Metacaracteres y Clases Predefinidas

Son atajos rápidos para tipos de caracteres comunes:

| Metacarácter | Coincide con... | Ejemplo de coincidencia |
| :---: | :--- | :--- |
| **`.`** | Cualquier carácter excepto salto de línea | En `c.sa` coincide con `casa`, `cosa`, `c1sa`. |
| **`\d`** | Cualquier **dígito** (`0-9`) *(d = digit)* | En `año 2026` coincide con `2`, `0`, `2`, `6`. |
| **`\D`** | Cualquier carácter que **NO sea un dígito** | En `123A` coincide con `A`. |
| **`\w`** | Cualquier carácter de **palabra** (`a-z`, `A-Z`, `0-9`, `_`) *(w = word)* | Letras, números y guion bajo. |
| **`\W`** | Cualquier carácter que **NO sea de palabra** | Puntos, espacios, símbolos (`@`, `!`, `#`, ` `). |
| **`\s`** | Cualquier **espacio en blanco** (espacio, tabulación `\t`, salto de línea `\n`) | Espacios y saltos. |
| **`\S`** | Cualquier carácter que **NO sea espacio en blanco** | Letras, números, signos. |

> 💡 **Regla mnemotécnica:** La letra minúscula (`\d`, `\w`, `\s`) selecciona el tipo; la letra MAYÚSCULA (`\D`, `\W`, `\S`) representa su **opuesto/negación**.

---

## 4. Conjuntos y Rangos `[...]`

Los corchetes definen un grupo donde **cualquiera de los caracteres dentro es válido**:

| Sintaxis | Significado | Explicación |
| :--- | :--- | :--- |
| `[aeiou]` | Coincide con **cualquier vocal**. | `p[aeiou]n` coincide con `pan`, `pen`, `pin`, `pon`, `pun`. |
| `[a-z]` | Cualquier letra minúscula de la `a` a la `z`. | Rangos alfabéticos. |
| `[A-Z]` | Cualquier letra mayúscula de la `A` a la `Z`. | Rangos mayúsculos. |
| `[0-9]` | Cualquier dígito del 0 al 9 (equivalente a `\d`). | Rangos numéricos. |
| `[a-zA-Z0-9]` | Cualquier letra o número alfanumérico. | Combinación de rangos. |
| **`[^abc]`** | **Negación:** Coincide con cualquier carácter **EXCEPTO** `a`, `b` o `c`. | El `^` dentro de corchetes niega la lista. |

---

## 5. Cuantificadores: ¿Cuántas veces se repite?

Indican cuántas veces debe repetirse el carácter o grupo anterior:

| Cuantificador | Cantidad | Explicación |
| :---: | :--- | :--- |
| **`*`** | **0 o más veces** | Puede no estar o repetirse infinitamente. `/ab*c/` coincide con `ac`, `abc`, `abbc`. |
| **`+`** | **1 o más veces** | Debe estar **al menos una vez**. `/ab+c/` coincide con `abc`, `abbc`, pero **no** con `ac`. |
| **`?`** | **0 o 1 vez (Opcional)** | El elemento puede estar o no. `/colou?r/` coincide con `color` y `colour`. |
| **`{n}`** | **Exactamente n veces** | `/\d{4}/` busca exactamente 4 dígitos (ej: `2026`). |
| **`{n,}`** | **Al menos n veces** | `/\d{2,}/` busca 2 o más dígitos. |
| **`{n,m}`** | **Entre n y m veces** | `/\d{2,4}/` busca entre 2 y 4 dígitos. |

### 🛑 Comportamiento "Codicioso" (*Greedy*) vs "Perezoso" (*Lazy*)
Por defecto, los cuantificadores son **codiciosos**: consumen todo el texto posible.
- Si en `<b>Hola</b> y <b>Mundo</b>` buscas `/<.*>/`, coincidirá con: `<b>Hola</b> y <b>Mundo</b>` (todo de golpe).
- Si agregas un `?` después del cuantificador (`/<.*?>/`), se vuelve **perezoso** y coincide solo con el primer cierre: `<b>` y luego `</b>`.

---

## 6. Anclas de Posición (`^`, `$`, `\b`)

Las anclas no coinciden con caracteres físicos, sino con **posiciones dentro del texto**:

| Ancla | Posición | Ejemplo |
| :---: | :--- | :--- |
| **`^`** | **Inicio** de la cadena o línea | `/^Hola/` solo coincide si el texto **empieza** por "Hola". |
| **`$`** | **Fin** de la cadena o línea | `/adiós$/` solo coincide si el texto **termina** en "adiós". |
| **`^patrón$`** | **Cadena exacta completa** | `/^\d{4}$/` valida que el texto contenga **únicamente 4 dígitos** de principio a fin. |
| **`\b`** | **Límite de palabra** (*word boundary*) | `/\bgato\b/` coincide con la palabra exacta "gato", pero **no** con "gatuno" ni con "lengato". |
| **`\B`** | **No límite de palabra** | `/\Bman/` coincide con "humano" pero no con "manzana". |

---

## 7. Grupos de Captura y Alternancia `(...)`, `|`

### 7.1 Grupos de Captura `(...)`
Permiten aislar partes de una coincidencia para extraerlas o reutilizarlas:
- En la fecha `(\d{4})-(\d{2})-(\d{2})`:
  - Coincidencia completa: `2026-09-05`
  - Grupo 1 (`$1`): `2026` (Año)
  - Grupo 2 (`$2`): `09` (Mes)
  - Grupo 3 (`$3`): `05` (Día)

### 7.2 Grupo de No Captura `(?:...)`
Si solo quieres agrupar para aplicar un cuantificador sin guardar el valor en memoria:
- `/(?:https?:\/\/)?(www\.[a-z]+\.com)/`

### 7.3 Alternancia `|` (Operador OR)
Funciona como un "O" lógico:
- `/(gato|perro|pájaro)/` coincide con cualquiera de los tres animales.
- `/(jpg|jpeg|png|webp)/i` valida extensiones de imagen.

---

## 8. Aserciones Avanzadas: Lookaheads y Lookbehinds

Permiten verificar si un texto está precedido o seguido por otro **sin incluirlo en la coincidencia final**.

| Tipo | Sintaxis | Significado | Ejemplo | Coincidencia en: |
| :--- | :---: | :--- | :--- | :--- |
| **Positive Lookahead** | `(?=...)` | Seguido por... | `/\d+(?=€)/` | En `100€`, coincide con `100` (el `€` no se incluye). |
| **Negative Lookahead** | `(?!...)` | **NO** seguido por... | `/\d+(?!€)/` | En `100$`, coincide con `100` porque no le sigue un euro. |
| **Positive Lookbehind** | `(?<=...)` | Precedido por... | `/(?<=\$)\d+/` | En `$50`, coincide con `50` (el `$` no se incluye). |
| **Negative Lookbehind** | `(?<!...)` | **NO** precedido por... | `/(?<!\$)\d+/` | Coincide con números que no tengan signo `$`. |

---

## 9. Recetas Listas para Usar en Proyectos Reales

Patrones probados y listos para copiar y pegar en tus validaciones:

### 1. Validar Correo Electrónico (Email)
```text
/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/
```
- Valida el usuario, arroba, dominio y extensión de al menos 2 letras.

### 2. Contraseña Fuerte (Seguridad Alta)
Mínimo 8 caracteres, al menos una mayúscula, una minúscula, un número y un carácter especial:
```text
/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/
```

### 3. Cédula Dominicana (000-0000000-0)
```text
/^\d{3}-?\d{7}-?\d{1}$/
```
- Valida 11 dígitos, con o sin guiones.

### 4. Número de Teléfono Internacional (Formato E.164)
```text
/^\+?[1-9]\d{1,14}$/
```
- Ejemplo: `+18091234567` o `+34600112233`.

### 5. Formato de Fecha (YYYY-MM-DD)
```text
/^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/
```
- Valida año de 4 cifras, mes del 01 al 12 y días del 01 al 31.

### 6. Extraer URLs (HTTP / HTTPS)
```text
/https?:\/\/[^\s/$.?#].[^\s]*/gi
```

---

## 10. Cómo usar RegEx en JavaScript, PHP y Python

### En JavaScript / TypeScript:

```javascript
const email = "usuario@empresa.com";
const regexEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// 1. Validar (devuelve true o false):
if (regexEmail.test(email)) {
    console.log("Email válido");
}

// 2. Extraer datos con grupos de captura:
const fecha = "2026-09-05";
const match = fecha.match(/^(\d{4})-(\d{2})-(\d{2})$/);
if (match) {
    console.log(`Año: ${match[1]}, Mes: ${match[2]}, Día: ${match[3]}`);
}

// 3. Reemplazar texto:
const texto = "El precio es 50 dolares y 100 dolares";
const nuevoTexto = texto.replace(/dolares/g, "USD");
console.log(nuevoTexto); // "El precio es 50 USD y 100 USD"
```

---

### En PHP:

```php
<?php
$telefono = "809-555-1234";

// 1. Validar con preg_match:
if (preg_match('/^\d{3}-\d{3}-\d{4}$/', $telefono)) {
    echo "Teléfono válido";
}

// 2. Reemplazar caracteres no numéricos:
$soloNumeros = preg_replace('/\D/', '', $telefono);
echo $soloNumeros; // "8095551234"
```

---

### En Python:

```python
import re

texto = "Mi correo es dev@test.com y mi año de nacimiento es 1995"

# 1. Buscar coincidencias:
coincidencia = re.search(r'\d{4}', texto)
if coincidencia:
    print("Año encontrado:", coincidencia.group()) # 1995

# 2. Extraer todos los correos:
emails = re.findall(r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}', texto)
print(emails) # ['dev@test.com']
```

---

## 11. Tabla Resumen (Cheat Sheet)

| Símbolo | Significado |
| :--- | :--- |
| `^` | Inicio del texto / línea. |
| `$` | Fin del texto / línea. |
| `.` | Cualquier carácter (excepto salto de línea). |
| `\d` / `\D` | Dígito / NO dígito. |
| `\w` / `\W` | Carácter de palabra (`[a-zA-Z0-9_]`) / NO palabra. |
| `\s` / `\S` | Espacio en blanco / NO espacio. |
| `\b` | Límite de palabra. |
| `[abc]` | Cualquier carácter entre los corchetes (`a`, `b` o `c`). |
| `[^abc]` | Cualquier carácter EXCEPTO `a`, `b` o `c`. |
| `*` | 0 o más repeticiones. |
| `+` | 1 o más repeticiones. |
| `?` | 0 o 1 repetición (opcional). |
| `{n,m}` | Entre `n` y `m` repeticiones. |
| `(a\|b)` | `a` O `b`. |
| `(?=abc)` | Seguido de `abc` (Positive Lookahead). |

---

## 🌐 Herramientas Recomendadas para Probar RegEx Visualmente

- **[Regex101.com](https://regex101.com/):** La mejor herramienta web del mundo para probar y depurar tus expresiones regulares con explicación paso a paso de cada carácter y resaltado de colores en tiempo real.
- **[RegExr.com](https://regexr.com/):** Excelente interfaz interactiva con hojas de trucos integradas.
