# Módulo 3: Funciones, Agregación, `GROUP BY` y `HAVING`

Hasta ahora has leído filas tal cual están guardadas. En este módulo aprendes a **transformarlas** (funciones de texto, número y fecha), a tomar **decisiones** dentro de la consulta (`CASE`) y a **resumir** miles de filas en informes (`GROUP BY`): ventas por mes, clientes por país, ticket medio...

---

## 1. Funciones de texto

| Función | Ejemplo | Resultado |
| :--- | :--- | :--- |
| `CONCAT(a, b, ...)` | `CONCAT(nombre, ' ', apellido)` | `Laura Gómez` (si algún valor es `NULL`, el resultado es `NULL`) |
| `CONCAT_WS(sep, a, b)` | `CONCAT_WS(', ', ciudad, pais)` | `Madrid, ES` (ignora los `NULL`) |
| `UPPER()` / `LOWER()` | `UPPER('sku-1')` | `SKU-1` |
| `LENGTH()` / `CHAR_LENGTH()` | `LENGTH('ñ')`, `CHAR_LENGTH('ñ')` | `2` bytes / `1` carácter |
| `TRIM()`, `LTRIM()`, `RTRIM()` | `TRIM('  hola ')` | `hola` |
| `SUBSTRING(t, inicio, largo)` | `SUBSTRING('LAP-001', 1, 3)` | `LAP` |
| `LEFT(t, n)` / `RIGHT(t, n)` | `RIGHT('LAP-001', 3)` | `001` |
| `SUBSTRING_INDEX(t, sep, n)` | `SUBSTRING_INDEX('ana@mail.com', '@', -1)` | `mail.com` |
| `REPLACE(t, buscar, poner)` | `REPLACE('Portátil', 'á', 'a')` | `Portatil` |
| `LOCATE(sub, t)` / `INSTR(t, sub)` | `LOCATE('@', 'ana@mail.com')` | `4` |
| `LPAD(t, n, relleno)` / `RPAD` | `LPAD(7, 5, '0')` | `00007` |
| `REVERSE()`, `REPEAT()` | `REPEAT('*', 3)` | `***` |
| `FORMAT(n, dec, 'es_ES')` | `FORMAT(1234567.891, 2, 'es_ES')` | `1.234.567,89` |

```sql
-- Nombre completo y dominio del email de cada empleado
SELECT CONCAT(nombre, ' ', apellido)          AS empleado,
       SUBSTRING_INDEX(email, '@', -1)        AS dominio,
       UPPER(LEFT(apellido, 3))               AS codigo
FROM empleados;

-- Número de pedido "bonito": PED-00001
SELECT CONCAT('PED-', LPAD(id, 5, '0')) AS numero, total FROM pedidos;
```

---

## 2. Funciones numéricas

| Función | Ejemplo | Resultado |
| :--- | :--- | :--- |
| `ROUND(n, d)` | `ROUND(59.987, 2)` | `59.99` |
| `TRUNCATE(n, d)` | `TRUNCATE(59.987, 2)` | `59.98` (corta, no redondea) |
| `CEIL()` / `FLOOR()` | `CEIL(4.1)`, `FLOOR(4.9)` | `5`, `4` |
| `ABS()` | `ABS(-7)` | `7` |
| `MOD(a, b)` o `a % b` | `MOD(10, 3)` | `1` |
| `POWER(a, b)`, `SQRT()` | `POWER(2, 10)` | `1024` |
| `RAND()` | `ORDER BY RAND() LIMIT 1` | Fila aleatoria (lento en tablas grandes) |
| `GREATEST()` / `LEAST()` | `GREATEST(stock - 5, 0)` | Nunca negativo |

```sql
-- Precio con IVA redondeado y número de cajas de 10 unidades necesarias para el stock
SELECT nombre,
       ROUND(precio * 1.21, 2)   AS con_iva,
       CEIL(stock / 10)          AS cajas
FROM productos;
```
> ⚠️ **División entre cero:** en MySQL `10 / 0` devuelve `NULL` (con un *warning*); en PostgreSQL y SQL Server **da error**. Protégete con `NULLIF`: `total / NULLIF(cantidad, 0)`.

---

## 3. Funciones de fecha y hora

