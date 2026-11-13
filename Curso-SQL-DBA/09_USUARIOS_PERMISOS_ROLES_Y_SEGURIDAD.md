# Módulo 9: Usuarios, Permisos, Roles y Seguridad

Una base de datos con datos de clientes es uno de los activos más atacados de cualquier empresa. El DBA decide **quién puede conectarse, desde dónde y qué puede hacer**. La regla que guía todo este módulo es el **principio de mínimo privilegio**: cada usuario o aplicación tiene **solo** los permisos que necesita, ni uno más.

---

## 1. Cómo identifica MySQL a un usuario: `'usuario'@'host'`

En MySQL un usuario **no** es solo un nombre: es la combinación **nombre + desde dónde se conecta**. `'ana'@'localhost'` y `'ana'@'%'` son **dos usuarios distintos**, con contraseñas y permisos distintos.

| Host | Significa |
| :--- | :--- |
| `'localhost'` | Solo desde la misma máquina (socket o 127.0.0.1) |
| `'192.168.1.50'` | Solo desde esa IP |
| `'192.168.1.%'` | Desde cualquier IP de esa red |
| `'%.miempresa.com'` | Desde cualquier equipo de ese dominio (requiere resolución DNS) |
| `'%'` | **Desde cualquier lugar.** Evítalo para usuarios con permisos altos. |

```sql
-- ¿Quién soy?
SELECT USER(), CURRENT_USER();

-- Todos los usuarios del servidor
SELECT user, host, account_locked, password_expired, plugin
FROM mysql.user
ORDER BY user, host;
```

---

## 2. Crear, modificar y eliminar usuarios

```sql
-- Usuario para la aplicación web (solo desde el propio servidor)
CREATE USER 'app_tienda'@'localhost' IDENTIFIED BY 'Cambia3sta!Clave#2025';

-- Usuario de informes, desde la red interna, que debe cambiar la clave al primer acceso
CREATE USER 'reportes'@'192.168.1.%' IDENTIFIED BY 'Temporal#2025' PASSWORD EXPIRE;

-- Usuario con límites de recursos
CREATE USER 'api_publica'@'%' IDENTIFIED BY 'Otra!Clave$Segura9'
    WITH MAX_QUERIES_PER_HOUR 5000
         MAX_USER_CONNECTIONS 20;

-- Cambiar contraseña
ALTER USER 'app_tienda'@'localhost' IDENTIFIED BY 'NuevaClave!Segura#2026';

-- Bloquear / desbloquear (sin borrarlo: útil cuando alguien deja la empresa)
ALTER USER 'reportes'@'192.168.1.%' ACCOUNT LOCK;
ALTER USER 'reportes'@'192.168.1.%' ACCOUNT UNLOCK;

-- Renombrar
RENAME USER 'api_publica'@'%' TO 'api_publica'@'10.0.0.%';

-- Eliminar
DROP USER IF EXISTS 'api_publica'@'10.0.0.%';
```
> 🔐 Para cambiar **tu propia** contraseña: `ALTER USER USER() IDENTIFIED BY '...';`.
> El tutorial [cambiar-contrasena-root-mysql-vps.md](../cambiar-contrasena-root-mysql-vps.md) explica cómo recuperar el acceso de `root` si se pierde la contraseña.

### Plugins de autenticación
| Plugin | Notas |
| :--- | :--- |
| `caching_sha2_password` | Por defecto en MySQL 8+. Seguro. Algunos clientes antiguos (PHP < 7.4, conectores viejos) no lo soportan. |
| `mysql_native_password` | El de MySQL 5.x. Obsoleto: desactivado por defecto en MySQL 8.4 y eliminado en 9.0. |
| `auth_socket` / `unix_socket` | Sin contraseña: autentica con el usuario del sistema operativo (típico de `root` en Ubuntu/MariaDB). |

---

## 3. Permisos: `GRANT` y `REVOKE`

```text
GRANT privilegios ON ámbito TO 'usuario'@'host';
REVOKE privilegios ON ámbito FROM 'usuario'@'host';
```

### Niveles (ámbitos) de permisos
| Ámbito | Sintaxis | Ejemplo |
| :--- | :--- | :--- |
| Global (todo el servidor) | `ON *.*` | Administradores |
| Base de datos | `ON tienda_dba.*` | La aplicación |
| Tabla | `ON tienda_dba.productos` | Un servicio que solo gestiona el catálogo |
| Columna | `GRANT SELECT (id, nombre, email) ON tienda_dba.clientes` | Ocultar columnas sensibles |
| Rutina | `ON PROCEDURE tienda_dba.sp_crear_pedido` | Solo ejecutar un procedimiento |

