# Módulo 2: CRUD — `INSERT`, `SELECT`, `UPDATE` y `DELETE`

**CRUD** = *Create, Read, Update, Delete*: las cuatro operaciones que harás miles de veces. En este módulo las dominas a fondo, incluidas las variantes que muchos programadores no conocen y los hábitos de seguridad que distinguen a un DBA.

> Asegúrate de tener cargada la base de datos del [índice](./00_INDICE_Y_BASE_DE_DATOS_DE_PRACTICA.md) y ejecuta `USE tienda_dba;`.

---

## 1. `INSERT`: Crear filas

```sql
-- Forma recomendada: indicando SIEMPRE las columnas
INSERT INTO clientes (nombre, email, ciudad, pais, fecha_registro)
VALUES ('Lola Méndez', 'lola@mail.com', 'Sevilla', 'ES', CURDATE());

-- Varias filas en una sola sentencia (mucho más rápido que muchos INSERT sueltos)
INSERT INTO clientes (nombre, email, ciudad, pais, fecha_registro) VALUES
('Tomás Gil',   'tomas@mail.com',  'Valencia', 'ES', '2025-01-10'),
('Renata Silva','renata@mail.com', 'Santiago', 'CL', '2025-01-11');

-- ¿Qué ID se acaba de generar? (el de la PRIMERA fila del último INSERT de TU conexión)
SELECT LAST_INSERT_ID();
```
> ❌ **Evita** `INSERT INTO clientes VALUES (...)` sin nombrar columnas: si mañana alguien añade una columna, tu sentencia se rompe o, peor, mete los datos en la columna equivocada.

### `INSERT ... SELECT`: copiar datos de una consulta
```sql
CREATE TABLE clientes_espana LIKE clientes;

INSERT INTO clientes_espana
SELECT * FROM clientes WHERE pais = 'ES';
```

### Qué hacer si la fila ya existe (claves `UNIQUE` o `PRIMARY KEY` duplicadas)
```sql
-- 1) INSERT IGNORE: si choca con un duplicado, la salta en silencio (convierte errores en avisos)
INSERT IGNORE INTO clientes (nombre, email, fecha_registro)
VALUES ('Sofía Duplicada', 'sofia@mail.com', CURDATE());   -- 0 filas afectadas
SHOW WARNINGS;

-- 2) ON DUPLICATE KEY UPDATE: "inserta, y si ya existe, actualiza" (UPSERT)
INSERT INTO productos (sku, nombre, categoria_id, precio, stock)
VALUES ('RAT-001', 'Ratón Gaming 26K DPI', 7, 49.90, 10)
ON DUPLICATE KEY UPDATE
    stock = stock + VALUES(stock);   -- Suma 10 al stock existente
-- MySQL 8.0.19+ (sintaxis moderna con alias; VALUES() está marcado como obsoleto desde 8.0.20
-- aunque sigue funcionando, y es la ÚNICA forma que acepta MariaDB):
-- VALUES (...) AS nuevo ON DUPLICATE KEY UPDATE stock = stock + nuevo.stock;

-- 3) REPLACE: si existe, BORRA la fila vieja e inserta la nueva (¡cambia el id y dispara ON DELETE CASCADE!)
REPLACE INTO departamentos (id, nombre) VALUES (4, 'Atención al Cliente');
```
| Opción | Si existe el duplicado... | Peligro |
| :--- | :--- | :--- |
| `INSERT` normal | Error `1062 Duplicate entry` | — |
| `INSERT IGNORE` | No hace nada | Oculta **también** otros errores (datos truncados, etc.). |
| `ON DUPLICATE KEY UPDATE` | Actualiza las columnas que digas | La opción más segura para *upserts*. |
| `REPLACE` | Borra + inserta | Borra hijos en cascada, pierde columnas no indicadas. |

> 🐘 **PostgreSQL:** `INSERT ... ON CONFLICT (email) DO UPDATE SET ...` o `ON CONFLICT DO NOTHING`. **SQL Server / estándar:** `MERGE`.

---

## 2. `SELECT`: Leer datos

### Orden en que se ESCRIBE vs orden en que se EJECUTA
```
Se escribe:   SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
Se ejecuta:   FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```
Entender esto explica muchos errores: por ejemplo, **no puedes usar en `WHERE` un alias definido en `SELECT`**, porque `WHERE` se ejecuta antes.

### Lo básico
```sql
SELECT * FROM productos;                          -- Todas las columnas (evítalo en producción)
SELECT sku, nombre, precio FROM productos;        -- Solo las que necesitas

-- Alias de columna y expresiones calculadas
SELECT nombre,
       precio,
       precio * 1.21        AS precio_con_iva,
       stock * precio       AS valor_inventario
FROM productos;

-- Alias de tabla (imprescindible cuando hay JOINs)
SELECT p.nombre, p.precio FROM productos AS p;

-- Valores únicos
SELECT DISTINCT pais FROM clientes;
SELECT DISTINCT pais, ciudad FROM clientes;       -- Combinaciones únicas
```

