# Módulo 7: Transacciones, Bloqueos y Concurrencia

En producción no hay **un** usuario: hay cientos de conexiones leyendo y escribiendo **a la vez**. ¿Qué pasa si dos clientes compran la última unidad en el mismo milisegundo? ¿Y si el servidor se apaga a mitad de una transferencia? Este módulo explica cómo garantiza MySQL (InnoDB) que los datos sigan siendo correctos, y cómo diagnosticar los bloqueos y *deadlocks* que todo DBA acaba encontrando.

---

## 1. ¿Qué es una transacción? ACID

Una **transacción** es un grupo de sentencias que se ejecutan como **una sola unidad**: o se aplican **todas**, o **ninguna**.

El ejemplo clásico: traspasar 100 € de la cuenta A a la B son dos `UPDATE`. Si el servidor se cae entre los dos, el dinero desaparece... salvo que estén en una transacción.

| Propiedad | Significado | En la práctica |
| :--- | :--- | :--- |
| **A**tomicidad | Todo o nada. | Si falla el segundo `UPDATE`, el primero se deshace. |
| **C**onsistencia | La base de datos pasa de un estado válido a otro válido. | Se respetan claves foráneas, `CHECK`, `UNIQUE`. |
| **I**slamiento (*Isolation*) | Las transacciones simultáneas no se "pisan". | Lo controlan los **niveles de aislamiento** y los **bloqueos**. |
| **D**urabilidad | Lo confirmado (`COMMIT`) sobrevive a un apagón. | InnoDB lo escribe primero en el *redo log* en disco. |

> ⚠️ Solo **InnoDB** es transaccional. En tablas MyISAM, `ROLLBACK` no deshace nada.

---

## 2. Comandos de control de transacciones (TCL)

```sql
START TRANSACTION;          -- o BEGIN;

UPDATE productos SET stock = stock - 1 WHERE id = 3;
INSERT INTO pedidos (cliente_id, fecha, estado, total) VALUES (2, NOW(), 'pagado', 649.90);
INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad, precio_unitario)
VALUES (LAST_INSERT_ID(), 3, 1, 649.90);

COMMIT;                     -- Confirma: los cambios son permanentes y visibles para todos
-- o
-- ROLLBACK;                -- Deshace TODO desde el START TRANSACTION
```

### `autocommit`: cada sentencia suelta es una transacción
```sql
SELECT @@autocommit;        -- 1 (por defecto): cada sentencia se confirma sola al terminar
SET autocommit = 0;         -- A partir de aquí todo queda pendiente hasta que hagas COMMIT o ROLLBACK
SET autocommit = 1;
```
> 😱 **Peligro clásico:** te conectas con un cliente que tiene `autocommit = 0` (algunos IDE lo traen así), haces un `UPDATE`, te vas a comer... y la fila queda **bloqueada** para todos durante horas. Revisa siempre la configuración de tu herramienta.

### `SAVEPOINT`: puntos de guardado parciales
```sql
START TRANSACTION;
UPDATE productos SET precio = precio * 0.90 WHERE categoria_id = 6;   -- Descuento teclados
SAVEPOINT despues_teclados;

UPDATE productos SET precio = precio * 0.50 WHERE categoria_id = 7;   -- ¡Uy! era 0.95, no 0.50
ROLLBACK TO SAVEPOINT despues_teclados;                               -- Deshace SOLO lo de los ratones

UPDATE productos SET precio = precio * 0.95 WHERE categoria_id = 7;
COMMIT;                                                               -- Teclados -10% y ratones -5%
```