### Privilegios más comunes
| Privilegio | Permite |
| :--- | :--- |
| `SELECT`, `INSERT`, `UPDATE`, `DELETE` | Leer y modificar datos |
| `CREATE`, `ALTER`, `DROP`, `INDEX` | Cambiar la estructura |
| `CREATE VIEW`, `SHOW VIEW` | Vistas |
| `CREATE ROUTINE`, `ALTER ROUTINE`, `EXECUTE` | Procedimientos y funciones |
| `TRIGGER`, `EVENT` | *Triggers* y eventos |
| `LOCK TABLES`, `REFERENCES` | Bloquear tablas, crear claves foráneas |
| `PROCESS` | Ver las consultas de todos (`SHOW PROCESSLIST`) |
| `RELOAD` | `FLUSH ...` |
| `REPLICATION SLAVE`, `REPLICATION CLIENT` | Replicación y su monitoreo |
| `CREATE USER` | Gestionar usuarios |
| `SUPER` / privilegios dinámicos (`SYSTEM_VARIABLES_ADMIN`, `CONNECTION_ADMIN`...) | Administración del servidor |
| `ALL PRIVILEGES` | Todo (en el ámbito indicado) |
| `GRANT OPTION` | Poder dar a otros los permisos que uno tiene |

### Ejemplos reales de mínimo privilegio
```sql
-- 1) La aplicación web: solo datos (nunca DROP, ALTER, ni acceso a otras bases de datos)
GRANT SELECT, INSERT, UPDATE, DELETE ON tienda_dba.* TO 'app_tienda'@'localhost';
GRANT EXECUTE ON tienda_dba.* TO 'app_tienda'@'localhost';   -- Si usa procedimientos

-- 2) Usuario de informes: solo lectura
GRANT SELECT ON tienda_dba.* TO 'reportes'@'192.168.1.%';

-- 3) Lectura de clientes SIN poder ver el email (permiso por columna)
CREATE USER 'marketing'@'localhost' IDENTIFIED BY 'Mk!2025#Clave';
GRANT SELECT (id, nombre, ciudad, pais, fecha_registro) ON tienda_dba.clientes TO 'marketing'@'localhost';

-- 4) Usuario de migraciones/despliegues: puede cambiar la estructura de SU base de datos
CREATE USER 'deploy'@'localhost' IDENTIFIED BY 'D3ploy!Clave#2025';
GRANT ALL PRIVILEGES ON tienda_dba.* TO 'deploy'@'localhost';

-- 5) Usuario para copias de seguridad con mysqldump (Módulo 10)
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'B4ckup!Clave#2025';
GRANT SELECT, SHOW VIEW, TRIGGER, EVENT, LOCK TABLES, RELOAD, PROCESS, REPLICATION CLIENT ON *.* TO 'backup'@'localhost';

-- 6) Usuario de monitoreo (Prometheus, Zabbix, PMM...)
CREATE USER 'monitor'@'localhost' IDENTIFIED BY 'M0nitor!Clave#2025' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'monitor'@'localhost';
GRANT SELECT ON performance_schema.* TO 'monitor'@'localhost';
```

### Ver y quitar permisos
```sql
SHOW GRANTS FOR 'app_tienda'@'localhost';
SHOW GRANTS;                                             -- Los tuyos

REVOKE DELETE ON tienda_dba.* FROM 'app_tienda'@'localhost';
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'deploy'@'localhost';
```
> 💡 `FLUSH PRIVILEGES` **solo** es necesario si modificas las tablas `mysql.user`, `mysql.db`... directamente con `INSERT`/`UPDATE` (no lo hagas). Con `GRANT`, `REVOKE`, `CREATE USER` y `ALTER USER` los cambios se aplican al instante.
>
> ⚠️ Los permisos nuevos afectan a las **conexiones nuevas** para los de nivel global/base de datos en algunos casos; si un cambio "no hace efecto", pide a la aplicación que reconecte.

---

## 4. Roles (MySQL 8+, MariaDB 10.0.5+)

Un **rol** es un conjunto de permisos con nombre. En vez de dar 8 permisos a cada uno de los 20 analistas, creas el rol `rol_analista` y lo asignas.

