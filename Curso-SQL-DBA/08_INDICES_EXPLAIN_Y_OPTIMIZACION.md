# Módulo 8: Índices, `EXPLAIN` y Optimización de Consultas

"La base de datos va lenta" es la frase que más escucha un DBA. En el 90 % de los casos la causa es una consulta sin el **índice** adecuado o escrita de forma que no puede usarlo. En este módulo aprendes cómo funcionan los índices por dentro, a leer el plan de ejecución (`EXPLAIN`) y a encontrar y arreglar las consultas lentas.

---

## 0. Preparar una tabla grande para practicar

Con 10 pedidos todo es instantáneo y `EXPLAIN` no enseña nada. Vamos a generar **200.000 pedidos** en una tabla aparte:

```sql
USE tienda_dba;

DROP TABLE IF EXISTS pedidos_big;
CREATE TABLE pedidos_big (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    cliente_id  INT UNSIGNED NOT NULL,
    fecha       DATETIME NOT NULL,
    estado      VARCHAR(10) NOT NULL,
    pais        CHAR(2) NOT NULL,
    total       DECIMAL(10,2) NOT NULL,
    email       VARCHAR(120) NOT NULL
) ENGINE=InnoDB;

SET SESSION cte_max_recursion_depth = 1000000;

INSERT INTO pedidos_big (cliente_id, fecha, estado, pais, total, email)
WITH RECURSIVE n AS (SELECT 1 AS i UNION ALL SELECT i + 1 FROM n WHERE i < 200000)
SELECT 1 + (i * 7919) % 20000,                                         -- 20.000 clientes distintos
       '2022-01-01' + INTERVAL (i * 37) % (3 * 365 * 24 * 60) MINUTE,   -- fechas repartidas en 3 años
       ELT(1 + i % 4, 'pendiente', 'pagado', 'enviado', 'cancelado'),
       ELT(1 + i % 5, 'ES', 'MX', 'CO', 'PE', 'CL'),
       ROUND(5 + (i * 13) % 2000 + (i % 100) / 100, 2),
       CONCAT('cliente', 1 + (i * 7919) % 20000, '@mail.com')
FROM n;

ANALYZE TABLE pedidos_big;    -- Actualiza las estadísticas que usa el optimizador
SELECT COUNT(*) FROM pedidos_big;   -- 200000
```

---

## 1. ¿Qué es un índice? (la analogía del libro)

Para encontrar "Deadlock" en un libro de 800 páginas puedes:
* **Leer todas las páginas** → en SQL: *full table scan* (`type: ALL` en `EXPLAIN`).
* **Ir al índice alfabético** del final, que te dice "página 412" → en SQL: **búsqueda por índice**.

Un índice es una **estructura ordenada aparte** con los valores de una o varias columnas y un puntero a cada fila. InnoDB usa **árboles B+ (B-Tree)**:

```
                        [ 5000 | 10000 | 15000 ]               ← raíz
                       /        |        |        \
        [1000|2000|...]  [6000|7000|...]  [...]  [16000|...]   ← nodos intermedios
            /  |  \                                    \
     [hojas ordenadas: valor → clave primaria de la fila]       ← hojas (enlazadas entre sí)
```
* Encontrar un valor entre millones requiere solo **3-4 saltos** (la altura del árbol).
* Como las hojas están **ordenadas y enlazadas**, también sirve para rangos (`BETWEEN`, `>`, `<`), `ORDER BY` y `MIN`/`MAX`.

### Índice agrupado (*clustered*) y secundarios en InnoDB
* La **clave primaria ES la tabla**: las filas se guardan físicamente ordenadas por la PK, dentro del propio árbol (*clustered index*).
* Cada **índice secundario** guarda el valor indexado + **la clave primaria**. Para obtener el resto de columnas hace un segundo salto a la PK (*bookmark lookup*).
* Consecuencias:
  * Usa PKs **cortas** (`INT`/`BIGINT`), porque se copian en todos los índices secundarios.
  * Usa PKs **crecientes** (`AUTO_INCREMENT`). Un UUID aleatorio como PK hace que cada `INSERT` caiga en un sitio al azar del árbol (fragmentación, inserciones lentas). Si necesitas UUID, usa UUID v7 / ULID (ordenables) o guárdalo en `BINARY(16)` con `UUID_TO_BIN(uuid, 1)`.

