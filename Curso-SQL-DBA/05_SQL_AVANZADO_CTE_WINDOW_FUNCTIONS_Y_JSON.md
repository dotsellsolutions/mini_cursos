# Módulo 5: SQL Avanzado — CTE, Funciones de Ventana, Pivotes y JSON

Este módulo separa a quien "sabe SQL" de quien **domina** SQL. Las CTE y las funciones de ventana resuelven en una consulta legible problemas que antes requerían subconsultas ilegibles, tablas temporales o código en el programa: rankings, acumulados, comparaciones con el mes anterior, árboles jerárquicos...

> Requiere **MySQL 8.0+** o **MariaDB 10.2+**. En MySQL 5.7 nada de esto existe.

---

## 1. CTE (*Common Table Expressions*): `WITH`

Una CTE es una **subconsulta con nombre** que defines al principio y usas después como si fuera una tabla. Hace las consultas complejas legibles "de arriba abajo".

```sql
-- Sin CTE: tabla derivada anidada (se lee de dentro hacia fuera)
SELECT * FROM (SELECT cliente_id, SUM(total) AS gastado FROM pedidos GROUP BY cliente_id) t WHERE gastado > 1000;

-- Con CTE: se lee como una receta
WITH gasto_por_cliente AS (
    SELECT cliente_id, SUM(total) AS gastado
    FROM pedidos
    WHERE estado <> 'cancelado'
    GROUP BY cliente_id
)
SELECT c.nombre, g.gastado
FROM gasto_por_cliente g
JOIN clientes c ON c.id = g.cliente_id
WHERE g.gastado > 1000
ORDER BY g.gastado DESC;
```

### Varias CTE encadenadas (cada una puede usar las anteriores)
```sql
WITH
ventas AS (                                   -- Paso 1: ventas válidas
    SELECT * FROM pedidos WHERE estado <> 'cancelado'
),
por_cliente AS (                              -- Paso 2: agrupar
    SELECT cliente_id, COUNT(*) AS pedidos, SUM(total) AS gastado
    FROM ventas GROUP BY cliente_id
),
media AS (                                    -- Paso 3: un único valor
    SELECT AVG(gastado) AS media_gasto FROM por_cliente
)
SELECT c.nombre, pc.pedidos, pc.gastado,
       ROUND(pc.gastado - m.media_gasto, 2) AS vs_media
FROM por_cliente pc
JOIN clientes c ON c.id = pc.cliente_id
CROSS JOIN media m
ORDER BY pc.gastado DESC;
```

---

## 2. CTE recursivas: árboles y jerarquías

Una CTE recursiva se llama **a sí misma**. Tiene dos partes unidas por `UNION ALL`:
1. **Ancla:** el punto de partida (por ejemplo, las categorías raíz).
2. **Parte recursiva:** cómo llegar al siguiente nivel a partir del anterior. Se repite hasta que no devuelve filas.

### Árbol de categorías con su ruta completa
```sql
WITH RECURSIVE arbol AS (
    -- 1) ANCLA: categorías sin padre (nivel 1)
    SELECT id, nombre, padre_id, 1 AS nivel,
           CAST(nombre AS CHAR(500)) AS ruta
    FROM categorias
    WHERE padre_id IS NULL

    UNION ALL

    -- 2) RECURSIVA: hijos de las filas que ya están en "arbol"
    SELECT c.id, c.nombre, c.padre_id, a.nivel + 1,
           CONCAT(a.ruta, ' > ', c.nombre)
    FROM categorias c
    JOIN arbol a ON c.padre_id = a.id
)
SELECT CONCAT(REPEAT('    ', nivel - 1), nombre) AS categoria, nivel, ruta
FROM arbol
ORDER BY ruta;
```
```
categoria                    nivel  ruta
Informática                  1      Informática
    Componentes              2      Informática > Componentes
        Tarjetas gráficas    3      Informática > Componentes > Tarjetas gráficas
    Portátiles               2      Informática > Portátiles
Periféricos                  1      Periféricos
    Ratones                  2      Periféricos > Ratones
    Teclados                 2      Periféricos > Teclados
```
> 💡 El `CAST(... AS CHAR(500))` en el ancla es necesario: el tipo de la columna lo decide el ancla, y si no lo amplías, las rutas largas se cortarían.