```sql
-- 1) Crear roles y darles permisos
CREATE ROLE IF NOT EXISTS 'rol_lectura', 'rol_escritura', 'rol_admin_tienda';

GRANT SELECT                          ON tienda_dba.* TO 'rol_lectura';
GRANT INSERT, UPDATE, DELETE, EXECUTE ON tienda_dba.* TO 'rol_escritura';
GRANT ALL PRIVILEGES                  ON tienda_dba.* TO 'rol_admin_tienda';

-- 2) Asignar roles a usuarios
CREATE USER 'lucia'@'localhost' IDENTIFIED BY 'Lucia!Clave#2025';
GRANT 'rol_lectura', 'rol_escritura' TO 'lucia'@'localhost';

-- 3) ¡Importante! Activar los roles por defecto (si no, el usuario los tiene pero NO activos)
SET DEFAULT ROLE ALL TO 'lucia'@'localhost';

-- Ver los permisos que le dan sus roles
SHOW GRANTS FOR 'lucia'@'localhost' USING 'rol_lectura', 'rol_escritura';

-- Quitar un rol
REVOKE 'rol_escritura' FROM 'lucia'@'localhost';

-- El usuario puede ver/activar sus roles en su sesión
-- SELECT CURRENT_ROLE();   SET ROLE ALL;
```
> Alternativa global: `SET GLOBAL activate_all_roles_on_login = ON;` activa siempre todos los roles asignados al iniciar sesión.

---

## 5. Políticas de contraseñas

### Componente `validate_password` (MySQL)
```sql
-- Instalarlo (MySQL 8+)
INSTALL COMPONENT 'file://component_validate_password';

SHOW VARIABLES LIKE 'validate_password%';
SET GLOBAL validate_password.length = 12;
SET GLOBAL validate_password.policy = 'STRONG';   -- LOW | MEDIUM | STRONG

-- Puntúa una contraseña de 0 a 100
SELECT VALIDATE_PASSWORD_STRENGTH('123456'), VALIDATE_PASSWORD_STRENGTH('Cambia3sta!Clave#2025');
```

### Caducidad, historial e intentos fallidos
```sql
-- Caducidad cada 90 días y no reutilizar las últimas 5
ALTER USER 'reportes'@'192.168.1.%'
    PASSWORD EXPIRE INTERVAL 90 DAY
    PASSWORD HISTORY 5;

-- Bloquear la cuenta 1 día tras 5 intentos fallidos (MySQL 8.0.19+)
ALTER USER 'app_tienda'@'localhost'
    FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;
```
> ⚠️ **No pongas caducidad a usuarios de aplicaciones** (`app_tienda`): si la contraseña caduca un sábado a las 3 AM, la web se cae. Las cuentas de servicio se rotan de forma planificada.

---

## 6. Asegurar un servidor nuevo (*hardening*)

### `mysql_secure_installation`
En cualquier servidor recién instalado, ejecuta en la terminal:
```bash
sudo mysql_secure_installation
```
Te guía para: poner contraseña a `root`, **eliminar usuarios anónimos**, **impedir que `root` se conecte remotamente**, **borrar la base de datos `test`** y recargar privilegios.

### Auditoría rápida de cuentas peligrosas
```sql
-- Usuarios sin contraseña
SELECT user, host FROM mysql.user
WHERE (authentication_string = '' OR authentication_string IS NULL)
  AND plugin NOT IN ('auth_socket', 'unix_socket');

-- Usuarios anónimos (user = '')
SELECT user, host FROM mysql.user WHERE user = '';

-- Quién puede conectarse desde CUALQUIER sitio
SELECT user, host FROM mysql.user WHERE host = '%';

-- Quién tiene permisos globales peligrosos
SELECT user, host, Super_priv, Grant_priv, File_priv, Create_user_priv, Shutdown_priv
FROM mysql.user
WHERE Super_priv = 'Y' OR Grant_priv = 'Y' OR File_priv = 'Y' OR Create_user_priv = 'Y' OR Shutdown_priv = 'Y';

-- Todos los permisos a nivel de base de datos
SELECT grantee, table_schema, privilege_type
FROM information_schema.schema_privileges
ORDER BY grantee, table_schema;
```

