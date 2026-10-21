# Mini-Curso de Linux para Administrar un VPS: Los Comandos Más y Menos Necesarios

Linux contiene más de 2,000 comandos distintos. Sin embargo, en el mundo real, **el 90% del tiempo de un desarrollador o administrador de servidores se reduce a solo unos 25 comandos clave**.

Este curso práctico va al grano: clasifica los comandos por su **grado de utilidad real en un VPS**, explica qué hace cada uno con sus banderas más usadas, y te enseña cómo reaccionar ante emergencias típicas (disco lleno, servidor caído o puertos ocupados).

---

## 📋 Tabla de Contenidos
1. [Filosofía y Mapa del Sistema Operativo en un VPS](#1-filosofía-y-mapa-del-sistema-operativo-en-un-vps)
2. [Nivel 1: Los IMPRESCINDIBLES (Uso Diario - 90% de tu tiempo)](#2-nivel-1-los-imprescindibles-uso-diario---90-de-tu-tiempo)
   - [Navegación y Archivos](#21-navegación-y-archivos)
   - [Gestión de Paquetes y Software](#22-gestión-de-paquetes-y-software)
   - [Control de Servicios (Systemd)](#23-control-de-servicios-systemd)
   - [Monitoreo de Recursos en Vivo](#24-monitoreo-de-recursos-en-vivo)
   - [Seguridad y Firewall](#25-seguridad-y-firewall)
3. [Nivel 2: Los INTERMEDIOS (Diagnóstico y Emergencias - 8% de tu tiempo)](#3-nivel-2-los-intermedios-diagnóstico-y-emergencias---8-de-tu-tiempo)
   - [Lectura y Monitoreo de Logs en Vivo](#31-lectura-y-monitoreo-de-logs-en-vivo)
   - [Búsqueda de Archivos y Texto](#32-búsqueda-de-archivos-y-texto)
   - [Redes, Puertos y Conectividad](#33-redes-puertos-y-conectividad)
   - [Procesos Trabados y Memoria](#34-procesos-trabados-y-memoria)
   - [Compresión y Respaldos Rápidos](#35-compresión-y-respaldos-rápidos)
4. [Nivel 3: Los MENOS NECESARIOS (Rara vez usados o prescindibles - 2%)](#4-nivel-3-los-menos-necesarios-rara-vez-usados-o-prescindibles---2)
   - [Comandos con mejores alternativas modernas](#41-comandos-con-mejores-alternativas-modernas)
   - [Comandos antiguos o ultra especializados](#42-comandos-antiguos-o-ultra-especializados)
   - [Comandos de ALTO RIESGO que debes evitar](#43-comandos-de-alto-riesgo-que-debes-evitar)
5. [Guía de Emergencias: Qué Comandos Correr Cuando Algo Falla](#5-guía-de-emergencias-qué-comandos-correr-cuando-algo-falla)
   - [Emergencia A: "El disco de mi VPS está al 100%"](#emergencia-a-el-disco-de-mi-vps-está-al-100)
   - [Emergencia B: "Mi web da Error 502 Bad Gateway"](#emergencia-b-mi-web-da-error-502-bad-gateway)
   - [Emergencia C: "El puerto 3000 o 3306 ya está ocupado"](#emergencia-c-el-puerto-3000-o-3306-ya-está-ocupado)
6. [Resumen Final: La Pirámide de Comandos](#6-resumen-final-la-pirámide-de-comandos)

---

## 1. Filosofía y Mapa del Sistema Operativo en un VPS

En Linux hay una máxima famosa: **"En Linux, todo es un archivo"** (los discos, los procesos, las configuraciones y los logs son simplemente archivos que puedes leer y editar).

### Las 5 carpetas que SIEMPRE debes tener ubicadas en tu VPS:
- **`/var/www/`** o **`/home/`**: Aquí viven tus aplicaciones web, proyectos de Node.js, Laravel, WordPress, HTML y Vite.
- **`/etc/`**: La carpeta de las **configuraciones**. Aquí están los archivos de Nginx (`/etc/nginx/`), de MySQL (`/etc/mysql/`), de SSH (`/etc/ssh/`) y los servicios del sistema (`/etc/systemd/system/`).
- **`/var/log/`**: Donde se guardan todos los **registros de error y acceso**. Si tu sitio no abre o la base de datos se cayó, la respuesta está aquí dentro.
- **`/tmp/`**: Archivos temporales que el sistema borra automáticamente en cada reinicio.
- **`/root`** y **`/home/usuario`**: Las carpetas personales (*home*) de cada usuario del sistema.

---

## 2. Nivel 1: Los IMPRESCINDIBLES (Uso Diario - 90% de tu tiempo)

Estos son los comandos que usarás todos los días sin excepción.

---

### 2.1 Navegación y Archivos

#### `pwd` *(Print Working Directory)*
Te dice exactamente en qué carpeta estás parado.
```bash
pwd
# Salida: /home/deployer/htdocs/mi-app.com
```

#### `ls -la` *(List All Long)*
Lista los archivos con todos sus detalles (permisos, dueño, peso y fecha).
```bash
ls -la
```
- `-l`: Formato de lista detallada.
- `-a` *(all)*: Muestra también los archivos ocultos (los que empiezan con un punto, como `.env`, `.gitignore`, `.htaccess`).
- `-lh` *(human readable)*: Muestra el tamaño de los archivos en KB, MB o GB en vez de bytes.

#### `cd` *(Change Directory)*
Para moverte entre carpetas:
```bash
cd /var/www/html      # Ir a una ruta absoluta
cd ..                 # Subir un nivel de carpeta
cd ~                  # Ir directo a tu carpeta personal (Home)
cd -                  # Volver a la carpeta anterior donde estabas
```

#### `mkdir -p` *(Make Directory)*
Crea carpetas. Con la bandera `-p` crea toda la estructura intermedia si no existe:
```bash
mkdir -p /var/www/mi-app/storage/logs
```

#### `cp -r` *(Copy)*
Copia archivos o carpetas:
```bash
cp archivo.env .env.backup          # Copiar un archivo
cp -r carpeta-original copia/       # -r (recursivo) copia la carpeta entera con su contenido
```

#### `mv` *(Move / Rename)*
Mueve archivos de lugar o les cambia de nombre:
```bash
mv dist/ /var/www/html/             # Mover una carpeta
mv archivo_viejo.txt nuevo.txt      # Renombrar un archivo
```

#### `rm -rf` *(Remove)*
Elimina archivos o carpetas:
```bash
rm archivo.txt                      # Borrar un archivo
rm -rf carpeta-vieja/               # Borrar una carpeta completa y todo lo que tenga dentro
```
- `-r` *(recursive)*: Borra el directorio y todo su contenido.
- `-f` *(force)*: No pide confirmación ni muestra errores si el archivo no existe.
> ⚠️ **Precaución:** En Linux no existe la "Papelera de reciclaje". Lo que borras con `rm -rf` desaparece para siempre.

#### `nano`
El editor de texto en terminal más rápido y amigable:
```bash
nano .env
```
- Guardar: `Ctrl + O` ➡️ `Enter`.
- Salir: `Ctrl + X`.

---

### 2.2 Gestión de Paquetes y Software

En Ubuntu / Debian, el gestor oficial es `apt`:

```bash
# 1. Actualizar la lista de programas disponibles
sudo apt update

# 2. Actualizar los programas instalados a su última versión
sudo apt upgrade -y

# 3. Instalar un nuevo programa (ej. Nginx, Git, Htop, Unzip)
sudo apt install -y nginx git htop unzip

# 4. Desinstalar un programa y limpiar basura sobrante
sudo apt remove nginx
sudo apt autoremove -y
```

---

### 2.3 Control de Servicios (Systemd)

Casi todo programa en un VPS (Nginx, MySQL, Node, PHP-FPM, Docker) corre como un servicio administrado por **`systemctl`**:

```bash
sudo systemctl status nginx     # Ver si el servicio está activo (verde) o caído (rojo)
sudo systemctl start nginx      # Encender el servicio
sudo systemctl stop nginx       # Apagar el servicio
sudo systemctl restart nginx    # Reiniciar el servicio (apaga y prende)
sudo systemctl reload nginx     # Recargar configuración sin cortar conexiones activas
sudo systemctl enable nginx     # Hacer que arranque automáticamente si el VPS se reinicia
```

---

### 2.4 Monitoreo de Recursos en Vivo

#### `df -h` *(Disk Free)*
Te muestra cuánto espacio te queda en el disco duro:
```bash
df -h
```
- `-h` *(human readable)*: En gigabytes (GB) y megabytes (MB). Revisa la fila `/` (raíz). Si ves más del 90%, tu servidor está en peligro de colapso.

#### `free -h` *(Memory Free)*
Muestra el consumo de memoria RAM y Swap:
```bash
free -h
```
- Permite ver cuánta RAM total tienes, cuánta está ocupada (`used`) y cuánta disponible (`available`).

#### `htop`
El administrador de tareas interactivo para Linux:
```bash
htop
```
- Muestra una gráfica en tiempo real de cada núcleo de la CPU, consumo de RAM y la lista de todos los procesos corriendo.
- Presiona `F6` para ordenar por consumo de memoria o CPU.
- Presiona `q` para salir.

---

### 2.5 Seguridad y Firewall

#### `ufw` *(Uncomplicated Firewall)*
El cortafuegos indispensable en cualquier VPS:
```bash
sudo ufw status                # Ver qué puertos están abiertos
sudo ufw allow 22/tcp          # Abrir el puerto SSH (¡obligatorio antes de activarlo!)
sudo ufw allow 80/tcp          # Abrir HTTP
sudo ufw allow 443/tcp         # Abrir HTTPS
sudo ufw enable                # Activar el firewall
```

---

## 3. Nivel 2: Los INTERMEDIOS (Diagnóstico y Emergencias - 8% de tu tiempo)

Los usas cuando algo falla, para investigar causas o transferir datos.

---

### 3.1 Lectura y Monitoreo de Logs en Vivo

#### `tail -f` *(Follow)*
Muestra las últimas líneas de un archivo y se queda escuchando en vivo los nuevos errores que ocurren:
```bash
# Ver errores de Nginx en vivo mientras pruebas tu web en el navegador:
sudo tail -f /var/log/nginx/error.log

# Ver las últimas 50 líneas:
sudo tail -n 50 /var/log/nginx/error.log
```

#### `journalctl -u`
Consulta los registros del diario de systemd para cualquier servicio:
```bash
# Ver los logs en vivo de un servicio Node.js o MySQL:
sudo journalctl -u mi-api-node -f
```

---

### 3.2 Búsqueda de Archivos y Texto

#### `grep -rnw`
Busca una palabra dentro de cientos de archivos a la vez:
```bash
grep -rnw '/var/www/mi-app' -e 'DB_PASSWORD'
```
- `-r`: Recursivo (busca dentro de todas las subcarpetas).
- `-n`: Muestra el número de línea exacto donde encontró la palabra.
- `-w`: Busca la palabra exacta.

#### `find`
Encuentra archivos por su nombre o tipo:
```bash
# Buscar dónde está el archivo de configuración de PHP en todo el VPS:
sudo find /etc -name "php.ini"

# Buscar todos los archivos que pesen más de 100 MB en el servidor:
sudo find / -type f -size +100M
```

---

### 3.3 Redes, Puertos y Conectividad

#### `ss -tulpn` o `netstat -tulpn`
Muestra qué programas están escuchando en qué puertos:
```bash
sudo ss -tulpn
```
- `-t`: TCP.
- `-u`: UDP.
- `-l`: Puertos en escucha (*listening*).
- `-p`: Muestra el nombre del proceso y el ID (`PID`).
- `-n`: Números de puerto (muestra `3306` en lugar de la palabra `mysql`).

#### `lsof -i :PUERTO`
Responde a la pregunta: *"¿Quién demonios tiene ocupado el puerto 3000 o el puerto 80?"*:
```bash
sudo lsof -i :3000
# Salida: node 18452 deployer ... LISTEN
```

#### `curl -I`
Prueba las cabeceras HTTP de una web o API desde la terminal:
```bash
curl -I https://google.com
# Salida rápida: HTTP/2 200, fecha, servidor, content-type
```

---

### 3.4 Procesos Trabados y Memoria

#### `ps aux | grep ...`
Busca un proceso específico que esté corriendo:
```bash
ps aux | grep node
```

#### `kill -9 PID` y `pkill`
Mata procesos que congelaron el servidor:
```bash
# Matar por el número de ID del proceso (PID):
sudo kill -9 18452

# Matar todos los procesos que se llamen "node" de golpe:
sudo pkill -f node
```

---

### 3.5 Compresión y Respaldos Rápidos

#### `tar` (El estándar universal de Linux)
```bash
# Comprimir una carpeta entera en un archivo .tar.gz:
tar -czvf respaldo_sitio.tar.gz /var/www/mi-sitio/

# Descomprimir un archivo .tar.gz:
tar -xzvf respaldo_sitio.tar.gz
```
- `-c`: Crear (*create*).
- `-x`: Extraer (*extract*).
- `-z`: Comprimir con `gzip`.
- `-v`: Modo visual (*verbose*, te muestra cada archivo comprimido).
- `-f`: Especifica el nombre del archivo (*file*).

---

## 4. Nivel 3: Los MENOS NECESARIOS (Rara vez usados o prescindibles - 2%)

Muchos tutoriales te enseñan estos comandos, pero en el 99% del trabajo diario en un VPS moderno **casi nunca los vas a necesitar** o existen formas mucho más prácticas de hacerlo.

---

### 4.1 Comandos con mejores alternativas modernas

| Comando antiguo | ¿Para qué servía? | ¿Por qué es prescindible? | Alternativa moderna recomendada |
| :--- | :--- | :--- | :--- |
| **`netstat`** | Ver conexiones de red. | Desactualizado en distribuciones modernas; muchos Linux ya no lo traen instalado por defecto. | Usa **`ss`** (más rápido y nativo). |
| **`traceroute`** | Ver los saltos de red de una IP. | La mayoría de routers de internet bloquean paquetes ICMP/UDP de traceroute hoy en día. | Usa **`mtr`** o un simple `ping`. |
| **`sed` / `awk`** | Procesar y reemplazar flujos de texto en consola con scripts regex complejos. | Son poderosos pero tienen una curva de aprendizaje enorme y críptica. Para modificar un archivo, un desarrollador prefiere abrirlo con `nano`. | **`nano`** o scripts cortos en Node / Python. |
| **`ifconfig`** | Ver interfaces de red e IPs. | Obsoleto (*deprecated*) desde hace años. | Usa **`ip a`** o **`ip route`**. |

---

### 4.2 Comandos antiguos o ultra especializados

- **`uname -a`**: Muestra la versión del kernel de Linux. Lo usas una sola vez cada 6 meses si necesitas saber si tu Linux es de 64 bits.
- **`whoami`** o **`id`**: Te dice con qué usuario estás conectado. Si configuras bien el prompt de tu terminal, siempre verás tu usuario antes del `@` (ej. `deployer@vps:`).
- **`dmesg`**: Muestra el log de inicio de hardware del kernel de Linux. En un VPS en la nube (AWS, DigitalOcean, Linode, Hetzner), el hardware está virtualizado, así que casi nunca revisarás esto salvo falla crítica del hipervisor.
- **`nice` / `renice`**: Cambia la prioridad de uso de CPU de un proceso. En un servidor web moderno, los planificadores de Linux y los servidores web manejan la carga automáticamente.
- **`chattr +i` / `lsattr`**: Atributos especiales (como hacer un archivo indestructible contra escritura). Extremadamente raro de usar en desarrollo web estándar.

---

### 4.3 Comandos de ALTO RIESGO que debes evitar

❌ **`rm -rf /*`** o **`rm -rf /`**: Borra el sistema operativo entero desde la raíz.
❌ **`chmod -R 777 /`**: Rompe la seguridad completa de Linux; el sistema ni siquiera te dejará conectar por SSH tras reiniciar porque SSH rechaza configuraciones con permisos 777.
❌ **`:(){ :|:& };:`** *(Fork bomb)*: Un bucle infinito que crea procesos hasta saturar la memoria y congelar el VPS obligándote a forzar un reinicio físico.

---

## 5. Guía de Emergencias: Qué Comandos Correr Cuando Algo Falla

Guarda esta sección como referencia rápida para cuando tengas un problema en producción.

---

### Emergencia A: "El disco de mi VPS está al 100%"

Cuando el disco se llena al 100%, MySQL colapsa y no te deja ni crear archivos temporales.

**Paso 1:** Confirma el espacio:
```bash
df -h
```

**Paso 2:** Averigua **qué carpeta gigante** está consumiendo todo tu disco con este truco profesional:
```bash
sudo du -sh /* 2>/dev/null | sort -h
```
*(Analiza cada carpeta de la raíz y las ordena de menor a mayor tamaño en GB).*

**Paso 3:** Entra a la carpeta más pesada (generalmente `/var/log` o `/var/lib/docker`) y borra logs acumulados de meses:
```bash
# Limpiar logs de systemd antiguos:
sudo journalctl --vacuum-time=3d

# Limpiar caché de paquetes de Ubuntu:
sudo apt clean
```

---

### Emergencia B: "Mi web da Error 502 Bad Gateway"

El error `502` significa: **"Nginx está vivo, pero la app que está detrás (Node.js o PHP-FPM) se murió o no contesta"**.

**Paso 1:** Revisa el estado de tu backend:
```bash
# Si es Node con PM2:
pm2 status

# Si es PHP-FPM:
sudo systemctl status php8.2-fpm

# Si es un servicio Systemd:
sudo systemctl status mi-api
```

**Paso 2:** Si está caído, reinícialo:
```bash
sudo systemctl restart php8.2-fpm
# o: pm2 restart all
```

**Paso 3:** Lee el log de error de Nginx para ver el motivo exacto del rechazo:
```bash
sudo tail -n 20 /var/log/nginx/error.log
```

---

### Emergencia C: "El puerto 3000 o 3306 ya está ocupado"

Ocurre cuando intentas arrancar Node o un contenedor y dice `EADDRINUSE: address already in use :::3000`.

**Paso 1:** Identifica qué proceso lo tiene secuestrado:
```bash
sudo lsof -i :3000
```
Verás una tabla con el nombre del comando y un número en la columna `PID` (ej: `14205`).

**Paso 2:** Fuerza el cierre del proceso con su PID:
```bash
sudo kill -9 14205
```

**Paso 3:** Vuelve a arrancar tu aplicación con normalidad.

---

## 6. Resumen Final: La Pirámide de Comandos

```text
                  ▲
                 / \
                /   \     Nivel 3: 2% (Casi nunca los usas)
               /  *  \    [traceroute, sed, awk, dmesg, strace, ifconfig]
              /-------\
             /         \    Nivel 2: 8% (Diagnóstico y Emergencias)
            /     *     \   [tail -f, grep, find, lsof, ss, kill -9, tar]
           /-------------\
          /               \   Nivel 1: 90% (El pan de cada día)
         /        *        \  [cd, ls -la, nano, cp, mv, rm -rf, apt,
        /___________________\  systemctl, df -h, free -h, htop, ufw, ssh]
```

Dominando únicamente los comandos del **Nivel 1** y teniendo a mano los del **Nivel 2**, tendrás el **100% de la capacidad técnica** necesaria para administrar cualquier VPS en producción de forma profesional y segura.