### El coste de los índices
Los índices aceleran las lecturas pero **cada `INSERT`, `UPDATE` y `DELETE` debe actualizar todos los índices** de la tabla, y ocupan disco y memoria. Una tabla con 15 índices escribe muy lento. **Indexa lo que realmente se consulta.**

---

## 2. Crear, ver y borrar índices

```sql
-- Índice simple
CREATE INDEX idx_cliente ON pedidos_big (cliente_id);

-- Índice compuesto (varias columnas, el ORDEN importa — ver apartado 4)
CREATE INDEX idx_estado_fecha ON pedidos_big (estado, fecha);

-- Índice único (además impide duplicados)
-- CREATE UNIQUE INDEX uq_email ON clientes (email);

-- Índice por prefijo (solo los primeros 20 caracteres de un texto largo)
CREATE INDEX idx_email_pref ON pedidos_big (email(20));

-- Con ALTER TABLE (varios a la vez = una sola reconstrucción)
ALTER TABLE pedidos_big ADD INDEX idx_pais (pais), ADD INDEX idx_total (total);

-- Ver los índices y su cardinalidad (número estimado de valores distintos)
SHOW INDEX FROM pedidos_big;

-- Borrar
DROP INDEX idx_total ON pedidos_big;          -- o: ALTER TABLE pedidos_big DROP INDEX idx_total;
```

### Tipos de índice
| Tipo | Sintaxis | Uso |
| :--- | :--- | :--- |
| `PRIMARY KEY` | `PRIMARY KEY (id)` | Identificador; es el índice agrupado. |
| `UNIQUE` | `UNIQUE (email)` | Búsqueda + garantía de no duplicados. |
| `INDEX` / `KEY` | `INDEX (cliente_id)` | Búsqueda normal. |
| `FULLTEXT` | `FULLTEXT (nombre, descripcion)` | Búsqueda de palabras en textos (apartado 7). |
| `SPATIAL` | `SPATIAL (ubicacion)` | Datos geográficos. |
| Funcional (MySQL 8.0.13+) | `INDEX ((LOWER(email)))` | Indexa el resultado de una expresión. |
| Descendente (MySQL 8+) | `INDEX (fecha DESC)` | Ordenaciones mixtas `ORDER BY a ASC, b DESC`. |
| Invisible (MySQL 8+) | `ALTER TABLE t ALTER INDEX idx INVISIBLE` | Probar si se puede borrar un índice sin borrarlo (apartado 9). |

---

## 3. `EXPLAIN`: cómo piensa ejecutar MySQL tu consulta

Antepón `EXPLAIN` a cualquier `SELECT`, `UPDATE` o `DELETE`. **No ejecuta la consulta**, muestra el plan:

```sql
EXPLAIN SELECT * FROM pedidos_big WHERE email = 'cliente500@mail.com';
```

### Las columnas que importan
| Columna | Qué significa | Qué buscar |
| :--- | :--- | :--- |
| `type` | Cómo accede a la tabla (de mejor a peor ↓) | |
| | `system` / `const` | 1 fila por PK o UNIQUE. Perfecto. |
| | `eq_ref` | 1 fila por cada fila de la tabla anterior en un JOIN (por PK/UNIQUE). Excelente. |
| | `ref` | Varias filas por un índice no único. Bien. |
| | `range` | Rango de un índice (`BETWEEN`, `>`, `IN`). Bien. |
| | `index` | Recorre **todo el índice**. Regular. |
| | **`ALL`** | **Recorre toda la tabla.** 🚨 En tablas grandes es el problema. |
| `possible_keys` | Índices que podría usar | |
| `key` | Índice que **eligió** | `NULL` = ninguno |
| `key_len` | Bytes del índice usados | Indica **cuántas columnas** de un índice compuesto aprovecha |
| `rows` | Filas que **estima** examinar | Cuanto menor, mejor |
| `filtered` | % de esas filas que pasarán el `WHERE` | |
| `Extra` | Detalles adicionales | ↓ |
| | `Using index` | ✅ **Índice cubriente**: no necesita leer la tabla. |
| | `Using where` | Filtra filas después de leerlas. |
| | `Using index condition` | ✅ Filtra dentro del índice (ICP). |
| | `Using temporary` | ⚠️ Crea una tabla temporal (típico de `GROUP BY`/`DISTINCT` sin índice). |
| | `Using filesort` | ⚠️ Ordena aparte (el `ORDER BY` no aprovecha un índice). |