### Organigrama: todos los subordinados (directos e indirectos) de un jefe
```sql
WITH RECURSIVE equipo AS (
    SELECT id, nombre, jefe_id, 0 AS profundidad
    FROM empleados WHERE id = 1               -- Laura (directora)
    UNION ALL
    SELECT e.id, e.nombre, e.jefe_id, eq.profundidad + 1
    FROM empleados e
    JOIN equipo eq ON e.jefe_id = eq.id
)
SELECT * FROM equipo WHERE profundidad > 0;
```

### Generar series (fechas o números) sin tablas
```sql
-- Todos los meses de 2024, para un informe que muestre también los meses SIN ventas
WITH RECURSIVE meses AS (
    SELECT DATE('2024-01-01') AS mes
    UNION ALL
    SELECT mes + INTERVAL 1 MONTH FROM meses WHERE mes < '2024-12-01'
)
SELECT DATE_FORMAT(m.mes, '%Y-%m') AS mes,
       COUNT(p.id)                 AS pedidos,
       COALESCE(SUM(p.total), 0)   AS facturado
FROM meses m
LEFT JOIN pedidos p
       ON p.fecha >= m.mes AND p.fecha < m.mes + INTERVAL 1 MONTH
      AND p.estado <> 'cancelado'
GROUP BY m.mes
ORDER BY m.mes;
```
> 🛑 **Protección contra bucles infinitos:** MySQL corta la recursión a los 1000 niveles (`cte_max_recursion_depth`). Si tus datos tienen un ciclo (A es jefe de B y B de A), obtendrás ese error en vez de colgar el servidor.

---

## 3. Funciones de ventana (*Window Functions*)

Una función de agregación con `GROUP BY` **colapsa** las filas (10 pedidos → 1 fila por cliente). Una **función de ventana** calcula sobre un grupo de filas **sin colapsarlas**: cada fila conserva su detalle y además recibe el cálculo.

```sql
SELECT id, cliente_id, total,
       SUM(total) OVER (PARTITION BY cliente_id) AS total_del_cliente,
       ROUND(total / SUM(total) OVER (PARTITION BY cliente_id) * 100, 1) AS pct_del_cliente
FROM pedidos
ORDER BY cliente_id, id;
```

### Anatomía de `OVER (...)`
```text
funcion() OVER (
    PARTITION BY columna      -- "Reinicia" el cálculo para cada grupo (opcional)
    ORDER BY columna          -- Orden dentro de cada grupo (necesario para rankings, LAG, acumulados)
    ROWS BETWEEN ... AND ...  -- Qué filas exactas forman la ventana (opcional, "frame")
)
```

### A) Rankings: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`
```sql
SELECT nombre, precio,
       ROW_NUMBER() OVER (ORDER BY precio DESC) AS num_fila,    -- 1,2,3,4... (nunca repite)
       -- (¡No uses "row_number" o "rank" como alias: son palabras reservadas desde MySQL 8!)
       RANK()       OVER (ORDER BY precio DESC) AS rnk,         -- 1,2,2,4... (empates comparten y saltan)
       DENSE_RANK() OVER (ORDER BY precio DESC) AS dense,       -- 1,2,2,3... (empates comparten, no saltan)
       NTILE(4)     OVER (ORDER BY precio DESC) AS cuartil,     -- Divide en 4 grupos iguales
       PERCENT_RANK() OVER (ORDER BY precio)    AS percentil
FROM productos;
```
> 🧪 En `tienda_dba` no hay dos precios iguales, así que las tres columnas coinciden. Para ver la diferencia, crea un empate y repite la consulta: `UPDATE productos SET precio = 649.90 WHERE id = 1;`.

