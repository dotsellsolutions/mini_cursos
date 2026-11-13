# Módulo 10: Backups, Recuperación, Replicación y Monitoreo

Este es el módulo que define a un DBA. Todo lo anterior sirve de poco si un `DELETE` sin `WHERE`, un disco que muere o un *ransomware* borran los datos y **no hay forma de recuperarlos**. Aquí aprendes a hacer copias de seguridad, a **restaurarlas** (lo que de verdad importa), a recuperar datos hasta el segundo exacto antes de un desastre, a montar réplicas y a vigilar la salud del servidor.

> 💡 **Frase de DBA:** *"Nadie quiere backups. Todo el mundo quiere restauraciones."* Un backup que nunca has probado a restaurar **no es un backup**: es una esperanza.

---

## 1. Estrategia: RPO, RTO y la regla 3-2-1

Antes de elegir herramientas, responde dos preguntas con el negocio:

| Concepto | Pregunta | Ejemplo |
| :--- | :--- | :--- |
| **RPO** (*Recovery Point Objective*) | ¿Cuántos datos podemos permitirnos **perder**? | "Como máximo 5 minutos de pedidos" → backups diarios **no** bastan: necesitas *binary logs*. |
| **RTO** (*Recovery Time Objective*) | ¿Cuánto tiempo podemos estar **caídos**? | "Máximo 1 hora" → restaurar un dump de 200 GB tarda más: necesitas backup físico o una réplica lista. |

**Regla 3-2-1:** **3** copias de los datos, en **2** soportes distintos, **1** de ellas fuera de la oficina o del proveedor (*offsite*), idealmente inmutable (no la puede borrar un atacante con acceso al servidor).

### Tipos de backup
| Tipo | Herramienta | Ventajas | Inconvenientes |
| :--- | :--- | :--- | :--- |
| **Lógico** (sentencias SQL) | `mysqldump`, `mysqlpump`, `mydumper`, MySQL Shell `util.dumpInstance()` | Portable entre versiones y sistemas, legible, permite restaurar una sola tabla | Lento de restaurar en bases grandes (re-ejecuta todos los `INSERT` y reconstruye índices) |
| **Físico** (copia de archivos) | **Percona XtraBackup**, MySQL Enterprise Backup, *snapshots* LVM/ZFS/nube | Muy rápido de hacer y restaurar en bases de cientos de GB | Misma versión de MySQL, menos portable |
| **Incremental** | XtraBackup, *binary logs* | Solo los cambios desde el último backup | La restauración encadena varios pasos |

---

## 2. Backup lógico con `mysqldump`

```bash
# Una base de datos completa (estructura + datos)
mysqldump -u backup -p --single-transaction --routines --triggers --events \
          tienda_dba > tienda_dba_2025-02-14.sql

# Varias bases de datos
mysqldump -u backup -p --single-transaction --databases tienda_dba otra_bd > varias.sql

# Todo el servidor (incluye usuarios y permisos de la base mysql)
mysqldump -u backup -p --single-transaction --all-databases --routines --events > servidor_completo.sql

# Solo algunas tablas
mysqldump -u backup -p --single-transaction tienda_dba clientes pedidos > clientes_pedidos.sql

# Solo la estructura (sin datos)
mysqldump -u backup -p --no-data tienda_dba > estructura.sql

# Solo los datos (sin CREATE TABLE)
mysqldump -u backup -p --no-create-info tienda_dba > datos.sql

# Filas concretas
mysqldump -u backup -p tienda_dba pedidos --where="fecha >= '2025-01-01'" > pedidos_2025.sql

# Comprimido al vuelo (un dump de texto se comprime un 80-90 %)
mysqldump -u backup -p --single-transaction --routines --triggers --events tienda_dba | gzip > tienda_dba.sql.gz
```

