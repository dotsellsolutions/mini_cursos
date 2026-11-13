# Módulo 1: Fundamentos — Bases de Datos, Tablas y Tipos de Datos (DDL)

En este módulo aprendes a **construir** la estructura: conectarte al servidor, crear bases de datos, elegir el tipo de dato correcto para cada columna, crear tablas con sus reglas (restricciones) y modificarlas después sin perder datos.

---

## 1. Conectarte al servidor desde la consola

Un DBA trabaja casi siempre por **SSH en la consola**, sin interfaz gráfica. Domina el cliente `mysql`:

```bash
# Conexión local pidiendo la contraseña
mysql -u root -p

# Conexión a otro servidor y puerto, directamente a una base de datos
mysql -h 192.168.1.50 -P 3306 -u admin -p tienda_dba

# Ejecutar una sola consulta sin entrar a la consola
mysql -u root -p -e "SHOW DATABASES;"

# Ejecutar un archivo .sql completo
mysql -u root -p tienda_dba < script.sql
```
> 🪟 **En WAMP (Windows)** el ejecutable está en `C:\wamp64\bin\mysql\mysqlX.Y.Z\bin\mysql.exe`. Añade esa carpeta al `PATH` de Windows para poder escribir solo `mysql`.
>
> 🐬 **MariaDB:** el cliente se llama `mariadb` (aunque `mysql` sigue funcionando como alias). En WAMP, MariaDB suele escuchar en el puerto **3307** y MySQL en el **3306**.

### Comandos útiles dentro de la consola `mysql>`
| Comando | ¿Qué hace? |
| :--- | :--- |
| `SELECT VERSION();` | Versión del servidor. |
| `STATUS;` o `\s` | Usuario conectado, base de datos actual, versión, *charset*, tiempo encendido. |
| `SELECT DATABASE();` | En qué base de datos estás. |
| `SELECT USER(), CURRENT_USER();` | Con qué usuario te conectaste y con cuál te reconoció el servidor. |
| `SOURCE C:/ruta/script.sql;` | Ejecuta un archivo `.sql` desde dentro de la consola. |
| `\G` al final (en lugar de `;`) | Muestra cada fila **en vertical**. Imprescindible para tablas con muchas columnas. |
| `\c` | Cancela la consulta que estás escribiendo. |
| `exit;` o `\q` | Salir. |
| `HELP SELECT;` | Ayuda integrada de cualquier comando. |

### Comentarios en SQL
```sql
-- Comentario de una línea (¡ojo: lleva un espacio después de los dos guiones!)
#  Comentario de una línea (solo MySQL/MariaDB)
/* Comentario
   de varias líneas */
```

---

## 2. Bases de datos: crear, ver, usar y borrar

```sql
-- Ver todas las bases de datos del servidor
SHOW DATABASES;

-- Crear una base de datos (IF NOT EXISTS evita el error si ya existe)
CREATE DATABASE IF NOT EXISTS tienda_dba
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- Entrar en ella (todas las consultas siguientes se ejecutan aquí)
USE tienda_dba;

-- Ver cómo fue creada
SHOW CREATE DATABASE tienda_dba;

-- Cambiar su juego de caracteres por defecto (afecta a las tablas NUEVAS, no a las existentes)
ALTER DATABASE tienda_dba CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

-- Borrar la base de datos COMPLETA con todas sus tablas y datos (¡irreversible!)
DROP DATABASE IF EXISTS tienda_vieja;
```
> 🐘 **PostgreSQL:** no existe `USE`; te conectas a la base con `\c tienda_dba` en `psql`. Las bases de datos se listan con `\l`.

### *Charset* y *Collation*: el detalle que causa los "Ã±" y los "????"
* **Charset (juego de caracteres):** qué símbolos se pueden guardar. Usa **siempre `utf8mb4`**.
  * ⚠️ El antiguo `utf8` de MySQL (`utf8mb3`) **no** guarda emojis 😀 ni algunos caracteres asiáticos. `utf8mb4` es el UTF-8 completo.
* **Collation (intercalación):** cómo se **comparan y ordenan** los textos.
  * `_ci` = *case insensitive*: `'MADRID' = 'madrid'` es verdadero.
  * `_ai` = *accent insensitive*: `'Martín' = 'Martin'` es verdadero.
  * `_bin` / `_cs` = distingue mayúsculas (útil para códigos o contraseñas de sistemas externos).
  * `utf8mb4_0900_ai_ci` es el valor por defecto en MySQL 8+; `utf8mb4_unicode_ci` es compatible con MySQL 5.7 y MariaDB.