### B) El clásico "Top N por grupo"
```sql
-- El producto más caro de cada categoría
WITH ranking AS (
    SELECT categoria_id, nombre, precio,
           ROW_NUMBER() OVER (PARTITION BY categoria_id ORDER BY precio DESC) AS pos
    FROM productos
)
SELECT categoria_id, nombre, precio
FROM ranking
WHERE pos = 1;          -- Cambia a "pos <= 3" para el top 3 de cada categoría
```
> ⚠️ No puedes filtrar por una función de ventana en el `WHERE` de la misma consulta (las ventanas se calculan **después** del `WHERE`). Por eso se envuelve en una CTE o tabla derivada.

### C) Comparar con la fila anterior/siguiente: `LAG` y `LEAD`
```sql
-- Días transcurridos entre un pedido y el anterior del MISMO cliente
SELECT cliente_id, id, DATE(fecha) AS fecha, total,
       LAG(total)  OVER w                                     AS total_anterior,
       total - LAG(total) OVER w                              AS diferencia,
       DATEDIFF(fecha, LAG(fecha) OVER w)                     AS dias_desde_anterior,
       LEAD(DATE(fecha)) OVER w                               AS siguiente_pedido
FROM pedidos
WINDOW w AS (PARTITION BY cliente_id ORDER BY fecha)          -- Ventana con nombre reutilizable
ORDER BY cliente_id, fecha;
```
`LAG(col, 2, 0)` = valor de 2 filas atrás, con `0` si no existe.

### D) Acumulados y medias móviles (con *frame*)
```sql
SELECT id, DATE(fecha) AS fecha, total,
       -- Total acumulado desde el primer pedido hasta este
       SUM(total) OVER (ORDER BY fecha ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS acumulado,
       -- Media móvil de los últimos 3 pedidos (el actual y los 2 anteriores)
       ROUND(AVG(total) OVER (ORDER BY fecha ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS media_movil_3
FROM pedidos
WHERE estado <> 'cancelado'
ORDER BY fecha;
```
| *Frame* | Significado |
| :--- | :--- |
| `UNBOUNDED PRECEDING` | Desde la primera fila de la partición |
| `n PRECEDING` / `n FOLLOWING` | n filas antes / después |
| `CURRENT ROW` | La fila actual |
| `UNBOUNDED FOLLOWING` | Hasta la última fila de la partición |

### E) `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`
```sql
-- Para cada empleado: el salario más alto de su departamento y quién lo cobra
SELECT nombre, departamento_id, salario,
       FIRST_VALUE(nombre) OVER (PARTITION BY departamento_id ORDER BY salario DESC) AS mejor_pagado,
       MAX(salario)        OVER (PARTITION BY departamento_id)                       AS salario_max_depto
FROM empleados
WHERE departamento_id IS NOT NULL;
```
> ⚠️ `LAST_VALUE` con `ORDER BY` sorprende: el *frame* por defecto termina en la fila actual. Usa `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

### F) Eliminar duplicados conservando uno (tarea típica de DBA)
```sql
-- Supón una tabla con emails repetidos por un bug de importación
CREATE TABLE suscriptores (id INT AUTO_INCREMENT PRIMARY KEY, email VARCHAR(120), creado DATETIME);
INSERT INTO suscriptores (email, creado) VALUES
('a@mail.com','2024-01-01'), ('b@mail.com','2024-01-02'),
('a@mail.com','2024-02-01'), ('a@mail.com','2024-03-01'), ('b@mail.com','2024-03-05');

-- 1) Ver cuáles sobran (todas menos la más antigua de cada email)
WITH dup AS (
    SELECT id, email, ROW_NUMBER() OVER (PARTITION BY email ORDER BY creado) AS n
    FROM suscriptores
)
SELECT * FROM dup WHERE n > 1;

