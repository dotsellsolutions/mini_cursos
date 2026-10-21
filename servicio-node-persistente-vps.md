# Guía Paso a Paso: Cómo Crear un Servicio Node.js Persistente en un VPS (Con Explicación de Comandos)

Esta guía te enseñará desde cero cómo desplegar una aplicación Node.js en un servidor virtual (**VPS con Ubuntu/Debian**) y configurarla para que se ejecute en segundo plano de forma persistente (que no se detenga al cerrar la terminal y se reinicie automáticamente si el servidor o la app fallan).

Además de los comandos, **cada paso incluye una explicación detallada de qué hace cada comando y cada parámetro/bandera**, para que entiendas a fondo cómo funciona la administración de servidores Linux.

---

## 📋 Tabla de Contenidos
1. [Requisitos Previos](#1-requisitos-previos)
2. [Paso 1: Conexión y Preparación del Servidor](#paso-1-conexión-y-preparación-del-servidor)
3. [Paso 2: Instalación de Node.js y npm](#paso-2-instalación-de-nodejs-y-npm)
4. [Paso 3: Creación de la Aplicación Node.js de Prueba](#paso-3-creación-de-la-aplicación-nodejs-de-prueba)
5. [Paso 4: Método A — Persistencia con PM2 (Recomendado)](#paso-4-método-a--persistencia-con-pm2-recomendado)
6. [Paso 5: Método B — Persistencia Nativa con Systemd](#paso-5-método-b--persistencia-nativa-con-systemd)
7. [Paso 6: Reverse Proxy con Nginx y Certificado SSL](#paso-6-reverse-proxy-con-nginx-y-certificado-ssl-opcional-pero-recomendado)
8. [Paso 7: Seguridad y Firewall (UFW)](#paso-7-seguridad-y-firewall-ufw)
9. [Glosario y Resumen de Comandos](#glosario-y-resumen-de-comandos)

---

## 1. Requisitos Previos

- Un VPS con **Ubuntu (20.04 / 22.04 / 24.04 LTS)** o **Debian (11 / 12)**.
- Acceso SSH con usuario `root` o permisos de administrador.
- Una terminal (PowerShell, Git Bash, WSL, terminal de macO0.
. S/Linux o PuTTY).

---

## Paso 1: Conexión y Preparación del Servidor

### 1.1 Conectarse por SSH
Abre tu terminal local y ejecuta:

```bash
ssh usuario@IP_DEL_VPS
# O si entras como root:
ssh root@IP_DEL_VPS
```

#### 🔍 ¿Qué hace este comando?
- `ssh`: Invoca el protocolo **Secure Shell**, un canal cifrado para comunicarse de forma segura con servidores remotos.
- `usuario@IP_DEL_VPS`: Especifica con qué usuario y a qué dirección IP pública de Internet deseas autenticarte.

---

### 1.2 Actualizar el sistema operativo
Una vez dentro del servidor, actualiza el índice de paquetes y el software existente:

```bash
sudo apt update && sudo apt upgrade -y
```

#### 🔍 ¿Qué hace este comando?
- `sudo`: *(SuperUser DO)* Ejecuta el comando con permisos de administrador (root).
- `apt`: El gestor de paquetes de distribuciones basadas en Debian/Ubuntu.
- `update`: Descarga la lista actualizada de los paquetes disponibles en los repositorios remotos (no instala nada aún, solo actualiza el catálogo).
- `&&`: Operador lógico que indica: *"si el primer comando finaliza con éxito (sin errores), ejecuta inmediatamente el siguiente"*.
- `upgrade`: Compara los programas instalados con el catálogo nuevo y descarga/instala las versiones más recientes.
- `-y`: Responde automáticamente *"Yes"* a todas las preguntas de confirmación de descarga e instalación, evitando que el proceso se detenga a pedir interacción.

---

### 1.3 Crear un usuario no-root (Seguridad)
Trabajar todo el tiempo como `root` es peligroso porque cualquier error puede dañar el sistema operativo completo. Creamos un usuario dedicado:

```bash
adduser deployer
```

#### 🔍 ¿Qué hace este comando?
- `adduser deployer`: Crea un nuevo usuario en el sistema llamado `deployer`, crea su carpeta personal `/home/deployer` y te pedirá asignarle una contraseña segura.

Ahora le damos permisos de administrador (`sudo`):

```bash
usermod -aG sudo deployer
```

#### 🔍 ¿Qué hace este comando?
- `usermod`: Modifica las propiedades de una cuenta de usuario existente.
- `-a` *(append)*: Añade al usuario al grupo especificado sin sacarlo de sus grupos actuales.
- `-G sudo`: Especifica el grupo objetivo (`sudo`), lo que le permite ejecutar comandos administrativos.

Cambiamos a la sesión del nuevo usuario:

```bash
su - deployer
```

#### 🔍 ¿Qué hace este comando?
- `su` *(switch user)*: Cambia la sesión actual a otro usuario.
- `-` *(guion)*: Inicia una sesión de login completa, cargando las variables de entorno, alias y configuración del directorio `HOME` del nuevo usuario (`/home/deployer`).

---

## Paso 2: Instalación de Node.js y npm

Para entornos de producción se recomienda la versión **LTS (Long Term Support)** de Node.js a través del repositorio oficial de NodeSource.

### 2.1 Instalar herramientas necesarias
```bash
sudo apt install -y curl build-essential
```

#### 🔍 ¿Qué hace este comando?
- `curl`: Herramienta para transferir datos y descargar archivos desde URLs a través de la terminal.
- `build-essential`: Paquete que incluye compiladores como `gcc`, `g++` y `make`, necesarios si alguna librería de Node.js compila módulos nativos en C/C++.

---

### 2.2 Descargar e integrar el repositorio de Node.js v20 (LTS)
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
```

#### 🔍 ¿Qué hace este comando?
- `curl -fsSL`:
  - `-f` *(fail silently)*: Si hay un error HTTP (ej. 404 o 500), no descarga páginas de error HTML.
  - `-s` *(silent)*: Modo silencioso, oculta la barra de progreso.
  - `-S` *(show error)*: Si ocurre un error, lo muestra aunque esté en modo silencioso.
  - `-L` *(location)*: Sigue redirecciones HTTP si la URL cambia.
- `|` *(pipe / tubería)*: Toma la salida del comando de la izquierda y la envía como entrada al comando de la derecha.
- `sudo -E bash -`:
  - `-E` *(preserve environment)*: Conserva las variables de entorno del usuario actual al ejecutar `sudo`.
  - `bash -`: Ejecuta el script descargado directamente en el intérprete de comandos `bash`.

---

### 2.3 Instalar Node.js y npm
```bash
sudo apt install -y nodejs
```

#### 🔍 ¿Qué hace este comando?
- Instala el binario de Node.js y el gestor de paquetes `npm` de forma conjunta.

Verifica que estén instalados correctamente:
```bash
node -v
npm -v
```

---

## Paso 3: Creación de la Aplicación Node.js de Prueba

### 3.1 Crear la carpeta del proyecto
```bash
mkdir -p ~/apps/mi-api-node
cd ~/apps/mi-api-node
```

#### 🔍 ¿Qué hace este comando?
- `mkdir -p`: Crea directorios. La bandera `-p` *(parents)* crea toda la ruta de carpetas intermedias si no existen, y no arroja error si ya existían.
- `~/`: Representa el directorio personal del usuario actual (ej. `/home/deployer`).
- `cd`: *(change directory)* Entra en la carpeta recién creada.

---

### 3.2 Inicializar el proyecto e instalar Express
```bash
npm init -y
npm install express
```

#### 🔍 ¿Qué hace este comando?
- `npm init -y`: Genera un archivo `package.json` con los valores por defecto sin hacer preguntas.
- `npm install express`: Descarga el framework web `express` en `node_modules` y lo registra como dependencia en `package.json`.

---

### 3.3 Crear el código del servidor
Creamos el archivo `app.js`:

```bash
nano app.js
```

#### 🔍 ¿Qué hace este comando?
- `nano`: Editor de texto en consola sencillo y rápido.

Pega el siguiente código en el editor:

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/', (req, res) => {
    res.json({
        status: 'online',
        message: 'Servicio Node.js persistente en ejecución',
        serverTime: new Date().toISOString()
    });
});

app.get('/health', (req, res) => {
    res.status(200).send('OK');
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Servidor Node escuchando en el puerto ${PORT}`);
});
```

> **Para guardar en Nano:** Presiona `Ctrl + O`, luego `Enter` para confirmar el nombre del archivo, y `Ctrl + X` para salir.

---

## Paso 4: Método A — Persistencia con PM2 (Recomendado)

Si ejecutas `node app.js`, la aplicación se cerrará en cuanto desconectes la sesión SSH. **PM2** es un gestor de procesos profesional que mantiene la aplicación viva en segundo plano.

### 4.1 Instalar PM2 de forma global
```bash
sudo npm install -g pm2
```

#### 🔍 ¿Qué hace este comando?
- `-g` *(global)*: Instala el paquete a nivel del sistema operativo para que el comando `pm2` esté disponible en cualquier terminal y usuario.

---

### 4.2 Iniciar la aplicación con PM2
Asegúrate de estar en `~/apps/mi-api-node` y ejecuta:

```bash
pm2 start app.js --name "mi-api"
```

#### 🔍 ¿Qué hace este comando?
- `pm2 start app.js`: Ejecuta `app.js` como un proceso en segundo plano (daemon).
- `--name "mi-api"`: Asigna un nombre descriptivo al proceso para no tener que identificarlo por su ID numérico.

*(Opcional - Modo Clúster)*:
```bash
pm2 start app.js --name "mi-api" -i max
```
- `-i max` *(instances max)*: Detecta cuántos núcleos de CPU tiene tu VPS y lanza una instancia del servidor por cada núcleo con balanceo de carga automático.

---

### 4.3 Hacer que PM2 inicie automáticamente con el sistema operativo
Este es el paso fundamental para la persistencia. Si el servidor se apaga o reinicia, PM2 volverá a levantar tus apps.

**1. Generar el script de inicio:**
```bash
pm2 startup
```

#### 🔍 ¿Qué hace este comando?
- Analiza qué sistema de inicio usa tu Linux (generalmente `systemd`) y genera una línea de comando específica configurada con la ruta absoluta de Node y tu usuario.

**2. Ejecutar el comando generado:**
La terminal te mostrará un comando largo similar a este (cópialo y ejecútalo):
```bash
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2-startup install -u deployer --hp /home/deployer
```

**3. Guardar el estado actual de procesos:**
```bash
pm2 save
```

#### 🔍 ¿Qué hace este comando?
- Toma una "fotografía" de todos los procesos que están corriendo en PM2 en este momento y los guarda en un archivo de volcado (`~/.pm2/dump.pm2`). Cuando el VPS inicie, restaurará exactamente esta lista.

---

### 4.4 Comandos útiles de PM2 explicados

```bash
# Ver una tabla con el estado, uso de CPU, memoria y uptime de tus apps
pm2 list

# Abrir un panel visual interactivo con métricas en tiempo real
pm2 monit

# Ver los logs en tiempo real (equivalente a un "tail -f" de consola)
pm2 logs mi-api

# Reiniciar la aplicación (detiene y arranca el proceso)
pm2 restart mi-api

# Recarga sin tiempo de inactividad (recarga una a una las instancias en modo clúster)
pm2 reload mi-api

# Detener temporalmente el proceso sin eliminarlo de la lista
pm2 stop mi-api

# Eliminar el proceso de la lista de PM2
pm2 delete mi-api
```

---

## Paso 5: Método B — Persistencia Nativa con Systemd

Si prefieres no usar dependencias como PM2 y utilizar las herramientas nativas del núcleo Linux, puedes crear una unidad de servicio con **systemd**.

### 5.1 Crear el archivo del servicio
```bash
sudo nano /etc/systemd/system/mi-api.service
```

#### 🔍 ¿Qué hace este comando?
- Los archivos en `/etc/systemd/system/` definen servicios del sistema gestionados por el init de Linux (`systemd`).

---

### 5.2 Contenido y explicación del archivo `.service`
Pega la siguiente configuración:

```ini
[Unit]
Description=Servicio Node.js - Mi API
After=network.target

[Service]
Type=simple
User=deployer
WorkingDirectory=/home/deployer/apps/mi-api-node
ExecStart=/usr/bin/node /home/deployer/apps/mi-api-node/app.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production PORT=3000
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=mi-api-node

[Install]
WantedBy=multi-user.target
```

#### 🔍 Explicación línea por línea de la configuración:
- **`[Unit]`**: Metadatos e información de dependencias.
  - `Description`: Nombre legible para humanos del servicio.
  - `After=network.target`: Le indica a Linux que no intente arrancar este servicio hasta que la red esté completamente activa y lista.
- **`[Service]`**: Define cómo se ejecuta el proceso.
  - `Type=simple`: El proceso principal iniciado por `ExecStart` es el servicio en sí.
  - `User=deployer`: Ejecuta el código con los permisos del usuario `deployer`, nunca como `root`.
  - `WorkingDirectory`: La carpeta base donde se ubica el proyecto (para que las rutas relativas funcionen).
  - `ExecStart`: El comando exacto con **rutas absolutas** para arrancar la aplicación (`which node` para ver la ruta de Node).
  - `Restart=always`: Si la aplicación colapsa (crash, excepción no controlada o cierre inesperado), systemd la reiniciará automáticamente.
  - `RestartSec=10`: Espera 10 segundos antes de intentar el reinicio para evitar bucles infinitos de consumo de CPU si hay un error fatal de inicio.
  - `Environment`: Variables de entorno pasadas al proceso (`process.env`).
  - `StandardOutput` y `StandardError`: Envía los `console.log` y errores a los registros del sistema (`syslog`).
  - `SyslogIdentifier`: Etiqueta para filtrar los logs fácilmente.
- **`[Install]`**:
  - `WantedBy=multi-user.target`: Configura el servicio para que inicie en el modo multiusuario estándar (cuando el sistema operativo arranca normalmente).

---

### 5.3 Registrar, activar e iniciar el servicio

```bash
# 1. Recargar el demonio de systemd para que reconozca el nuevo archivo
sudo systemctl daemon-reload

# 2. Habilitar el servicio para que arranque automáticamente al encender el VPS
sudo systemctl enable mi-api

# 3. Iniciar el servicio inmediatamente
sudo systemctl start mi-api

# 4. Comprobar que está corriendo en verde (active / running)
sudo systemctl status mi-api
```

#### 🔍 ¿Qué hace `systemctl`?
- `systemctl`: La herramienta de control del sistema de servicios de systemd.
- `daemon-reload`: Vuelve a leer todos los archivos `.service` tras crear o modificar uno.
- `enable`: Crea los enlaces simbólicos necesarios en `/etc/systemd/system/multi-user.target.wants/` para el autoarranque.
- `start` / `status`: Arranca el servicio y muestra su estado actual, memoria y últimas líneas de log.

---

### 5.4 Comandos de gestión y logs con `journalctl`

```bash
# Ver los logs en tiempo real (similar a tail -f)
sudo journalctl -u mi-api -f

# Ver las últimas 100 líneas sin paginador
sudo journalctl -u mi-api -n 100 --no-pager

# Reiniciar el servicio tras hacer cambios en el código
sudo systemctl restart mi-api

# Detener el servicio
sudo systemctl stop mi-api
```

#### 🔍 ¿Qué hace `journalctl`?
- `journalctl`: Herramienta para consultar los registros del diario de systemd.
- `-u mi-api` *(unit)*: Filtra exclusivamente los logs de esa unidad/servicio.
- `-f` *(follow)*: Deja la terminal abierta mostrando nuevos logs a medida que se generan.
- `-n 100` *(lines)*: Muestra las últimas 100 entradas.
- `--no-pager`: Imprime la salida directamente en pantalla sin abrir el visor interactivo `less`.

---

## Paso 6: Reverse Proxy con Nginx y Certificado SSL (Opcional pero Recomendado)

En producción no se expone el puerto `3000` directamente a internet. Se utiliza **Nginx** en el puerto 80 (HTTP) y 443 (HTTPS) para recibir el tráfico y redirigirlo internamente a Node.js.

### 6.1 Instalar Nginx
```bash
sudo apt install -y nginx
```

---

### 6.2 Configurar el bloque de servidor
Creamos un archivo de configuración para nuestro sitio:

```bash
sudo nano /etc/nginx/sites-available/mi-api
```

Pega la siguiente configuración:

```nginx
server {
    listen 80;
    server_name tu-dominio.com www.tu-dominio.com; # Si no tienes dominio, coloca la IP del VPS

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 🔍 ¿Qué significan estas directivas de Nginx?
- `listen 80`: Escucha peticiones en el puerto estándar HTTP.
- `server_name`: El dominio o IP que responderá a esta configuración.
- `proxy_pass http://127.0.0.1:3000`: Toma la solicitud externa y la reenvía a tu app Node.js que corre localmente en el puerto 3000.
- `proxy_http_version 1.1`: Usa HTTP/1.1 (necesario para WebSockets y conexiones persistentes).
- `proxy_set_header Upgrade` y `Connection 'upgrade'`: Permite el uso de WebSockets si tu aplicación los necesita (ej. Socket.io).
- `proxy_set_header Host $host`: Pasa el encabezado de host original que el cliente solicitó a Node.js.
- `proxy_set_header X-Real-IP $remote_addr`: Le pasa a Node.js la dirección IP real del visitante (en lugar de la IP local 127.0.0.1).
- `proxy_set_header X-Forwarded-Proto $scheme`: Informa a Node si la petición original entró por `http` o `https`.

---

### 6.3 Habilitar el sitio y verificar Nginx
```bash
# 1. Crear enlace simbólico para activar el sitio
sudo ln -s /etc/nginx/sites-available/mi-api /etc/nginx/sites-enabled/

# 2. Probar que no haya errores de sintaxis en los archivos de configuración
sudo nginx -t

# 3. Recargar Nginx con la nueva configuración
sudo systemctl restart nginx
```

#### 🔍 ¿Qué hace `ln -s` y `nginx -t`?
- `ln -s` *(symbolic link)*: Crea un acceso directo en `sites-enabled/` que apunta al archivo real en `sites-available/`. Nginx solo lee las configuraciones que están en `sites-enabled/`.
- `nginx -t` *(test)*: Valida la sintaxis de todos los archivos de configuración de Nginx antes de recargarlo, evitando que el servidor web se caiga por un error tipográfico.

---

### 6.4 Instalar Certificado SSL Gratuito con Certbot (HTTPS)
Si tienes un dominio apuntando a la IP de tu VPS:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d tu-dominio.com -d www.tu-dominio.com
```

#### 🔍 ¿Qué hace este comando?
- `certbot`: Herramienta automática de Let's Encrypt para emitir y renovar certificados SSL/TLS gratuitos.
- `--nginx`: Usa el plugin de Nginx para modificar automáticamente el archivo de configuración, habilitar el puerto 443 (HTTPS) y configurar la redirección automática de HTTP a HTTPS.
- `-d` *(domain)*: Especifica para qué dominios o subdominios se emitirá el certificado.

---

## Paso 7: Seguridad y Firewall (UFW)

El cortafuegos (**UFW** - Uncomplicated Firewall) bloquea accesos no autorizados a puertos que no deben estar expuestos públicamente.

```bash
# 1. Permitir conexiones SSH (¡CRUCIAL! Si no lo permites, perderás el acceso al servidor)
sudo ufw allow ssh

# 2. Permitir tráfico web HTTP (Puerto 80)
sudo ufw allow http

# 3. Permitir tráfico web HTTPS (Puerto 443)
sudo ufw allow https

# 4. Activar el Firewall
sudo ufw enable

# 5. Consultar el estado y reglas activas
sudo ufw status verbose
```

#### 🔍 ¿Qué hace este comando?
- `ufw allow ssh`: Abre el puerto 22 (o el configurado para SSH) para que puedas seguir conectándote remotamente.
- `ufw allow http` / `https`: Abre los puertos 80 y 443 para que el público pueda consultar tu API a través de Nginx.
- El puerto 3000 de Node **no** se abre, manteniéndolo protegido dentro de la red local del servidor (`127.0.0.1`).

---

## Glosario y Resumen de Comandos

| Comando | Función principal |
| :--- | :--- |
| `pm2 start app.js --name "api"` | Inicia la app en segundo plano con PM2. |
| `pm2 save` | Guarda la lista de procesos para el reinicio del servidor. |
| `pm2 startup` | Genera y configura el script de inicio en el arranque del sistema. |
| `pm2 logs` | Muestra los registros de salida (`stdout` y `stderr`) en tiempo real. |
| `systemctl daemon-reload` | Notifica a systemd sobre cambios en archivos de servicio `.service`. |
| `systemctl enable <servicio>` | Habilita el inicio automático de un servicio al encender el VPS. |
| `journalctl -u <servicio> -f` | Monitorea los logs del servicio systemd en tiempo real. |
| `nginx -t` | Verifica la sintaxis de la configuración de Nginx. |
| `ufw status` | Muestra el estado del firewall y los puertos abiertos. |
