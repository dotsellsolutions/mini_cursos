# Guía Paso a Paso: Cómo Instalar y Ejecutar n8n Localmente para Pruebas

**n8n** es una plataforma de automatización de flujos de trabajo (*workflow automation*) de código abierto, extensible y auto-hospedada (la alternativa libre a Zapier o Make). 

Esta guía te enseñará desde cero cómo instalar y ejecutar n8n en tu propia computadora local (Windows, macOS o Linux) para realizar pruebas, crear automatizaciones y conectar APIs sin costo.

---

## 📋 Tabla de Contenidos
1. [Requisitos Previos](#1-requisitos-previos)
2. [Método 1: Ejecución Rápida con Node.js / npx (El más fácil)](#método-1-ejecución-rápida-con-nodejs--npx-el-más-fácil)
   - [Opción A: Ejecutar sin instalar nada adicional (`npx`)](#opción-a-ejecutar-sin-instalar-nada-adicional-npx)
   - [Opción B: Instalación global con `npm`](#opción-b-instalación-global-con-npm)
3. [Método 2: Ejecución con Docker (El más limpio y recomendado)](#método-2-ejecución-con-docker-el-más-limpio-y-recomendado)
   - [Opción A: Con un solo comando `docker run`](#opción-a-con-un-solo-comando-docker-run)
   - [Opción B: Con `docker-compose.yml` (Persistente)](#opción-b-con-docker-composeyml-persistente)
4. [Paso 3: Primer Acceso y Configuración Inicial](#paso-3-primer-acceso-y-configuración-inicial)
5. [Paso 4: El Truco Pro — Recibir Webhooks en Local (Túnel Integrado)](#paso-4-el-truco-pro--recibir-webhooks-en-local-túnel-integrado)
6. [Manejo de Errores y Preguntas Frecuentes](#manejo-de-errores-y-preguntas-frecuentes)
7. [Resumen Rápido de Comandos](#resumen-rápido-de-comandos)

---

## 1. Requisitos Previos

Dependiendo del método que elijas, necesitarás:

- **Para Método 1 (Node.js):** Tener instalado **Node.js (versión 18.17+ o 20 LTS)** y `npm`. Puedes verificarlo en tu terminal con:
  ```bash
  node -v
  npm -v
  ```
  *(Si no lo tienes instalado, descárgalo desde [nodejs.org](https://nodejs.org)).*
- **Para Método 2 (Docker):** Tener instalado y abierto **Docker Desktop** ([docker.com](https://www.docker.com/products/docker-desktop/)).

---

## Método 1: Ejecución Rápida con Node.js / npx (El más fácil)

Este método es ideal si ya eres desarrollador y tienes Node.js instalado en tu máquina.

---

### Opción A: Ejecutar sin instalar nada adicional (`npx`)

`npx` permite descargar y ejecutar paquetes temporales de npm sin dejarlos instalados permanentemente.

Abre tu terminal (PowerShell, CMD, Git Bash o Terminal de macOS/Linux) y ejecuta:

```bash
npx n8n
```

#### 🔍 ¿Qué hace este comando?
- Descarga la versión más reciente de n8n en una caché temporal.
- Inicia el servidor local de n8n en el puerto `5678`.
- Muestra en consola la URL para entrar: `Editor is now accessible via: http://localhost:5678`.

> 💡 **Para detenerlo:** Simplemente presiona `Ctrl + C` en la terminal.

---

### Opción B: Instalación global con `npm`

Si planeas usar n8n frecuentemente en local y quieres que inicie al instante:

1. **Instalar n8n de forma global:**
   ```bash
   npm install -g n8n
   ```

   #### 🔍 Explicación de banderas:
   - `-g` *(global)*: Instala el ejecutable en tu sistema para que el comando `n8n` esté disponible desde cualquier carpeta de tu terminal.

2. **Iniciar n8n:**
   ```bash
   n8n
   ```

3. **Para actualizar n8n a la última versión en el futuro:**
   ```bash
   npm update -g n8n
   ```

---

## Método 2: Ejecución con Docker (El más limpio y recomendado)

Docker es el método preferido por la comunidad porque aísla n8n de tu sistema operativo y no requiere instalar Node.js ni lidiar con dependencias.

---

### Opción A: Con un solo comando `docker run`

Asegúrate de que Docker Desktop esté abierto y ejecuta en tu terminal:

```bash
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

#### 🔍 Explicación detallada de cada bandera:
- `docker run`: Descarga la imagen (si no la tienes) y crea un contenedor en ejecución.
- `-it`: Modo interactivo con terminal asignada para ver los logs en vivo.
- `--rm`: Elimina el contenedor al detenerlo (para no acumular contenedores basura). Como usamos un volumen persistente, tus workflows no se pierden.
- `--name n8n`: Asigna un nombre fácil de recordar al contenedor (`n8n`).
- `-p 5678:5678`: Mapea el puerto local `5678` de tu computadora al puerto interno `5678` del contenedor.
- `-v n8n_data:/home/node/.n8n`: Crea un volumen llamado `n8n_data` que guarda de forma persistente tus workflows, credenciales y base de datos SQLite en tu disco duro.
- `docker.n8n.io/n8nio/n8n`: La imagen oficial de n8n.

---

### Opción B: Con `docker-compose.yml` (Persistente en Segundo Plano)

Si quieres dejar n8n configurado en una carpeta para encenderlo y apagarlo fácilmente:

1. Crea una carpeta para tu proyecto n8n:
   ```bash
   mkdir n8n-local
   cd n8n-local
   ```

2. Crea un archivo llamado `docker-compose.yml`:
   ```yaml
   services:
     n8n:
       image: docker.n8n.io/n8nio/n8n:latest
       container_name: n8n_local
       restart: unless-stopped
       ports:
         - "5678:5678"
       environment:
         - N8N_DEFAULT_BINARY_DATA_MODE=filesystem
         - GENERIC_TIMEZONE=America/Santo_Domingo # Cambia por tu zona horaria
         - TZ=America/Santo_Domingo
       volumes:
         - n8n_storage:/home/node/.n8n

   volumes:
     n8n_storage:
   ```

3. **Iniciar n8n en segundo plano:**
   ```bash
   docker compose up -d
   ```
   - `-d` *(detached)*: Corre el contenedor en segundo plano, liberando tu terminal.

4. **Comandos útiles con Docker Compose:**
   ```bash
   # Ver los logs en tiempo real
   docker compose logs -f

   # Detener n8n
   docker compose stop

   # Iniciar de nuevo n8n
   docker compose start

   # Apagar y destruir el contenedor (los datos se quedan guardados en el volumen)
   docker compose down
   ```

---

## Paso 3: Primer Acceso y Configuración Inicial

1. Abre tu navegador web favorito e ingresa a:
   👉 **`http://localhost:5678`**

2. **Crear la cuenta de Administrador (Owner Account):**
   - Como es tu primera vez, n8n te pedirá crear una cuenta local:
     - **Email:** Tu correo (solo se usará para iniciar sesión local).
     - **First / Last Name:** Tu nombre.
     - **Password:** Contraseña para proteger tu instancia local.
   - Haz clic en **Next / Set up owner account**.

3. **Encuesta opcional:** Puedes saltarla o llenarla según prefieras.

4. ¡Listo! Ya estás dentro del lienzo interactivo de n8n listo para crear automatizaciones.

---

## Paso 4: El Truco Pro — Recibir Webhooks en Local (Túnel Integrado)

Uno de los mayores dolores de cabeza al probar n8n en local es: **¿Cómo recibo webhooks de Stripe, GitHub, WhatsApp o APIs externas si estoy en `localhost`?**

Normalmente tendrías que instalar herramientas como `ngrok` o `Cloudflare Tunnels`, pero n8n incluye un **túnel gratuito oficial integrado**.

### ¿Cómo activar el túnel de n8n?

Si usas el método de Node.js:
```bash
n8n start --tunnel
```

Si usas Docker:
```bash
docker run -it --rm --name n8n -p 5678:5678 -e WEBHOOK_URL=https://$(curl -s ifconfig.me)/ -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n start --tunnel
```

#### 🔍 ¿Qué hace este comando?
- n8n se conecta a los servidores de n8n Cloud y genera automáticamente una URL pública segura (ejemplo: `https://ancient-fox-82.hooks.n8n.cloud/webhook/...`).
- Cualquier servicio en Internet puede enviar peticiones a esa URL y llegarán directamente a tu n8n corriendo en tu PC.

> ⚠️ **Aviso:** El túnel es solo para pruebas de desarrollo. Para producción 24/7 debes desplegar n8n en un VPS.

---

## Manejo de Errores y Preguntas Frecuentes

### ❌ 1. Error: `Port 5678 is already in use`
- **Causa:** Tienes otra instancia de n8n u otro programa usando el puerto 5678.
- **Solución:** Cambia el puerto en la variable de entorno al iniciar:
  ```bash
  # En Node.js (PowerShell):
  $env:N8N_PORT=5679; n8n

  # En Node.js (Bash / Linux / Mac):
  N8N_PORT=5679 n8n

  # En Docker:
  docker run -it --rm -p 5679:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
  ```
  *(Luego ingresarías por `http://localhost:5679`).*

### ❌ 2. ¿Dónde se guardan mis workflows y credenciales?
- **En Node / npm:** En tu carpeta de usuario del sistema operativo:
  - Windows: `C:\Users\TuUsuario\.n8n\`
  - Linux / macOS: `~/.n8n/`
  Ahí dentro hay un archivo llamado `database.sqlite` que contiene todos tus flujos y contraseñas cifradas.
- **En Docker:** Dentro del volumen de Docker `n8n_data` o `n8n_storage`.

### ❌ 3. ¿Cómo restablecer la contraseña si la olvidé en local?
Puedes resetear la contraseña del usuario propietario ejecutando en tu terminal:
```bash
n8n user-management:reset
```
*(O en Docker: `docker exec -it n8n n8n user-management:reset`).*

---

## Resumen Rápido de Comandos

| Acción | Comando |
| :--- | :--- |
| **Prueba ultra rápida (sin instalar nada)** | `npx n8n` |
| **Instalar global en tu sistema** | `npm install -g n8n` |
| **Iniciar n8n instalado** | `n8n` |
| **Iniciar n8n con Túnel público para Webhooks** | `n8n start --tunnel` |
| **Correr en Docker con persistencia** | `docker run -it --rm -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n` |
| **URL de Acceso** | `http://localhost:5678` |