-- 2) Borrarlas
DELETE FROM suscriptores
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY creado) AS n
        FROM suscriptores
    ) AS x
    WHERE n > 1
);

-- 3) Evitar que vuelva a pasar
ALTER TABLE suscriptores ADD UNIQUE (email);
```
*(La subconsulta doble `SELECT id FROM (...) AS x` es necesaria porque MySQL no permite borrar de una tabla que lees directamente en la misma sentencia: error 1093).*

---

## 4. Tablas pivote (filas → columnas)

MySQL no tiene `PIVOT` (SQL Server sí). Se hace con agregación condicional:
```sql
-- Facturación por cliente y año, con un año en cada columna
SELECT c.nombre,
       SUM(CASE WHEN YEAR(p.fecha) = 2024 THEN p.total ELSE 0 END) AS `2024`,
       SUM(CASE WHEN YEAR(p.fecha) = 2025 THEN p.total ELSE 0 END) AS `2025`,
       SUM(p.total)                                                AS total
FROM pedidos p
JOIN clientes c ON c.id = p.cliente_id
WHERE p.estado <> 'cancelado'
GROUP BY c.id, c.nombre
ORDER BY total DESC;
```
Para columnas **dinámicas** (no sabes de antemano qué años hay) se genera el SQL con `GROUP_CONCAT` y se ejecuta con `PREPARE` (lo verás en el Módulo 6).

### Unpivot (columnas → filas)
```sql
SELECT id, 'precio' AS atributo, precio AS valor FROM productos WHERE id = 1
UNION ALL
SELECT id, 'stock', stock FROM productos WHERE id = 1;
```

---

## 5. Columnas generadas (calculadas)

Columnas cuyo valor **calcula el propio servidor** a partir de otras:
```sql
ALTER TABLE detalle_pedidos
    ADD COLUMN subtotal DECIMAL(12,2)
        AS (cantidad * precio_unitario) STORED;   -- STORED: se guarda en disco (se puede indexar)

ALTER TABLE empleados
    ADD COLUMN nombre_completo VARCHAR(130)
        AS (CONCAT(nombre, ' ', apellido)) VIRTUAL; -- VIRTUAL: se calcula al leer (no ocupa espacio)

SELECT pedido_id, producto_id, subtotal FROM detalle_pedidos;
SELECT nombre_completo FROM empleados;
-- INSERT/UPDATE no pueden escribir en ellas: siempre están sincronizadas
```
> 💡 Truco de rendimiento: crea una columna generada `VIRTUAL` con la expresión que filtras a menudo (`YEAR(fecha)`, `LOWER(email)`) y **ponle un índice**. MySQL 8.0.13+ también permite **índices funcionales** directamente: `CREATE INDEX idx ON pedidos ((YEAR(fecha)));`.

---

## 6. JSON en MySQL

Desde MySQL 5.7 / MariaDB 10.2 existe el tipo `JSON`: útil para atributos variables (especificaciones técnicas, configuraciones), **no** para reemplazar el diseño relacional.

```sql
ALTER TABLE productos ADD COLUMN specs JSON NULL;

UPDATE productos SET specs = '{"marca": "ASUS", "ram_gb": 16, "colores": ["negro", "plata"]}' WHERE id = 1;
UPDATE productos SET specs = JSON_OBJECT('marca', 'MSI', 'ram_gb', 32, 'colores', JSON_ARRAY('negro')) WHERE id = 2;
UPDATE productos SET specs = '{"marca": "Logitech", "inalambrico": true}' WHERE id = 9;
```

### Leer valores
```sql
SELECT nombre,
       specs->'$.marca'                     AS marca_json,     -- "ASUS"  (con comillas, tipo JSON)
       specs->>'$.marca'                    AS marca,          -- ASUS    (texto limpio) = JSON_UNQUOTE(JSON_EXTRACT(...))
       specs->>'$.ram_gb'                   AS ram,
       specs->>'$.colores[0]'               AS primer_color,   -- Arrays: índice desde 0
       JSON_LENGTH(specs->'$.colores')      AS num_colores
