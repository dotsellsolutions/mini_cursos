# Módulo 4: `JOIN`s, Subconsultas y `UNION`

En una base de datos bien diseñada (normalizada), la información está **repartida en varias tablas**: el pedido guarda el `cliente_id`, no el nombre del cliente. Para responder preguntas reales ("¿qué compró Sofía?") hay que **unir tablas**. Este es el módulo más importante para escribir SQL del mundo real.

---

## 1. `INNER JOIN`: solo las filas que coinciden en ambas tablas

```sql
-- Cada pedido con el nombre de su cliente
SELECT p.id, p.fecha, c.nombre AS cliente, p.total
FROM pedidos AS p
INNER JOIN clientes AS c ON c.id = p.cliente_id
ORDER BY p.id;
```
```
 pedidos                     clientes
 ┌────┬────────────┐         ┌────┬──────────────┐
 │ id │ cliente_id │         │ id │ nombre       │
 ├────┼────────────┤         ├────┼──────────────┤
 │ 1  │ 1  ────────┼────────►│ 1  │ Sofía        │
 │ 2  │ 2  ────────┼────────►│ 2  │ Javier       │
 │ 3  │ 1  ────────┼───┘     │ 6  │ Andrés  ✗    │ ← sin pedidos: NO aparece en INNER JOIN
 └────┴────────────┘         └────┴──────────────┘
```
* `JOIN` a secas significa `INNER JOIN`.
* Usa **siempre alias** (`p`, `c`) y antepón el alias a cada columna: evita ambigüedades (las dos tablas tienen `id`).

### Unir 3, 4 y 5 tablas
```sql
-- Detalle completo: pedido, cliente, producto, categoría y vendedor
SELECT p.id                         AS pedido,
       DATE(p.fecha)                AS fecha,
       c.nombre                     AS cliente,
       pr.nombre                    AS producto,
       cat.nombre                   AS categoria,
       d.cantidad,
       d.precio_unitario,
       d.cantidad * d.precio_unitario AS subtotal,
       CONCAT(e.nombre, ' ', e.apellido) AS vendedor
FROM pedidos          AS p
JOIN clientes         AS c   ON c.id   = p.cliente_id
JOIN detalle_pedidos  AS d   ON d.pedido_id = p.id
JOIN productos        AS pr  ON pr.id  = d.producto_id
JOIN categorias       AS cat ON cat.id = pr.categoria_id
LEFT JOIN empleados   AS e   ON e.id   = p.empleado_id   -- LEFT: el pedido 8 no tiene vendedor
ORDER BY p.id, pr.nombre;
```

### `USING`: atajo cuando las columnas se llaman igual
```sql
-- Cada línea de detalle junto al número de líneas de su pedido.
-- USING (pedido_id) equivale a ON d.pedido_id = r.pedido_id (y muestra la columna una sola vez)
SELECT pedido_id, d.producto_id, r.lineas
FROM detalle_pedidos AS d
JOIN (SELECT pedido_id, COUNT(*) AS lineas
      FROM detalle_pedidos
      GROUP BY pedido_id) AS r USING (pedido_id);
```
*(En `tienda_dba` las claves foráneas se llaman `cliente_id`, `pedido_id`... y las primarias `id`, así que normalmente usaremos `ON`.)*

---

## 2. `LEFT JOIN` y `RIGHT JOIN`: conservar las filas sin pareja

**`LEFT JOIN`** devuelve **todas** las filas de la tabla de la izquierda, tengan o no pareja. Donde no la hay, las columnas de la derecha salen como `NULL`.

```sql
-- TODOS los clientes, con su número de pedidos (incluidos los que nunca compraron)
SELECT c.nombre, COUNT(p.id) AS pedidos       -- ¡COUNT(p.id), no COUNT(*)! (COUNT(*) contaría 1 para quien no tiene pedidos)
FROM clientes AS c
LEFT JOIN pedidos AS p ON p.cliente_id = c.id
GROUP BY c.id, c.nombre
ORDER BY pedidos DESC;
```

### El patrón "anti-join": encontrar lo que NO tiene pareja
```sql
-- Clientes que NUNCA han hecho un pedido
SELECT c.id, c.nombre
FROM clientes AS c
LEFT JOIN pedidos AS p ON p.cliente_id = c.id
WHERE p.id IS NULL;           -- Andrés Castro y Hugo Romero

-- Productos que nunca se han vendido
SELECT pr.sku, pr.nombre
FROM productos AS pr
LEFT JOIN detalle_pedidos AS d ON d.producto_id = pr.id
WHERE d.producto_id IS NULL;  -- TEC-002
```