### Opciones imprescindibles
| Opción | Por qué |
| :--- | :--- |
| `--single-transaction` | **Copia consistente de InnoDB sin bloquear las tablas** (usa una transacción `REPEATABLE READ`). Imprescindible en producción. |
| `--routines` | Incluye procedimientos y funciones (por defecto **no** se incluyen). |
| `--triggers` | Incluye *triggers* (activo por defecto, pero ponlo explícito). |
| `--events` | Incluye los eventos programados (por defecto **no**). |
| `--source-data=2` (MySQL 8.0.26+; antes y en MariaDB: `--master-data=2`) | Anota en el dump la posición del *binary log* (necesario para recuperación a un punto en el tiempo y para crear réplicas). |
| `--set-gtid-purged=OFF` | Evita errores al restaurar en un servidor con GTID distinto (ver apartado 6). |
| `--hex-blob` | Exporta columnas binarias de forma segura. |
| `--default-character-set=utf8mb4` | Evita problemas de codificación. |

> 🔐 **No escribas la contraseña en la línea de comandos** (`-pMiClave` queda en el historial y es visible en `ps`). Para scripts automáticos usa un archivo de opciones protegido:
> ```ini
> # ~/.my.cnf   (chmod 600 ~/.my.cnf)
> [mysqldump]
> user=backup
> password=B4ckup!Clave#2025
> ```
> O, mejor aún en MySQL 8: `mysql_config_editor set --login-path=backup --user=backup --password` y luego `mysqldump --login-path=backup ...`.

---

## 3. Restaurar un backup lógico

```bash
# Restaurar en una base de datos (debe existir)
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS tienda_dba_restaurada"
mysql -u root -p tienda_dba_restaurada < tienda_dba_2025-02-14.sql

# Desde un archivo comprimido
gunzip < tienda_dba.sql.gz | mysql -u root -p tienda_dba_restaurada

# Desde dentro de la consola mysql
#   mysql> USE tienda_dba_restaurada;
#   mysql> SOURCE /ruta/tienda_dba_2025-02-14.sql;
```

### Restaurar UNA sola tabla de un dump completo
```bash
# Extraer el bloque de la tabla "clientes" del dump (desde su DROP TABLE hasta la siguiente tabla)
sed -n '/^-- Table structure for table `clientes`/,/^-- Table structure for table/p' tienda_dba.sql > solo_clientes.sql
```
> 💡 **Restaura siempre en una base de datos aparte** (`tienda_dba_restaurada`) y copia desde ahí solo lo que necesitas a producción. Restaurar encima de producción puede **sobrescribir** los datos buenos de las últimas horas.

### Acelerar restauraciones grandes
```sql
-- En la sesión que restaura (no en producción con tráfico):
SET SESSION foreign_key_checks = 0;
SET SESSION unique_checks = 0;
SET SESSION sql_log_bin = 0;          -- No replicar ni registrar en binlog la restauración (¡solo si sabes por qué!)
-- ... SOURCE dump.sql ...
SET SESSION foreign_key_checks = 1;
SET SESSION unique_checks = 1;
```
Para bases de datos de decenas de GB, usa herramientas paralelas: **`mydumper`/`myloader`** o **MySQL Shell** (`util.dumpSchemas()` / `util.loadDump()`), que exportan y cargan en varios hilos a la vez.

---

## 4. Backup físico con Percona XtraBackup (bases grandes)

```bash
# 1) Copia en caliente (sin parar MySQL ni bloquear InnoDB)
xtrabackup --backup --user=backup --password=*** --target-dir=/backups/full_2025-02-14

# 2) "Preparar" la copia: aplica los cambios que ocurrieron mientras se copiaba
xtrabackup --prepare --target-dir=/backups/full_2025-02-14

# 3) Restaurar: con MySQL PARADO y el datadir vacío
systemctl stop mysql
xtrabackup --copy-back --target-dir=/backups/full_2025-02-14
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```
Incrementales: `--incremental-basedir=/backups/full_...` copia solo las páginas que cambiaron desde el backup base.
> 🐬 **MariaDB** usa su propia versión: `mariabackup` (misma sintaxis).

---

## 5. *Binary logs* y recuperación a un punto en el tiempo (PITR)

El **binary log (binlog)** registra **todos los cambios** de datos (`INSERT`, `UPDATE`, `DELETE`, DDL) en orden. Sirve para dos cosas críticas:
1. **Replicación** (apartado 6).
2. **Point-In-Time Recovery:** restaurar el backup de anoche y "reproducir" los cambios hasta el segundo **anterior** al desastre.