FROM productos
WHERE specs IS NOT NULL;
```

### Filtrar y buscar
```sql
SELECT nombre FROM productos WHERE specs->>'$.marca' = 'ASUS';
SELECT nombre FROM productos WHERE CAST(specs->>'$.ram_gb' AS UNSIGNED) >= 16;
SELECT nombre FROM productos WHERE JSON_CONTAINS(specs->'$.colores', '"plata"');
SELECT nombre FROM productos WHERE JSON_CONTAINS_PATH(specs, 'one', '$.inalambrico');
-- MySQL 8.0.17+: buscar dentro de arrays
SELECT nombre FROM productos WHERE 'negro' MEMBER OF (specs->'$.colores');
```

### Modificar parte de un JSON
```sql
UPDATE productos SET specs = JSON_SET(specs, '$.ram_gb', 32, '$.garantia', 24) WHERE id = 1;   -- crea o reemplaza
UPDATE productos SET specs = JSON_REMOVE(specs, '$.garantia') WHERE id = 1;
UPDATE productos SET specs = JSON_ARRAY_APPEND(specs, '$.colores', 'azul') WHERE id = 1;
```

### Convertir JSON en filas (`JSON_TABLE`, MySQL 8+)
```sql
SELECT p.nombre, c.color
FROM productos p,
     JSON_TABLE(p.specs, '$.colores[*]' COLUMNS (color VARCHAR(30) PATH '$')) AS c
WHERE p.specs IS NOT NULL;
```

### Generar JSON desde SQL (para APIs)
```sql
SELECT JSON_OBJECT(
           'pedido',   p.id,
           'cliente',  c.nombre,
           'total',    p.total,
           'lineas',   (SELECT JSON_ARRAYAGG(JSON_OBJECT('producto_id', d.producto_id, 'cantidad', d.cantidad))
                        FROM detalle_pedidos d WHERE d.pedido_id = p.id)
       ) AS documento
FROM pedidos p
JOIN clientes c ON c.id = p.cliente_id
WHERE p.id = 4;
```

### Indexar un campo de un JSON
```sql
ALTER TABLE productos
    ADD COLUMN marca VARCHAR(50) AS (specs->>'$.marca') VIRTUAL,
    ADD INDEX idx_marca (marca);