### ⚠️ El error clásico: filtrar la tabla derecha en el `WHERE`
```sql
-- ❌ Quieres todos los clientes y sus pedidos de 2025... pero el WHERE elimina a los que no tienen pedido
SELECT c.nombre, p.id
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
WHERE YEAR(p.fecha) = 2025;            -- Convierte el LEFT JOIN en un INNER JOIN sin que te des cuenta

-- ✅ La condición sobre la tabla derecha va en el ON
SELECT c.nombre, p.id
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
                   AND p.fecha >= '2025-01-01';
```

**`RIGHT JOIN`** es lo mismo que `LEFT JOIN` con las tablas al revés. En la práctica casi todo el mundo escribe `LEFT JOIN` y ordena las tablas para que la "principal" quede a la izquierda.

### `FULL OUTER JOIN` (no existe en MySQL): se emula con `UNION`
```sql
-- Todos los empleados y todos los departamentos, tengan o no relación
SELECT e.nombre AS empleado, d.nombre AS departamento
FROM empleados e LEFT JOIN departamentos d ON d.id = e.departamento_id
UNION
SELECT e.nombre, d.nombre
FROM empleados e RIGHT JOIN departamentos d ON d.id = e.departamento_id;
```
> 🐘 **PostgreSQL / SQL Server** sí tienen `FULL OUTER JOIN` directamente.

---

## 3. `CROSS JOIN`: todas las combinaciones

Combina **cada fila** de una tabla con **cada fila** de la otra (producto cartesiano: 8 × 4 = 32 filas).
```sql
-- Matriz de todos los departamentos × todos los estados de pedido (para un informe con ceros)
SELECT d.nombre, s.estado
FROM departamentos d
CROSS JOIN (SELECT 'pendiente' AS estado UNION ALL SELECT 'pagado'
            UNION ALL SELECT 'enviado' UNION ALL SELECT 'cancelado') AS s;
```
> 💣 Un `JOIN` sin condición `ON` (o con la condición equivocada) produce un `CROSS JOIN` accidental: con dos tablas de 100.000 filas obtienes **10.000 millones** de filas y tumbas el servidor.

---

## 4. *Self join*: unir una tabla consigo misma

Necesario cuando una tabla se referencia a sí misma, como `empleados.jefe_id`:
```sql
-- Cada empleado con el nombre de su jefe
SELECT CONCAT(e.nombre, ' ', e.apellido)      AS empleado,
       COALESCE(CONCAT(j.nombre, ' ', j.apellido), '— (sin jefe)') AS jefe
FROM empleados AS e
LEFT JOIN empleados AS j ON j.id = e.jefe_id;

-- Empleados que ganan MÁS que su jefe
SELECT e.nombre, e.salario, j.nombre AS jefe, j.salario AS salario_jefe
FROM empleados e
JOIN empleados j ON j.id = e.jefe_id
WHERE e.salario > j.salario;

-- Categoría con su categoría padre
SELECT hijo.nombre AS categoria, padre.nombre AS padre
FROM categorias hijo
LEFT JOIN categorias padre ON padre.id = hijo.padre_id;
```
*(Para recorrer el árbol completo a cualquier profundidad necesitarás una **CTE recursiva**, en el Módulo 5).*

---

## 5. Subconsultas

Una subconsulta es un `SELECT` dentro de otra consulta.

### A) Subconsulta escalar (devuelve 1 valor)
```sql
-- Productos más caros que la media
SELECT nombre, precio
FROM productos
WHERE precio > (SELECT AVG(precio) FROM productos);

-- Como columna: diferencia de cada salario con la media de la empresa
SELECT nombre, salario,
       salario - (SELECT AVG(salario) FROM empleados) AS diferencia_con_media
FROM empleados;
```

### B) Subconsulta con `IN` (devuelve una lista)
```sql
-- Clientes que han comprado alguna tarjeta gráfica (categoría 4)
SELECT nombre FROM clientes
WHERE id IN (
    SELECT p.cliente_id
    FROM pedidos p
    JOIN detalle_pedidos d ON d.pedido_id = p.id
    JOIN productos pr     ON pr.id = d.producto_id
    WHERE pr.categoria_id = 4
);
```

### C) `EXISTS` / `NOT EXISTS` (¿existe al menos una fila?)
```sql
-- Clientes con al menos un pedido pendiente
SELECT c.nombre
FROM clientes c
WHERE EXISTS (
    SELECT 1 FROM pedidos p
    WHERE p.cliente_id = c.id AND p.estado = 'pendiente'
);

-- Clientes SIN pedidos (alternativa segura al NOT IN, que falla con NULL — ver Módulo 2)
SELECT c.nombre
FROM clientes c
WHERE NOT EXISTS (SELECT 1 FROM pedidos p WHERE p.cliente_id = c.id);
```
> 💡 `EXISTS` se detiene en cuanto encuentra la primera fila: suele ser muy eficiente. El `SELECT 1` es una convención: lo que se devuelve no importa.