```sql
SELECT NOW(),            -- 2025-02-14 13:00:00  (fecha y hora actual)
       CURDATE(),        -- 2025-02-14
       CURTIME(),        -- 13:00:00
       UTC_TIMESTAMP();  -- Fecha y hora en UTC
```
| Función | Ejemplo | Resultado |
| :--- | :--- | :--- |
| `YEAR()`, `MONTH()`, `DAY()`, `HOUR()` | `YEAR('2024-06-14')` | `2024` |
| `DAYOFWEEK()` / `WEEKDAY()` | `DAYOFWEEK('2024-06-14')` | `6` (1 = domingo) |
| `DAYNAME()` / `MONTHNAME()` | `MONTHNAME('2024-06-14')` | `June` (usa `SET lc_time_names = 'es_ES';` para español) |
| `DATE()` | `DATE('2024-06-14 20:10:00')` | `2024-06-14` |
| `DATE_ADD(f, INTERVAL n unidad)` | `DATE_ADD('2024-01-31', INTERVAL 1 MONTH)` | `2024-02-29` |
| `DATE_SUB()` o `f - INTERVAL n unidad` | `NOW() - INTERVAL 7 DAY` | Hace una semana |
| `DATEDIFF(f1, f2)` | `DATEDIFF('2025-01-01', '2024-12-25')` | `7` días |
| `TIMESTAMPDIFF(unidad, f1, f2)` | `TIMESTAMPDIFF(YEAR, '2018-03-01', CURDATE())` | Años completos (antigüedad, edad) |
| `DATE_FORMAT(f, formato)` | `DATE_FORMAT(fecha, '%d/%m/%Y %H:%i')` | `14/06/2024 20:10` |
| `STR_TO_DATE(texto, formato)` | `STR_TO_DATE('14/06/2024', '%d/%m/%Y')` | `2024-06-14` |
| `LAST_DAY(f)` | `LAST_DAY('2024-02-10')` | `2024-02-29` |
| `UNIX_TIMESTAMP()` / `FROM_UNIXTIME()` | | Conversión a/desde segundos desde 1970 |
| `CONVERT_TZ(f, 'UTC', 'Europe/Madrid')` | | Cambio de zona horaria (requiere tablas de zonas cargadas) |

```sql
-- Antigüedad de cada empleado en años
SELECT CONCAT(nombre, ' ', apellido) AS empleado,
       fecha_contratacion,
       TIMESTAMPDIFF(YEAR, fecha_contratacion, CURDATE()) AS anios
FROM empleados
ORDER BY anios DESC;

-- Pedidos de los últimos 90 días (relativo a hoy)
SELECT id, fecha FROM pedidos WHERE fecha >= NOW() - INTERVAL 90 DAY;
```
> 🐢 **Regla de rendimiento:** no apliques funciones a la columna en el `WHERE`. `WHERE YEAR(fecha) = 2024` **no puede usar el índice** de `fecha`. Escribe `WHERE fecha >= '2024-01-01' AND fecha < '2025-01-01'` (Módulo 8).

---

## 4. Conversión de tipos

```sql
SELECT CAST('2024-06-14' AS DATE),
       CAST(precio AS SIGNED)           AS precio_entero,   -- SIGNED / UNSIGNED / DECIMAL(10,2) / CHAR / DATE / DATETIME / JSON
       CONVERT('123', UNSIGNED)         AS numero,
       CAST(stock AS CHAR)              AS stock_texto
FROM productos LIMIT 1;
```
> ⚠️ MySQL hace **conversiones implícitas** sin avisar: `'5abc' + 1` da `6`, y `WHERE sku = 0` compara convirtiendo cada SKU a número (lento y con resultados sorprendentes). Compara siempre con el tipo correcto.

---

## 5. Lógica condicional: `CASE`, `IF`, `COALESCE`, `NULLIF`

```sql
-- CASE "buscado": la forma más flexible (estándar SQL, funciona en todos los motores)
SELECT nombre, stock,
       CASE
           WHEN stock = 0   THEN 'Agotado'
           WHEN stock < 10  THEN 'Pocas unidades'
           ELSE 'Disponible'
       END AS disponibilidad
FROM productos;

-- CASE "simple": compara una expresión con valores
SELECT id,
       CASE estado
           WHEN 'pendiente' THEN '⏳'
           WHEN 'pagado'    THEN '💳'
           WHEN 'enviado'   THEN '🚚'
           ELSE '❌'
       END AS icono
FROM pedidos;

-- IF(condición, si_verdadero, si_falso): atajo de MySQL
SELECT nombre, IF(activo, 'Sí', 'No') AS activo FROM clientes;

-- COALESCE: devuelve el primer valor que no sea NULL (estándar SQL)
SELECT nombre, COALESCE(ciudad, 'Sin ciudad') AS ciudad FROM clientes;
-- IFNULL(a, b) es la versión de 2 argumentos de MySQL

-- NULLIF(a, b): devuelve NULL si a = b (ideal para evitar divisiones por cero)
SELECT 100 / NULLIF(0, 0);   -- NULL en lugar de error
```

---

## 6. Funciones de agregación

Resumen **muchas filas en un solo valor**:

| Función | ¿Qué calcula? | ¿Cuenta los `NULL`? |
| :--- | :--- | :--- |
| `COUNT(*)` | Número de filas | Sí (cuenta filas) |
| `COUNT(columna)` | Filas donde la columna **no** es `NULL` | No |
| `COUNT(DISTINCT columna)` | Valores distintos | No |
| `SUM(col)` | Suma | Ignora los `NULL` |
| `AVG(col)` | Media | Ignora los `NULL` (¡no los cuenta como 0!) |
| `MIN(col)` / `MAX(col)` | Mínimo / máximo (también en textos y fechas) | Ignora los `NULL` |
| `GROUP_CONCAT(col)` | Une los valores en un texto separado por comas | Ignora los `NULL` |
| `STDDEV()`, `VARIANCE()` | Desviación típica y varianza | Ignora los `NULL` |

```sql
SELECT COUNT(*)               AS total_clientes,
       COUNT(ciudad)          AS con_ciudad,         -- 7 (Hugo no tiene)
       COUNT(DISTINCT pais)   AS paises
FROM clientes;

SELECT SUM(total)  AS facturado,
       AVG(total)  AS ticket_medio,
       MIN(fecha)  AS primer_pedido,
       MAX(fecha)  AS ultimo_pedido
FROM pedidos
WHERE estado <> 'cancelado';
```

---

## 7. `GROUP BY`: agrupar y resumir

`GROUP BY` forma **un grupo por cada valor distinto** y aplica las funciones de agregación a cada grupo.

```sql
-- Clientes por país
SELECT pais, COUNT(*) AS clientes
FROM clientes
GROUP BY pais
ORDER BY clientes DESC;

-- Ventas por estado de pedido
SELECT estado, COUNT(*) AS pedidos, SUM(total) AS importe
FROM pedidos
GROUP BY estado;

-- Ventas por año y mes (agrupar por expresiones)
SELECT DATE_FORMAT(fecha, '%Y-%m') AS mes,
       COUNT(*)                     AS pedidos,
       SUM(total)                   AS facturado
FROM pedidos
WHERE estado <> 'cancelado'
GROUP BY mes
ORDER BY mes;
```

### La regla de oro de `GROUP BY`
**Cada columna del `SELECT` debe estar en el `GROUP BY` o dentro de una función de agregación.**
```sql
-- ❌ ¿Qué "nombre" debería mostrar si hay 3 empleados en Ventas?
SELECT departamento_id, nombre, AVG(salario) FROM empleados GROUP BY departamento_id;
-- ERROR 1055: ... this is incompatible with sql_mode=only_full_group_by
```
MySQL 5.7+ activa `ONLY_FULL_GROUP_BY` por defecto para protegerte de resultados aleatorios.