### `WHERE`: filtrar filas
| Operador | Ejemplo |
| :--- | :--- |
| `=`, `<>` o `!=`, `<`, `>`, `<=`, `>=` | `WHERE precio >= 100` |
| `AND`, `OR`, `NOT` | `WHERE activo = TRUE AND stock > 0` |
| `BETWEEN a AND b` (incluye los extremos) | `WHERE precio BETWEEN 50 AND 150` |
| `IN (...)` / `NOT IN (...)` | `WHERE pais IN ('MX','CO','PE')` |
| `LIKE` con `%` (cualquier texto) y `_` (un carácter) | `WHERE nombre LIKE 'Teclado%'` |
| `IS NULL` / `IS NOT NULL` | `WHERE ciudad IS NULL` |
| `REGEXP` / `RLIKE` | `WHERE sku REGEXP '^(LAP|GPU)-'` |

```sql
-- Productos activos con stock, de más de 100 € o de la categoría 6 (teclados)
SELECT nombre, precio, stock, categoria_id
FROM productos
WHERE activo = TRUE
  AND stock > 0
  AND (precio > 100 OR categoria_id = 6);   -- ¡Los paréntesis importan! AND se evalúa antes que OR

-- Pedidos de 2024 (forma correcta para fechas con hora)
SELECT id, fecha, total FROM pedidos
WHERE fecha >= '2024-01-01' AND fecha < '2025-01-01';

-- Nombres que contienen "gaming" (sin importar mayúsculas con collation _ci)
SELECT nombre FROM productos WHERE nombre LIKE '%gaming%';

-- Buscar un % o _ literal: se escapan con \
SELECT 'Descuento 50% hoy' LIKE '%50\%%' AS contiene_50_por_ciento;   -- 1 (verdadero)
```

### ⚠️ La trampa del `NULL`
`NULL` significa "desconocido". **Cualquier comparación con `NULL` da `NULL`** (ni verdadero ni falso):
```sql
SELECT nombre FROM clientes WHERE ciudad = NULL;      -- ❌ Devuelve 0 filas SIEMPRE
SELECT nombre FROM clientes WHERE ciudad IS NULL;     -- ✅ Hugo Romero

SELECT nombre FROM clientes WHERE ciudad <> 'Madrid'; -- ❌ ¡Hugo NO aparece! (NULL <> 'Madrid' es NULL)
SELECT nombre FROM clientes WHERE ciudad <> 'Madrid' OR ciudad IS NULL;  -- ✅
SELECT nombre FROM clientes WHERE NOT (ciudad <=> 'Madrid');             -- ✅ <=> es el "igual seguro con NULL" de MySQL

-- ¡Peligro con NOT IN y NULL! Si la subconsulta devuelve algún NULL, NOT IN devuelve 0 filas
SELECT nombre FROM empleados
WHERE id NOT IN (SELECT jefe_id FROM empleados);                          -- ❌ 0 filas (hay jefe_id NULL)
SELECT nombre FROM empleados
WHERE id NOT IN (SELECT jefe_id FROM empleados WHERE jefe_id IS NOT NULL); -- ✅ los que no son jefes
```

### `ORDER BY` y `LIMIT`
```sql
-- Ordenar por varias columnas
SELECT nombre, categoria_id, precio
FROM productos
ORDER BY categoria_id ASC, precio DESC;

-- Los 3 productos más caros
SELECT nombre, precio FROM productos ORDER BY precio DESC LIMIT 3;

-- Paginación: página 2 de 4 en 4 (salta 4, toma 4)
SELECT id, nombre FROM productos ORDER BY id LIMIT 4 OFFSET 4;   -- o: LIMIT 4, 4

-- Ordenar poniendo los NULL al final (en MySQL los NULL van primero en ASC)
SELECT nombre, ciudad FROM clientes ORDER BY ciudad IS NULL, ciudad;

-- Orden personalizado
SELECT id, estado FROM pedidos
ORDER BY FIELD(estado, 'pendiente', 'pagado', 'enviado', 'cancelado');
```
> 🐢 **`OFFSET` grande es lento:** `LIMIT 20 OFFSET 500000` lee y descarta 500.000 filas. Para paginar tablas enormes usa **paginación por clave** (*keyset*): `WHERE id > :ultimo_id_visto ORDER BY id LIMIT 20`.
>
> 🐘 **SQL Server:** `SELECT TOP 3 ...` u `OFFSET 4 ROWS FETCH NEXT 4 ROWS ONLY`. **PostgreSQL:** igual que MySQL (`LIMIT ... OFFSET ...`) y además `NULLS LAST`.

---

## 3. `UPDATE`: Modificar filas

```sql
-- Subir un 5% el precio de los teclados
UPDATE productos
SET precio = ROUND(precio * 1.05, 2)
WHERE categoria_id = 6;

-- Varias columnas a la vez
UPDATE clientes
SET ciudad = 'Madrid', activo = TRUE
WHERE email = 'hugo@mail.com';

-- Con ORDER BY y LIMIT (solo MySQL/MariaDB): desactivar los 2 productos con menos stock
UPDATE productos SET activo = FALSE
WHERE stock < 5
ORDER BY stock
LIMIT 2;
```

