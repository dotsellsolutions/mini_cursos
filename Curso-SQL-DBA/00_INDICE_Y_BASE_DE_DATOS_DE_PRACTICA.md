# Curso Completo de SQL para DBA (De Cero a Administrador de Bases de Datos)
## Base de datos de práctica: **`tienda_dba`**

Este curso te lleva desde el primer `SELECT` hasta las tareas que hace a diario un **DBA** (*Database Administrator*, Administrador de Bases de Datos): optimizar consultas lentas, gestionar usuarios y permisos, hacer copias de seguridad, recuperar datos borrados por error, configurar réplicas y vigilar la salud del servidor.

> 🐬 **Motor usado:** Todos los ejemplos están escritos para **MySQL 8** y **MariaDB 10.6+** (lo que traen WAMP, XAMPP, Laragon y la mayoría de VPS). Cuando un comando es distinto en **PostgreSQL** o **SQL Server**, lo verás indicado con una nota 🐘.

---

## 1. ¿Qué hace un DBA? (y por qué necesitas más que `SELECT * FROM`)

Un programador usa la base de datos. Un DBA es **responsable de que la base de datos funcione**, sea rápida, esté segura y nunca pierda datos. Sus tareas se agrupan en cuatro áreas:

| Área | Pregunta que responde | Módulos |
| :--- | :--- | :--- |
| **Diseño y lenguaje SQL** | ¿Cómo guardo y consulto los datos correctamente? | 1 al 5 |
| **Lógica en la base de datos** | ¿Cómo automatizo reglas dentro del servidor? | 6 y 7 |
| **Rendimiento** | ¿Por qué esta consulta tarda 30 segundos y cómo la bajo a 30 ms? | 8 |
| **Seguridad y operación** | ¿Quién puede ver qué? ¿Qué pasa si el disco muere a las 3 AM? | 9 y 10 |

### Las 5 familias de comandos SQL
| Familia | Significado | Comandos | ¿Qué hace? |
| :--- | :--- | :--- | :--- |
| **DDL** | *Data Definition Language* | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` | Define la **estructura**: bases de datos, tablas, índices. |
| **DML** | *Data Manipulation Language* | `INSERT`, `UPDATE`, `DELETE`, `MERGE`/`REPLACE` | Modifica los **datos**. |
| **DQL** | *Data Query Language* | `SELECT` | **Consulta** datos. |
| **DCL** | *Data Control Language* | `GRANT`, `REVOKE` | Controla **permisos**. |
| **TCL** | *Transaction Control Language* | `START TRANSACTION`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` | Controla **transacciones**. |

---

## 2. Mapa de Módulos del Curso

### 🟢 Nivel Básico
1. **[01_FUNDAMENTOS_BASES_DE_DATOS_TABLAS_Y_TIPOS.md](./01_FUNDAMENTOS_BASES_DE_DATOS_TABLAS_Y_TIPOS.md)**
   * Conectarte desde la consola, crear/borrar bases de datos, *charset* y *collation* (`utf8mb4`).
   * Todos los tipos de datos, `CREATE TABLE` con restricciones (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `DEFAULT`).
   * `ALTER TABLE` completo (añadir, modificar, renombrar y borrar columnas), `DROP` vs `TRUNCATE` vs `DELETE`.
2. **[02_CRUD_INSERT_SELECT_UPDATE_DELETE.md](./02_CRUD_INSERT_SELECT_UPDATE_DELETE.md)**
   * `INSERT` (simple, múltiple, `INSERT ... SELECT`, `ON DUPLICATE KEY UPDATE`, `INSERT IGNORE`, `REPLACE`).
   * `SELECT` con `WHERE`, operadores, `LIKE`, `IN`, `BETWEEN`, `NULL`, `ORDER BY`, `LIMIT`, `DISTINCT`.
   * `UPDATE` y `DELETE` seguros (y cómo no borrar toda la tabla por accidente).
3. **[03_FUNCIONES_AGREGACION_GROUP_BY_Y_HAVING.md](./03_FUNCIONES_AGREGACION_GROUP_BY_Y_HAVING.md)**
   * Funciones de texto, números, fechas y conversión. `CASE`, `IF`, `COALESCE`.
   * `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP BY`, `HAVING`, `WITH ROLLUP`, `GROUP_CONCAT`.