```sql
-- Ver el charset y collation del servidor y de la conexión
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

-- Ver collations disponibles para utf8mb4
SHOW COLLATION WHERE Charset = 'utf8mb4' AND Collation LIKE '%spanish%';
```

---

## 3. Tipos de datos: elige bien desde el principio

Elegir mal un tipo de dato es uno de los errores más caros: desperdicia espacio en disco y memoria, hace los índices más lentos y a veces **corrompe datos** (como guardar dinero con `FLOAT`).

### A) Números enteros
| Tipo | Bytes | Rango con signo | Rango `UNSIGNED` | Uso típico |
| :--- | :---: | :--- | :--- | :--- |
| `TINYINT` | 1 | -128 a 127 | 0 a 255 | Edad, estrellas (1-5), flags |
| `SMALLINT` | 2 | -32.768 a 32.767 | 0 a 65.535 | Cantidades pequeñas, años |
| `MEDIUMINT` | 3 | ±8,3 millones | 0 a 16,7 millones | Contadores medianos |
| `INT` | 4 | ±2.147 millones | 0 a 4.294 millones | IDs de la mayoría de tablas |
| `BIGINT` | 8 | ±9,2 trillones | 0 a 18,4 trillones | IDs de tablas gigantes (logs, eventos) |

> 💡 **`UNSIGNED`** (sin signo) duplica el rango positivo. Úsalo en IDs, stock y cantidades que nunca pueden ser negativas. *(En `INT(11)`, el `11` **no** es el tamaño: es un ancho de visualización obsoleto. Ignóralo.)*

### B) Números con decimales
| Tipo | Precisión | Uso |
| :--- | :--- | :--- |
| `DECIMAL(p, s)` | **Exacta.** `p` dígitos totales, `s` decimales. `DECIMAL(10,2)` → hasta 99.999.999,99 | **Dinero, siempre.** Precios, salarios, saldos. |
| `FLOAT` / `DOUBLE` | **Aproximada** (coma flotante) | Medidas científicas, coordenadas GPS, estadísticas. |

```sql
-- Por qué NUNCA usar FLOAT para dinero:
SELECT CAST(0.1 AS FLOAT) + CAST(0.2 AS FLOAT);       -- 0.30000000447034836  ❌
SELECT CAST(0.1 AS DECIMAL(10,2)) + CAST(0.2 AS DECIMAL(10,2)); -- 0.30 ✅
```

### C) Texto
| Tipo | Máximo | Uso |
| :--- | :--- | :--- |
| `CHAR(n)` | 255 caracteres, **longitud fija** | Códigos de longitud fija: país (`ES`), moneda (`EUR`), hashes de tamaño fijo. |
| `VARCHAR(n)` | 65.535 bytes (compartidos por toda la fila) | Casi todo: nombres, emails, títulos. |
| `TINYTEXT` / `TEXT` / `MEDIUMTEXT` / `LONGTEXT` | 255 B / 64 KB / 16 MB / 4 GB | Descripciones, artículos, HTML. No admiten `DEFAULT` (antes de MySQL 8.0.13) y se indexan solo por prefijo. |
| `ENUM('a','b','c')` | Una opción de la lista | Estados fijos (`'pendiente','pagado'`). Añadir opciones requiere `ALTER TABLE`. |
| `SET('a','b')` | Varias opciones de la lista | Rara vez recomendable; mejor una tabla aparte. |

### D) Fechas y horas
| Tipo | Formato | Rango | Notas |
| :--- | :--- | :--- | :--- |
| `DATE` | `2025-02-14` | 1000-01-01 a 9999-12-31 | Solo fecha. |
| `TIME` | `13:45:00` | ±838 horas | Hora o duración. |
| `DATETIME` | `2025-02-14 13:45:00` | Año 1000 a 9999 | **No** cambia con la zona horaria. |
| `TIMESTAMP` | `2025-02-14 13:45:00` | 1970 a **2038** | Se guarda en UTC y se convierte a la zona horaria de la sesión. Cuidado con el límite de 2038. |
| `YEAR` | `2025` | 1901 a 2155 | Solo año. |

> Para fechas de negocio (fecha de pedido, nacimiento) usa `DATE`/`DATETIME`. Para "cuándo se creó o modificó esta fila" es común `TIMESTAMP DEFAULT CURRENT_TIMESTAMP`. Añade `(3)` o `(6)` para milisegundos/microsegundos: `DATETIME(3)`.

