# Módulo 6: Vistas, Procedimientos Almacenados, Funciones, Triggers y Eventos

Hasta ahora todo el SQL lo enviabas tú desde fuera. En este módulo aprendes a **guardar lógica dentro del servidor**: consultas reutilizables (vistas), programas con parámetros (procedimientos), cálculos propios (funciones), reacciones automáticas a cambios (*triggers*) y tareas programadas (eventos).

> 💡 **¿Lógica en la base de datos o en la aplicación?** Regla práctica de DBA: pon en la base de datos lo que protege la **integridad de los datos** (auditoría, reglas que nunca deben saltarse, mantenimiento) y deja en la aplicación la **lógica de negocio cambiante**. Los procedimientos son difíciles de versionar, testear y depurar si se abusa de ellos.

---

## 1. Vistas (`VIEW`): consultas guardadas con nombre

Una vista es un `SELECT` guardado que se usa **como si fuera una tabla**. No guarda datos: cada vez que la consultas, ejecuta su `SELECT`.

```sql
CREATE OR REPLACE VIEW v_pedidos_detalle AS
SELECT p.id                               AS pedido_id,
       p.fecha,
       p.estado,
       c.nombre                           AS cliente,
       c.pais,
       CONCAT(e.nombre, ' ', e.apellido)  AS vendedor,
       p.total
FROM pedidos p
JOIN clientes c       ON c.id = p.cliente_id
LEFT JOIN empleados e ON e.id = p.empleado_id;

-- Se consulta igual que una tabla (y se puede filtrar, unir, agrupar...)
SELECT cliente, SUM(total) FROM v_pedidos_detalle WHERE estado = 'enviado' GROUP BY cliente;
```

### ¿Para qué sirven?
1. **Simplificar:** el equipo usa `v_pedidos_detalle` en vez de repetir 3 `JOIN` en cada informe.
2. **Seguridad:** das permiso sobre la **vista** pero no sobre la tabla (Módulo 9). Por ejemplo, una vista de empleados **sin** la columna `salario`:
   ```sql
   CREATE VIEW v_directorio AS
   SELECT CONCAT(nombre, ' ', apellido) AS empleado, email, departamento_id FROM empleados;
   ```
3. **Compatibilidad:** si renombras o divides una tabla, una vista con el nombre antiguo evita romper aplicaciones viejas.

### Vistas actualizables y `WITH CHECK OPTION`
Si la vista es simple (una tabla, sin `GROUP BY`, `DISTINCT`, `UNION`...), puedes hacer `INSERT`/`UPDATE` a través de ella:
```sql
CREATE OR REPLACE VIEW v_productos_activos AS
SELECT id, sku, nombre, precio, stock, activo, categoria_id
FROM productos
WHERE activo = TRUE
WITH CHECK OPTION;   -- Impide modificar una fila de forma que "salga" de la vista

UPDATE v_productos_activos SET precio = 55.00 WHERE sku = 'TEC-001';  -- ✅
UPDATE v_productos_activos SET activo = FALSE WHERE sku = 'TEC-001';
-- ❌ ERROR 1369: CHECK OPTION failed (la fila dejaría de cumplir activo = TRUE)
```

### Administrar vistas
```sql
SHOW FULL TABLES WHERE Table_type = 'VIEW';
SHOW CREATE VIEW v_pedidos_detalle\G
DROP VIEW IF EXISTS v_directorio;
```
> 🐢 **Rendimiento:** una vista no es más rápida que su consulta. Vistas que llaman a vistas que llaman a vistas son una fuente clásica de consultas lentas. Revisa siempre su `EXPLAIN` (Módulo 8).
>
> 🐘 **PostgreSQL / Oracle** tienen **vistas materializadas** (guardan el resultado y se refrescan con `REFRESH MATERIALIZED VIEW`). En MySQL se simulan con una tabla normal + un evento que la recalcula (apartado 6).

---

## 2. Variables