```sql
SHOW VARIABLES LIKE 'log_bin';                 -- ON por defecto en MySQL 8
SHOW BINARY LOGS;                              -- Lista de archivos binlog y su tamaño
SHOW BINARY LOG STATUS;                        -- Archivo y posición actual (MySQL 8.2+; en 8.0 y MariaDB: SHOW MASTER STATUS)
SHOW VARIABLES LIKE 'binlog_format';           -- ROW (recomendado)
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';  -- Cuánto tiempo se conservan (por defecto 30 días)
```
> 🪟 **En WAMP el binlog suele venir desactivado** (`log_bin = OFF`, por `skip-log-bin` en `my.ini`) y `SHOW BINARY LOG STATUS` responde *"You are not using binary logging"*. Para practicar este apartado, quita `skip-log-bin`/`disable-log-bin` de `my.ini`, añade `log-bin = mysql-bin` y `server-id = 1` en `[mysqld]` y reinicia MySQL. En servidores Linux con MySQL 8 viene **activado** por defecto.

```ini
[mysqld]
server_id                  = 1
log_bin                    = /var/log/mysql/mysql-bin
binlog_format              = ROW
binlog_expire_logs_seconds = 604800     # 7 días (deben cubrir, como mínimo, el tiempo entre dos backups completos)
sync_binlog                = 1          # Durabilidad: escribe el binlog a disco en cada COMMIT
```

### 🚑 Caso real: alguien ejecutó `DELETE FROM pedidos;` a las 11:42
**Situación:** tienes un backup completo de las 03:00 (hecho con `--source-data=2`) y los binlogs del día.

```bash
# 1) Mira en el dump en qué binlog y posición se hizo el backup
grep -m1 "CHANGE REPLICATION SOURCE TO\|CHANGE MASTER TO" tienda_dba_0300.sql
#   -- CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='mysql-bin.000042', SOURCE_LOG_POS=157;

# 2) Localiza el momento exacto del DELETE en los binlogs
mysqlbinlog --base64-output=DECODE-ROWS -v \
    --start-datetime="2025-02-14 11:30:00" --stop-datetime="2025-02-14 11:50:00" \
    /var/log/mysql/mysql-bin.000043 | grep -n -B5 "DELETE FROM \`tienda_dba\`.\`pedidos\`" | head
#   # at 98765432   ← posición donde empieza la transacción del DELETE

# 3) Restaura el backup de las 03:00 en una base de datos aparte
mysql -u root -p -e "CREATE DATABASE tienda_dba_pitr"
mysql -u root -p tienda_dba_pitr < tienda_dba_0300.sql

# 4) Reproduce los cambios desde el backup HASTA justo antes del DELETE
#    --rewrite-db  : aplica los cambios en tienda_dba_pitr en vez de en tienda_dba
#    --database    : se evalúa DESPUÉS del rewrite, por eso lleva el nombre NUEVO
#    --skip-gtids  : imprescindible con GTID activo; si no, el servidor ve que esas transacciones
#                    "ya se ejecutaron" (en tienda_dba) y las SALTA sin avisar
mysqlbinlog --skip-gtids --start-position=157 /var/log/mysql/mysql-bin.000042 \
            --rewrite-db='tienda_dba->tienda_dba_pitr' --database=tienda_dba_pitr | mysql -u root -p
mysqlbinlog --skip-gtids --stop-position=98765432 /var/log/mysql/mysql-bin.000043 \
            --rewrite-db='tienda_dba->tienda_dba_pitr' --database=tienda_dba_pitr | mysql -u root -p

# 5) Verifica tienda_dba_pitr.pedidos y devuelve las filas a producción
#    INSERT INTO tienda_dba.pedidos SELECT * FROM tienda_dba_pitr.pedidos WHERE ...;
```
> ⏱️ El RPO de esta estrategia es **segundos**: solo pierdes el propio `DELETE`. Practica este procedimiento **antes** de necesitarlo: bajo presión y con el jefe detrás no es momento de leer la documentación.

---

## 6. Replicación

La replicación mantiene **copias en vivo** de la base de datos en otros servidores (réplicas), que reciben los cambios del servidor principal (*source*, antes llamado *master*) a través del binlog.

```
             escrituras                      binlog
 App ───────────────────► [ SOURCE ] ─────────────────────► [ RÉPLICA 1 ]  ← lecturas / informes
                               │                           [ RÉPLICA 2 ]  ← backups sin cargar el source
                               └────────────────────────►  [ RÉPLICA 3 ]  ← en otro centro de datos (DR)
```