### Configuración de red (`my.cnf`)
```ini
[mysqld]
# Escuchar solo en local (si la aplicación está en el mismo servidor)
bind-address = 127.0.0.1
# O en la IP privada si la aplicación está en otro servidor de la red interna:
# bind-address = 10.0.0.5

# Impide LOAD DATA INFILE / SELECT ... INTO OUTFILE fuera de esta carpeta
secure_file_priv = /var/lib/mysql-files

# No resolver nombres DNS de los clientes (más rápido y evita depender del DNS)
skip_name_resolve = ON

# Desactivar la lectura de archivos locales del cliente
local_infile = OFF
```
* **Nunca** expongas el puerto 3306 a Internet. Usa el cortafuegos (`ufw`, *security groups*) para permitir solo las IP de tus servidores de aplicación, o conéctate por **túnel SSH**:
  ```bash
  ssh -L 3307:127.0.0.1:3306 usuario@mi-servidor.com
  # Ahora tu DBeaver/Workbench se conecta a localhost:3307 y viaja cifrado por SSH
  ```
* Protege también el acceso SSH con **Fail2ban** ([tutorial-fail2ban-seguridad-vps.md](../tutorial-fail2ban-seguridad-vps.md)).

---

## 7. Cifrado

### En tránsito (SSL/TLS)
```sql
SHOW VARIABLES LIKE '%ssl%';
SHOW STATUS LIKE 'Ssl_cipher';            -- Vacío = tu conexión NO está cifrada

-- Obligar a un usuario a conectarse siempre por TLS
ALTER USER 'reportes'@'192.168.1.%' REQUIRE SSL;
```
```ini
[mysqld]
require_secure_transport = ON     # Rechaza toda conexión no cifrada
```

### En reposo (datos en disco)
* **InnoDB tablespace encryption** (MySQL 8 con *keyring*): `ALTER TABLE clientes ENCRYPTION = 'Y';`. Protege si alguien roba el disco o los archivos de datos.
* **Cifrar columnas sensibles** desde la aplicación (IBAN, documentos de identidad): la base de datos solo ve texto cifrado. Las funciones `AES_ENCRYPT()`/`AES_DECRYPT()` de MySQL existen, pero obligan a enviar la clave en cada consulta (y puede acabar en logs).
* **Contraseñas de usuarios de tu aplicación:** **nunca** en texto plano ni cifradas de forma reversible: guarda un **hash** con *bcrypt* o *Argon2* generado por la aplicación. Nunca `MD5()` ni `SHA1()`.

### Los backups también son datos
Una copia de seguridad sin cifrar en un *bucket* público es una filtración. Cifra los backups (Módulo 10) y restringe quién puede leerlos.

---

## 8. Inyección SQL: el ataque número 1

Ocurre cuando una aplicación **concatena** texto del usuario dentro de una consulta:
```php
// ❌ PHP VULNERABLE
$sql = "SELECT * FROM clientes WHERE email = '" . $_GET['email'] . "'";
```
Si el atacante envía como email:  `' OR '1'='1`
```sql
SELECT * FROM clientes WHERE email = '' OR '1'='1';   -- ¡Devuelve TODOS los clientes!
```
Y con `'; DROP TABLE clientes; -- ` podría borrar la tabla (si el driver permite varias sentencias).

### La defensa: consultas preparadas (parámetros)
```php
// ✅ PHP con PDO: el valor viaja SEPARADO de la consulta, nunca se interpreta como SQL
$stmt = $pdo->prepare('SELECT * FROM clientes WHERE email = ?');
$stmt->execute([$_GET['email']]);
```
```python
# ✅ Python
cursor.execute("SELECT * FROM clientes WHERE email = %s", (email,))
```
Defensas adicionales que dependen del DBA:
1. **Mínimo privilegio:** si la app solo tiene `SELECT, INSERT, UPDATE, DELETE` en su base de datos, una inyección **no puede** hacer `DROP TABLE`, leer `mysql.user` ni escribir archivos.
2. **Usuarios distintos** por aplicación/servicio (si roban uno, el daño se limita).
3. Nombres de tablas/columnas dinámicos → **lista blanca**, nunca del usuario.
4. Revisar el log de errores y el *slow log* en busca de consultas extrañas.

---

## 9. Auditoría: ¿quién hizo qué?