### 🟡 Nivel Intermedio
4. **[04_JOINS_SUBCONSULTAS_Y_UNION.md](./04_JOINS_SUBCONSULTAS_Y_UNION.md)**
   * Todos los `JOIN` (`INNER`, `LEFT`, `RIGHT`, `CROSS`, *self join*, `FULL OUTER` emulado).
   * Subconsultas (escalares, `IN`, `EXISTS`, correlacionadas, tablas derivadas), `UNION`, `INTERSECT`, `EXCEPT`.
   * `UPDATE` y `DELETE` con `JOIN`.
5. **[05_SQL_AVANZADO_CTE_WINDOW_FUNCTIONS_Y_JSON.md](./05_SQL_AVANZADO_CTE_WINDOW_FUNCTIONS_Y_JSON.md)**
   * `WITH` (CTE) y **CTE recursivas** (jerarquías y árboles).
   * **Funciones de ventana:** `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, acumulados, *top N por grupo*.
   * Tablas pivote, columnas generadas y funciones **JSON**.
6. **[06_VISTAS_PROCEDIMIENTOS_FUNCIONES_TRIGGERS_Y_EVENTOS.md](./06_VISTAS_PROCEDIMIENTOS_FUNCIONES_TRIGGERS_Y_EVENTOS.md)**
   * Vistas, procedimientos almacenados (parámetros `IN`/`OUT`, variables, bucles, cursores, manejo de errores).
   * Funciones propias, *triggers* (auditoría automática) y eventos programados.

### 🔴 Nivel Avanzado (DBA)
7. **[07_TRANSACCIONES_BLOQUEOS_Y_CONCURRENCIA.md](./07_TRANSACCIONES_BLOQUEOS_Y_CONCURRENCIA.md)**
   * ACID, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, niveles de aislamiento.
   * Bloqueos (`FOR UPDATE`, `SKIP LOCKED`), *deadlocks* y cómo diagnosticarlos.
8. **[08_INDICES_EXPLAIN_Y_OPTIMIZACION.md](./08_INDICES_EXPLAIN_Y_OPTIMIZACION.md)**
   * Cómo funcionan los índices (B-Tree), índices compuestos, *covering*, únicos, `FULLTEXT`, invisibles.
   * Leer `EXPLAIN` y `EXPLAIN ANALYZE`, *slow query log*, antipatrones y particionado.
9. **[09_USUARIOS_PERMISOS_ROLES_Y_SEGURIDAD.md](./09_USUARIOS_PERMISOS_ROLES_Y_SEGURIDAD.md)**
   * `CREATE USER`, `GRANT`, `REVOKE`, roles, políticas de contraseñas, SSL, mínimo privilegio, inyección SQL.
10. **[10_BACKUPS_RECUPERACION_REPLICACION_Y_MONITOREO.md](./10_BACKUPS_RECUPERACION_REPLICACION_Y_MONITOREO.md)**
    * `mysqldump`, restauración, *binary logs* y **recuperación a un punto en el tiempo**.
    * Replicación (*source/replica*, GTID), monitoreo (`PROCESSLIST`, `performance_schema`, `sys`), configuración (`my.cnf`) y checklist de mantenimiento.

> 💪 **Cada módulo termina con ejercicios y sus soluciones** sobre la base de datos `tienda_dba`. Intenta resolverlos antes de mirar la solución.

---

## 3. La Base de Datos de Práctica: `tienda_dba`

Todos los ejemplos del curso usan la misma base de datos: una tienda con empleados, clientes, productos en categorías (con subcategorías) y pedidos.

```
┌──────────────────┐        ┌──────────────────────────┐
│  departamentos   │◄───────│        empleados         │◄──┐ jefe_id
└──────────────────┘   N:1  │ (salario, jefe_id → self) │───┘ (auto-relación)
                            └────────────┬─────────────┘
                                         │ 1:N (vendedor del pedido)
┌──────────────────┐  1:N   ┌────────────▼─────────────┐  1:N  ┌──────────────────┐
│     clientes     │───────►│         pedidos          │──────►│ detalle_pedidos  │
└──────────────────┘        └──────────────────────────┘       │ (cantidad,precio)│
                                                               └────────┬─────────┘