### Antes y después de un índice
```sql
-- 1) Sin índice: le pedimos a MySQL que IGNORE el índice por prefijo que creamos en el apartado 2
EXPLAIN SELECT id, total FROM pedidos_big IGNORE INDEX (idx_email_pref)
WHERE email = 'cliente500@mail.com';
-- type: ALL | key: NULL | rows: ~200000  🚨 lee la tabla entera para encontrar 10 filas

-- 2) Con el índice idx_email_pref (primeros 20 caracteres del email)
EXPLAIN SELECT id, total FROM pedidos_big WHERE email = 'cliente500@mail.com';
-- type: ref | key: idx_email_pref | rows: 10  ✅ va directo a las 10 filas
```

### Pistas al optimizador (*index hints*)
Normalmente el optimizador elige bien. Cuando no, o para comparar planes como arriba, puedes guiarlo:
```text
SELECT ... FROM pedidos_big USE INDEX (idx_fecha)    WHERE ...;   -- "considera solo este índice"
SELECT ... FROM pedidos_big FORCE INDEX (idx_fecha)  WHERE ...;   -- "úsalo aunque creas que un full scan es mejor"
SELECT ... FROM pedidos_big IGNORE INDEX (idx_email) WHERE ...;   -- "no uses este índice"
```
> ⚠️ Una pista que hoy acelera la consulta puede frenarla mañana, cuando cambien los datos. Úsalas como último recurso y documenta por qué.

### `EXPLAIN ANALYZE` (MySQL 8.0.18+): ejecuta y mide el tiempo REAL
```sql
EXPLAIN ANALYZE
SELECT estado, COUNT(*), SUM(total)
FROM pedidos_big
WHERE fecha >= '2024-01-01'
GROUP BY estado;
```
```
-> Group aggregate: count(0), sum(pedidos_big.total)  (cost=35451 rows=3) (actual time=171..341 rows=4 loops=1)
    -> Filter: (pedidos_big.fecha >= TIMESTAMP'2024-01-01 00:00:00')  (cost=20153 rows=66395) (actual time=89.8..328 rows=57946 loops=1)
        -> Index scan on pedidos_big using idx_estado_fecha  (cost=20153 rows=199206) (actual time=4.43..315 rows=200000 loops=1)
```
*(Tus tiempos y costes serán distintos según tu equipo.)* Se lee **de dentro hacia fuera**: recorre los 200.000 registros del índice `idx_estado_fecha` (porque así le sale el `GROUP BY estado` ya ordenado), filtra hasta 57.946 filas de 2024 en adelante y agrupa en 4 resultados. Recorrer 200.000 entradas para quedarse con un 29 % es mejorable: en el apartado 4 verás por qué el orden de las columnas del índice importa.
* `actual time=inicio..fin` en milisegundos, `rows` reales, `loops` = cuántas veces se ejecutó ese paso.
* `rows=66395` (estimado) vs `rows=57946` (real): una estimación razonable.
* Compara `rows` estimadas vs reales: una diferencia enorme indica **estadísticas desactualizadas** → `ANALYZE TABLE`.
* ⚠️ `EXPLAIN ANALYZE` **ejecuta** la consulta: cuidado con consultas muy pesadas en producción.

Otros formatos: `EXPLAIN FORMAT=TREE ...` (árbol) y `EXPLAIN FORMAT=JSON ...` (con costes detallados). MySQL Workbench y DBeaver lo dibujan gráficamente (*Visual Explain*).

> 🐘 **PostgreSQL:** `EXPLAIN (ANALYZE, BUFFERS) SELECT ...` (muy completo). **SQL Server:** "Plan de ejecución real" en SSMS.

---

## 4. Índices compuestos: la regla del prefijo izquierdo

Un índice `(estado, fecha)` es como una guía telefónica ordenada por **apellido y luego nombre**: puedes buscar por apellido, o por apellido + nombre, pero **no** solo por nombre.

