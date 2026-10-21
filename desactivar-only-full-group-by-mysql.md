# Guía Paso a Paso: Solucionar Errores de SQL Mode en MySQL (ONLY_FULL_GROUP_BY y Field Default Value)

Esta guía explica en detalle cómo solucionar los dos errores más comunes que ocurren al migrar una aplicación desde un entorno local (como **WampServer** o **XAMPP**) a un servidor **VPS de producción** con MySQL / MariaDB:

1. **Error 1055 (`ONLY_FULL_GROUP_BY`):** Bloquea consultas con `GROUP BY` donde se seleccionan columnas no agregadas.
2. **Error 1364 (`STRICT_TRANS_TABLES`):** Bloquea inserciones (`INSERT`) cuando un campo booleano o de texto no recibe valor y no tiene `DEFAULT` explícito.

---

## 📋 Tabla de Contenidos
1. [Error 1: Error 1055 — ONLY_FULL_GROUP_BY](#1-error-1-error-1055--only_full_group_by)
2. [Error 2: Error 1364 — Field doesn't have a default value](#2-error-2-error-1364--field-doesnt-have-a-default-value)
   - [¿Por qué en WampServer funciona y en el VPS falla?](#por-qué-en-wampserver-funciona-y-en-el-vps-falla)
3. [Solución Definitiva en el VPS (Para ambos errores)](#3-solución-definitiva-en-el-vps-para-ambos-errores-recomendado)
   - [Opción A: Comando automático de 1 línea](#opción-a-comando-automático-de-1-sola-línea-más-rápido)
   - [Opción B: Edición manual de archivo de configuración](#opción-b-edición-manual-del-archivo-de-configuración)
4. [Solución a nivel de Aplicación (Laravel / PHP PDO)](#4-solución-a-nivel-de-aplicación-laravel--php-pdo)
5. [Solución a nivel de Estructura de Base de Datos y Consultas](#5-solución-a-nivel-de-estructura-de-base-de-datos-y-consultas)
6. [Resumen y Comprobación](#6-resumen-y-comprobación)

---

## 1. Error 1: Error 1055 — `ONLY_FULL_GROUP_BY`

### Mensaje de error típico:
```text
PDOException: SQLSTATE[42000]: Syntax error or access violation: 1055 Expression #1 of SELECT list 
is not in GROUP BY clause and contains nonaggregated column 'swimingschool.students.name' which 
is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

### 🔍 ¿Por qué ocurre?
MySQL 5.7 y 8.0+ exigen que si agrupas filas con `GROUP BY`, todas las columnas que pidas en el `SELECT` deben estar en el `GROUP BY` o dentro de una función de agregación (`MAX()`, `COUNT()`, `SUM()`, `GROUP_CONCAT()`). Si no lo están, MySQL no sabe qué fila elegir y lanza este error.

---

## 2. Error 2: Error 1364 — `Field doesn't have a default value`

### Mensaje de error típico:
```text
PDOException: SQLSTATE[HY000]: General error: 1364 Field 'alerted' doesn't have a default value
```

### 🔍 ¿Por qué en WampServer funciona y en el VPS falla?

- **En WampServer / Local:** El modo estricto de tablas (`STRICT_TRANS_TABLES`) suele venir **desactivado**. Cuando haces un `INSERT` omitiendo el campo `alerted` (aunque esté como `NOT NULL`), MySQL le asigna automáticamente el valor cero (`0` / `false` para booleanos, `''` para texto) sin arrojar error.
- **En el VPS (Producción):** MySQL viene configurado con **Modo Estricto (`STRICT_TRANS_TABLES`)** activado. Si una columna está definida como `NOT NULL` y no tiene un valor `DEFAULT` explícito en la tabla, MySQL rechaza la inserción con el error 1364.

---

## 3. Solución Definitiva en el VPS (Para ambos errores - Recomendado)

Podemos relajar el `sql_mode` en el VPS para que el servidor acepte inserciones sin valores por defecto (igual que WampServer) y permita consultas `GROUP BY` flexibles.

---

### Opción A: Comando automático de 1 sola línea (Más rápido)

Conéctate a la terminal de tu VPS por SSH y ejecuta este comando:

```bash
echo -e "[mysqld]\nsql_mode = \"NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION\"" | sudo tee /etc/mysql/conf.d/disable_strict_mode.cnf
```

#### 🔍 ¿Qué hace este comando?
- Define las reglas globales de MySQL omitiendo **`ONLY_FULL_GROUP_BY`** y **`STRICT_TRANS_TABLES`**.
- Crea el archivo de configuración `/etc/mysql/conf.d/disable_strict_mode.cnf` que MySQL lee automáticamente al arrancar.

Ahora reinicia el servicio MySQL para aplicar los cambios:

```bash
sudo systemctl restart mysql
```

---

### Opción B: Edición manual del archivo de configuración

Si prefieres editar el archivo principal a mano:

1. Abre el archivo de configuración:
   ```bash
   sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
   # O en otros sistemas: sudo nano /etc/mysql/my.cnf
   ```

2. Ve al final del archivo y agrega las siguientes dos líneas:
   ```ini
   [mysqld]
   sql_mode = "NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"
   ```

3. Guarda con `Ctrl + O`, presiona `Enter` y sal con `Ctrl + X`.

4. Reinicia el servicio:
   ```bash
   sudo systemctl restart mysql
   ```

---

## 4. Solución a nivel de Aplicación (Laravel / PHP PDO)

Si estás usando **Laravel**, puedes desactivar el modo estricto directamente en tu proyecto sin necesidad de tocar la configuración global del VPS:

### En Laravel (`config/database.php`):
Abre `config/database.php` y en la conexión `'mysql'`:

```php
'mysql' => [
    'driver' => 'mysql',
    'url' => env('DB_URL'),
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE', 'tu_base_datos'),
    'username' => env('DB_USERNAME', 'tu_usuario'),
    'password' => env('DB_PASSWORD', 'tu_clave'),
    'unix_socket' => env('DB_SOCKET', ''),
    'charset' => env('DB_CHARSET', 'utf8mb4'),
    'collation' => env('DB_COLLATION', 'utf8mb4_unicode_ci'),
    'prefix' => '',
    'prefix_indexes' => true,
    'strict' => false, // <-- Cambia esto de true a false
    'engine' => null,
],
```

> 💡 **Nota:** En Laravel, al establecer `'strict' => false`, Laravel desactiva automáticamente tanto `ONLY_FULL_GROUP_BY` como `STRICT_TRANS_TABLES` en cada conexión.

Luego limpia la caché de Laravel en el VPS:
```bash
php artisan config:clear
```

---

## 5. Solución a nivel de Estructura de Base de Datos y Consultas

Si prefieres mantener el modo estricto en el servidor y corregir la base de datos según las mejores prácticas:

### Para el Error 1364 (Field doesn't have a default value):
Asigna un valor por defecto explícito a la columna en la tabla:

```sql
-- En MySQL / phpMyAdmin:
ALTER TABLE students MODIFY alerted TINYINT(1) NOT NULL DEFAULT 0;
```

O si usas **migraciones en Laravel**:
```php
$table->boolean('alerted')->default(false)->change();
```

---

### Para el Error 1055 (ONLY_FULL_GROUP_BY):
Adapta la consulta para agrupar por la clave primaria o usar funciones de agregación:

```sql
-- Agrupando por clave primaria (id):
SELECT students.id, students.name, COUNT(enrollments.id) 
FROM students 
JOIN enrollments ON enrollments.student_id = students.id
GROUP BY students.id;

-- O usando GROUP_CONCAT / MAX:
SELECT GROUP_CONCAT(students.name) AS alumnos, students.course_id, COUNT(*) 
FROM students 
GROUP BY students.course_id;
```

---

## 6. Resumen y Comprobación

Para verificar qué modos tiene activos tu MySQL actualmente en el VPS:

```bash
mysql -u root -p -e "SELECT @@GLOBAL.sql_mode;"
```

✅ **Resultado esperado tras aplicar la solución:**
```text
NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```
*(Ni `ONLY_FULL_GROUP_BY` ni `STRICT_TRANS_TABLES` estarán presentes, permitiendo que tu aplicación funcione exactamente igual que en WampServer).*