## 4. `DELETE`: Borrar filas

```sql
DELETE FROM clientes WHERE email = 'lola@mail.com';

-- Borrar en lotes (tablas enormes: no bloquea la tabla durante minutos ni llena el log)
DELETE FROM clientes_espana WHERE fecha_registro < '2024-01-01' LIMIT 1000;
-- (repetir hasta que afecte a 0 filas)
```

### "Soft delete": no borrar de verdad
En muchos sistemas no se borra, se **marca** como borrado para poder recuperarlo y mantener el historial:
```sql
ALTER TABLE clientes ADD COLUMN borrado_en DATETIME NULL;
UPDATE clientes SET borrado_en = NOW() WHERE id = 6;
SELECT * FROM clientes WHERE borrado_en IS NULL;   -- Las consultas normales ignoran los "borrados"
```

---

## 5. 🛡️ Hábitos de seguridad de un DBA antes de `UPDATE`/`DELETE`

El error más famoso de la historia de las bases de datos: **olvidar el `WHERE`**.

```sql
UPDATE empleados SET salario = 1000;   -- 😱 Acabas de cambiar el salario de TODA la empresa
```

**1. Haz primero un `SELECT` con el mismo `WHERE`:**
```sql
SELECT COUNT(*) FROM productos WHERE categoria_id = 6;   -- ¿Cuántas filas voy a tocar? ¿Son las que espero?
-- Si el número cuadra, cambia SELECT COUNT(*) por UPDATE/DELETE manteniendo el WHERE
```

**2. Trabaja dentro de una transacción** (Módulo 7): si algo sale mal, `ROLLBACK`.
```sql
START TRANSACTION;
UPDATE productos SET precio = precio * 0.9 WHERE categoria_id = 7;
SELECT nombre, precio FROM productos WHERE categoria_id = 7;   -- Revisa el resultado
ROLLBACK;   -- o COMMIT; si todo está bien
```

**3. Activa el modo seguro en tu sesión de consola:**
```sql
SET SQL_SAFE_UPDATES = 1;
UPDATE productos SET activo = FALSE;
-- ERROR 1175: You are using safe update mode and you tried to update a table without a WHERE that uses a KEY column
```
También puedes arrancar la consola con `mysql --safe-updates` (alias: `--i-am-a-dummy` 😄).

**4. Haz un respaldo rápido de las filas que vas a tocar:**
```sql
CREATE TABLE bkp_productos_20250214 AS SELECT * FROM productos WHERE categoria_id = 6;
```

---

## 🏋️ Ejercicios del Módulo 2

> Reinicia la base de datos con el script del índice antes de empezar.

1. Lista los clientes de España o México registrados en 2024, ordenados por fecha de registro (más recientes primero).
2. Muestra nombre, precio y stock de los productos **activos** sin stock.
3. Muestra los productos cuyo SKU empieza por `TEC` o `RAT` y cuyo precio está entre 35 y 55 €, usando `LIKE` y `BETWEEN`.
4. Lista los clientes que **no** son de Madrid (incluido el que no tiene ciudad).
5. Inserta un producto con SKU `CMP-001` y stock 5. Si el SKU ya existe, en vez de fallar **suma** 5 al stock y actualiza el precio a 84.95.
6. Muestra la "página 3" del catálogo ordenado por nombre, con 3 productos por página.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
SELECT nombre, pais, fecha_registro
FROM clientes
WHERE pais IN ('ES', 'MX')
  AND fecha_registro BETWEEN '2024-01-01' AND '2024-12-31'
ORDER BY fecha_registro DESC;
-- Hugo Romero, Isabella Vargas, Camila Ortega

-- 2
SELECT nombre, precio, stock FROM productos WHERE activo = TRUE AND stock = 0;
-- Tarjeta gráfica RX 7800

-- 3
SELECT sku, nombre, precio
FROM productos
WHERE (sku LIKE 'TEC%' OR sku LIKE 'RAT%')
  AND precio BETWEEN 35 AND 55;
-- TEC-002 (39.50), RAT-001 (49.90)

-- 4
SELECT nombre, ciudad FROM clientes WHERE ciudad <> 'Madrid' OR ciudad IS NULL;

-- 5
INSERT INTO productos (sku, nombre, categoria_id, precio, stock)
VALUES ('CMP-001', 'SSD NVMe 1TB', 3, 84.95, 5)
ON DUPLICATE KEY UPDATE
    stock  = stock + VALUES(stock),
    precio = VALUES(precio);
-- stock pasa de 40 a 45

-- 6 (página 3 → saltar 2 páginas × 3 = 6)
SELECT nombre FROM productos ORDER BY nombre LIMIT 3 OFFSET 6;
```
</details>

---

👉 **Siguiente paso:** **[Módulo 3: Funciones, Agregación, GROUP BY y HAVING](./03_FUNCIONES_AGREGACION_GROUP_BY_Y_HAVING.md)**