**Para qué sirve:**
* **Escalar lecturas:** los informes pesados van a una réplica.
* **Alta disponibilidad / recuperación ante desastres:** si el *source* muere, se **promociona** una réplica.
* **Backups** desde una réplica sin afectar a producción.
> ⚠️ **La replicación NO es un backup.** Un `DROP TABLE` en el *source* se replica a todas las réplicas en milisegundos. (Una **réplica retrasada** con `SOURCE_DELAY = 3600` sí da una hora de margen).

### Montar una réplica con GTID (MySQL 8)
**En el *source* (`my.cnf`)**:
```ini
[mysqld]
server_id                = 1
log_bin                  = mysql-bin
binlog_format            = ROW
gtid_mode                = ON
enforce_gtid_consistency = ON
```
```sql
-- Usuario para que la réplica se conecte
CREATE USER 'replicador'@'10.0.0.%' IDENTIFIED BY 'R3plica!Clave#2025';
GRANT REPLICATION SLAVE ON *.* TO 'replicador'@'10.0.0.%';
```
**En la réplica (`my.cnf`)**:
```ini
[mysqld]
server_id                = 2          # ÚNICO en todo el grupo
gtid_mode                = ON
enforce_gtid_consistency = ON
read_only                = ON
super_read_only          = ON         # Ni siquiera root puede escribir por accidente
relay_log                = relay-bin
```
```bash
# 1) Copia inicial de los datos del source a la réplica
mysqldump -h 10.0.0.1 -u backup -p --single-transaction --source-data=1 --routines --triggers --events \
          --all-databases > inicial.sql
mysql -u root -p < inicial.sql           # (en la réplica)
```
```sql
-- 2) En la réplica: apuntar al source y arrancar (MySQL 8.0.23+)
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.0.1',
    SOURCE_USER = 'replicador',
    SOURCE_PASSWORD = 'R3plica!Clave#2025',
    SOURCE_AUTO_POSITION = 1,            -- Con GTID no hace falta indicar archivo/posición
    SOURCE_SSL = 1;

START REPLICA;
```
> Sintaxis antigua (MySQL < 8.0.22 y MariaDB): `CHANGE MASTER TO MASTER_HOST=...`, `START SLAVE`, `SHOW SLAVE STATUS`.

### Monitorear la replicación
```sql
SHOW REPLICA STATUS\G
```
| Campo | Valor sano |
| :--- | :--- |
| `Replica_IO_Running` | `Yes` (recibe el binlog) |
| `Replica_SQL_Running` | `Yes` (aplica los cambios) |
| `Seconds_Behind_Source` | `0` o pocos segundos (**retraso**) |
| `Last_IO_Error` / `Last_SQL_Error` | vacíos |

Problemas típicos:
* **Retraso creciente:** la réplica no da abasto (consultas pesadas en ella, disco lento, transacciones enormes en el *source*). Activa la aplicación paralela: `replica_parallel_workers = 4`.
* **`Replica_SQL_Running: No` por un error** (ej. clave duplicada porque alguien escribió directamente en la réplica): investiga la causa; la "solución" de saltarse la transacción hace que la réplica **deje de ser idéntica**. Por eso `super_read_only = ON`.

### Topologías y alta disponibilidad
| Solución | Descripción |
| :--- | :--- |
| Source → réplicas (asíncrona) | La clásica. Puede perder las últimas transacciones si el *source* muere. |
| Semisíncrona | El *source* espera a que al menos una réplica reciba el cambio antes de confirmar. |
| **MySQL InnoDB Cluster / Group Replication** | Grupo de 3+ nodos con elección automática de nuevo primario. Con **MySQL Router** la app no se entera del cambio. |
| **Galera Cluster** (MariaDB, Percona XtraDB Cluster) | Multi-primario síncrono. |
| Servicios gestionados | Amazon RDS/Aurora, Google Cloud SQL, Azure: replicación, backups y *failover* automáticos. |

---

## 7. Monitoreo: la salud del servidor