```sql
-- Variables de usuario (@): viven mientras dure tu conexión
SET @iva = 0.21;
SELECT nombre, precio, ROUND(precio * (1 + @iva), 2) AS con_iva FROM productos;

SELECT MAX(total) INTO @mayor FROM pedidos;
SELECT @mayor;

-- Variables de sistema: configuran el servidor (@@)
SELECT @@version, @@autocommit, @@max_connections;
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';   -- solo tu conexión
-- SET GLOBAL ... afecta a todas las conexiones nuevas (requiere privilegios)
-- SET PERSIST ... (MySQL 8) además lo guarda para que sobreviva a un reinicio
```
Dentro de procedimientos y funciones se usan variables **locales** con `DECLARE` (sin `@`).

---

## 3. Procedimientos almacenados (`PROCEDURE`)

Un procedimiento es un **programa guardado en el servidor** que se ejecuta con `CALL`. Puede recibir parámetros, ejecutar varias sentencias, usar condiciones y bucles y devolver resultados.

### El truco del `DELIMITER`
Dentro del procedimiento hay muchos `;`. Para que la consola no crea que el `CREATE PROCEDURE` termina en el primer `;`, se cambia temporalmente el delimitador:

```sql
DELIMITER //

CREATE PROCEDURE sp_pedidos_de_cliente(IN p_cliente_id INT)
BEGIN
    SELECT id, fecha, estado, total
    FROM pedidos
    WHERE cliente_id = p_cliente_id
    ORDER BY fecha DESC;
END //

DELIMITER ;

CALL sp_pedidos_de_cliente(1);
```
> `DELIMITER` es un comando **del cliente** `mysql` (y de Workbench, HeidiSQL, DBeaver), no del servidor. Desde un lenguaje de programación (PHP, Python...) se envía el `CREATE PROCEDURE` completo sin `DELIMITER`.

### Parámetros `IN`, `OUT` e `INOUT`
```sql
DELIMITER //
CREATE PROCEDURE sp_resumen_cliente(
    IN  p_cliente_id INT,
    OUT p_pedidos    INT,
    OUT p_gastado    DECIMAL(12,2)
)
BEGIN
    SELECT COUNT(*), COALESCE(SUM(total), 0)
    INTO p_pedidos, p_gastado
    FROM pedidos
    WHERE cliente_id = p_cliente_id AND estado <> 'cancelado';
END //
DELIMITER ;

CALL sp_resumen_cliente(1, @n, @total);
SELECT @n AS pedidos, @total AS gastado;     -- 3 | 2579.86
```

### Variables locales, `IF`, `CASE` y bucles
```sql
DELIMITER //
CREATE PROCEDURE sp_reponer_stock(IN p_minimo INT, IN p_cantidad INT, OUT p_repuestos INT)
BEGIN
    DECLARE v_total INT DEFAULT 0;

    IF p_cantidad <= 0 THEN
        -- Lanzar un error propio
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La cantidad a reponer debe ser positiva';
    END IF;

    UPDATE productos
    SET stock = stock + p_cantidad
    WHERE activo = TRUE AND stock < p_minimo;

    SET v_total = ROW_COUNT();       -- Filas afectadas por la última sentencia
    SET p_repuestos = v_total;
END //
DELIMITER ;

CALL sp_reponer_stock(5, 20, @r);  SELECT @r;   -- 2 productos repuestos
CALL sp_reponer_stock(5, 0, @r);                -- ERROR 1644 (45000): La cantidad a reponer debe ser positiva
```

| Estructura | Sintaxis |
| :--- | :--- |
| Condición | `IF cond THEN ... ELSEIF cond THEN ... ELSE ... END IF;` |
| Selección | `CASE WHEN cond THEN ...; ELSE ...; END CASE;` |
| Bucle con condición al inicio | `WHILE cond DO ... END WHILE;` |
| Bucle con condición al final | `REPEAT ... UNTIL cond END REPEAT;` |
| Bucle con salida manual | `etiqueta: LOOP ... IF cond THEN LEAVE etiqueta; END IF; END LOOP;` |
| Saltar a la siguiente vuelta | `ITERATE etiqueta;` |