### E) Otros tipos
| Tipo | Uso |
| :--- | :--- |
| `BOOLEAN` / `BOOL` | En MySQL es un alias de `TINYINT(1)`: guarda `0` o `1`. |
| `JSON` | Documentos JSON validados (Módulo 5). |
| `BINARY(n)` / `VARBINARY(n)` / `BLOB` | Datos binarios (UUID en binario, archivos pequeños). Guardar archivos grandes en la BD casi nunca es buena idea. |
| `GEOMETRY`, `POINT`... | Datos geográficos (mapas, distancias). |

> 🐘 **PostgreSQL:** usa `SERIAL`/`GENERATED AS IDENTITY` en vez de `AUTO_INCREMENT`, `BOOLEAN` real, `TEXT` sin penalización, `TIMESTAMPTZ` para fechas con zona horaria, y no existe `UNSIGNED`.

---

## 4. Crear tablas con restricciones (*constraints*)

Las **restricciones** son reglas que la base de datos hace cumplir **siempre**, aunque el programa tenga un bug. Son tu última línea de defensa para la calidad de los datos.

```sql
CREATE TABLE IF NOT EXISTS proveedores (
    -- PRIMARY KEY: identifica cada fila; implica NOT NULL + UNIQUE.
    -- AUTO_INCREMENT: el servidor asigna 1, 2, 3...
    id            INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,

    -- NOT NULL: obligatorio
    nombre        VARCHAR(100) NOT NULL,

    -- UNIQUE: no se puede repetir (pero sí admite varios NULL)
    cif           CHAR(9) UNIQUE,
    email         VARCHAR(120) NOT NULL,

    -- DEFAULT: valor si no se envía ninguno
    pais          CHAR(2) NOT NULL DEFAULT 'ES',
    activo        BOOLEAN NOT NULL DEFAULT TRUE,

    -- CHECK: condición que debe cumplir el valor (MySQL 8.0.16+, MariaDB 10.2+)
    dias_entrega  TINYINT UNSIGNED NOT NULL DEFAULT 3 CHECK (dias_entrega BETWEEN 1 AND 60),

    -- Fechas automáticas de creación y modificación
    creado_en       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    actualizado_en  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Restricciones con nombre (más fáciles de borrar o identificar en los errores)
    CONSTRAINT uq_proveedor_email UNIQUE (email),
    CONSTRAINT chk_email CHECK (email LIKE '%_@_%._%')
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Proveedores de productos';
```

### Claves foráneas (`FOREIGN KEY`) y sus acciones
Una clave foránea garantiza que el valor **exista en otra tabla** (integridad referencial):

```sql
CREATE TABLE productos_proveedores (
    producto_id   INT UNSIGNED NOT NULL,
    proveedor_id  INT UNSIGNED NOT NULL,
    precio_compra DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (producto_id, proveedor_id),               -- Clave primaria COMPUESTA
    CONSTRAINT fk_pp_producto  FOREIGN KEY (producto_id)  REFERENCES productos(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    CONSTRAINT fk_pp_proveedor FOREIGN KEY (proveedor_id) REFERENCES proveedores(id)
        ON DELETE RESTRICT
);
```
| Acción (`ON DELETE` / `ON UPDATE`) | Si se borra/modifica la fila "padre"... |
| :--- | :--- |
| `RESTRICT` / `NO ACTION` (por defecto) | **Se impide** la operación mientras existan hijos. |
| `CASCADE` | Se borran/actualizan también los hijos. Potente y peligroso. |
| `SET NULL` | Los hijos quedan con la columna en `NULL` (debe ser *nullable*). |
| `SET DEFAULT` | No soportado por InnoDB. |

> ⚠️ **Requisitos de una FK en MySQL:** ambas tablas deben ser **InnoDB**, las columnas deben tener **exactamente el mismo tipo** (`INT UNSIGNED` ↔ `INT UNSIGNED`; un `INT` con signo no vale) y la columna referenciada debe tener índice (la PK lo tiene).