> ⚠️ **¿No te da el error?** Algunos paquetes como **WAMP** traen el `sql_mode` **vacío** y la consulta "funciona" devolviendo un nombre cualquiera. Compruébalo con `SELECT @@sql_mode;` y, para practicar con el comportamiento estándar, ejecuta al principio de tu sesión `SET SESSION sql_mode = DEFAULT;` si tu servidor ya tiene el modo estándar, o explícitamente:
> `SET SESSION sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';` (ver [Módulo 1, apartado 9](./01_FUNDAMENTOS_BASES_DE_DATOS_TABLAS_Y_TIPOS.md#9-sql_mode-qué-tan-estricto-es-tu-servidor)). **No lo desactives**: corrige la consulta. *(En el tutorial [desactivar-only-full-group-by-mysql.md](../desactivar-only-full-group-by-mysql.md) verás cómo se desactiva para sistemas heredados, pero la solución correcta es esta.)*

### `HAVING`: filtrar **grupos** (no filas)
* `WHERE` filtra **filas antes** de agrupar.
* `HAVING` filtra **grupos después** de agrupar (y puede usar funciones de agregación).

```sql
-- Clientes que han gastado más de 1.000 € en pedidos no cancelados
SELECT cliente_id,
       COUNT(*)   AS pedidos,
       SUM(total) AS gastado
FROM pedidos
WHERE estado <> 'cancelado'      -- 1º: descarta filas de pedidos cancelados
GROUP BY cliente_id               -- 2º: agrupa por cliente
HAVING SUM(total) > 1000          -- 3º: se queda con los grupos que superan 1.000
ORDER BY gastado DESC;
```

### `WITH ROLLUP`: subtotales y total general
```sql
SELECT pais, ciudad, COUNT(*) AS clientes
FROM clientes
GROUP BY pais, ciudad WITH ROLLUP;
-- Añade filas con ciudad = NULL (subtotal de cada país) y una fila con pais = NULL (total)
```
Para distinguir la fila de subtotal de un `NULL` real usa `GROUPING()` (MySQL 8+):
```sql
SELECT IF(GROUPING(pais), 'TOTAL', pais) AS pais, COUNT(*) AS clientes
FROM clientes
GROUP BY pais WITH ROLLUP;
```

### `GROUP_CONCAT`: listas dentro de un grupo
```sql
-- Productos de cada pedido en una sola línea
SELECT pedido_id,
       GROUP_CONCAT(producto_id ORDER BY producto_id SEPARATOR ', ') AS productos,
       SUM(cantidad) AS unidades
FROM detalle_pedidos
GROUP BY pedido_id;
```
> ⚠️ `GROUP_CONCAT` corta el resultado a **1024 caracteres** por defecto sin avisar. Amplíalo con `SET SESSION group_concat_max_len = 100000;`.
> 🐘 **PostgreSQL / SQL Server:** `STRING_AGG(col, ', ')`.

### Agregación condicional (un "mini pivote")
```sql
-- Número de pedidos por estado para cada cliente, en columnas
SELECT cliente_id,
       COUNT(*)                              AS total,
       SUM(estado = 'enviado')               AS enviados,     -- En MySQL una comparación vale 1 o 0
       SUM(CASE WHEN estado = 'pendiente' THEN 1 ELSE 0 END) AS pendientes,  -- Versión estándar SQL
       SUM(IF(estado = 'cancelado', total, 0))               AS importe_cancelado
FROM pedidos
GROUP BY cliente_id;
```

---

## 🏋️ Ejercicios del Módulo 3

1. Muestra el nombre completo de cada empleado y su salario anual (14 pagas) formateado en estilo español (`72.800,00`).
2. ¿Cuántos productos hay por categoría y cuál es el precio medio (redondeado a 2 decimales) de cada una?
3. Muestra los departamentos (`departamento_id`) con **más de 1 empleado** y su salario medio, del más alto al más bajo. Excluye a los empleados sin departamento.
4. Facturación por año (`YEAR`) de los pedidos que no están cancelados, con una fila final de total general.
5. Clasifica a los clientes según su antigüedad de registro hasta el `2025-01-01`: "Veterano" (más de 18 meses), "Regular" (6 a 18) y "Nuevo" (menos de 6).
6. Para cada cliente con pedidos, muestra en una sola columna la lista de sus IDs de pedido (ej: `1, 3, 10`) y cuántos están `pendiente`.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
SELECT CONCAT(nombre, ' ', apellido) AS empleado,
       FORMAT(salario * 14, 2, 'es_ES') AS salario_anual
FROM empleados;

-- 2
SELECT categoria_id,
       COUNT(*)              AS productos,
       ROUND(AVG(precio), 2) AS precio_medio
FROM productos
GROUP BY categoria_id;

-- 3
SELECT departamento_id,
       COUNT(*)               AS empleados,
       ROUND(AVG(salario), 2) AS salario_medio
FROM empleados
WHERE departamento_id IS NOT NULL
GROUP BY departamento_id
HAVING COUNT(*) > 1
ORDER BY salario_medio DESC;
-- Dirección (1) tiene solo 1 → no aparece. Almacén (3): 2200.00, Ventas (2): 2483.33

-- 4 (calculamos el año en una subconsulta para poder usar GROUPING() sobre una columna:
--    GROUPING(YEAR(fecha)) no es aceptado con ONLY_FULL_GROUP_BY)
SELECT IF(GROUPING(anio), 'TOTAL', anio) AS anio,
       SUM(total) AS facturado
FROM (
    SELECT YEAR(fecha) AS anio, total
    FROM pedidos
    WHERE estado <> 'cancelado'
) AS t
GROUP BY anio WITH ROLLUP;
-- 2024: 5467.51 | 2025: 1659.69 | TOTAL: 7127.20

-- 5
SELECT nombre, fecha_registro,
       CASE
           WHEN TIMESTAMPDIFF(MONTH, fecha_registro, '2025-01-01') > 18 THEN 'Veterano'
           WHEN TIMESTAMPDIFF(MONTH, fecha_registro, '2025-01-01') >= 6 THEN 'Regular'
           ELSE 'Nuevo'
       END AS segmento
FROM clientes;

-- 6
SELECT cliente_id,
       GROUP_CONCAT(id ORDER BY id SEPARATOR ', ') AS pedidos,
       SUM(estado = 'pendiente')                   AS pendientes
FROM pedidos
GROUP BY cliente_id;
```
</details>

---

👉 **Siguiente paso:** **[Módulo 4: JOINs, Subconsultas y UNION](./04_JOINS_SUBCONSULTAS_Y_UNION.md)**