### Ejemplo con bucle: generar datos de prueba
```sql
DELIMITER //
CREATE PROCEDURE sp_generar_clientes(IN p_cantidad INT)
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= p_cantidad DO
        INSERT INTO clientes (nombre, email, ciudad, pais, fecha_registro)
        VALUES (CONCAT('Cliente prueba ', i),
                CONCAT('prueba', i, '_', UNIX_TIMESTAMP(), '@test.com'),
                ELT(1 + FLOOR(RAND() * 3), 'Madrid', 'CDMX', 'Bogotá'),   -- Ciudad al azar
                'ES',
                CURDATE() - INTERVAL FLOOR(RAND() * 365) DAY);
        SET i = i + 1;
    END WHILE;
END //
DELIMITER ;

CALL sp_generar_clientes(100);
```

### Cursores: recorrer un resultado fila a fila
Úsalos solo cuando no puedas resolverlo con una sentencia SQL normal (casi siempre puedes, y es mucho más rápido).
```sql
DELIMITER //
CREATE PROCEDURE sp_recalcular_totales()
BEGIN
    DECLARE v_id INT;
    DECLARE v_fin BOOLEAN DEFAULT FALSE;

    -- 1) Declarar el cursor
    DECLARE cur CURSOR FOR SELECT id FROM pedidos;
    -- 2) Qué hacer cuando no queden filas
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_fin = TRUE;

    OPEN cur;
    bucle: LOOP
        FETCH cur INTO v_id;
        IF v_fin THEN LEAVE bucle; END IF;

        UPDATE pedidos
        SET total = (SELECT COALESCE(SUM(cantidad * precio_unitario), 0)
                     FROM detalle_pedidos WHERE pedido_id = v_id)
        WHERE id = v_id;
    END LOOP;
    CLOSE cur;
END //
DELIMITER ;
-- (El mismo resultado se consigue con un solo UPDATE ... JOIN del Módulo 4, que es muchísimo más rápido)
```

### Manejo de errores y transacciones dentro de un procedimiento
El patrón profesional: si algo falla, deshacer todo y propagar el error.
```sql
DELIMITER //
CREATE PROCEDURE sp_crear_pedido(
    IN  p_cliente_id  INT,
    IN  p_producto_id INT,
    IN  p_cantidad    INT,
    OUT p_pedido_id   INT
)
BEGIN
    DECLARE v_precio DECIMAL(10,2);
    DECLARE v_stock  INT;

    -- Si ocurre CUALQUIER error SQL: deshacer y relanzar el error original
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    -- Bloquea la fila del producto hasta el COMMIT (Módulo 7) para que nadie venda ese stock a la vez
    SELECT precio, stock INTO v_precio, v_stock
    FROM productos WHERE id = p_producto_id AND activo = TRUE
    FOR UPDATE;

    IF v_precio IS NULL THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Producto inexistente o inactivo';
    END IF;
    IF v_stock < p_cantidad THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Stock insuficiente';
    END IF;

    INSERT INTO pedidos (cliente_id, fecha, estado, total)
    VALUES (p_cliente_id, NOW(), 'pendiente', v_precio * p_cantidad);
    SET p_pedido_id = LAST_INSERT_ID();

    INSERT INTO detalle_pedidos (pedido_id, producto_id, cantidad, precio_unitario)
    VALUES (p_pedido_id, p_producto_id, p_cantidad, v_precio);

    UPDATE productos SET stock = stock - p_cantidad WHERE id = p_producto_id;

    COMMIT;
END //
DELIMITER ;

CALL sp_crear_pedido(5, 9, 2, @nuevo);   SELECT @nuevo;           -- ✅ nuevo pedido (id 11)
CALL sp_crear_pedido(5, 9, 999, @nuevo);                           -- ❌ Stock insuficiente (y no se guarda NADA)
CALL sp_crear_pedido(5, 8, 1, @nuevo);                             -- ❌ Producto inexistente o inactivo
```
| Handler | Comportamiento |
| :--- | :--- |
| `DECLARE CONTINUE HANDLER FOR ...` | Ejecuta el bloque y **sigue** con la siguiente sentencia. |
| `DECLARE EXIT HANDLER FOR ...` | Ejecuta el bloque y **sale** del `BEGIN ... END` actual. |
| `FOR SQLEXCEPTION` / `SQLWARNING` / `NOT FOUND` | Cualquier error / aviso / "no hay más filas". |
| `FOR 1062` o `FOR SQLSTATE '23000'` | Un error concreto (ej. clave duplicada). |
| `SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '...'` | Lanza tu propio error. |
| `RESIGNAL` | Relanza el error capturado. |
| `GET DIAGNOSTICS CONDITION 1 @msg = MESSAGE_TEXT;` | Lee el mensaje del error capturado. |