| Opción | Qué registra | Notas |
| :--- | :--- | :--- |
| *General query log* | **Todas** las consultas | Solo para depurar unos minutos: llena el disco y frena el servidor. |
| *Triggers* de auditoría (Módulo 6) | Cambios en tablas concretas | Gratis y selectivo. |
| **MariaDB Audit Plugin** (`server_audit`) | Conexiones, consultas, tablas | Gratuito; funciona también en algunas versiones de MySQL. |
| **MySQL Enterprise Audit** / **Percona Audit Log** | Auditoría completa, filtrable | Para cumplimiento normativo (RGPD, PCI-DSS, ISO 27001). |

```sql
-- Activar el general log temporalmente para depurar (¡y apagarlo después!)
SET GLOBAL general_log_file = '/var/log/mysql/general.log';
SET GLOBAL general_log = ON;
-- ...
SET GLOBAL general_log = OFF;

-- Intentos de conexión fallidos (performance_schema)
SELECT * FROM performance_schema.host_cache WHERE sum_connect_errors > 0;
```

---

## 10. Checklist de seguridad del DBA

* [ ] `root` sin acceso remoto y con contraseña fuerte (o `auth_socket`).
* [ ] Sin usuarios anónimos ni base de datos `test`.
* [ ] Un usuario **por aplicación**, con permisos solo sobre su base de datos y sin `DROP`/`ALTER` en producción.
* [ ] Ningún usuario con privilegios altos en `'%'`.
* [ ] `bind-address` restringido y puerto 3306 cerrado en el cortafuegos.
* [ ] TLS obligatorio para conexiones que salgan del servidor.
* [ ] Política de contraseñas y bloqueo tras intentos fallidos para personas.
* [ ] Backups cifrados y probados (Módulo 10).
* [ ] Auditoría de cambios en datos sensibles.
* [ ] MySQL actualizado (parches de seguridad).

---

## 🏋️ Ejercicios del Módulo 9

1. Crea el usuario `'almacen'@'localhost'` que solo pueda **leer** productos y categorías y **modificar** la columna `stock` de `productos` (ninguna otra columna).
2. Crea el rol `rol_soporte` con permiso de lectura sobre `clientes`, `pedidos` y `detalle_pedidos`, asígnalo a un nuevo usuario `'soporte1'@'localhost'` y haz que se active automáticamente al iniciar sesión.
3. Escribe la consulta que lista todos los usuarios que pueden conectarse desde cualquier host (`%`) y si su cuenta está bloqueada.
4. Quita al usuario `marketing` el permiso que tenga y bloquea su cuenta.
5. Elimina todos los usuarios y roles creados en este módulo.

<details>
<summary>👀 Ver soluciones</summary>

```sql
-- 1
CREATE USER 'almacen'@'localhost' IDENTIFIED BY 'Alm4cen!Clave#2025';
GRANT SELECT ON tienda_dba.productos  TO 'almacen'@'localhost';
GRANT SELECT ON tienda_dba.categorias TO 'almacen'@'localhost';
GRANT UPDATE (stock) ON tienda_dba.productos TO 'almacen'@'localhost';
SHOW GRANTS FOR 'almacen'@'localhost';

-- 2
CREATE ROLE 'rol_soporte';
GRANT SELECT ON tienda_dba.clientes        TO 'rol_soporte';
GRANT SELECT ON tienda_dba.pedidos         TO 'rol_soporte';
GRANT SELECT ON tienda_dba.detalle_pedidos TO 'rol_soporte';
CREATE USER 'soporte1'@'localhost' IDENTIFIED BY 'S0porte!Clave#2025';
GRANT 'rol_soporte' TO 'soporte1'@'localhost';
SET DEFAULT ROLE 'rol_soporte' TO 'soporte1'@'localhost';

-- 3
SELECT user, host, account_locked FROM mysql.user WHERE host = '%';

-- 4
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'marketing'@'localhost';
ALTER USER 'marketing'@'localhost' ACCOUNT LOCK;

-- 5
DROP USER IF EXISTS
    'app_tienda'@'localhost', 'reportes'@'192.168.1.%', 'marketing'@'localhost',
    'deploy'@'localhost', 'backup'@'localhost', 'monitor'@'localhost',
    'lucia'@'localhost', 'almacen'@'localhost', 'soporte1'@'localhost';
DROP ROLE IF EXISTS 'rol_lectura', 'rol_escritura', 'rol_admin_tienda', 'rol_soporte';
```
</details>

---

👉 **Siguiente paso:** **[Módulo 10: Backups, Recuperación, Replicación y Monitoreo](./10_BACKUPS_RECUPERACION_REPLICACION_Y_MONITOREO.md)**