```sql
-- Índice: idx_estado_fecha (estado, fecha)

EXPLAIN SELECT * FROM pedidos_big WHERE estado = 'pagado';                              -- ✅ usa (estado)
EXPLAIN SELECT * FROM pedidos_big WHERE estado = 'pagado' AND fecha >= '2024-06-01';    -- ✅ usa (estado, fecha)
EXPLAIN SELECT * FROM pedidos_big WHERE fecha >= '2024-06-01';                          -- ❌ no puede usarlo (falta "estado")
```

### Cómo ordenar las columnas de un índice compuesto
1. Primero las columnas con **igualdad** (`=`, `IN`).
2. Después la columna de **rango** (`>`, `<`, `BETWEEN`, `LIKE 'abc%'`). **Tras un rango, las columnas siguientes del índice ya no sirven para filtrar.**
3. Si además hay `ORDER BY`, que coincida con el orden del índice tras las igualdades.

```sql
-- Consulta frecuente:
-- SELECT * FROM pedidos_big WHERE pais = 'MX' AND estado = 'pagado' AND fecha >= '2024-01-01' ORDER BY fecha;

-- Índice ideal: igualdades (pais, estado) + rango/orden (fecha)
CREATE INDEX idx_pais_estado_fecha ON pedidos_big (pais, estado, fecha);

EXPLAIN SELECT * FROM pedidos_big
WHERE pais = 'MX' AND estado = 'pagado' AND fecha >= '2024-01-01'
ORDER BY fecha;
-- type: range | key: idx_pais_estado_fecha | Extra: Using index condition (sin "Using filesort": ya sale ordenado)
-- Este índice ADEMÁS cubre las consultas que filtran solo por (pais) o por (pais, estado).
-- → El índice idx_pais (pais) que creamos antes ahora es REDUNDANTE: bórralo.
DROP INDEX idx_pais ON pedidos_big;
```

### Índices cubrientes (*covering index*)
Si el índice contiene **todas** las columnas que la consulta necesita, MySQL no toca la tabla (`Extra: Using index`). Es la optimización más potente para consultas muy frecuentes:
```sql
-- Total facturado por un cliente
EXPLAIN SELECT SUM(total) FROM pedidos_big WHERE cliente_id = 123;
-- key: idx_cliente, pero tiene que ir a la tabla a leer "total" por cada fila

CREATE INDEX idx_cliente_total ON pedidos_big (cliente_id, total);
EXPLAIN SELECT SUM(total) FROM pedidos_big WHERE cliente_id = 123;
-- key: idx_cliente_total | Extra: Using index ✅ (y idx_cliente se vuelve redundante)
```

---

## 5. Por qué MySQL NO usa tu índice (antipatrones)

| ❌ Escrito así (no usa índice) | ✅ Escríbelo así |
| :--- | :--- |
| `WHERE YEAR(fecha) = 2024` | `WHERE fecha >= '2024-01-01' AND fecha < '2025-01-01'` |
| `WHERE DATE(fecha) = '2024-06-14'` | `WHERE fecha >= '2024-06-14' AND fecha < '2024-06-15'` |
| `WHERE total * 1.21 > 100` | `WHERE total > 100 / 1.21` |
| `WHERE LOWER(email) = 'a@b.com'` | Collation `_ci` ya ignora mayúsculas: `WHERE email = 'a@b.com'` (o índice funcional) |
| `WHERE email LIKE '%@gmail.com'` | Comodín al **inicio** = no hay índice posible. Guarda el dominio en otra columna, o usa `FULLTEXT`. |
| `WHERE cliente_id = '123'` si la columna es `VARCHAR` y comparas con número (`= 123`) | Compara con el **mismo tipo** que la columna: la conversión implícita anula el índice. |
| `WHERE estado = 'pagado' OR cliente_id = 5` | A veces usa *index merge*; si no, reescribe como `UNION ALL` de dos consultas. |
| `SELECT *` cuando solo necesitas 3 columnas | Selecciona solo lo necesario: permite índices cubrientes y mueve menos datos. |
| `ORDER BY RAND() LIMIT 1` en tablas grandes | Elige un `id` aleatorio en la aplicación y búscalo por PK. |
| `LIMIT 20 OFFSET 100000` | Paginación por clave: `WHERE id > :ultimo ORDER BY id LIMIT 20`. |
| Subconsulta correlacionada sobre una tabla enorme | `JOIN` con tabla derivada o función de ventana (Módulo 5). |

