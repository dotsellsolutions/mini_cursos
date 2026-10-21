# Guía Paso a Paso: Cómo Restablecer y Cambiar la Contraseña de Root en MySQL (VPS Linux)

Esta guía explica detalladamente cómo recuperar o cambiar la contraseña del usuario `root` de MySQL / MariaDB / Percona Server en un VPS (Ubuntu, Debian, CentOS) cuando la has olvidado o cuando el sistema te arroja el error `ERROR 1045 (28000): Access denied for user 'root'@'localhost'`.

Incluye la solución a problemas comunes como el error de directorio de socket faltante, el error `1396 (HY000)` al ejecutar `ALTER USER`, y la explicación de cada comando.

---

## 📋 Tabla de Contenidos
1. [¿Por qué falla el acceso si soy root en Linux?](#1-por-qué-falla-el-acceso-si-soy-root-en-linux)
2. [Procedimiento Paso a Paso de Recuperación](#2-procedimiento-paso-a-paso-de-recuperación)
   - [Paso 1: Detener el servicio MySQL](#paso-1-detener-el-servicio-mysql)
   - [Paso 2: Crear y asegurar la carpeta del Socket](#paso-2-crear-y-asegurar-la-carpeta-del-socket)
   - [Paso 3: Iniciar MySQL en Modo Seguro Temporal](#paso-3-iniciar-mysql-en-modo-seguro-temporal)
   - [Paso 4: Conectarse e identificar la cuenta root exacta](#paso-4-conectarse-e-identificar-la-cuenta-root-exacta)
   - [Paso 5: Asignar la nueva contraseña](#paso-5-asignar-la-nueva-contraseña)
   - [Paso 6: Cerrar el modo seguro y reiniciar MySQL](#paso-6-cerrar-el-modo-seguro-y-reiniciar-mysql)
   - [Paso 7: Probar el acceso con la nueva contraseña](#paso-7-probar-el-acceso-con-la-nueva-contraseña)
3. [Preguntas Frecuentes y Errores Típicos](#3-preguntas-frecuentes-y-errores-típicos)
4. [Glosario de Comandos Utilizados](#4-glosario-de-comandos-utilizados)

---

## 1. ¿Por qué falla el acceso si soy root en Linux?

En un servidor VPS existen **dos usuarios `root` totalmente independientes**:

1. **`root` de Linux / Sistema Operativo:** Es el superadministrador del servidor completo (acceso SSH, archivos del sistema).
2. **`root` de MySQL:** Es el usuario administrador interno dentro del motor de base de datos.

Cuando ejecutas `sudo mysql`, el sistema operativo te permite invocar el programa, pero si MySQL está configurado para exigir contraseña a su usuario interno `root`, rechazará la conexión con el mensaje:
> `ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)`

---

## 2. Procedimiento Paso a Paso de Recuperación

> ⚠️ **Nota:** Durante este proceso de 2 a 3 minutos, el servicio MySQL estará detenido temporalmente, por lo que las páginas web conectadas a la base de datos mostrarán errores de conexión hasta que finalices.

---

### Paso 1: Detener el servicio MySQL
Debemos detener el servidor de base de datos para poder arrancarlo en modo de mantenimiento sin comprobación de permisos.

```bash
sudo systemctl stop mysql
```

#### 🔍 ¿Qué hace este comando?
- `systemctl stop mysql`: Envía una señal ordenada de apagado al servicio MySQL para cerrar las conexiones activas y guardar las transacciones en disco.

---

### Paso 2: Crear y asegurar la carpeta del Socket
Un error muy frecuente al iniciar el modo seguro es que falte la carpeta temporal `/var/run/mysqld`. La preparamos con permisos correctos:

```bash
sudo install -d -o mysql -g mysql /var/run/mysqld
```

#### 🔍 ¿Qué hace este comando?
- `install -d`: Crea el directorio `/var/run/mysqld` si no existe.
- `-o mysql`: Asigna como propietario (*owner*) al usuario de sistema `mysql`.
- `-g mysql`: Asigna como grupo (*group*) al grupo `mysql`.
- *Evita el error:* `mysqld_safe Directory '/var/run/mysqld' for UNIX socket file doesn't exist`.

---

### Paso 3: Iniciar MySQL en Modo Seguro Temporal
Iniciaremos el motor de MySQL saltándonos la tabla de contraseñas y desactivando la red externa (para que nadie más pueda conectarse sin clave mientras estamos en mantenimiento):

```bash
sudo mysqld_safe --skip-grant-tables --skip-networking --user=mysql >/tmp/mysql-reset.log 2>&1 &
```

#### 🔍 ¿Qué hace cada parte de este comando?
- `mysqld_safe`: Script seguro para arrancar el binario `mysqld` con monitoreo.
- `--skip-grant-tables`: **La bandera clave.** Desactiva la verificación de usuarios y contraseñas. Cualquiera que se conecte localmente tiene acceso total.
- `--skip-networking`: Cierra los puertos de red (puerto 3306) para que nadie desde Internet pueda aprovechar que las contraseñas están desactivadas.
- `--user=mysql`: Ejecuta el proceso con los privilegios seguros del usuario del sistema `mysql`.
- `>/tmp/mysql-reset.log 2>&1`: Redirige tanto la salida estándar como los mensajes de error a un archivo temporal en `/tmp/`.
- `&`: Ejecuta el proceso en segundo plano para no bloquear tu terminal.

Espera unos 5 segundos a que termine de inicializarse:
```bash
sleep 5
```

---

### Paso 4: Conectarse e identificar la cuenta root exacta
Entra a la consola de MySQL (ahora entrará directo sin pedir clave):

```bash
mysql -u root
```

Una vez dentro del prompt `mysql>`, recarga los privilegios para poder modificarlos e identifica cómo está registrado el usuario `root`:

```sql
FLUSH PRIVILEGES;
SELECT User, Host FROM mysql.user WHERE User = 'root';
```

#### 🔍 ¿Qué hace este comando?
- `FLUSH PRIVILEGES`: Le indica a MySQL que lea las tablas de privilegios en memoria para que acepte comandos de modificación de cuentas (`ALTER USER`).
- `SELECT User, Host...`: Muestra las cuentas de root existentes. En MySQL, un usuario no es solo `root`, sino el par `User@Host`. Por ejemplo:
  - `root` @ `localhost` (conexión vía socket Unix)
  - `root` @ `127.0.0.1` (conexión vía protocolo TCP/IP local)
  - `root` @ `%` (conexión desde cualquier dirección IP)

---

### Paso 5: Asignar la nueva contraseña

Elige el comando según lo que viste en la consulta `SELECT`:

#### Caso A: Si en la tabla aparece `root` con Host `localhost`
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'TuNuevaContrasenaSegura2026!';
```

#### Caso B: Si en la tabla aparece `root` con Host `127.0.0.1`
*(Esto ocurre comúnmente en instalaciones de CloudPanel, Percona o configuraciones personalizadas)*:
```sql
ALTER USER 'root'@'127.0.0.1' IDENTIFIED BY 'TuNuevaContrasenaSegura2026!';
```

#### Caso C: Si no aparece ningún usuario `root` (Crearlo desde cero)
```sql
CREATE USER 'root'@'localhost' IDENTIFIED BY 'TuNuevaContrasenaSegura2026!';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
```

Aplica los cambios finales y sal de la consola:
```sql
FLUSH PRIVILEGES;
exit
```

> 💡 **Nota sobre el mensaje `Query OK, 0 rows affected`:**
> Esto **NO es un error**. `ALTER USER` modifica los metadatos de autenticación del servidor, no filas de una tabla convencional de datos, por lo que reportar 0 filas modificadas es la respuesta estándar de éxito.

---

### Paso 6: Cerrar el modo seguro y reiniciar MySQL

Ahora debemos apagar el proceso temporal en segundo plano y arrancar el servicio normal de MySQL para que vuelva a escuchar peticiones de tus sitios web:

```bash
# 1. Detener el proceso seguro temporal
sudo pkill -f mysqld

# 2. Iniciar el servicio normal de MySQL
sudo systemctl start mysql
```

#### 🔍 ¿Qué hace este comando?
- `pkill -f mysqld`: Busca y finaliza todos los procesos que tengan `mysqld` en su nombre (cerrando la sesión segura).
- `systemctl start mysql`: Inicia el servicio estándar de producción con la red activa y la nueva contraseña requerida.

---

### Paso 7: Probar el acceso con la nueva contraseña

Prueba conectarte ingresando tu nueva contraseña:

```bash
# Para conexiones por socket local:
sudo mysql -u root -p

# O si tu cuenta root está asociada a 127.0.0.1 (TCP):
mysql -u root -h 127.0.0.1 -p
```

Escribe tu nueva contraseña cuando te la solicite *(recuerda que en Linux los caracteres no se muestran en pantalla al escribir contraseñas por seguridad)*.

Si ves el prompt `mysql>`, **¡felicidades, has recuperado el control total de MySQL!**

---

## 3. Preguntas Frecuentes y Errores Típicos

### ❌ Error `ERROR 1396 (HY000): Operation ALTER USER failed for 'root'@'localhost'`
- **Causa:** El usuario `root` no está registrado con el Host `localhost`, sino con `127.0.0.1` o con `%`.
- **Solución:** Ejecuta `SELECT User, Host FROM mysql.user WHERE User = 'root';` y usa en `ALTER USER` el valor de Host exacto que devuelva la consulta.

### ❌ Error `ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1:3306' (111)`
- **Causa:** MySQL aún se está iniciando (el motor InnoDB puede tardar unos segundos en verificar las tablas al arrancar) o el servicio está detenido.
- **Solución:** Espera 10 segundos y verifica el estado con `sudo systemctl status mysql`.

### ❌ Error `mysqld_safe Directory '/var/run/mysqld' for UNIX socket file doesn't exist`
- **Causa:** En sistemas Ubuntu/Debian modernos, la carpeta `/var/run` se monta en memoria RAM (`tmpfs`), por lo que al detener el servicio la carpeta del socket puede desaparecer.
- **Solución:** Ejecutar `sudo install -d -o mysql -g mysql /var/run/mysqld` antes de `mysqld_safe`.

---

## 4. Glosario de Comandos Utilizados

| Comando | Propósito |
| :--- | :--- |
| `sudo systemctl stop mysql` | Detiene el servicio MySQL en producción. |
| `install -d -o mysql -g mysql <ruta>` | Crea carpetas asignando dueño y grupo en un solo paso. |
| `mysqld_safe --skip-grant-tables` | Inicia MySQL omitiendo el sistema de contraseñas. |
| `--skip-networking` | Bloquea conexiones remotas externas durante el mantenimiento. |
| `FLUSH PRIVILEGES` | Fuerza al servidor a recargar las tablas de permisos en memoria. |
| `ALTER USER 'usr'@'host' IDENTIFIED BY 'pass'` | Modifica la contraseña del usuario especificado. |
| `sudo pkill -f mysqld` | Fuerza el cierre de todos los procesos de MySQL en ejecución. |