### SQL dinámico con `PREPARE`
Para construir sentencias en tiempo de ejecución (nombres de tabla variables, pivotes dinámicos):
```sql
SET @tabla = 'productos';
SET @sql = CONCAT('SELECT COUNT(*) AS filas FROM ', @tabla);
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;

-- Con parámetros (?): seguro contra inyección SQL en los VALORES
PREPARE stmt FROM 'SELECT nombre, precio FROM productos WHERE precio > ? AND categoria_id = ?';
SET @min = 50, @cat = 2;
EXECUTE stmt USING @min, @cat;
DEALLOCATE PREPARE stmt;
```
> 🛡️ Nunca concatenes en `@sql` texto que venga de un usuario: es **inyección SQL** (Módulo 9). Los `?` solo sirven para valores, no para nombres de tablas o columnas; esos valídalos contra una lista blanca.

### Administrar procedimientos
```sql
SHOW PROCEDURE STATUS WHERE Db = 'tienda_dba';
SHOW CREATE PROCEDURE sp_crear_pedido\G
DROP PROCEDURE IF EXISTS sp_generar_clientes;
```
*(No existe `ALTER PROCEDURE` para cambiar el código: se hace `DROP` + `CREATE`. MariaDB admite `CREATE OR REPLACE PROCEDURE`.)*

---

## 4. Funciones propias (`FUNCTION`)

A diferencia de un procedimiento, una función **devuelve un único valor** y se usa **dentro de las consultas**, como `UPPER()` o `ROUND()`.

```sql
DELIMITER //
CREATE FUNCTION fn_precio_con_iva(p_precio DECIMAL(10,2), p_pais CHAR(2))
RETURNS DECIMAL(10,2)
DETERMINISTIC          -- Mismos parámetros → mismo resultado (permite optimizaciones)
NO SQL                 -- No lee ni escribe tablas
BEGIN
    DECLARE v_iva DECIMAL(4,2);
    SET v_iva = CASE p_pais
                    WHEN 'ES' THEN 0.21
                    WHEN 'MX' THEN 0.16
                    WHEN 'CO' THEN 0.19
                    ELSE 0
                END;
    RETURN ROUND(p_precio * (1 + v_iva), 2);
END //
DELIMITER ;

SELECT nombre, precio,
       fn_precio_con_iva(precio, 'ES') AS espana,
       fn_precio_con_iva(precio, 'MX') AS mexico
FROM productos;
```