### Sentencias que hacen `COMMIT` implícito
**Toda sentencia DDL** (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME`) confirma la transacción en curso automáticamente y **no se puede deshacer**:
```sql
CREATE TABLE prueba_commit (id INT);
INSERT INTO prueba_commit VALUES (1), (2), (3);

START TRANSACTION;
DELETE FROM prueba_commit;
CREATE TABLE otra_tabla (id INT);   -- ← COMMIT implícito: el DELETE ya es definitivo
ROLLBACK;                           -- No recupera nada
SELECT COUNT(*) FROM prueba_commit; -- 0 😱

DROP TABLE prueba_commit, otra_tabla;
```
También hacen `COMMIT` implícito: `LOCK TABLES`, `CREATE USER`, `GRANT`, `SET autocommit = 1`, `START TRANSACTION` (si ya había una abierta).

---

## 3. Problemas de concurrencia (lo que el aislamiento evita)

| Problema | Qué ocurre |
| :--- | :--- |
| **Lectura sucia** (*dirty read*) | Lees datos que otra transacción ha cambiado pero **todavía no ha confirmado** (y que quizá deshaga). |
| **Lectura no repetible** (*non-repeatable read*) | Lees una fila, otra transacción la modifica y confirma, la vuelves a leer y **ha cambiado**. |
| **Lectura fantasma** (*phantom read*) | Repites un `SELECT ... WHERE` y **aparecen filas nuevas** que otra transacción insertó. |
| **Actualización perdida** (*lost update*) | Dos transacciones leen stock = 1, ambas restan 1 y guardan 0: se vendieron 2 unidades de 1. |

## 4. Niveles de aislamiento

| Nivel | Lectura sucia | No repetible | Fantasma | Notas |
| :--- | :---: | :---: | :---: | :--- |
| `READ UNCOMMITTED` | ❌ Posible | ❌ Posible | ❌ Posible | Casi nunca se usa. |
| `READ COMMITTED` | ✅ | ❌ Posible | ❌ Posible | Por defecto en PostgreSQL, SQL Server y Oracle. |
| **`REPEATABLE READ`** | ✅ | ✅ | ✅* | **Por defecto en MySQL/InnoDB.** *InnoDB evita fantasmas en lecturas normales gracias a MVCC y a los *gap locks*. |
| `SERIALIZABLE` | ✅ | ✅ | ✅ | Convierte cada `SELECT` en `SELECT ... FOR SHARE`. Máxima seguridad, mínima concurrencia. |

```sql
SELECT @@transaction_isolation;                                -- REPEATABLE-READ
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;       -- Para tu conexión
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;                 -- Solo para la PRÓXIMA transacción
```

### MVCC: por qué leer no bloquea
InnoDB usa **MVCC** (*Multi-Version Concurrency Control*): guarda versiones antiguas de las filas (en el *undo log*). En `REPEATABLE READ`, tu transacción ve una **"foto" (snapshot)** de los datos tal como estaban en su primera lectura:

| Tiempo | Sesión A | Sesión B |
| :---: | :--- | :--- |
| 1 | `START TRANSACTION;` | |
| 2 | `SELECT stock FROM productos WHERE id = 5;` → **40** | |
| 3 | | `UPDATE productos SET stock = 10 WHERE id = 5;` (autocommit) |
| 4 | `SELECT stock FROM productos WHERE id = 5;` → **40** *(sigue viendo su foto)* | |
| 5 | `COMMIT;` | |
| 6 | `SELECT stock FROM productos WHERE id = 5;` → **10** | |

> 🧪 **Pruébalo tú:** abre **dos consolas** `mysql` (o dos pestañas de tu IDE) y ejecuta los pasos en orden. Es la mejor forma de entender este módulo.

Ventaja: los `SELECT` normales **nunca esperan** por los `UPDATE` de otros. Inconveniente: una transacción abierta durante horas obliga a InnoDB a guardar todas las versiones antiguas (el *undo log* crece, el rendimiento cae: lo verás como *"History list length"* alta en el Módulo 10).

---

## 5. Bloqueos (*locks*)

### Lecturas bloqueantes: `FOR UPDATE` y `FOR SHARE`
Para evitar la **actualización perdida**, lee la fila **bloqueándola** hasta el `COMMIT`:
```sql
START TRANSACTION;

-- Bloqueo EXCLUSIVO: nadie más puede modificar ni bloquear esta fila hasta mi COMMIT
SELECT stock FROM productos WHERE id = 3 FOR UPDATE;

-- (la aplicación comprueba que stock >= cantidad)
UPDATE productos SET stock = stock - 1 WHERE id = 3;
COMMIT;
```
| Tiempo | Sesión A | Sesión B |
| :---: | :--- | :--- |
| 1 | `START TRANSACTION;` | `START TRANSACTION;` |
| 2 | `SELECT stock ... WHERE id = 3 FOR UPDATE;` → 7 | |
| 3 | | `SELECT stock ... WHERE id = 3 FOR UPDATE;` ⏳ **espera...** |
| 4 | `UPDATE ... stock = stock - 1;` `COMMIT;` | |
| 5 | | ...se desbloquea → **6** (ve el valor ya actualizado) |

| Cláusula | Tipo de bloqueo | Otros pueden leer con `SELECT` normal | Otros pueden `FOR SHARE` | Otros pueden modificar/`FOR UPDATE` |
| :--- | :--- | :---: | :---: | :---: |
| `FOR SHARE` (antes `LOCK IN SHARE MODE`) | Compartido (S) | ✅ | ✅ | ❌ |
| `FOR UPDATE` | Exclusivo (X) | ✅ (ven la versión anterior) | ❌ | ❌ |

### Alternativa sin bloqueo explícito: `UPDATE` condicional atómico
Para casos simples, un único `UPDATE` es atómico y más eficiente:
```sql
UPDATE productos SET stock = stock - 1 WHERE id = 3 AND stock >= 1;
SELECT ROW_COUNT();   -- 1 = vendido; 0 = no había stock (nadie restó de más)
```

### `NOWAIT` y `SKIP LOCKED` (MySQL 8+): colas de trabajo
```sql
-- NOWAIT: si la fila está bloqueada, falla al instante en vez de esperar
SELECT * FROM productos WHERE id = 3 FOR UPDATE NOWAIT;
-- ERROR 3572: Statement aborted because lock(s) could not be acquired immediately and NOWAIT is set.

-- SKIP LOCKED: salta las filas bloqueadas. Ideal para que varios "workers" tomen tareas distintas
START TRANSACTION;
SELECT id FROM pedidos
WHERE estado = 'pagado'
ORDER BY fecha
LIMIT 1
FOR UPDATE SKIP LOCKED;          -- Cada worker obtiene un pedido diferente, sin esperas
-- ... procesar el envío ...
UPDATE pedidos SET estado = 'enviado' WHERE id = /* el id obtenido */ 3;
COMMIT;
```

### Tipos de bloqueo internos de InnoDB
| Bloqueo | Qué bloquea | Cuándo aparece |
| :--- | :--- | :--- |
| **Record lock** | Una entrada de índice (una fila) | `UPDATE ... WHERE id = 3` |
| **Gap lock** | El "hueco" entre dos valores del índice | Rangos en `REPEATABLE READ`: impide insertar en ese hueco (evita fantasmas) |
| **Next-key lock** | Fila + hueco anterior | El tipo por defecto en `REPEATABLE READ` |
| **Insert intention** | Intención de insertar en un hueco | `INSERT` concurrentes |
| **Table lock (IS/IX)** | Marcas de intención a nivel tabla | Siempre, de forma ligera |
| **Metadata lock (MDL)** | La estructura de la tabla | Un `ALTER TABLE` espera a que terminen las transacciones que usan la tabla... y bloquea a las nuevas |

> 🔑 **Los bloqueos se ponen sobre ÍNDICES, no sobre filas.** Si tu `UPDATE ... WHERE email = 'x'` no tiene índice en `email`, InnoDB recorre (y bloquea) **toda la tabla**. Un índice adecuado (Módulo 8) reduce drásticamente los bloqueos.

### Bloqueo de tablas completo (raramente necesario)
```sql
LOCK TABLES productos WRITE, categorias READ;
-- ... operaciones de mantenimiento ...
UNLOCK TABLES;

FLUSH TABLES WITH READ LOCK;   -- Congela TODAS las escrituras del servidor (usado por algunas herramientas de backup)
UNLOCK TABLES;
```

---

## 6. *Deadlocks* (interbloqueos)

Un *deadlock* ocurre cuando dos transacciones se esperan **mutuamente** para siempre:

| Tiempo | Sesión A | Sesión B |
| :---: | :--- | :--- |
| 1 | `START TRANSACTION;` | `START TRANSACTION;` |
| 2 | `UPDATE productos SET stock = stock - 1 WHERE id = 1;` 🔒 fila 1 | |
| 3 | | `UPDATE productos SET stock = stock - 1 WHERE id = 2;` 🔒 fila 2 |
| 4 | `UPDATE productos SET stock = stock - 1 WHERE id = 2;` ⏳ espera a B | |
| 5 | | `UPDATE productos SET stock = stock - 1 WHERE id = 1;` ⏳ espera a A → 💥 |

InnoDB lo detecta al instante, **elige una víctima** (la transacción "más barata" de deshacer) y le hace `ROLLBACK`:
```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

### Cómo diagnosticarlos
```sql
-- El último deadlock ocurrido, con las sentencias y bloqueos implicados (sección "LATEST DETECTED DEADLOCK")
SHOW ENGINE INNODB STATUS\G
```
Para registrar **todos** los *deadlocks* en el log de errores (y no solo el último):
```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

### Cómo prevenirlos
1. **Accede a las filas siempre en el mismo orden** (por ejemplo, ordena por `id` los productos de un carrito antes de bloquearlos).
2. **Transacciones cortas:** nada de esperar respuestas de APIs externas o del usuario con una transacción abierta.
3. **Índices adecuados** para que los `UPDATE`/`DELETE` bloqueen pocas filas.
4. Considera `READ COMMITTED` si tu aplicación lo tolera (menos *gap locks*).
5. **Reintenta** en la aplicación: un *deadlock* ocasional es normal en sistemas concurrentes; la transacción víctima simplemente se repite.

---

## 7. Ver quién bloquea a quién (diagnóstico en vivo)

```sql
-- Conexiones activas y qué están haciendo
SHOW FULL PROCESSLIST;

-- Transacciones abiertas en este momento, desde cuándo y cuántas filas bloquean
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS segundos_abierta,
       trx_mysql_thread_id AS conexion,
       trx_rows_locked, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started;

-- ¿Quién espera a quién? (vista del esquema sys, MySQL 8)
SELECT waiting_pid       AS espera_conexion,
       waiting_query     AS consulta_que_espera,
       blocking_pid      AS bloquea_conexion,
       blocking_query    AS consulta_que_bloquea,
       wait_age          AS esperando_desde
FROM sys.innodb_lock_waits;

-- Bloqueos actuales en detalle (MySQL 8)
SELECT engine_transaction_id, object_name, index_name, lock_type, lock_mode, lock_status, lock_data
FROM performance_schema.data_locks;

-- Metadata locks (cuando un ALTER TABLE se queda "Waiting for table metadata lock")
SELECT * FROM sys.schema_table_lock_waits;
```

### Resolver: matar la conexión que bloquea
```sql
KILL 1234;          -- Cierra la conexión 1234 (su transacción hace ROLLBACK)
KILL QUERY 1234;    -- Solo cancela la consulta en curso, mantiene la conexión
```
> ⚠️ Antes de hacer `KILL`, mira **qué** está haciendo esa transacción. Matar una transacción que lleva una hora modificando millones de filas provoca un `ROLLBACK` que puede durar **otra hora**.

### Tiempo máximo de espera de un bloqueo
```sql
SELECT @@innodb_lock_wait_timeout;              -- 50 segundos por defecto
SET SESSION innodb_lock_wait_timeout = 5;       -- Falla antes en tu sesión
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

---

## 8. Buenas prácticas de transacciones

* ✅ Mantén las transacciones **lo más cortas posible**.
* ✅ Haz las lecturas que no necesitan bloqueo **antes** de abrir la transacción.
* ✅ Usa `FOR UPDATE` solo en las filas que vas a modificar después.
* ✅ Maneja en la aplicación los errores **1213** (*deadlock*) y **1205** (*timeout*) con reintentos.
* ❌ Nunca abras una transacción y esperes una acción del usuario o la respuesta de una API externa.
* ❌ No mezcles DDL dentro de transacciones (`COMMIT` implícito).
* ❌ No dejes clientes de base de datos con `autocommit = 0` y sesiones abiertas.

---

## 🏋️ Ejercicios del Módulo 7

1. Dentro de una transacción, baja un 20 % el precio de todos los productos, comprueba el resultado con un `SELECT` y deshaz los cambios. Verifica que los precios volvieron a su valor original.
2. Usando `SAVEPOINT`, inserta dos clientes nuevos, deshaz solo el segundo y confirma el primero.
3. Escribe la venta segura de **2 unidades** del producto 7 sin `SELECT ... FOR UPDATE`, garantizando que el stock nunca quede negativo, e indica cómo sabe la aplicación si se vendió.
4. **(Dos consolas)** Provoca un *deadlock* entre dos sesiones siguiendo la tabla del apartado 6 y localízalo después en `SHOW ENGINE INNODB STATUS`.
5. Escribe la consulta que muestre las transacciones que llevan abiertas **más de 60 segundos**, con el ID de conexión para poder hacer `KILL`.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
START TRANSACTION;
UPDATE productos SET precio = ROUND(precio * 0.80, 2);
SELECT sku, precio FROM productos;
ROLLBACK;
SELECT sku, precio FROM productos;    -- Precios originales

-- 2
START TRANSACTION;
INSERT INTO clientes (nombre, email, fecha_registro) VALUES ('Cliente Uno', 'uno@test.com', CURDATE());
SAVEPOINT tras_uno;
INSERT INTO clientes (nombre, email, fecha_registro) VALUES ('Cliente Dos', 'dos@test.com', CURDATE());
ROLLBACK TO SAVEPOINT tras_uno;
COMMIT;
SELECT email FROM clientes WHERE email IN ('uno@test.com', 'dos@test.com');   -- solo uno@test.com

-- 3 (UPDATE condicional atómico)
UPDATE productos SET stock = stock - 2 WHERE id = 7 AND stock >= 2;
SELECT ROW_COUNT();   -- 1 → vendido | 0 → stock insuficiente

-- 4: ver la tabla del apartado 6. Después:
SHOW ENGINE INNODB STATUS\G   -- buscar "LATEST DETECTED DEADLOCK"

-- 5
SELECT trx_mysql_thread_id AS conexion,
       trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS segundos,
       trx_rows_locked,
       trx_query
FROM information_schema.innodb_trx
WHERE trx_started < NOW() - INTERVAL 60 SECOND
ORDER BY trx_started;
```
</details>

---

👉 **Siguiente paso:** **[Módulo 8: Índices, EXPLAIN y Optimización de Consultas](./08_INDICES_EXPLAIN_Y_OPTIMIZACION.md)**
