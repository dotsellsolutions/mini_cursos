# Guía Completa: Permisos de Carpetas y Archivos en Linux (`chmod`, `chown` y Notación Octal)

Comprender cómo funcionan los permisos en Linux es indispensable para administrar un servidor VPS. Te permite evitar fallos de seguridad (como ataques por carpetas abiertas a todo el mundo) y resolver errores típicos como `403 Forbidden`, `Permission Denied` o problemas de subida de archivos en servidores web.

---

## 📋 Tabla de Contenidos
1. [Anatomía de los Permisos en Linux (`ls -l`)](#1-anatomía-de-los-permisos-en-linux-ls--l)
2. [Las 3 Identidades: Usuario, Grupo y Otros](#2-las-3-identidades-usuario-grupo-y-otros)
3. [Los 3 Tipos de Permisos y su Valor Numérico (`rwx`)](#3-los-3-tipos-de-permisos-y-su-valor-numérico-rwx)
4. [¿Por qué los permisos en carpetas son diferentes a los de archivos?](#4-por-qué-los-permisos-en-carpetas-son-diferentes-a-los-de-archivos)
5. [El Comando `chmod`: Cambiar Permisos](#5-el-comando-chmod-cambiar-permisos)
   - [Método A: Notación Numérica (Octal)](#método-a-notación-numérica-octal)
   - [Método B: Notación Simbólica (`u`, `g`, `o`, `+`, `-`)](#método-b-notación-simbólica-u-g-o---)
   - [🔥 El Truco Pro: Permisos recursivos sin romper carpetas](#-el-truco-pro-permisos-recursivos-sin-romper-carpetas)
6. [El Comando `chown`: Cambiar Dueño y Grupo](#6-el-comando-chown-cambiar-dueño-y-grupo)
7. [⚠️ Por qué NUNCA debes usar `chmod 777`](#7-️-por-qué-nunca-debes-usar-chmod-777)
8. [Recetas Listas para Servidores Web (Laravel, Nginx, WordPress, SSH)](#8-recetas-listas-para-servidores-web-laravel-nginx-wordpress-ssh)
9. [Tabla Resumen / Cheat Sheet](#9-tabla-resumen--cheat-sheet)

---

## 1. Anatomía de los Permisos en Linux (`ls -l`)

Cuando ejecutas `ls -la` en tu terminal, ves líneas como esta:

```text
-rwxr-xr--  1 deployer www-data 4096 Sep 13 18:00 index.php
drwxr-xr-x  2 deployer www-data 4096 Sep 13 18:00 public/
```

Analicemos la primera columna de 10 caracteres (`-rwxr-xr--`):

| Carácter 1 | Caracteres 2, 3, 4 | Caracteres 5, 6, 7 | Caracteres 8, 9, 10 |
| :---: | :---: | :---: | :---: |
| **Tipo de archivo** | **Permisos del DUEÑO (User)** | **Permisos del GRUPO (Group)** | **Permisos del RESTO (Others)** |
| `-` (archivo) o `d` (directorio) | `r w x` | `r - x` | `r - -` |

- **Primer carácter:**
  - `-` = Archivo normal.
  - `d` = Directorio (carpeta).
  - `l` = Enlace simbólico (*symbolic link*).
- **Siguientes 9 caracteres:** Se dividen en 3 bloques de 3 caracteres cada uno:
  1. Bloque 1: **Dueño** (`rwx` = puede leer, escribir y ejecutar).
  2. Bloque 2: **Grupo** (`r-x` = puede leer y ejecutar, pero no escribir).
  3. Bloque 3: **Otros** (`r--` = solo puede leer).

---

## 2. Las 3 Identidades: Usuario, Grupo y Otros

En Linux, cada archivo tiene asociado un **propietario** y un **grupo**:

1. **`u` - User (Dueño):** La cuenta de usuario que creó el archivo o a quien se le asignó la propiedad.
2. **`g` - Group (Grupo):** Conjunto de usuarios que comparten permisos en común (ej. el grupo `www-data` de Nginx/Apache o el grupo `sudo`).
3. **`o` - Others (Otros / Mundo):** Cualquier otro usuario o proceso del sistema operativo que no sea el dueño ni pertenezca al grupo.
4. **`a` - All (Todos):** Representa a los tres anteriores a la vez (`u + g + o`).

---

## 3. Los 3 Tipos de Permisos y su Valor Numérico (`rwx`)

Cada letra tiene un valor matemático basado en el sistema binario:

| Letra | Significado | Valor Numérico | ¿Qué permite en un archivo? |
| :---: | :---: | :---: | :--- |
| **`r`** | *Read* (Lectura) | **4** | Abrir y leer el contenido del archivo. |
| **`w`** | *Write* (Escritura) | **2** | Modificar, editar o sobrescribir el archivo. |
| **`x`** | *eXecute* (Ejecución) | **1** | Ejecutar el archivo como un script o binario (`./script.sh`). |
| **`-`** | *Sin permiso* | **0** | No se permite esa acción. |

### 🧮 ¿Cómo se calcula el número de 3 dígitos (ej: 755)?
Simplemente sumas los valores de cada bloque:

| Combinación | Suma | Significado |
| :---: | :---: | :--- |
| `rwx` | 4 + 2 + 1 = **7** | Permisos totales (leer, escribir y ejecutar). |
| `rw-` | 4 + 2 + 0 = **6** | Lectura y escritura (estándar para archivos de texto/código). |
| `r-x` | 4 + 0 + 1 = **5** | Lectura y ejecución (estándar para carpetas públicas). |
| `r--` | 4 + 0 + 0 = **4** | Solo lectura. |
| `---` | 0 + 0 + 0 = **0** | Ningún permiso (acceso completamente denegado). |

### Ejemplo del clásico `755`:
- Primer dígito (**7** = `rwx`): El **Dueño** puede hacer todo.
- Segundo dígito (**5** = `r-x`): El **Grupo** puede leer y entrar a la carpeta.
- Tercer dígito (**5** = `r-x`): Los **Demás** pueden leer y entrar a la carpeta.

### Ejemplo del clásico `644`:
- Primer dígito (**6** = `rw-`): El **Dueño** puede leer y modificar.
- Segundo dígito (**4** = `r--`): El **Grupo** solo puede leer.
- Tercer dígito (**4** = `r--`): Los **Demás** solo pueden leer.

---

## 4. ¿Por qué los permisos en carpetas son diferentes a los de archivos?

Este es un concepto crítico que confunde a muchos principiantes:

| Permiso | En un Archivo (`documento.txt`) | En una Carpeta (`/fotos/`) |
| :---: | :--- | :--- |
| **`r` (Lectura)** | Ver el contenido del archivo. | Listar los nombres de archivos que están dentro (`ls`). |
| **`w` (Escritura)** | Modificar el contenido del archivo. | Crear, borrar o renombrar archivos **dentro** de la carpeta. |
| **`x` (Ejecución)** | Correr el archivo como programa. | **Atravesar y entrar a la carpeta (`cd`), y leer metadatos de los archivos.** |

> ⚠️ **Regla de oro:** Si una carpeta **NO** tiene permiso `x` (ejecución), no podrás hacer `cd` hacia ella ni tu servidor web podrá servir las imágenes o páginas dentro de ella, aunque los archivos individuales tengan permiso de lectura.

---

## 5. El Comando `chmod`: Cambiar Permisos

`chmod` significa *Change Mode* (cambiar modo de acceso).

---

### Método A: Notación Numérica (Octal)

Es la forma más rápida y común:

```bash
# Asignar permisos estándar a una carpeta (755)
chmod 755 /var/www/mi-sitio

# Asignar permisos estándar a un archivo (644)
chmod 644 index.php

# Darle permisos estrictos solo al dueño (700 en carpeta o 600 en archivo)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
```

---

### Método B: Notación Simbólica (`u`, `g`, `o`, `+`, `-`)

Ideal cuando solo quieres agregar o quitar un permiso específico sin alterar el resto:

```bash
# Darle permiso de ejecución al dueño para correr un script:
chmod u+x script.sh

# Darle permiso de ejecución a todos (all):
chmod +x script.sh

# Quitarle permiso de escritura al grupo:
chmod g-w archivo.txt

# Quitarle todos los permisos a los demás (others):
chmod o-rwx confidencial.txt

# Asignar lectura y escritura al dueño y grupo, nada a otros:
chmod ug=rw,o= archivo.env
```

---

### 🔥 El Truco Pro: Permisos recursivos sin romper carpetas

Un error muy común es hacer `chmod -R 755 mi-sitio/` o peor aún `chmod -R 644 mi-sitio/`.
- Si aplicas `644` a todo recursivamente, **¡le quitas el permiso `x` a las carpetas y tu web dejará de funcionar!**
- Si aplicas `755` a todo recursivamente, le darás permiso de ejecución innecesario a imágenes, CSS y archivos de texto.

**El comando profesional para asignar permisos correctos de forma masiva:**

```bash
# 1. Aplicar 755 ÚNICAMENTE a los directorios (carpetas):
find /ruta/proyecto -type d -exec chmod 755 {} +

# 2. Aplicar 644 ÚNICAMENTE a los archivos normales:
find /ruta/proyecto -type f -exec chmod 644 {} +
```

#### 🔍 ¿Qué hace este comando?
- `find /ruta/proyecto`: Busca dentro de la carpeta indicada y todas sus subcarpetas.
- `-type d`: Filtra solo los directorios (*directories*).
- `-type f`: Filtra solo los archivos (*files*).
- `-exec chmod ... {} +`: Aplica el comando en lote sobre los elementos encontrados de forma ultra eficiente.

---

## 6. El Comando `chown`: Cambiar Dueño y Grupo

`chown` significa *Change Owner* (cambiar propietario).

### Sintaxis básica:
```bash
# Cambiar solo el usuario propietario:
sudo chown deployer archivo.txt

# Cambiar el usuario y el grupo a la vez (separados por dos puntos :):
sudo chown deployer:www-data /var/www/mi-sitio

# Cambiar de forma recursiva a toda una carpeta y su contenido (-R):
sudo chown -R deployer:www-data /var/www/mi-sitio
```

#### 🔍 ¿Qué hace `-R`?
- `-R` *(recursive)*: Aplica el cambio de propietario a la carpeta seleccionada, a todos sus archivos internos y a todas las subcarpetas.

---

## 7. ⚠️ Por qué NUNCA debes usar `chmod 777`

Cuando un desarrollador tiene un error de permisos en su servidor, la tentación rápida es ejecutar:
```bash
# ❌ PELIGRO: NUNCA HAGAS ESTO EN PRODUCCIÓN
sudo chmod -R 777 /var/www/mi-sitio
```

### ¿Por qué `777` es una catástrofe de seguridad?
1. `777` significa que **CUALQUIERA** (cualquier usuario del servidor, cualquier proceso infectado o cualquier hacker que encuentre una vulnerabilidad mínima) puede:
   - Modificar tus archivos `.php` o `.js` para inyectar código malicioso (*web shells*, mineros de criptomonedas o redirecciones de spam).
   - Sobrescribir o borrar la aplicación completa.
   - Leer tus archivos `.env` con contraseñas de bases de datos y claves de API.
2. **La solución real nunca es `777`**, sino asignar el **propietario correcto con `chown`** y permisos equilibrados (`755` para carpetas y `644` para archivos).

---

## 8. Recetas Listas para Servidores Web (Laravel, Nginx, WordPress, SSH)

### Receta 1: Servidor Web Estándar (Nginx / CloudPanel / Node / PHP)
```bash
# Asignar dueño a tu usuario de despliegue y grupo al servidor web
sudo chown -R deployer:www-data /var/www/mi-app

# Carpetas en 755 y archivos en 644
find /var/www/mi-app -type d -exec chmod 755 {} +
find /var/www/mi-app -type f -exec chmod 644 {} +
```

---

### Receta 2: Proyecto Laravel (Permisos de Storage y Caché)
En Laravel, el servidor web necesita escribir logs y caché dentro de `storage/` y `bootstrap/cache/`:

```bash
cd /var/www/mi-proyecto-laravel

# Asignar propiedad
sudo chown -R deployer:www-data .

# Permisos base
find . -type d -exec chmod 755 {} +
find . -type f -exec chmod 644 {} +

# Dar permisos de escritura al grupo www-data en las carpetas que lo requieren:
sudo chgrp -R www-data storage bootstrap/cache
sudo chmod -R ug+rwx storage bootstrap/cache
```

---

### Receta 3: Llaves SSH y Seguridad Personal
El servicio OpenSSH rechaza llaves privadas si tienen permisos inseguros:

```bash
# La carpeta .ssh debe ser accesible solo por su dueño
chmod 700 ~/.ssh

# La llave privada NUNCA debe ser leída por nadie más (600)
chmod 600 ~/.ssh/id_rsa

# La llave pública sí puede ser leída por otros (644)
chmod 644 ~/.ssh/id_rsa.pub

# El archivo de llaves autorizadas para entrar al VPS:
chmod 600 ~/.ssh/authorized_keys
```

---

## 9. Tabla Resumen / Cheat Sheet

| Permiso | Octal | ¿Quién tiene acceso? | ¿Dónde se usa comúnmente? |
| :---: | :---: | :--- | :--- |
| `rwxr-xr-x` | **755** | Dueño: todo. Grupo y otros: leer y navegar. | **Carpetas web y ejecutables.** |
| `rw-r--r--` | **644** | Dueño: editar. Grupo y otros: solo leer. | **Archivos HTML, PHP, JS, imágenes, CSS.** |
| `rw-------` | **600** | Solo el dueño puede leer y editar. | **Archivos `.env`, llaves privadas SSH, certificados.** |
| `rwx------` | **700** | Solo el dueño puede entrar y operar. | **Carpeta `~/.ssh`, carpetas privadas de usuario.** |
| `rwxrwxrwx` | **777** | Todos pueden hacer todo sin restricciones. | **❌ NUNCA en producción.** |