```sql
-- Compruébalo:
EXPLAIN SELECT COUNT(*) FROM pedidos_big WHERE YEAR(fecha) = 2024;                         -- type: index/ALL
CREATE INDEX idx_fecha ON pedidos_big (fecha);
EXPLAIN SELECT COUNT(*) FROM pedidos_big WHERE fecha >= '2024-01-01' AND fecha < '2025-01-01'; -- type: range ✅
-- (MySQL puede elegir idx_fecha o incluso idx_estado_fecha con "Using index for skip scan" (MySQL 8.0.13+):
--  salta por los 4 valores de "estado" y busca el rango de fechas dentro de cada uno)
```

### Selectividad: cuándo un índice no compensa
Si el `WHERE` devuelve una gran parte de la tabla (por ejemplo `estado = 'pagado'`, un 25 %), al optimizador le sale más barato leer la tabla entera que saltar del índice a la tabla 50.000 veces. Los índices brillan con condiciones **selectivas** (pocas filas). Por eso un índice sobre una columna booleana o con 3-4 valores **solo** suele ser inútil (salvo como primera columna de un índice compuesto).

```sql
-- Cardinalidad (valores distintos) de cada columna: cuanto más alta, más selectivo
SELECT COUNT(DISTINCT estado) AS estados, COUNT(DISTINCT pais) AS paises,
       COUNT(DISTINCT cliente_id) AS clientes, COUNT(DISTINCT email) AS emails
FROM pedidos_big;
```

---

## 6. Optimizar `JOIN`, `GROUP BY` y `ORDER BY`

* **JOIN:** indexa las columnas del `ON` en la tabla "de destino". Las claves foráneas de InnoDB crean índice automáticamente en la columna hija.
* **ORDER BY ... LIMIT:** si un índice coincide con el orden, MySQL lee las primeras N entradas del índice y para (sin `filesort`):
  ```sql
  EXPLAIN SELECT id, fecha, total FROM pedidos_big ORDER BY fecha DESC LIMIT 10;   -- usa idx_fecha, sin filesort
  ```
* **GROUP BY:** un índice que empiece por las columnas agrupadas evita `Using temporary`:
  ```sql
  EXPLAIN SELECT pais, estado, COUNT(*) FROM pedidos_big GROUP BY pais, estado;   -- usa idx_pais_estado_fecha
  ```
* **COUNT(*) en InnoDB** no está precalculado: recorre el índice más pequeño. Para contadores en tiempo real de tablas gigantes, mantén una tabla de resumen.

---

## 7. Búsqueda de texto: `FULLTEXT`

`LIKE '%texto%'` no puede usar índices. Para buscar palabras en descripciones o artículos:
```sql
ALTER TABLE productos ADD FULLTEXT INDEX ft_nombre (nombre);

-- Modo natural: relevancia
SELECT nombre, MATCH(nombre) AGAINST ('teclado gaming') AS relevancia
FROM productos
WHERE MATCH(nombre) AGAINST ('teclado gaming')
ORDER BY relevancia DESC;

-- Modo booleano: + obligatorio, - excluido, * prefijo
SELECT nombre FROM productos
WHERE MATCH(nombre) AGAINST ('+tarjeta -RX' IN BOOLEAN MODE);
```
> Ignora palabras de menos de 3 caracteres (`innodb_ft_min_token_size`) y las *stopwords*. Para buscadores serios (tolerancia a errores, sinónimos, facetas) se usa un motor externo: Elasticsearch, OpenSearch, Meilisearch o Typesense.

---

## 8. Encontrar las consultas lentas

### A) *Slow query log*: registra las consultas que superan un tiempo
```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;                       -- Más de 1 segundo
SET GLOBAL log_queries_not_using_indexes = ON;        -- Opcional (genera mucho ruido)
SHOW VARIABLES LIKE 'slow_query_log_file';            -- Dónde se guarda
```
Para que sea permanente, en `my.cnf` / `my.ini`:
```ini
[mysqld]
slow_query_log      = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time     = 1
```
Analiza el log con **`mysqldumpslow`** (incluido con MySQL) o **`pt-query-digest`** (Percona Toolkit), que agrupan consultas iguales y las ordenan por tiempo total:
```bash
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log      # Top 10 por tiempo total
pt-query-digest /var/log/mysql/slow.log > informe.txt
```