```
> 🐘 **PostgreSQL** tiene `JSONB` con índices GIN y operadores `->`, `->>`, `@>`; es el motor más potente para JSON.

---

## 🏋️ Ejercicios del Módulo 5

1. Con una CTE, muestra los clientes cuyo gasto total (pedidos no cancelados) supera la media de gasto de los clientes que compraron.
2. Para cada departamento, muestra el ranking de salarios de sus empleados (1 = mejor pagado) usando `DENSE_RANK`.
3. Muestra el producto **más vendido en unidades** de cada categoría padre (Informática y Periféricos), en pedidos no cancelados.
4. Para cada pedido no cancelado muestra su total, el total del pedido anterior (por fecha, de toda la tienda) y el porcentaje de cambio.
5. Con una CTE recursiva, muestra la cadena de mando completa de Jorge Díaz hasta la directora (`Jorge → Ana → Laura`).
6. Crea un informe con una fila por mes de 2024 (los 12 meses, aunque no haya ventas) y el facturado acumulado del año.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
WITH gasto AS (
    SELECT cliente_id, SUM(total) AS gastado
    FROM pedidos WHERE estado <> 'cancelado'
    GROUP BY cliente_id
)
SELECT c.nombre, g.gastado
FROM gasto g JOIN clientes c ON c.id = g.cliente_id
WHERE g.gastado > (SELECT AVG(gastado) FROM gasto);
-- Sofía (2579.86), Valentina (1936.60), Javier (1607.79)

-- 2
SELECT departamento_id, nombre, salario,
       DENSE_RANK() OVER (PARTITION BY departamento_id ORDER BY salario DESC) AS puesto
FROM empleados
WHERE departamento_id IS NOT NULL;

-- 3
WITH RECURSIVE raiz AS (
    SELECT id, id AS raiz_id FROM categorias WHERE padre_id IS NULL
    UNION ALL
    SELECT c.id, r.raiz_id FROM categorias c JOIN raiz r ON c.padre_id = r.id
),
ventas AS (
    SELECT r.raiz_id, pr.nombre, SUM(d.cantidad) AS unidades
    FROM detalle_pedidos d
    JOIN pedidos p   ON p.id = d.pedido_id AND p.estado <> 'cancelado'
    JOIN productos pr ON pr.id = d.producto_id
    JOIN raiz r       ON r.id = pr.categoria_id
    GROUP BY r.raiz_id, pr.id, pr.nombre
),
ranking AS (
    SELECT v.*, ROW_NUMBER() OVER (PARTITION BY raiz_id ORDER BY unidades DESC) AS pos
    FROM ventas v
)
SELECT c.nombre AS categoria_padre, r.nombre AS producto, r.unidades
FROM ranking r JOIN categorias c ON c.id = r.raiz_id
WHERE r.pos = 1;
-- Informática: Memoria RAM 32GB DDR5 (3) | Periféricos: Ratón Gaming 26K DPI (5)
-- ⚠️ En Informática hay EMPATE: el SSD NVMe 1TB también vendió 3. ROW_NUMBER elige uno al azar;
--    cambia ROW_NUMBER() por RANK() para que aparezcan todos los empatados.

-- 4
SELECT id, DATE(fecha) AS fecha, total,
       LAG(total) OVER (ORDER BY fecha) AS anterior,
       ROUND((total - LAG(total) OVER (ORDER BY fecha)) / LAG(total) OVER (ORDER BY fecha) * 100, 1) AS pct_cambio
FROM pedidos
WHERE estado <> 'cancelado'
ORDER BY fecha;

-- 5
WITH RECURSIVE cadena AS (
    SELECT id, nombre, jefe_id, CAST(nombre AS CHAR(200)) AS ruta
    FROM empleados WHERE email = 'jorge@tienda.com'
    UNION ALL
    SELECT e.id, e.nombre, e.jefe_id, CONCAT(c.ruta, ' → ', e.nombre)
    FROM empleados e JOIN cadena c ON e.id = c.jefe_id
)
SELECT ruta FROM cadena WHERE jefe_id IS NULL;
-- Jorge → Ana → Laura

-- 6
WITH RECURSIVE meses AS (
    SELECT DATE('2024-01-01') AS mes
    UNION ALL
    SELECT mes + INTERVAL 1 MONTH FROM meses WHERE mes < '2024-12-01'
),
por_mes AS (
    SELECT m.mes, COALESCE(SUM(p.total), 0) AS facturado
    FROM meses m
    LEFT JOIN pedidos p ON p.fecha >= m.mes AND p.fecha < m.mes + INTERVAL 1 MONTH
                       AND p.estado <> 'cancelado'
    GROUP BY m.mes
)
SELECT DATE_FORMAT(mes, '%Y-%m') AS mes, facturado,
       SUM(facturado) OVER (ORDER BY mes) AS acumulado
FROM por_mes;
-- Diciembre termina en 5467.51
```
</details>

---

👉 **Siguiente paso:** **[Módulo 6: Vistas, Procedimientos, Funciones, Triggers y Eventos](./06_VISTAS_PROCEDIMIENTOS_FUNCIONES_TRIGGERS_Y_EVENTOS.md)**