┌──────────────────┐  1:N   ┌──────────────────────────┐   N:1          │
│    categorias    │───────►│        productos         │◄───────────────┘
│ (padre_id → self)│        └──────────────────────────┘
└──────────────────┘
```

### ⚙️ Antes de empezar: comprueba la configuración de tu servidor
Los paquetes de desarrollo (WAMP, XAMPP...) a veces traen una configuración **distinta a la de un servidor estándar**, y los ejemplos del curso no se comportarán igual. Ejecuta:
```sql
SELECT @@version, @@sql_mode, @@default_storage_engine;
```
| Variable | Valor esperado | Si no coincide |
| :--- | :--- | :--- |
| `@@sql_mode` | Contiene `STRICT_TRANS_TABLES` y `ONLY_FULL_GROUP_BY` | Ver [Módulo 1, apartado 9](./01_FUNDAMENTOS_BASES_DE_DATOS_TABLAS_Y_TIPOS.md#9-sql_mode-qué-tan-estricto-es-tu-servidor) |
| `@@default_storage_engine` | `InnoDB` | Ver [Módulo 1, apartado 8](./01_FUNDAMENTOS_BASES_DE_DATOS_TABLAS_Y_TIPOS.md#8-motores-de-almacenamiento) |

Para dejarlo como un servidor estándar, edita `my.ini` (en WAMP: clic en el icono → MySQL → `my.ini`), añade en la sección `[mysqld]` y reinicia MySQL:
```ini
[mysqld]
default-storage-engine = InnoDB
sql_mode = "ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"
```
> ⚠️ Si tienes **otros proyectos** en ese mismo MySQL (por ejemplo, aplicaciones antiguas que dependen del modo permisivo), el `sql_mode` estricto podría hacer fallar alguno. En ese caso, aplícalo solo a tu sesión de práctica con `SET SESSION ...` al conectarte.

### Script de creación (cópialo y ejecútalo completo)
Guárdalo como `tienda_dba.sql` y ejecútalo con `mysql -u root -p < tienda_dba.sql`, o pégalo en phpMyAdmin / HeidiSQL / DBeaver / MySQL Workbench.

```sql
-- =========================================================
--  BASE DE DATOS DE PRÁCTICA: tienda_dba
-- =========================================================
DROP DATABASE IF EXISTS tienda_dba;
CREATE DATABASE tienda_dba
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE tienda_dba;

-- ---------- DEPARTAMENTOS ----------
CREATE TABLE departamentos (
    id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nombre  VARCHAR(60) NOT NULL UNIQUE
) ENGINE=InnoDB;

-- ---------- EMPLEADOS (con auto-relación jefe_id) ----------
CREATE TABLE empleados (
    id                  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nombre              VARCHAR(60)  NOT NULL,
    apellido            VARCHAR(60)  NOT NULL,
    email               VARCHAR(120) NOT NULL UNIQUE,
    salario             DECIMAL(10,2) NOT NULL CHECK (salario > 0),
    fecha_contratacion  DATE NOT NULL,
    departamento_id     INT UNSIGNED NULL,
    jefe_id             INT UNSIGNED NULL,
    CONSTRAINT fk_emp_depto FOREIGN KEY (departamento_id) REFERENCES departamentos(id) ON DELETE SET NULL,
    CONSTRAINT fk_emp_jefe  FOREIGN KEY (jefe_id)         REFERENCES empleados(id)     ON DELETE SET NULL
) ENGINE=InnoDB;

-- ---------- CLIENTES ----------
CREATE TABLE clientes (
    id              INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,
    email           VARCHAR(120) NOT NULL UNIQUE,
    ciudad          VARCHAR(60),
    pais            CHAR(2) NOT NULL DEFAULT 'ES',
    fecha_registro  DATE NOT NULL,
    activo          BOOLEAN NOT NULL DEFAULT TRUE
) ENGINE=InnoDB;

