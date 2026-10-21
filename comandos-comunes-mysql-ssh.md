# Guía Práctica: Comandos Más Comunes para MySQL vía SSH (Terminal Linux)

Esta guía recopila los comandos esenciales para administrar, consultar, respaldar y optimizar bases de datos **MySQL / MariaDB** directamente desde la terminal por SSH en tu servidor VPS.

Cada comando incluye una **explicación detallada de sus parámetros y banderas** para que domines la línea de comandos.

---

## 📋 Tabla de Contenidos
1. [Conexión y Autenticación](#1-conexión-y-autenticación)
2. [Gestión de Bases de Datos](#2-gestión-de-bases-de-datos)
3. [Gestión e Inspección de Tablas](#3-gestión-e-inspección-de-tablas)
4. [Consultas Básicas y el Truco del Formato Vertical (`\G`)](#4-consultas-básicas-y-el-truco-del-formato-vertical-g)
5. [Respaldos y Restauraciones (`mysqldump` e Importación)](#5-respaldos-y-restauraciones-mysqldump-e-importación)
6. [Gestión de Usuarios y Permisos](#6-gestión-de-usuarios-y-permisos)
7. [Monitoreo, Procesos Colgados y Rendimiento](#7-monitoreo-procesos-colgados-y-rendimiento)
8. [Gestión del Servicio MySQL en el VPS (Linux)](#8-gestión-del-servicio-mysql-en-el-vps-linux)
9. [Tabla Rápida / Cheat Sheet](#9-tabla-rápida--cheat-sheet)

---

## 1. Conexión y Autenticación

### 1.1 Entrar a la consola interactiva de MySQL
```bash
mysql -u root -p
```
- `-u root`: Especifica el usuario de MySQL con el que te autenticas.
- `-p`: Solicita ingresar la contraseña de forma oculta en la terminal.

### 1.2 Entrar conectándose a un Host o Puerto específico (TCP)
```bash
mysql -u root -h 127.0.0.1 -P 3306 -p
```
- `-h 127.0.0.1`: Conecta vía red local TCP/IP en lugar del socket Unix predeterminado.
- `-P 3306`: Especifica el puerto (en mayúscula `-P`, para no confundir con `-p` de password).

### 1.3 Entrar directamente seleccionando una Base de Datos
```bash
mysql -u usuario -p nombre_de_tu_bd
```

### 1.4 Ejecutar un comando SQL sin entrar a la consola interactiva
Ideal para scripts bash o consultas rápidas:
```bash
mysql -u root -p -e "SHOW DATABASES;"
```
- `-e "..."` *(execute)*: Ejecuta la consulta SQL entre comillas y devuelve el resultado inmediatamente a la terminal de Linux.

### 1.5 Salir de la consola de MySQL
```sql
EXIT;
-- O también:
QUIT;
-- O el atajo rápido:
\q
```

---

## 2. Gestión de Bases de Datos

Una vez dentro del prompt `mysql>`:

### 2.1 Listar todas las bases de datos existentes
```sql
SHOW DATABASES;
```

### 2.2 Crear una nueva base de datos (con codificación UTF-8 recomendada)
```sql
CREATE DATABASE mi_proyecto CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
- `CHARACTER SET utf8mb4`: Permite almacenar cualquier caracter internacional y emojis sin corromper el texto.
- `COLLATE utf8mb4_unicode_ci`: Define las reglas de comparación y ordenamiento insensible a mayúsculas/minúsculas.

### 2.3 Seleccionar una base de datos para trabajar en ella
```sql
USE mi_proyecto;
```

### 2.4 Ver qué base de datos tienes seleccionada actualmente
```sql
SELECT DATABASE();
```

### 2.5 Eliminar una base de datos por completo
```sql
DROP DATABASE IF EXISTS nombre_bd_antigua;
```
- `IF EXISTS`: Evita que MySQL arroje un error fatal si la base de datos ya no existe.

---

## 3. Gestión e Inspección de Tablas

### 3.1 Listar todas las tablas de la base de datos actual
```sql
SHOW TABLES;
```

### 3.2 Ver la estructura de una tabla (columnas, tipos, nulos, llaves)
```sql
DESCRIBE students;
-- O la versión abreviada:
DESC students;
```

### 3.3 Ver el comando SQL exacto con el que se creó una tabla
Muy útil para revisar índices, claves foráneas (`FOREIGN KEY`) y valores por defecto:
```sql
SHOW CREATE TABLE students\G
```

### 3.4 Vaciar todos los datos de una tabla (reiniciando el autoincrementable)
```sql
TRUNCATE TABLE logs;
```

### 3.5 Eliminar una tabla
```sql
DROP TABLE IF EXISTS tabla_temporal;
```

---

## 4. Consultas Básicas y el Truco del Formato Vertical (`\G`)

### 4.1 El truco de oro en terminal: Formato vertical con `\G`
En terminales estrechas, una consulta `SELECT *` con muchas columnas se descuadra y resulta ilegible. Si terminas la consulta con **`\G`** (en lugar de punto y coma `;`), MySQL muestra cada fila en formato vertical:

```sql
SELECT * FROM users WHERE id = 1\G
```

**Resultado visual en terminal:**
```text
*************************** 1. row ***************************
           id: 1
         name: Juan Perez
        email: juan@example.com
      created: 2026-08-10 20:00:00
```

### 4.2 Consultas rápidas con límite
```sql
-- Ver los últimos 5 registros insertados
SELECT id, name, created_at FROM students ORDER BY id DESC LIMIT 5;

-- Contar total de registros
SELECT COUNT(*) FROM enrollments;
```

---

## 5. Respaldos y Restauraciones (`mysqldump` e Importación)

Estos comandos se ejecutan en la **terminal de Linux (Bash)**, NO dentro de la consola de MySQL.

---

### 5.1 Exportar / Respaldar una Base de Datos (Backup)
```bash
mysqldump -u usuario -p nombre_bd > respaldo_$(date +%F).sql
```
- `mysqldump`: Herramienta de volcado de datos de MySQL.
- `>`: Redirige la salida a un archivo `.sql`.
- `$(date +%F)`: Añade automáticamente la fecha actual al nombre del archivo (ej: `respaldo_2026-08-11.sql`).

### 5.2 Exportar y comprimir al vuelo (Ahorra hasta un 80% de espacio)
```bash
mysqldump -u usuario -p nombre_bd | gzip > respaldo.sql.gz
```
- `| gzip`: Comprime el flujo de datos en tiempo real antes de escribirlo en el disco.

### 5.3 Exportar solo la estructura (sin datos)
```bash
mysqldump -u usuario -p --no-data nombre_bd > estructura_tablas.sql
```
- `--no-data`: Exporta solo las sentencias `CREATE TABLE` sin los `INSERT`.

### 5.4 Exportar TODAS las bases de datos del servidor
```bash
mysqldump -u root -p --all-databases > backup_completo_servidor.sql
```

---

### 5.5 Importar / Restaurar una Base de Datos

Antes de importar, asegúrate de haber creado la base de datos vacía (`CREATE DATABASE nombre_bd;`).

**Para restaurar un archivo `.sql` plano:**
```bash
mysql -u usuario -p nombre_bd < respaldo.sql
```
- `<`: Lee el archivo `.sql` e introduce cada instrucción dentro de MySQL.

**Para restaurar un archivo comprimido `.sql.gz` directamente:**
```bash
gunzip < respaldo.sql.gz | mysql -u usuario -p nombre_bd
```

---

## 6. Gestión de Usuarios y Permisos

### 6.1 Listar todos los usuarios y sus hosts permitidos
```sql
SELECT User, Host FROM mysql.user;
```

### 6.2 Crear un nuevo usuario
```sql
CREATE USER 'mi_app_user'@'localhost' IDENTIFIED BY 'ClaveMuySegura_2026!';
```
- `'localhost'`: Permite que este usuario se conecte únicamente desde dentro del mismo VPS.
- *Si necesitas permitir conexión desde cualquier IP externa, usa `'mi_app_user'@'%'`.*

### 6.3 Otorgar permisos totales sobre una base de datos específica
```sql
GRANT ALL PRIVILEGES ON mi_proyecto.* TO 'mi_app_user'@'localhost';
FLUSH PRIVILEGES;
```
- `mi_proyecto.*`: Otorga permisos en todas las tablas de esa base de datos.
- `FLUSH PRIVILEGES`: Aplica de inmediato los cambios de permisos en la memoria del servidor.

### 6.4 Ver los permisos que tiene un usuario
```sql
SHOW GRANTS FOR 'mi_app_user'@'localhost';
```

### 6.5 Cambiar la contraseña de un usuario
```sql
ALTER USER 'mi_app_user'@'localhost' IDENTIFIED BY 'NuevaClaveSegura2026!';
FLUSH PRIVILEGES;
```

### 6.6 Eliminar un usuario
```sql
DROP USER IF EXISTS 'mi_app_user'@'localhost';
```

---

## 7. Monitoreo, Procesos Colgados y Rendimiento

### 7.1 Ver todas las consultas activas en tiempo real
```sql
SHOW FULL PROCESSLIST;
```
- Muestra el `Id`, usuario, tiempo de ejecución en segundos (`Time`), estado y la consulta SQL exacta que se está ejecutando.

### 7.2 Matar un proceso o consulta bloqueada
Si una consulta pesada está congelando el servidor o consumiendo el 100% de la CPU:
```sql
KILL 1482;
```
*(Donde `1482` es el número de `Id` que obtuviste con `SHOW PROCESSLIST`).*

### 7.3 Ver el tamaño en Megabytes (MB) de cada una de tus bases de datos
Pega esta consulta para ver cuánto espacio en disco ocupa cada base de datos:

```sql
SELECT 
    table_schema AS "Base de Datos", 
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS "Tamaño en MB" 
FROM information_schema.TABLES 
GROUP BY table_schema;
```

---

## 8. Gestión del Servicio MySQL en el VPS (Linux)

Comandos para controlar el demonio de MySQL desde la terminal de Linux:

```bash
# Ver si MySQL está activo y corriendo en verde (active / running)
sudo systemctl status mysql

# Reiniciar el servicio (tras modificar archivos .cnf)
sudo systemctl restart mysql

# Detener el servicio
sudo systemctl stop mysql

# Iniciar el servicio
sudo systemctl start mysql

# Ver los últimos logs y errores en tiempo real
sudo tail -f /var/log/mysql/error.log
```

---

## 9. Tabla Rápida / Cheat Sheet

| Tarea | Comando |
| :--- | :--- |
| **Entrar a MySQL** | `mysql -u root -p` |
| **Listar bases de datos** | `SHOW DATABASES;` |
| **Seleccionar base de datos** | `USE nombre_bd;` |
| **Listar tablas** | `SHOW TABLES;` |
| **Ver columnas de tabla** | `DESCRIBE nombre_tabla;` |
| **Consulta en vertical** | `SELECT * FROM tabla LIMIT 1\G` |
| **Hacer Backup** | `mysqldump -u root -p bd > backup.sql` |
| **Restaurar Backup** | `mysql -u root -p bd < backup.sql` |
| **Ver procesos activos** | `SHOW FULL PROCESSLIST;` |
| **Matar consulta lenta** | `KILL <id_proceso>;` |
| **Reiniciar MySQL** | `sudo systemctl restart mysql` |
