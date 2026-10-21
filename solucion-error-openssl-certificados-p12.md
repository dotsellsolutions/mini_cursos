# Guía: Solución al Error de Contraseña / OpenSSL en Certificados PKCS#12 (.p12 / .pfx)

## 📌 Contexto del Problema

Al validar un certificado digital `.p12` o `.pfx` en PHP mediante la función `openssl_pkcs12_read()`:

```php
openssl_pkcs12_read($pkcs12Content, $certs, $password);
```

La función devuelve `false`, indicando falsamente que la contraseña es errónea a pesar de que está escrita correctamente.

### Causa Real
A partir de **PHP 8.1+** se utiliza **OpenSSL 3.x**. En esta versión, los algoritmos de cifrado heredados (*legacy*) como **Triple-DES (3DES)**, **RC2-40** y **SHA-1** están **desactivados por defecto**. 

Muchos certificados digitales (incluyendo los emitidos para la **DGII** por entidades como *Avansi*, *Cámara de Comercio de Santo Domingo*, o exportados desde Windows) utilizan estos algoritmos clásicos. Al intentar leerlos sin el proveedor *legacy* activo, OpenSSL arroja internamente:

```text
error:0308010C:digital envelope routines::unsupported
```

---

## 🔍 Paso 0: Cómo Diagnosticar el Error en PHP

Para confirmar que este es tu problema y no un fallo de contraseña real, captura los errores de OpenSSL en tu código:

```php
public function checkCertificatePassword(string $pkcs12, string $p12Password): bool
{
    // 1. Limpiar buffer de errores previos
    while (openssl_error_string());

    // 2. Intentar leer el certificado
    if (!openssl_pkcs12_read($pkcs12, $certs, $p12Password)) {
        $errores = [];
        while ($msg = openssl_error_string()) {
            $errores[] = $msg;
        }

        // Si estás depurando, imprime los errores:
        // dd($errores);

        return false;
    }

    return true;
}
```

> Si en la salida aparece `digital envelope routines::unsupported`, confirma que se debe a OpenSSL 3 y debes aplicar una de las siguientes soluciones.

---

## 💻 Entorno Local (Windows / Wamp / Laravel Artisan)

### Solución 1: Variable de entorno al correr `php artisan serve` (La más rápida)

Si ejecutas tu API localmente con `php artisan serve`, el servidor corre en el proceso de tu terminal (no pasa por Apache). 

Detén el servidor con `Ctrl + C` y arráncalo pasando la ruta de las librerías SSL de tu versión de PHP:

* **En PowerShell (Terminal de VS Code o Windows):**
  ```powershell
  $env:OPENSSL_MODULES = "C:\wamp64\bin\php\php8.4.0\extras\ssl"
  php artisan serve
  ```

* **En CMD (Símbolo del sistema):**
  ```cmd
  set OPENSSL_MODULES=C:\wamp64\bin\php\php8.4.0\extras\ssl && php artisan serve
  ```

---

### Solución 2: Configuración fija en `openssl.cnf` para Windows

Si quieres que funcione siempre sin tener que escribir la variable cada vez:

1. Abre el bloc de notas o editor como **Administrador**.
2. Abre el archivo `openssl.cnf` correspondiente:
   * **Si usas Wamp con Apache:** Tu `phpinfo()` indica que Apache busca en:
     ```text
     C:\Apache24\conf\openssl.cnf
     ```
   * **Si usas PHP CLI / Artisan Serve:** Busca por defecto en:
     ```text
     C:\Program Files\Common Files\SSL\openssl.cnf
     ```
   *(Si la carpeta no existe, créala y copia el `openssl.cnf` desde `C:\wamp64\bin\php\php8.4.0\extras\ssl\openssl.cnf`)*.

3. Al inicio del archivo, confirma que tenga:
   ```ini
   openssl_conf = openssl_init
   ```

4. Ubica la sección `[openssl_init]` y `[provider_sect]` (alrededor de la línea 53) y configúrala de la siguiente manera:
   ```ini
   [openssl_init]
   providers = provider_sect

   [provider_sect]
   default = default_sect
   legacy = legacy_sect

   [default_sect]
   activate = 1

   [legacy_sect]
   activate = 1
   module = C:/wamp64/bin/php/php8.4.0/extras/ssl/legacy
   ```

   > **Notas importantes:**
   > * En la línea `module`, usa barras inclinadas hacia adelante: `/`.
   > * **No** pongas la extensión `.dll` al final (OpenSSL la añade automáticamente).
   > * Si usas otra versión de PHP (por ejemplo 8.3), ajusta el nombre de la carpeta `php8.3.x`.

5. Guarda el archivo.
6. Si usas Wamp con Apache, haz clic en **Restart All Services**. Si usas `php artisan serve`, reinicia la terminal.

---

## 🚀 Servidor de Producción (VPS con Linux / CloudPanel)

En un VPS con **CloudPanel** (Ubuntu o Debian), Linux maneja las librerías de forma centralizada, por lo que **no** necesitas indicar rutas `module`.

### Paso 1: Conexión SSH
Conéctate a tu servidor como usuario con permisos `sudo` o `root`:
```bash
ssh root@IP_DE_TU_VPS
```

### Paso 2: Editar `openssl.cnf`
Abre el archivo de configuración del sistema:
```bash
sudo nano /etc/ssl/openssl.cnf
```

### Paso 3: Habilitar el módulo legacy
Justo después de los primeros comentarios, asegúrate de que esté la directiva:
```ini
openssl_conf = openssl_init
```

Luego baja a la sección de proveedores (puedes buscar con `Ctrl + W` escribiendo `openssl_init`) y déjalo así:

```ini
[openssl_init]
providers = provider_sect

[provider_sect]
default = default_sect
legacy = legacy_sect

[default_sect]
activate = 1

[legacy_sect]
activate = 1
```

* Para guardar en nano: presiona `Ctrl + O` y luego `Enter`.
* Para salir: presiona `Ctrl + X`.

### Paso 4: Reiniciar PHP-FPM en CloudPanel
Para que el cambio surta efecto en la web/API, reinicia el servicio PHP-FPM correspondiente a la versión de tu sitio:

* **Por comandos en la consola:**
  ```bash
  # Para PHP 8.2:
  sudo systemctl restart php8.2-fpm

  # Para PHP 8.3:
  sudo systemctl restart php8.3-fpm

  # Para PHP 8.4:
  sudo systemctl restart php8.4-fpm
  ```

* **O desde la interfaz web de CloudPanel:**
  1. Ve a **PHP** en el menú de la izquierda.
  2. Ubica la versión que utiliza tu sitio.
  3. Haz clic en **Restart**.

---

## 🔄 Alternativa: Modernizar el Certificado (AES-256)

Si prefieres no alterar la configuración del servidor y solo necesitas corregir un certificado puntual, puedes re-exportarlo con cifrado moderno desde la consola de Windows o Linux:

```powershell
# 1. Extraer llaves y certificados a un archivo temporal (usando el flag -legacy)
openssl pkcs12 -in "certificado_original.p12" -legacy -nodes -out temp.pem

# 2. Volver a empaquetar con el nuevo cifrado AES-256 (te pedirá la nueva contraseña)
openssl pkcs12 -export -in temp.pem -out "certificado_actualizado.p12"

# 3. Eliminar archivo temporal por seguridad
Remove-Item temp.pem
```

El nuevo archivo `certificado_actualizado.p12` será compatible nativamente con cualquier versión moderna de PHP y OpenSSL sin configuraciones adicionales.