### D) Subconsultas correlacionadas (se ejecutan "por cada fila")
Hacen referencia a la consulta exterior (`e.departamento_id`):
```sql
-- Empleados que ganan más que la media de SU departamento
SELECT e.nombre, e.departamento_id, e.salario
FROM empleados e
WHERE e.salario > (
    SELECT AVG(e2.salario)
    FROM empleados e2
    WHERE e2.departamento_id = e.departamento_id
);
```
> 🐢 Son cómodas pero pueden ser lentas en tablas grandes (conceptualmente se recalculan por cada fila). A menudo se reescriben con un `JOIN` a una tabla derivada o con funciones de ventana (Módulo 5).

### E) Tablas derivadas (subconsulta en el `FROM`)
```sql
-- Ticket medio de cada cliente, y luego solo los que superan los 500 €
SELECT t.cliente_id, t.pedidos, t.ticket_medio
FROM (
    SELECT cliente_id, COUNT(*) AS pedidos, AVG(total) AS ticket_medio
    FROM pedidos
    WHERE estado <> 'cancelado'
    GROUP BY cliente_id
) AS t                                  -- ¡En MySQL la tabla derivada necesita alias!
WHERE t.ticket_medio > 500;
```

### F) `ANY` / `ALL`
```sql
-- ALL: productos más caros que TODOS los de la categoría Tarjetas gráficas (4) → solo los portátiles
SELECT nombre, precio FROM productos
WHERE precio > ALL (SELECT precio FROM productos WHERE categoria_id = 4);

-- ANY: productos más baratos que ALGUNO de los ratones (7), es decir, más baratos que el más caro
SELECT nombre, precio FROM productos
WHERE precio < ANY (SELECT precio FROM productos WHERE categoria_id = 7);
```

---

## 6. `UNION`, `INTERSECT` y `EXCEPT`: combinar resultados

Las consultas deben tener **el mismo número de columnas** y tipos compatibles. Los nombres de columna salen de la primera consulta.

```sql
-- Directorio de contactos: clientes + empleados en una sola lista
SELECT nombre, email, 'Cliente' AS tipo FROM clientes
UNION ALL
SELECT CONCAT(nombre, ' ', apellido), email, 'Empleado' FROM empleados
ORDER BY tipo, nombre;          -- El ORDER BY final ordena el resultado combinado
```
| Operador | Resultado | Disponible en |
| :--- | :--- | :--- |
| `UNION ALL` | Todas las filas, **con** duplicados. **Más rápido.** | Todos |
| `UNION` | Todas las filas **sin** duplicados (hace un `DISTINCT`, más lento) | Todos |
| `INTERSECT` | Filas que están en **ambas** consultas | MySQL 8.0.31+, MariaDB 10.3+, PostgreSQL, SQL Server |
| `EXCEPT` | Filas de la primera que **no** están en la segunda (`MINUS` en Oracle) | MySQL 8.0.31+, MariaDB 10.3+, PostgreSQL, SQL Server |

```sql
-- Clientes que compraron en 2024 Y TAMBIÉN en 2025 (clientes recurrentes)
SELECT cliente_id FROM pedidos WHERE fecha <  '2025-01-01'
INTERSECT
SELECT cliente_id FROM pedidos WHERE fecha >= '2025-01-01';   -- 1 y 3

-- IDs de clientes registrados que no tienen pedidos
SELECT id FROM clientes
EXCEPT
SELECT cliente_id FROM pedidos;
```

---

## 7. `UPDATE` y `DELETE` con `JOIN` (MySQL/MariaDB)

```sql
-- Desactivar los productos de categorías hijas de "Periféricos" (5) que no tienen stock
UPDATE productos pr
JOIN categorias c ON c.id = pr.categoria_id
SET pr.activo = FALSE
WHERE c.padre_id = 5 AND pr.stock = 0;

-- Recalcular el total de todos los pedidos a partir de su detalle (con tabla derivada)
UPDATE pedidos p
JOIN (
    SELECT pedido_id, SUM(cantidad * precio_unitario) AS total_real
    FROM detalle_pedidos
    GROUP BY pedido_id
) AS t ON t.pedido_id = p.id
SET p.total = t.total_real;

-- Borrar el detalle de los pedidos cancelados (indicando de QUÉ tabla se borra)
DELETE d
FROM detalle_pedidos d
JOIN pedidos p ON p.id = d.pedido_id
WHERE p.estado = 'cancelado';
```
> 🐘 **PostgreSQL:** `UPDATE productos pr SET activo = FALSE FROM categorias c WHERE c.id = pr.categoria_id AND ...` y `DELETE FROM detalle_pedidos d USING pedidos p WHERE ...`.