```sql
-- Función que lee datos: antigüedad del cliente en días desde su primer pedido
DELIMITER //
CREATE FUNCTION fn_dias_como_cliente(p_cliente_id INT)
RETURNS INT
READS SQL DATA
BEGIN
    DECLARE v_primero DATETIME;
    SELECT MIN(fecha) INTO v_primero FROM pedidos WHERE cliente_id = p_cliente_id;
    RETURN IF(v_primero IS NULL, NULL, DATEDIFF(NOW(), v_primero));
END //
DELIMITER ;

SELECT nombre, fn_dias_como_cliente(id) AS dias FROM clientes;
```
> ⚠️ **Error 1418 al crear funciones:** si el *binary log* está activo (siempre en producción), MySQL exige declarar `DETERMINISTIC`, `NO SQL` o `READS SQL DATA`. Dilo honestamente: marcar como `DETERMINISTIC` una función que no lo es puede romper la replicación.
>
> 🐢 Una función que hace un `SELECT` se ejecuta **una vez por cada fila**: en una tabla de un millón de filas, son un millón de consultas. Para informes grandes, usa un `JOIN`.

---

## 5. *Triggers* (disparadores): reacciones automáticas

Un *trigger* se ejecuta **solo**, cuando ocurre un `INSERT`, `UPDATE` o `DELETE` en una tabla.

| Momento | Evento | Acceso a datos |
| :--- | :--- | :--- |
| `BEFORE` / `AFTER` | `INSERT` | `NEW.columna` (fila que se inserta) |
| `BEFORE` / `AFTER` | `UPDATE` | `OLD.columna` (antes) y `NEW.columna` (después) |
| `BEFORE` / `AFTER` | `DELETE` | `OLD.columna` (fila que se borra) |

* `BEFORE`: puedes **modificar** `NEW` (normalizar datos) o **impedir** la operación con `SIGNAL`.
* `AFTER`: la fila ya está guardada; ideal para auditoría o actualizar otras tablas.

### Ejemplo 1: Auditoría de cambios de precio
```sql
CREATE TABLE auditoria_precios (
    id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    producto_id   INT UNSIGNED NOT NULL,
    precio_antes  DECIMAL(10,2),
    precio_nuevo  DECIMAL(10,2),
    usuario       VARCHAR(100),
    fecha         DATETIME DEFAULT CURRENT_TIMESTAMP
);

DELIMITER //
CREATE TRIGGER trg_productos_audit_precio
AFTER UPDATE ON productos
FOR EACH ROW
BEGIN
    IF NOT (OLD.precio <=> NEW.precio) THEN          -- Solo si el precio cambió de verdad
        INSERT INTO auditoria_precios (producto_id, precio_antes, precio_nuevo, usuario)
        VALUES (NEW.id, OLD.precio, NEW.precio, CURRENT_USER());
    END IF;
END //
DELIMITER ;

UPDATE productos SET precio = 54.99 WHERE sku = 'TEC-001';
SELECT * FROM auditoria_precios;
```

### Ejemplo 2: Validar y normalizar antes de guardar
```sql
DELIMITER //
CREATE TRIGGER trg_clientes_before_insert
BEFORE INSERT ON clientes
FOR EACH ROW
BEGIN
    SET NEW.email = LOWER(TRIM(NEW.email));          -- Normaliza el email
    SET NEW.pais  = UPPER(NEW.pais);
    IF NEW.fecha_registro > CURDATE() THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La fecha de registro no puede ser futura';
    END IF;
END //
DELIMITER ;

INSERT INTO clientes (nombre, email, pais, fecha_registro) VALUES ('Eva Ríos', '  EVA@Mail.COM ', 'es', CURDATE());
SELECT email, pais FROM clientes WHERE nombre = 'Eva Ríos';   -- eva@mail.com | ES
```

### Ejemplo 3: Mantener el stock sincronizado
```sql
DELIMITER //
CREATE TRIGGER trg_detalle_after_insert
AFTER INSERT ON detalle_pedidos
FOR EACH ROW
BEGIN
    UPDATE productos SET stock = stock - NEW.cantidad WHERE id = NEW.producto_id;
END //

CREATE TRIGGER trg_detalle_after_delete
AFTER DELETE ON detalle_pedidos
FOR EACH ROW
BEGIN
    UPDATE productos SET stock = stock + OLD.cantidad WHERE id = OLD.producto_id;
END //
DELIMITER ;
```
> ⚠️ Si activas el Ejemplo 3, `sp_crear_pedido` descontaría el stock **dos veces** (una en el procedimiento y otra en el *trigger*). Decide en un solo sitio quién mantiene el stock. Esto ilustra el gran peligro de los *triggers*: **lógica invisible** que sorprende a quien no sabe que existe.