### Crear tablas a partir de otras
```sql
-- Copia la ESTRUCTURA (columnas, índices) sin datos
CREATE TABLE productos_respaldo LIKE productos;

-- Copia los DATOS de una consulta (pero NO copia índices, PK ni FKs)
CREATE TABLE productos_sin_stock AS
SELECT id, sku, nombre FROM productos WHERE stock = 0;

-- Tabla temporal: solo existe en tu sesión y se borra al desconectarte
CREATE TEMPORARY TABLE tmp_top_clientes AS
SELECT cliente_id, SUM(total) AS gastado FROM pedidos GROUP BY cliente_id;
```

---

## 5. Inspeccionar la estructura

```sql
SHOW TABLES;                          -- Tablas de la base de datos actual
SHOW FULL TABLES WHERE Table_type = 'VIEW';  -- Solo las vistas
DESCRIBE productos;                   -- Columnas, tipos, NULL, claves, defaults (alias: DESC productos)
SHOW COLUMNS FROM productos LIKE 'pre%';
SHOW CREATE TABLE productos\G         -- El CREATE TABLE completo (¡el más útil!)
SHOW INDEX FROM productos;            -- Índices de la tabla
SHOW TABLE STATUS LIKE 'productos'\G  -- Motor, filas aprox., tamaño, AUTO_INCREMENT actual
```

### `information_schema`: la base de datos que describe todas las demás
```sql
-- Todas las columnas de tipo DECIMAL de tienda_dba
SELECT table_name, column_name, column_type
FROM information_schema.columns
WHERE table_schema = 'tienda_dba' AND data_type = 'decimal';

-- Todas las claves foráneas: quién apunta a quién
SELECT table_name, column_name, constraint_name,
       referenced_table_name, referenced_column_name
FROM information_schema.key_column_usage
WHERE table_schema = 'tienda_dba' AND referenced_table_name IS NOT NULL;
```

---

## 6. `ALTER TABLE`: modificar tablas existentes

```sql
-- ===== COLUMNAS =====
-- Añadir (AFTER/FIRST controlan la posición)
ALTER TABLE clientes ADD COLUMN telefono VARCHAR(20) NULL AFTER email;

-- Añadir varias a la vez (más rápido: la tabla se reconstruye una sola vez)
ALTER TABLE clientes
    ADD COLUMN fecha_nacimiento DATE NULL,
    ADD COLUMN puntos INT UNSIGNED NOT NULL DEFAULT 0;

-- MODIFY: cambiar el tipo o las propiedades (se reescribe la definición COMPLETA)
-- ⚠️ Para pasar una columna a NOT NULL, primero rellena las filas que tengan NULL,
--    o MySQL (en modo estricto) responde: ERROR 1138 Invalid use of NULL value
UPDATE clientes SET telefono = '' WHERE telefono IS NULL;
ALTER TABLE clientes MODIFY COLUMN telefono VARCHAR(30) NOT NULL DEFAULT '';

-- CHANGE: renombrar Y redefinir a la vez
ALTER TABLE clientes CHANGE COLUMN telefono movil VARCHAR(30) NOT NULL DEFAULT '';

-- RENAME COLUMN: solo renombrar (MySQL 8+, MariaDB 10.5+)
ALTER TABLE clientes RENAME COLUMN movil TO telefono;

-- Cambiar solo el valor por defecto
ALTER TABLE clientes ALTER COLUMN pais SET DEFAULT 'MX';
ALTER TABLE clientes ALTER COLUMN pais DROP DEFAULT;

-- Eliminar columnas (¡se pierden sus datos!)
ALTER TABLE clientes DROP COLUMN fecha_nacimiento, DROP COLUMN puntos, DROP COLUMN telefono;

-- ===== RESTRICCIONES E ÍNDICES =====
ALTER TABLE clientes ADD CONSTRAINT chk_pais CHECK (CHAR_LENGTH(pais) = 2);
ALTER TABLE clientes DROP CHECK chk_pais;          -- MariaDB: DROP CONSTRAINT chk_pais

ALTER TABLE clientes ADD INDEX idx_ciudad (ciudad);
ALTER TABLE clientes DROP INDEX idx_ciudad;

ALTER TABLE productos_proveedores DROP FOREIGN KEY fk_pp_proveedor;
ALTER TABLE productos_proveedores
    ADD CONSTRAINT fk_pp_proveedor FOREIGN KEY (proveedor_id) REFERENCES proveedores(id);

-- ===== LA TABLA EN SÍ =====
ALTER TABLE productos_respaldo RENAME TO productos_backup;   -- o: RENAME TABLE a TO b;
ALTER TABLE productos_backup ENGINE = InnoDB;              -- cambiar motor / reconstruir tabla
ALTER TABLE productos_backup AUTO_INCREMENT = 1000;        -- próximo id que se asignará
ALTER TABLE productos_backup COMMENT = 'Copia de seguridad manual';
```