### A) ¿Qué está pasando AHORA?
```sql
SHOW FULL PROCESSLIST;

-- Versión filtrable: consultas activas (no dormidas) que llevan más de 5 segundos
SELECT id, user, host, db, command, time AS segundos, state, LEFT(info, 120) AS consulta
FROM information_schema.processlist
WHERE command <> 'Sleep' AND time > 5
ORDER BY time DESC;

-- Conexiones por usuario y origen
SELECT user, SUBSTRING_INDEX(host, ':', 1) AS origen, COUNT(*) AS conexiones
FROM information_schema.processlist
GROUP BY user, origen
ORDER BY conexiones DESC;
```

### B) Variables de estado clave (`SHOW GLOBAL STATUS`)
```sql
SHOW GLOBAL STATUS LIKE 'Uptime';                     -- Segundos encendido
SHOW GLOBAL STATUS LIKE 'Threads_connected';          -- Conexiones abiertas ahora
SHOW GLOBAL STATUS LIKE 'Max_used_connections';       -- Pico histórico (compáralo con max_connections)
SHOW GLOBAL STATUS LIKE 'Aborted_connects';           -- Intentos de conexión fallidos (¿contraseñas mal? ¿ataque?)
SHOW GLOBAL STATUS LIKE 'Slow_queries';               -- Consultas lentas registradas
SHOW GLOBAL STATUS LIKE 'Questions';                  -- Consultas totales (divídelo por Uptime = QPS medio)
SHOW GLOBAL STATUS LIKE 'Created_tmp_disk_tables';    -- Tablas temporales que tuvieron que ir a disco
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_waits';      -- Esperas por bloqueos

-- Deadlocks totales desde el arranque (MySQL: contador de innodb_metrics; MariaDB: SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks')
SELECT name, count FROM information_schema.innodb_metrics WHERE name = 'lock_deadlocks';
```

### C) La métrica reina: *buffer pool hit ratio*
El **buffer pool** es la memoria donde InnoDB guarda las páginas de datos e índices. Si los datos "calientes" caben en él, casi no se lee del disco.
```sql
SELECT ROUND(100 * (1 - (
         (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_reads') /
         (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Innodb_buffer_pool_read_requests')
       )), 2) AS hit_ratio_pct;
-- Ideal: > 99 %. Si es menor, el buffer pool es pequeño para tus datos.

SELECT @@innodb_buffer_pool_size / 1024 / 1024 / 1024 AS buffer_pool_gb;
```