### Limitaciones y administración
* Un *trigger* no puede modificar la **misma tabla** que lo disparó (salvo `NEW` en un `BEFORE`).
* No se disparan con `TRUNCATE`, ni por borrados en cascada de claves foráneas (`ON DELETE CASCADE`).
* No pueden devolver resultados ni usar `COMMIT`/`ROLLBACK` (forman parte de la transacción de la sentencia que los disparó: si el *trigger* falla, la sentencia original también se deshace).

```sql
SHOW TRIGGERS FROM tienda_dba;
SHOW CREATE TRIGGER trg_productos_audit_precio\G
DROP TRIGGER IF EXISTS trg_detalle_after_insert;
DROP TRIGGER IF EXISTS trg_detalle_after_delete;
```

---

## 6. Eventos programados (`EVENT`): el "cron" de MySQL

```sql
-- 1) El planificador debe estar encendido (ON por defecto en MySQL 8)
SHOW VARIABLES LIKE 'event_scheduler';
SET GLOBAL event_scheduler = ON;          -- Para que sea permanente: event_scheduler=ON en my.cnf

-- 2) Cancelar cada noche los pedidos pendientes con más de 30 días
CREATE EVENT IF NOT EXISTS ev_cancelar_pendientes
ON SCHEDULE EVERY 1 DAY
STARTS (CURRENT_DATE + INTERVAL 1 DAY + INTERVAL 3 HOUR)      -- Mañana a las 03:00
DO
    UPDATE pedidos
    SET estado = 'cancelado'
    WHERE estado = 'pendiente' AND fecha < NOW() - INTERVAL 30 DAY;
```

### "Vista materializada" casera: tabla de resumen recalculada cada hora
```sql
CREATE TABLE resumen_ventas_mes (
    mes        CHAR(7) PRIMARY KEY,
    pedidos    INT,
    facturado  DECIMAL(12,2),
    calculado  DATETIME
);

DELIMITER //
CREATE EVENT ev_resumen_ventas
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    REPLACE INTO resumen_ventas_mes (mes, pedidos, facturado, calculado)
    SELECT DATE_FORMAT(fecha, '%Y-%m'), COUNT(*), SUM(total), NOW()
    FROM pedidos
    WHERE estado <> 'cancelado'
    GROUP BY DATE_FORMAT(fecha, '%Y-%m');
END //
DELIMITER ;
```

### Evento de una sola vez y administración
```sql
CREATE EVENT ev_limpieza_unica
ON SCHEDULE AT NOW() + INTERVAL 10 MINUTE
ON COMPLETION NOT PRESERVE        -- Se borra solo al terminar
DO DELETE FROM auditoria_precios WHERE fecha < NOW() - INTERVAL 2 YEAR;

SHOW EVENTS FROM tienda_dba;
ALTER EVENT ev_resumen_ventas DISABLE;     -- Pausar
ALTER EVENT ev_resumen_ventas ENABLE;
ALTER EVENT ev_resumen_ventas ON SCHEDULE EVERY 30 MINUTE;
DROP EVENT IF EXISTS ev_limpieza_unica;
```
> Los errores de los eventos van al **log de errores** del servidor, no a ninguna pantalla. Revisa ese log o haz que el evento escriba en una tabla de log propia.

---

## 7. Resumen: ¿cuál uso?

| Necesito... | Usa |
| :--- | :--- |
| Reutilizar una consulta compleja u ocultar columnas | **Vista** |
| Una operación de varios pasos con parámetros (crear pedido, cierre de mes) | **Procedimiento** |
| Un cálculo reutilizable dentro de `SELECT`/`WHERE` | **Función** |
| Reaccionar **siempre** a un cambio en una tabla (auditoría, validación) | **Trigger** |
| Ejecutar algo periódicamente dentro de la base de datos | **Evento** (o `cron` del sistema operativo) |