-- ---------- CATEGORÍAS (árbol: categoría padre) ----------
CREATE TABLE categorias (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nombre     VARCHAR(60) NOT NULL,
    padre_id   INT UNSIGNED NULL,
    CONSTRAINT fk_cat_padre FOREIGN KEY (padre_id) REFERENCES categorias(id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- ---------- PRODUCTOS ----------
CREATE TABLE productos (
    id            INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sku           VARCHAR(20)  NOT NULL UNIQUE,
    nombre        VARCHAR(120) NOT NULL,
    categoria_id  INT UNSIGNED NOT NULL,
    precio        DECIMAL(10,2) NOT NULL CHECK (precio >= 0),
    stock         INT UNSIGNED NOT NULL DEFAULT 0,
    activo        BOOLEAN NOT NULL DEFAULT TRUE,
    creado_en     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_prod_cat FOREIGN KEY (categoria_id) REFERENCES categorias(id)
) ENGINE=InnoDB;

-- ---------- PEDIDOS ----------
CREATE TABLE pedidos (
    id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    cliente_id   INT UNSIGNED NOT NULL,
    empleado_id  INT UNSIGNED NULL,
    fecha        DATETIME NOT NULL,
    estado       ENUM('pendiente','pagado','enviado','cancelado') NOT NULL DEFAULT 'pendiente',
    total        DECIMAL(10,2) NOT NULL DEFAULT 0,
    CONSTRAINT fk_ped_cliente  FOREIGN KEY (cliente_id)  REFERENCES clientes(id),
    CONSTRAINT fk_ped_empleado FOREIGN KEY (empleado_id) REFERENCES empleados(id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- ---------- DETALLE DE PEDIDOS (clave primaria compuesta) ----------
CREATE TABLE detalle_pedidos (
    pedido_id        INT UNSIGNED NOT NULL,
    producto_id      INT UNSIGNED NOT NULL,
    cantidad         SMALLINT UNSIGNED NOT NULL CHECK (cantidad > 0),
    precio_unitario  DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (pedido_id, producto_id),
    CONSTRAINT fk_det_pedido   FOREIGN KEY (pedido_id)   REFERENCES pedidos(id) ON DELETE CASCADE,
    CONSTRAINT fk_det_producto FOREIGN KEY (producto_id) REFERENCES productos(id)
) ENGINE=InnoDB;

-- =========================================================
--  DATOS DE EJEMPLO
-- =========================================================
INSERT INTO departamentos (nombre) VALUES
('Dirección'), ('Ventas'), ('Almacén'), ('Soporte');

INSERT INTO empleados (nombre, apellido, email, salario, fecha_contratacion, departamento_id, jefe_id) VALUES
('Laura',   'Gómez',    'laura@tienda.com',   5200.00, '2018-03-01', 1, NULL),  -- 1 Directora
('Carlos',  'Ruiz',     'carlos@tienda.com',  3100.00, '2019-06-15', 2, 1),     -- 2 Jefe de ventas
('Marta',   'López',    'marta@tienda.com',   2200.00, '2021-01-10', 2, 2),     -- 3
('Pedro',   'Sánchez',  'pedro@tienda.com',   2150.00, '2022-09-05', 2, 2),     -- 4
('Ana',     'Martín',   'ana@tienda.com',     2600.00, '2020-02-20', 3, 1),     -- 5 Jefa de almacén
('Jorge',   'Díaz',     'jorge@tienda.com',   1800.00, '2023-04-11', 3, 5),     -- 6
('Lucía',   'Torres',   'lucia@tienda.com',   1950.00, '2023-11-02', 4, 1),     -- 7
('Diego',   'Navarro',  'diego@tienda.com',   1700.00, '2024-07-01', NULL, NULL); -- 8 sin departamento

INSERT INTO clientes (nombre, email, ciudad, pais, fecha_registro, activo) VALUES
('Sofía Herrera',     'sofia@mail.com',    'Madrid',    'ES', '2023-01-15', TRUE),
('Javier Molina',     'javier@mail.com',   'Barcelona', 'ES', '2023-03-22', TRUE),
('Valentina Rojas',   'valen@mail.com',    'Bogotá',    'CO', '2023-05-30', TRUE),
('Mateo Fernández',   'mateo@mail.com',    'CDMX',      'MX', '2023-08-08', TRUE),
('Camila Ortega',     'camila@mail.com',   'Madrid',    'ES', '2024-01-12', TRUE),
('Andrés Castro',     'andres@mail.com',   'Lima',      'PE', '2024-02-18', FALSE),
('Isabella Vargas',   'isabella@mail.com', 'CDMX',      'MX', '2024-06-03', TRUE),
('Hugo Romero',       'hugo@mail.com',     NULL,        'ES', '2024-10-25', TRUE);  -- sin pedidos

INSERT INTO categorias (nombre, padre_id) VALUES
('Informática', NULL),      -- 1
('Portátiles', 1),          -- 2
('Componentes', 1),         -- 3
('Tarjetas gráficas', 3),   -- 4
('Periféricos', NULL),      -- 5
('Teclados', 5),            -- 6
('Ratones', 5);             -- 7

INSERT INTO productos (sku, nombre, categoria_id, precio, stock, activo) VALUES
('LAP-001', 'Portátil Ultra 14"',        2, 899.99,  12, TRUE),
('LAP-002', 'Portátil Gamer 16"',        2, 1499.00,  4, TRUE),
('GPU-001', 'Tarjeta gráfica RTX 4070',  4, 649.90,   7, TRUE),
('GPU-002', 'Tarjeta gráfica RX 7800',   4, 529.00,   0, TRUE),
('CMP-001', 'SSD NVMe 1TB',              3, 89.95,   40, TRUE),
('CMP-002', 'Memoria RAM 32GB DDR5',     3, 119.00,  25, TRUE),
('TEC-001', 'Teclado Mecánico RGB',      6, 59.99,   30, TRUE),
('TEC-002', 'Teclado Inalámbrico Slim',  6, 39.50,    0, FALSE),
('RAT-001', 'Ratón Gaming 26K DPI',      7, 49.90,   55, TRUE),
('RAT-002', 'Ratón Ergonómico Vertical', 7, 34.00,   18, TRUE);

INSERT INTO pedidos (cliente_id, empleado_id, fecha, estado) VALUES
(1, 3, '2024-01-20 10:15:00', 'enviado'),    -- 1
(2, 4, '2024-02-03 17:40:00', 'enviado'),    -- 2
(1, 3, '2024-03-11 09:05:00', 'pagado'),     -- 3
(3, 3, '2024-04-25 12:30:00', 'enviado'),    -- 4
(4, 4, '2024-06-14 20:10:00', 'cancelado'),  -- 5
(5, 3, '2024-08-02 11:00:00', 'pagado'),     -- 6
(7, 4, '2024-09-19 15:45:00', 'enviado'),    -- 7
(2, NULL, '2024-11-29 00:05:00', 'pagado'),  -- 8 (venta online, sin vendedor)
(3, 4, '2025-01-07 18:20:00', 'pendiente'),  -- 9
(1, 3, '2025-02-14 13:00:00', 'pendiente');  -- 10

INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad, precio_unitario) VALUES
(1, 1, 1, 899.99), (1, 9, 1, 49.90),
(2, 3, 1, 649.90), (2, 5, 2, 89.95),
(3, 7, 2, 59.99),
(4, 2, 1, 1499.00), (4, 6, 2, 119.00), (4, 9, 1, 49.90),
(5, 4, 1, 529.00),
(6, 5, 1, 89.95), (6, 10, 1, 34.00),
(7, 1, 1, 879.00),                       -- se vendió con descuento
(8, 3, 1, 599.00), (8, 6, 1, 119.00), (8, 7, 1, 59.99),
(9, 9, 3, 49.90),
(10, 2, 1, 1450.00), (10, 7, 1, 59.99);

-- Calculamos el total de cada pedido a partir de su detalle
UPDATE pedidos p
SET p.total = (
    SELECT SUM(d.cantidad * d.precio_unitario)
    FROM detalle_pedidos d
    WHERE d.pedido_id = p.id
);
```

✅ **Comprueba que todo cargó bien:**
```sql
SELECT 'departamentos' AS tabla, COUNT(*) AS filas FROM departamentos
UNION ALL SELECT 'empleados',       COUNT(*) FROM empleados
UNION ALL SELECT 'clientes',        COUNT(*) FROM clientes
UNION ALL SELECT 'categorias',      COUNT(*) FROM categorias
UNION ALL SELECT 'productos',       COUNT(*) FROM productos
UNION ALL SELECT 'pedidos',         COUNT(*) FROM pedidos
UNION ALL SELECT 'detalle_pedidos', COUNT(*) FROM detalle_pedidos;
-- Esperado: 4, 8, 8, 7, 10, 10, 18
```

> 🔁 **¿Rompiste algo practicando?** Vuelve a ejecutar el script entero: empieza con `DROP DATABASE IF EXISTS`, así siempre vuelves al estado inicial.

---

## 4. Herramientas recomendadas

| Herramienta | Tipo | Para qué |
| :--- | :--- | :--- |
| **Cliente `mysql`** (consola) | Línea de comandos | Lo que usarás en servidores (por SSH). **Todo DBA debe dominarlo.** |
| **DBeaver** | Gráfica, gratis, multi-motor | MySQL, MariaDB, PostgreSQL, SQL Server, SQLite... en una sola app. |
| **MySQL Workbench** | Gráfica, oficial | Diagramas, `EXPLAIN` visual, administración. |
| **HeidiSQL** | Gráfica, ligera (Windows) | Rápida para el día a día. |
| **phpMyAdmin** | Web (viene con WAMP) | Útil, pero limitada para tareas de DBA. |

---

👉 **Empieza aquí:** **[Módulo 1: Fundamentos — Bases de Datos, Tablas y Tipos de Datos](./01_FUNDAMENTOS_BASES_DE_DATOS_TABLAS_Y_TIPOS.md)**