### B) `performance_schema` y `sys`: estadísticas sin activar nada
```sql
-- Top 10 consultas que más tiempo TOTAL consumen desde el arranque del servidor
SELECT query, db, exec_count, total_latency, avg_latency, rows_examined_avg
FROM sys.statement_analysis
ORDER BY total_latency DESC
LIMIT 10;

-- Consultas que hacen full table scan
SELECT query, db, exec_count, total_latency, no_index_used_count
FROM sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC
LIMIT 10;

-- Consultas que usan tablas temporales en disco u ordenaciones costosas
SELECT * FROM sys.statements_with_temp_tables LIMIT 10;
SELECT * FROM sys.statements_with_sorting LIMIT 10;
```

### C) Índices que sobran
```sql
-- Índices que no se han usado desde el último reinicio (¡revisa con un periodo largo!)
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'tienda_dba';

-- Índices redundantes (cubiertos por otro índice)
SELECT table_name, redundant_index_name, dominant_index_name, sql_drop_index
FROM sys.schema_redundant_indexes
WHERE table_schema = 'tienda_dba';
```

---

## 9. Índices invisibles: borrar sin miedo

Antes de borrar un índice en producción, hazlo **invisible**: el optimizador deja de usarlo, pero se sigue manteniendo. Si algo se pone lento, lo vuelves visible al instante (sin reconstruirlo):
```sql
ALTER TABLE pedidos_big ALTER INDEX idx_email_pref INVISIBLE;
-- ... observa el sistema unos días ...
ALTER TABLE pedidos_big ALTER INDEX idx_email_pref VISIBLE;   -- marcha atrás
-- o, si nadie lo echó de menos:
DROP INDEX idx_email_pref ON pedidos_big;
```

---

## 10. Estadísticas y mantenimiento de tablas

```sql
ANALYZE TABLE pedidos_big;     -- Recalcula estadísticas (cardinalidad) → mejores planes. Rápido.
OPTIMIZE TABLE pedidos_big;    -- Reconstruye la tabla y recupera espacio tras borrados masivos. Lento.
CHECK TABLE pedidos_big;       -- Comprueba errores de integridad.

-- Histogramas (MySQL 8): estadísticas de distribución para columnas SIN índice
ANALYZE TABLE pedidos_big UPDATE HISTOGRAM ON estado, pais WITH 16 BUCKETS;
```

### Cuánto espacio "libre" (fragmentado) tiene cada tabla
```sql
SELECT table_name,
       ROUND(data_length / 1024 / 1024, 1)  AS datos_mb,
       ROUND(index_length / 1024 / 1024, 1) AS indices_mb,
       ROUND(data_free / 1024 / 1024, 1)    AS libre_mb
FROM information_schema.tables
WHERE table_schema = 'tienda_dba'
ORDER BY data_length + index_length DESC;
```

---

## 11. Particionado (tablas enormes)

El particionado divide **físicamente** una tabla en trozos según una columna, aunque se sigue consultando como una sola. Útil en tablas de cientos de millones de filas con un patrón claro (por ejemplo, logs por mes):
```sql
CREATE TABLE logs_accesos (
    id       BIGINT UNSIGNED AUTO_INCREMENT,
    fecha    DATETIME NOT NULL,
    usuario  INT UNSIGNED,
    url      VARCHAR(255),
    PRIMARY KEY (id, fecha)                  -- La columna de partición DEBE estar en la PK/UNIQUE
)
PARTITION BY RANGE (YEAR(fecha)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);

-- Partition pruning: solo lee la partición p2024
EXPLAIN SELECT COUNT(*) FROM logs_accesos WHERE fecha >= '2024-03-01' AND fecha < '2024-04-01';  -- partitions: p2024

-- La gran ventaja para un DBA: borrar un año entero de logs es INSTANTÁNEO (no un DELETE de millones de filas)
ALTER TABLE logs_accesos DROP PARTITION p2023;
```
> ⚠️ El particionado **no** sustituye a los índices y puede empeorar consultas que no filtran por la columna de partición. Úsalo para **gestionar datos** (archivar/borrar por rangos), no como primera herramienta de rendimiento.