---

## 8. Resumen visual de los `JOIN`

```
 INNER JOIN          LEFT JOIN           RIGHT JOIN          FULL OUTER (UNION)    ANTI-JOIN (LEFT + IS NULL)
  ┌───┬───┐          ┌───┬───┐           ┌───┬───┐           ┌───┬───┐             ┌───┬───┐
  │ A │▓B │          │▓A▓│▓B │           │ A │▓B▓│           │▓A▓│▓B▓│             │▓A▓│ B │
  └───┴───┘          └───┴───┘           └───┴───┘           └───┴───┘             └───┴───┘
  solo la            todo A +            todo B +            todo A y              A sin pareja
  intersección       lo común de B       lo común de A       todo B                en B
```

---

## 🏋️ Ejercicios del Módulo 4

1. Lista cada producto con el nombre de su categoría y el de la categoría padre (si la tiene).
2. Muestra **todos** los empleados con el número de pedidos que han gestionado y el importe total vendido (pedidos no cancelados). Los que no tienen ventas deben aparecer con 0.
3. ¿Qué productos ha comprado Sofía Herrera? Muestra nombre del producto, unidades totales e importe total, sin repetir productos.
4. Lista los departamentos que no tienen ningún empleado y los empleados que no tienen departamento (una sola consulta).
5. Muestra los pedidos cuyo total es superior al ticket medio **de su propio cliente**.
6. Calcula, para cada categoría padre (`Informática`, `Periféricos`), las unidades vendidas en pedidos no cancelados.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
SELECT pr.nombre AS producto, c.nombre AS categoria, padre.nombre AS categoria_padre
FROM productos pr
JOIN categorias c          ON c.id = pr.categoria_id
LEFT JOIN categorias padre ON padre.id = c.padre_id;

-- 2
SELECT CONCAT(e.nombre, ' ', e.apellido) AS empleado,
       COUNT(p.id)                       AS pedidos,
       COALESCE(SUM(p.total), 0)         AS vendido
FROM empleados e
LEFT JOIN pedidos p ON p.empleado_id = e.id AND p.estado <> 'cancelado'
GROUP BY e.id, e.nombre, e.apellido
ORDER BY vendido DESC;
-- Marta López: 5 pedidos (4490.71) | Pedro Sánchez: 3 (1858.50) | resto: 0

-- 3
SELECT pr.nombre, SUM(d.cantidad) AS unidades, SUM(d.cantidad * d.precio_unitario) AS importe
FROM clientes c
JOIN pedidos p          ON p.cliente_id = c.id
JOIN detalle_pedidos d  ON d.pedido_id  = p.id
JOIN productos pr       ON pr.id        = d.producto_id
WHERE c.email = 'sofia@mail.com'
GROUP BY pr.id, pr.nombre;

-- 4
SELECT 'Departamento sin empleados' AS tipo, d.nombre
FROM departamentos d
LEFT JOIN empleados e ON e.departamento_id = d.id
WHERE e.id IS NULL
UNION ALL
SELECT 'Empleado sin departamento', CONCAT(nombre, ' ', apellido)
FROM empleados
WHERE departamento_id IS NULL;
-- Solo Diego Navarro (todos los departamentos tienen al menos un empleado)

-- 5 (subconsulta correlacionada)
SELECT p.id, p.cliente_id, p.total
FROM pedidos p
WHERE p.total > (SELECT AVG(p2.total) FROM pedidos p2 WHERE p2.cliente_id = p.cliente_id);
-- Pedidos 1 y 10 (Sofía), 2 (Javier) y 4 (Valentina).
-- Los clientes con un solo pedido nunca aparecen: su pedido ES su media.

-- 6
SELECT padre.nombre AS categoria_padre, SUM(d.cantidad) AS unidades
FROM detalle_pedidos d
JOIN pedidos p         ON p.id = d.pedido_id AND p.estado <> 'cancelado'
JOIN productos pr      ON pr.id = d.producto_id
JOIN categorias c      ON c.id = pr.categoria_id
LEFT JOIN categorias c2 ON c2.id = c.padre_id             -- padre directo
JOIN categorias padre  ON padre.id = COALESCE(c2.padre_id, c2.id, c.id)  -- sube hasta la raíz (máx. 3 niveles)
GROUP BY padre.id, padre.nombre;
-- Con más niveles de profundidad se usaría una CTE recursiva (Módulo 5)
```
</details>

---

👉 **Siguiente paso:** **[Módulo 5: SQL Avanzado — CTE, Funciones de Ventana y JSON](./05_SQL_AVANZADO_CTE_WINDOW_FUNCTIONS_Y_JSON.md)**