> 🛡️ **ALTER en producción (tablas con millones de filas):** muchas operaciones bloquean la tabla mientras se reconstruye. Pide expresamente que no bloquee y MySQL fallará en lugar de bloquear si no puede:
> ```sql
> ALTER TABLE pedidos ADD INDEX idx_fecha (fecha), ALGORITHM=INPLACE, LOCK=NONE;
> ALTER TABLE pedidos ADD COLUMN notas VARCHAR(255) NULL, ALGORITHM=INSTANT;  -- MySQL 8.0.29+: casi instantáneo
> ```
> Para cambios muy pesados, los DBA usan herramientas como **`pt-online-schema-change`** (Percona) o **`gh-ost`** (GitHub).

---

## 7. Borrar: `DROP` vs `TRUNCATE` vs `DELETE`

| Comando | ¿Qué borra? | ¿Se puede deshacer con `ROLLBACK`? | `AUTO_INCREMENT` | Velocidad | Dispara *triggers* |
| :--- | :--- | :---: | :--- | :--- | :---: |
| `DELETE FROM t WHERE ...` | Las filas que cumplan la condición | ✅ Sí | Se mantiene | Lenta en tablas grandes (fila a fila) | ✅ |
| `TRUNCATE TABLE t` | **Todas** las filas (la estructura queda) | ❌ No (es DDL) | Se reinicia a 1 | Muy rápida | ❌ |
| `DROP TABLE t` | La tabla **entera** (estructura + datos + índices) | ❌ No | — | Instantánea | ❌ |

```sql
DROP TABLE IF EXISTS productos_sin_stock, productos_backup;
TRUNCATE TABLE tmp_top_clientes;
```
> ⚠️ `TRUNCATE` falla si otra tabla tiene una clave foránea apuntando a esta. Truco (solo en desarrollo, nunca en producción):
> ```sql
> SET FOREIGN_KEY_CHECKS = 0;
> TRUNCATE TABLE detalle_pedidos;
> SET FOREIGN_KEY_CHECKS = 1;
> ```

---

## 8. Motores de almacenamiento

```sql
SHOW ENGINES;
```
| Motor | Transacciones | Claves foráneas | Bloqueo | Uso |
| :--- | :---: | :---: | :--- | :--- |
| **InnoDB** (por defecto) | ✅ | ✅ | Por fila | **Úsalo siempre**, salvo motivo muy concreto. |
| MyISAM | ❌ | ❌ | Por tabla | Legado. Se corrompe con apagones. Migra a InnoDB. |
| MEMORY | ❌ | ❌ | Por tabla | Datos temporales en RAM (se pierden al reiniciar). |
| Aria (MariaDB) | Parcial | ❌ | Por tabla | Tablas internas de MariaDB. |

> 🚨 **Revisa el motor por defecto de tu servidor.** Si creas una tabla sin `ENGINE=...`, se usa `default_storage_engine`. **Algunas instalaciones de WAMP lo traen en `MyISAM`**: tus tablas no tendrán transacciones y las `FOREIGN KEY` se ignorarán **sin dar ningún error**.
> ```sql
> SELECT @@default_storage_engine;                -- Debe ser InnoDB
> SET SESSION default_storage_engine = InnoDB;    -- Arreglo temporal para tu sesión
> ```
> Arreglo permanente en `my.ini` / `my.cnf`, sección `[mysqld]` (y reinicia el servicio): `default-storage-engine = InnoDB`.
>
> Por eso en este curso escribimos `ENGINE=InnoDB` explícitamente en las tablas importantes.

```sql
-- Encontrar tablas que todavía usan MyISAM (tarea típica de DBA)
SELECT table_schema, table_name, engine
FROM information_schema.tables
WHERE engine = 'MyISAM' AND table_schema NOT IN ('mysql','information_schema','performance_schema','sys');
```

---

## 9. `sql_mode`: ¿Qué tan estricto es tu servidor?

`sql_mode` decide si MySQL **rechaza** datos incorrectos o los **"arregla" en silencio**. Es de lo primero que revisa un DBA en un servidor nuevo:

```sql
SELECT @@GLOBAL.sql_mode, @@SESSION.sql_mode;
```