---

## 🏋️ Ejercicios del Módulo 6

1. Crea la vista `v_ventas_por_vendedor` con nombre del vendedor, número de pedidos y total vendido (pedidos no cancelados). Los pedidos sin vendedor deben salir como `'Online'`.
2. Crea la función `fn_nivel_cliente(cliente_id)` que devuelva `'ORO'` si ha gastado más de 2.000 €, `'PLATA'` si más de 500 € y `'BRONCE'` en otro caso. Úsala para listar todos los clientes.
3. Crea el procedimiento `sp_cambiar_estado(pedido_id, nuevo_estado)` que lance un error si el pedido no existe o si se intenta pasar de `'cancelado'` a otro estado.
4. Crea un *trigger* que impida borrar empleados que tengan pedidos asignados.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
CREATE OR REPLACE VIEW v_ventas_por_vendedor AS
SELECT COALESCE(CONCAT(e.nombre, ' ', e.apellido), 'Online') AS vendedor,
       COUNT(*)     AS pedidos,
       SUM(p.total) AS vendido
FROM pedidos p
LEFT JOIN empleados e ON e.id = p.empleado_id
WHERE p.estado <> 'cancelado'
GROUP BY e.id, e.nombre, e.apellido;

SELECT * FROM v_ventas_por_vendedor ORDER BY vendido DESC;

-- 2
DELIMITER //
CREATE FUNCTION fn_nivel_cliente(p_cliente_id INT)
RETURNS VARCHAR(6)
READS SQL DATA
BEGIN
    DECLARE v_gastado DECIMAL(12,2);
    SELECT COALESCE(SUM(total), 0) INTO v_gastado
    FROM pedidos WHERE cliente_id = p_cliente_id AND estado <> 'cancelado';
    RETURN CASE
               WHEN v_gastado > 2000 THEN 'ORO'
               WHEN v_gastado > 500  THEN 'PLATA'
               ELSE 'BRONCE'
           END;
END //
DELIMITER ;

SELECT nombre, fn_nivel_cliente(id) AS nivel FROM clientes;

-- 3
DELIMITER //
CREATE PROCEDURE sp_cambiar_estado(IN p_pedido_id INT, IN p_estado VARCHAR(20))
BEGIN
    DECLARE v_actual VARCHAR(20);
    SELECT estado INTO v_actual FROM pedidos WHERE id = p_pedido_id;

    IF v_actual IS NULL THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'El pedido no existe';
    ELSEIF v_actual = 'cancelado' AND p_estado <> 'cancelado' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Un pedido cancelado no puede reactivarse';
    END IF;

    UPDATE pedidos SET estado = p_estado WHERE id = p_pedido_id;
END //
DELIMITER ;

CALL sp_cambiar_estado(9, 'pagado');      -- ✅
CALL sp_cambiar_estado(5, 'enviado');     -- ❌ Un pedido cancelado no puede reactivarse

-- 4
DELIMITER //
CREATE TRIGGER trg_empleados_before_delete
BEFORE DELETE ON empleados
FOR EACH ROW
BEGIN
    IF EXISTS (SELECT 1 FROM pedidos WHERE empleado_id = OLD.id) THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'No se puede borrar: el empleado tiene pedidos asignados';
    END IF;
END //
DELIMITER ;

DELETE FROM empleados WHERE id = 3;   -- ❌ Marta tiene pedidos
DELETE FROM empleados WHERE id = 8;   -- ✅ Diego no tiene pedidos
```
</details>

---

👉 **Siguiente paso:** **[Módulo 7: Transacciones, Bloqueos y Concurrencia](./07_TRANSACCIONES_BLOQUEOS_Y_CONCURRENCIA.md)**