---

## 12. Checklist del DBA ante una consulta lenta

1. **Reproduce** y mide la consulta exacta (con sus valores reales).
2. `EXPLAIN` → ¿`type: ALL`? ¿`key: NULL`? ¿`Using filesort` / `Using temporary`? ¿`rows` enorme?
3. ¿El `WHERE` aplica funciones o conversiones a columnas? → reescríbelo.
4. ¿Falta un índice? Diseña el **compuesto** correcto (igualdades → rango → orden) y, si es muy frecuente, **cubriente**.
5. ¿Las estimaciones (`rows`) difieren mucho de la realidad? → `ANALYZE TABLE`.
6. `EXPLAIN ANALYZE` para confirmar la mejora con tiempos reales.
7. Revisa que el nuevo índice no duplique otro existente y que la tabla no tenga ya demasiados.
8. Si nada basta: caché en la aplicación, tabla de resumen (Módulo 6), réplicas de lectura (Módulo 10) o revisar el diseño.

---

## 🏋️ Ejercicios del Módulo 8

> Usa la tabla `pedidos_big` del apartado 0. Si la recreas, vuelve a crear los índices que necesites.

1. Escribe una consulta que cuente los pedidos de **marzo de 2024** de forma que pueda usar `idx_fecha`, y comprueba con `EXPLAIN` que `type` es `range`.
2. Diseña **un solo índice** para que esta consulta no lea la tabla (índice cubriente) y compruébalo con `EXPLAIN`:
   `SELECT fecha, total FROM pedidos_big WHERE cliente_id = 77 ORDER BY fecha DESC;`
3. Esta consulta no usa índices. Explica por qué y reescríbela: `SELECT * FROM pedidos_big WHERE DATE(fecha) = '2024-05-10';`
4. Usa `sys.schema_redundant_indexes` para listar los índices redundantes de `pedidos_big` después de los ejercicios anteriores.
5. Haz invisible `idx_estado_fecha`, comprueba con `EXPLAIN` qué índice usa ahora `WHERE estado = 'pagado' AND fecha >= '2024-06-01'` y vuelve a hacerlo visible.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
EXPLAIN SELECT COUNT(*) FROM pedidos_big
WHERE fecha >= '2024-03-01' AND fecha < '2024-04-01';
-- type: range | key: idx_fecha | Extra: Using where; Using index

-- 2: igualdad (cliente_id) + orden (fecha) + columna leída (total)
CREATE INDEX idx_cli_fecha_total ON pedidos_big (cliente_id, fecha, total);
EXPLAIN SELECT fecha, total FROM pedidos_big WHERE cliente_id = 77 ORDER BY fecha DESC;
-- key: idx_cli_fecha_total | Extra: Backward index scan; Using index  (sin filesort y sin leer la tabla)

-- 3: DATE(fecha) aplica una función a la columna → no puede usar el índice ordenado por fecha
SELECT * FROM pedidos_big
WHERE fecha >= '2024-05-10' AND fecha < '2024-05-11';

-- 4
SELECT redundant_index_name, redundant_index_columns, dominant_index_name, sql_drop_index
FROM sys.schema_redundant_indexes
WHERE table_schema = 'tienda_dba' AND table_name = 'pedidos_big';
-- Ej.: idx_cliente y idx_cliente_total quedan dominados por idx_cli_fecha_total

-- 5
ALTER TABLE pedidos_big ALTER INDEX idx_estado_fecha INVISIBLE;
EXPLAIN SELECT * FROM pedidos_big WHERE estado = 'pagado' AND fecha >= '2024-06-01';
-- Ahora usa idx_fecha (o hace un full scan): menos eficiente
ALTER TABLE pedidos_big ALTER INDEX idx_estado_fecha VISIBLE;
```
</details>

---

👉 **Siguiente paso:** **[Módulo 9: Usuarios, Permisos, Roles y Seguridad](./09_USUARIOS_PERMISOS_ROLES_Y_SEGURIDAD.md)**