**Valor por defecto en MySQL 8** (el recomendado):
```
ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE,
ERROR_FOR_DIVISION_BY_ZERO, NO_ENGINE_SUBSTITUTION
```
| Modo | Qué hace |
| :--- | :--- |
| `STRICT_TRANS_TABLES` | Un dato inválido (texto en un número, texto demasiado largo, falta un valor obligatorio) da **error** en lugar de guardarse recortado o con `0`. |
| `ONLY_FULL_GROUP_BY` | Prohíbe `GROUP BY` ambiguos (Módulo 3). |
| `NO_ZERO_DATE` / `NO_ZERO_IN_DATE` | Prohíbe fechas `0000-00-00` y `2024-00-10`. |
| `ERROR_FOR_DIVISION_BY_ZERO` | Dividir entre cero en un `INSERT`/`UPDATE` da error. |
| `NO_ENGINE_SUBSTITUTION` | Si pides un motor que no existe, da error en vez de usar otro. |

Mira la diferencia con una columna `VARCHAR(5)`:
```sql
CREATE TEMPORARY TABLE prueba_modo (codigo VARCHAR(5), cantidad INT);

SET SESSION sql_mode = '';                       -- Modo "permisivo" (lo que traen algunos WAMP/XAMPP)
INSERT INTO prueba_modo VALUES ('ABCDEFGH', 'diez');
SELECT * FROM prueba_modo;                       -- 'ABCDE' y 0  😱 datos corruptos, solo un warning

SET SESSION sql_mode = 'STRICT_TRANS_TABLES';
INSERT INTO prueba_modo VALUES ('ABCDEFGH', 10);
-- ERROR 1406: Data too long for column 'codigo'  ✅ el dato malo no entra
```
> ⚠️ **WAMP suele traer el `sql_mode` vacío.** Tu código puede "funcionar" en tu PC y fallar en el servidor de producción (o al revés, guardar datos corruptos en producción). Pon el mismo `sql_mode` en desarrollo y en producción, en el archivo de configuración (`my.ini` / `my.cnf`, sección `[mysqld]`):
> ```ini
> sql_mode = "ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"
> ```

---

## 🏋️ Ejercicios del Módulo 1

1. Crea la tabla `cupones` en `tienda_dba` con: `id` autoincremental, `codigo` (único, hasta 20 caracteres, obligatorio), `porcentaje` (entero 1-90 con `CHECK`), `valido_hasta` (fecha, obligatoria), `usos` (entero sin signo, por defecto 0) y `creado_en` (automático).
2. Añade a `pedidos` una columna `cupon_id` opcional con su clave foránea a `cupones` (si se borra el cupón, el pedido queda sin cupón).
3. Muestra con `information_schema` todas las tablas de `tienda_dba` con su número aproximado de filas y su tamaño en KB.
4. Cambia la columna `ciudad` de `clientes` a `VARCHAR(80)` manteniendo que admita `NULL`.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
CREATE TABLE cupones (
    id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    codigo       VARCHAR(20) NOT NULL UNIQUE,
    porcentaje   TINYINT UNSIGNED NOT NULL CHECK (porcentaje BETWEEN 1 AND 90),
    valido_hasta DATE NOT NULL,
    usos         INT UNSIGNED NOT NULL DEFAULT 0,
    creado_en    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 2 (el tipo debe coincidir exactamente: INT UNSIGNED)
ALTER TABLE pedidos
    ADD COLUMN cupon_id INT UNSIGNED NULL AFTER empleado_id,
    ADD CONSTRAINT fk_ped_cupon FOREIGN KEY (cupon_id) REFERENCES cupones(id) ON DELETE SET NULL;

-- 3
SELECT table_name,
       table_rows,
       ROUND((data_length + index_length) / 1024, 1) AS tamano_kb
FROM information_schema.tables
WHERE table_schema = 'tienda_dba'
ORDER BY tamano_kb DESC;

-- 4
ALTER TABLE clientes MODIFY COLUMN ciudad VARCHAR(80) NULL;
```
</details>

> 🔁 Si quieres dejar `tienda_dba` como al principio para el siguiente módulo, vuelve a ejecutar el script del [índice](./00_INDICE_Y_BASE_DE_DATOS_DE_PRACTICA.md#script-de-creación-cópialo-y-ejecútalo-completo).

---

👉 **Siguiente paso:** **[Módulo 2: CRUD — INSERT, SELECT, UPDATE y DELETE](./02_CRUD_INSERT_SELECT_UPDATE_DELETE.md)**