### D) Espacio en disco
```sql
-- Tamaño por base de datos
SELECT table_schema AS base_de_datos,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) AS tamano_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY tamano_mb DESC;

-- Las 10 tablas más grandes del servidor
SELECT table_schema, table_name, table_rows,
       ROUND((data_length + index_length) / 1024 / 1024, 1) AS total_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
ORDER BY data_length + index_length DESC
LIMIT 10;
```
> 🚨 Un disco lleno **detiene** MySQL. Los culpables habituales: binlogs que no caducan, *slow/general log* sin rotar, tablas de logs que nadie purga y `ibdata1`/*undo* crecidos por transacciones larguísimas.

### E) El esquema `sys`: informes listos para usar (MySQL 8)
```sql
SELECT * FROM sys.host_summary;                       -- Actividad por host cliente
SELECT * FROM sys.user_summary;                       -- Actividad por usuario
SELECT * FROM sys.schema_table_statistics LIMIT 10;   -- Tablas con más lecturas/escrituras y su latencia
SELECT * FROM sys.io_global_by_file_by_bytes LIMIT 10;-- Archivos con más E/S
SELECT * FROM sys.memory_global_by_current_bytes LIMIT 10;  -- Uso de memoria por componente
SELECT * FROM sys.schema_auto_increment_columns       -- ⚠️ Columnas AUTO_INCREMENT cerca de su máximo
WHERE auto_increment_ratio > 0.5;
```
> El último es un clásico: un `INT` con signo llega a 2.147.483.647 y a partir de ahí **todos los `INSERT` fallan**. Detéctalo meses antes y migra a `BIGINT`.

### F) Herramientas de monitoreo continuo
| Herramienta | Tipo |
| :--- | :--- |
| **Percona Monitoring and Management (PMM)** | Gratuita, muy completa para MySQL/MariaDB/PostgreSQL |
| **Prometheus + mysqld_exporter + Grafana** | Estándar *open source* |
| **Zabbix**, **Datadog**, **New Relic** | Monitoreo general con plantillas MySQL |
| **MySQL Enterprise Monitor** | Comercial de Oracle |
| **Laravel Pulse / Telescope** | Si la app es Laravel ([Curso Laravel, Módulo 10](../Curso-Laravel-API/10_BUENAS_PRACTICAS_Y_DESPLIEGUE_A_PRODUCCION.md)) |

---

## 8. Configuración del servidor (`my.cnf` / `my.ini`)

Ubicación: Linux `/etc/mysql/my.cnf` o `/etc/mysql/mysql.conf.d/mysqld.cnf`; WAMP `C:\wamp64\bin\mysql\mysqlX\my.ini`. Tras editar: `sudo systemctl restart mysql`.

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';       -- Ver un valor
SET PERSIST max_connections = 300;                    -- MySQL 8: cambia en caliente Y lo guarda en mysqld-auto.cnf
SELECT * FROM performance_schema.persisted_variables; -- Qué se ha cambiado con SET PERSIST
```

### Parámetros que todo DBA ajusta
```ini
[mysqld]
# --- Memoria ---
innodb_buffer_pool_size        = 6G      # 50-75 % de la RAM en un servidor DEDICADO a MySQL
innodb_buffer_pool_instances   = 4       # (MySQL < 8.4) 1 por cada GB, hasta 8

# --- Redo log / durabilidad ---
innodb_redo_log_capacity       = 2G      # MySQL 8.0.30+ (antes innodb_log_file_size)
innodb_flush_log_at_trx_commit = 1       # 1 = ACID total. 2 = más rápido, puede perder ~1 s si se cae el SO
sync_binlog                    = 1

# --- Conexiones ---
max_connections                = 300     # No lo subas "por si acaso": cada conexión consume memoria
wait_timeout                   = 600     # Cierra conexiones inactivas tras 10 minutos

# --- Tablas temporales ---
tmp_table_size                 = 64M
max_heap_table_size            = 64M

# --- Logs ---
slow_query_log                 = 1
long_query_time                = 1
log_error                      = /var/log/mysql/error.log

# --- Comportamiento estándar (Módulo 1) ---
default-storage-engine         = InnoDB
sql_mode                       = "ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"
character-set-server           = utf8mb4
collation-server               = utf8mb4_0900_ai_ci
```
> 🧰 **MySQLTuner** (`perl mysqltuner.pl`) analiza un servidor que lleva al menos 24 h funcionando y sugiere ajustes. Tómalo como pista, no como verdad absoluta.

---

## 9. Automatizar los backups (script + cron)

```bash
#!/bin/bash
# /usr/local/bin/backup_mysql.sh
set -euo pipefail

FECHA=$(date +%F_%H%M)
DESTINO=/backups/mysql
DIAS_RETENCION=14
mkdir -p "$DESTINO"

# 1) Backup de cada base de datos de usuario por separado (más fácil restaurar una sola)
for BD in $(mysql --login-path=backup -N -e "SHOW DATABASES" \
            | grep -Ev '^(information_schema|performance_schema|sys|mysql)$'); do
    mysqldump --login-path=backup --single-transaction --routines --triggers --events \
              --source-data=2 --set-gtid-purged=OFF "$BD" \
      | gzip > "$DESTINO/${BD}_${FECHA}.sql.gz"
done

# 2) Usuarios y permisos como sentencias CREATE USER + GRANT (más portable que copiar la base "mysql")
mysql --login-path=backup -N -e "SELECT CONCAT('SHOW CREATE USER ', QUOTE(user), '@', QUOTE(host), '; SHOW GRANTS FOR ', QUOTE(user), '@', QUOTE(host), ';') FROM mysql.user WHERE user NOT LIKE 'mysql.%'" \
  | mysql --login-path=backup -N | sed 's/$/;/' | gzip > "$DESTINO/usuarios_${FECHA}.sql.gz"

# 3) Verificar que los archivos no están vacíos ni corruptos
for F in "$DESTINO"/*_"$FECHA".sql.gz; do
    gzip -t "$F" || { echo "❌ Backup corrupto: $F"; exit 1; }
done

# 4) Copia fuera del servidor (regla 3-2-1): S3, Backblaze B2, otro servidor...
# rclone copy "$DESTINO" remoto:backups-mysql --include "*_${FECHA}.sql.gz"

# 5) Borrar backups antiguos
find "$DESTINO" -name "*.sql.gz" -mtime +$DIAS_RETENCION -delete

echo "✅ Backup terminado: $FECHA"
```
```bash
sudo chmod 700 /usr/local/bin/backup_mysql.sh
sudo crontab -e
# Todos los días a las 03:00, guardando el resultado en un log
0 3 * * * /usr/local/bin/backup_mysql.sh >> /var/log/backup_mysql.log 2>&1
```
> 🧪 **Prueba de restauración periódica:** programa (al menos una vez al mes) un script que restaure el último backup en un servidor de pruebas y compruebe el número de filas de las tablas principales. Si falla, que te avise. Es la única forma de **saber** que tus backups sirven.

---

## 10. Mantenimiento y actualizaciones

### Tareas periódicas del DBA
| Frecuencia | Tarea |
| :--- | :--- |
| **Diaria** | Revisar que el backup terminó bien · Revisar el log de errores · Estado de la replicación · Espacio en disco |
| **Semanal** | Top de consultas lentas (`pt-query-digest`) · Conexiones fallidas · Crecimiento de tablas |
| **Mensual** | **Prueba de restauración** · Revisar usuarios y permisos · Índices sin uso/redundantes · `AUTO_INCREMENT` cerca del límite · Purgar tablas de logs |
| **Trimestral** | Revisar configuración y capacidad (¿cabe el *working set* en el buffer pool?) · Parches de seguridad |

### Actualizar MySQL (por ejemplo 8.0 → 8.4 LTS)
1. Lee las **notas de la versión** (funciones eliminadas, cambios de comportamiento, `mysql_native_password`...).
2. Ejecuta el comprobador de MySQL Shell: `util.checkForServerUpgrade('root@localhost:3306')`.
3. **Backup completo** y prueba la restauración.
4. Prueba la aplicación contra la nueva versión en un entorno de *staging*.
5. Con replicación: actualiza primero las **réplicas**, promociona una y después actualiza el antiguo *source*.
6. Desde MySQL 8.0.16 el *upgrade* de las tablas del sistema es automático al arrancar la nueva versión; revisa el log de errores.

---

## 11. 🚑 Guía rápida de emergencias

| Síntoma | Primeros pasos |
| :--- | :--- |
| **"Too many connections"** | `SHOW PROCESSLIST` (MySQL reserva 1 conexión extra para usuarios con privilegio de admin) → ¿consultas bloqueadas? ¿la app no cierra conexiones? → `KILL` a las ociosas → revisar *pool* de la aplicación. |
| **El servidor va lento de repente** | `SHOW PROCESSLIST` → consultas largas y en qué `state` → `sys.innodb_lock_waits` → `EXPLAIN` de la sospechosa → ¿alguien lanzó un informe enorme o un `ALTER TABLE`? |
| **"Waiting for table metadata lock"** | Un `ALTER TABLE` espera a una transacción abierta: `sys.schema_table_lock_waits` → localiza y cierra la transacción. |
| **Disco lleno** | `df -h` → `du -sh /var/lib/mysql/* /var/log/mysql/*` → `PURGE BINARY LOGS BEFORE NOW() - INTERVAL 3 DAY;` (¡comprueba antes que las réplicas no los necesitan!) → rotar logs. |
| **MySQL no arranca** | Lee `log_error` (las últimas líneas dicen el motivo: permisos, disco, configuración inválida, corrupción). |
| **Borrado accidental de datos** | **No reinicies nada.** Detén las escrituras de la app si es posible → PITR (apartado 5) en una base aparte. |
| **Réplica parada** | `SHOW REPLICA STATUS\G` → `Last_SQL_Error` → corrige la causa → `START REPLICA;`. |
| **Sospecha de corrupción InnoDB** | `CHECK TABLE` → log de errores → restaurar desde backup. `innodb_force_recovery` (1-6) solo para extraer datos, en último recurso. |

---

## 🏋️ Ejercicios del Módulo 10

1. Escribe el comando para hacer un backup **consistente y comprimido** de `tienda_dba` que incluya procedimientos, *triggers* y eventos, y el comando para restaurarlo en una base nueva `tienda_dba_prueba`.
2. Haz el backup del ejercicio 1, **bórralo todo** (`DROP DATABASE tienda_dba;`), restáuralo y comprueba con `COUNT(*)` que las 7 tablas tienen las mismas filas que al principio.
3. Escribe una consulta que muestre las 5 conexiones activas que más tiempo llevan ejecutando una consulta, con su usuario y los primeros 100 caracteres de la consulta.
4. Escribe una consulta que calcule el tamaño de cada tabla de `tienda_dba` en KB, ordenado de mayor a menor, incluyendo el número aproximado de filas.
5. Con `mysqlbinlog`, ¿qué opciones usarías para ver en texto legible los cambios de fila ocurridos entre las 10:00 y las 10:15 de hoy?

<details>
<summary>👀 Ver soluciones</summary>

```bash
# 1
mysqldump -u root -p --single-transaction --routines --triggers --events tienda_dba | gzip > tienda_dba.sql.gz
mysql -u root -p -e "CREATE DATABASE tienda_dba_prueba CHARACTER SET utf8mb4"
gunzip < tienda_dba.sql.gz | mysql -u root -p tienda_dba_prueba

# 2 (Windows/WAMP sin gzip: sin comprimir)
mysqldump -u root -p --single-transaction --routines --triggers --events tienda_dba > tienda_dba.sql
mysql -u root -p -e "DROP DATABASE tienda_dba; CREATE DATABASE tienda_dba CHARACTER SET utf8mb4"
mysql -u root -p tienda_dba < tienda_dba.sql
# y luego el SELECT ... UNION ALL de COUNT(*) del índice del curso

# 5
mysqlbinlog --base64-output=DECODE-ROWS -v \
    --start-datetime="$(date +%F) 10:00:00" --stop-datetime="$(date +%F) 10:15:00" \
    /var/log/mysql/mysql-bin.0000NN
```
```sql
-- 3
SELECT id, user, time AS segundos, state, LEFT(info, 100) AS consulta
FROM information_schema.processlist
WHERE command = 'Query' AND id <> CONNECTION_ID()
ORDER BY time DESC
LIMIT 5;

-- 4
SELECT table_name, table_rows,
       ROUND((data_length + index_length) / 1024, 1) AS tamano_kb
FROM information_schema.tables
WHERE table_schema = 'tienda_dba'
ORDER BY data_length + index_length DESC;
```
</details>

---

## 🎓 Cierre del Curso: El Mapa del DBA

```
                  ┌────────────────────────────────────────────────┐
                  │                 LENGUAJE SQL                    │
                  │  DDL (M1) · CRUD (M2) · Funciones y GROUP BY (M3)│
                  │  JOINs (M4) · CTE, ventanas, JSON (M5)          │
                  └───────────────────────┬────────────────────────┘
                                          │
             ┌────────────────────────────┼────────────────────────────┐
             ▼                            ▼                            ▼
   ┌───────────────────┐       ┌────────────────────┐       ┌────────────────────┐
   │  LÓGICA EN LA BD  │       │    RENDIMIENTO     │       │ SEGURIDAD Y OPERAC.│
   │ Vistas, SP, funcs,│       │ Transacciones (M7) │       │ Usuarios/roles (M9)│
   │ triggers, eventos │       │ Índices, EXPLAIN,  │       │ Backups, PITR,     │
   │       (M6)        │       │ slow log (M8)      │       │ réplicas, monitoreo│
   └───────────────────┘       └────────────────────┘       │       (M10)        │
                                                            └────────────────────┘
```

### ¿Y ahora qué?
1. **Practica en la consola**, no solo en herramientas gráficas: en un servidor caído por SSH no habrá phpMyAdmin.
2. Monta en tu PC un **source + réplica** con Docker ([instalar-y-configurar-docker-en-windows.md](../instalar-y-configurar-docker-en-windows.md)) y rompe cosas a propósito: para la réplica, llena el disco, haz un `DELETE` y recupéralo con PITR.
3. Aprende **PostgreSQL**: el 80 % de este curso se aplica igual, y es el otro gran motor *open source*.
4. Certificaciones: *Oracle Certified Professional, MySQL 8.0 Database Administrator*; *MariaDB Certified DBA*.

¡Felicidades! 🎉 Ya conoces los comandos y las responsabilidades de un Administrador de Bases de Datos.
