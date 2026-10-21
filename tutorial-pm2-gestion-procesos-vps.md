# 🚀 Guía Definitiva de PM2: Gestión y Persistencia de Aplicaciones en Producción

PM2 (**Process Manager 2**) es el gestor de procesos en producción estándar para Node.js (y compatible con Python, Bash, PHP, binarios Go, etc.). Te permite mantener aplicaciones vivas 24/7, reiniciarlas automáticamente ante caídas imprevistas o reinicios del servidor VPS, escalar en clúster utilizando todos los núcleos de tu CPU, rotar logs y recargar código con **cero tiempo de inactividad** (*zero-downtime*).

---

## 📑 Tabla de Contenidos
1. [¿Por qué usar PM2?](#1-por-qué-usar-pm2)
2. [Instalación de PM2](#2-instalación-de-pm2)
3. [Comandos Esenciales del Ciclo de Vida](#3-comandos-esenciales-del-ciclo-de-vida)
4. [Persistencia Automática tras Reinicios del Servidor (Startup)](#4-persistencia-automática-tras-reinicios-del-servidor-startup)
5. [Gestión de Logs y Rotación Automática](#5-gestión-de-logs-y-rotación-automática)
6. [Monitoreo en Tiempo Real y Diagnóstico](#6-monitoreo-en-tiempo-real-y-diagnóstico)
7. [Archivo de Configuración Profesional (`ecosystem.config.js`)](#7-archivo-de-configuración-profesional-ecosystemconfigjs)
8. [Modo Clúster y Recargas con Cero Downtime](#8-modo-clúster-y-recargas-con-cero-downtime)
9. [Uso con Otros Lenguajes (Python, PHP, Scripts Bash)](#9-uso-con-otros-lenguajes-python-php-scripts-bash)
10. [Chuleta / Resumen Rápido de Comandos](#10-chuleta--resumen-rápido-de-comandos)

---

## 1. ¿Por qué usar PM2?

Cuando ejecutas una aplicación con `node index.js`:
- Si cierras la consola SSH, la aplicación se apaga.
- Si ocurre una excepción no capturada (`uncaughtException`), el proceso muere y tus usuarios ven un error 502/conexión rechazada.
- Si el VPS se reinicia por mantenimiento o corte de energía, nada vuelve a levantarse solo.
- Solo utilizas un único núcleo de tu CPU.

**PM2 soluciona todo esto:**
- Corre aplicaciones en segundo plano (*daemon*).
- Reinicia la aplicación instantáneamente ante un fallo o error fatal.
- Se integra con `systemd` para arrancar automáticamente cuando el servidor se encienda.
- Ofrece modo clúster para balancear carga entre todos los núcleos disponibles.

---

## 2. Instalación de PM2

PM2 se instala de forma global a través de `npm`:

```bash
sudo npm install -g pm2
```

### Verificar la instalación:
```bash
pm2 -v
```

> [!NOTE]
> Para actualizar PM2 en el futuro a una versión más reciente sin perder las aplicaciones que ya están corriendo:
> ```bash
> sudo npm install -g pm2 && pm2 update
> ```

---

## 3. Comandos Esenciales del Ciclo de Vida

### 3.1 Iniciar una aplicación

```bash
# Iniciar con un nombre descriptivo
pm2 start index.js --name "mi-api"

# Iniciar pasando argumentos a la aplicación
pm2 start index.js --name "mi-api" -- --port 3000

# Iniciar observando cambios en archivos (útil en pruebas, no recomendado en prod)
pm2 start app.js --name "frontend" --watch
```

#### 🔍 Explicación de banderas:
- `--name "nombre"`: Asigna un alias único al proceso. En lugar de referirte a él por un ID numérico (`0`, `1`), puedes usar su nombre en todos los comandos.
- `--watch`: Reinicia la app si algún archivo del directorio cambia.

---

### 3.2 Listar aplicaciones activas

```bash
pm2 list
# O su alias corto:
pm2 ls
```

Muestra una tabla con:
- **id**: Identificador numérico asignado por PM2.
- **name**: Nombre que le diste a la aplicación.
- **mode**: `fork` (modo estándar) o `cluster`.
- **status**: `online`, `stopping`, `stopped` o `errored`.
- **cpu**: Porcentaje actual de uso de procesador.
- **memory**: Consumo de memoria RAM.

---

### 3.3 Detener, Reiniciar y Eliminar procesos

Puedes usar el `id` o el `name` del proceso:

```bash
# Detener temporalmente (mantiene el registro en pm2 list)
pm2 stop mi-api

# Reiniciar el proceso
pm2 restart mi-api

# Eliminar del registro de PM2 (libera memoria y registros)
pm2 delete mi-api

# Aplicar acciones a TODOS los procesos simultáneamente:
pm2 stop all
pm2 restart all
pm2 delete all
```

---

## 4. Persistencia Automática tras Reinicios del Servidor (Startup)

Este es el paso más importante en un servidor de producción. Si el VPS se apaga o reinicia, PM2 debe levantarse automáticamente junto con todas tus apps.

### Paso 1: Generar el script de inicio de systemd
```bash
pm2 startup
```

PM2 detectará tu sistema operativo y generará una línea de comando que debes copiar y ejecutar con `sudo`. Por ejemplo:
```text
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u usuario --hp /home/usuario
```
*Copia ese comando exacto que te arrojó la terminal y ejecútalo.*

### Paso 2: Guardar la lista de procesos activos
Una vez que tus aplicaciones estén corriendo y en estado `online`:
```bash
pm2 save
```

> [!IMPORTANT]
> Cada vez que agregues una nueva app, elimines una vieja o cambies la configuración de procesos, **debes volver a ejecutar `pm2 save`**. De lo contrario, ante un reinicio del VPS se restaurará la lista guardada anteriormente.

---

## 5. Gestión de Logs y Rotación Automática

### 5.1 Ver logs en tiempo real

```bash
# Ver logs de todas las aplicaciones
pm2 logs

# Ver logs de una aplicación específica
pm2 logs mi-api

# Mostrar solo las últimas 100 líneas
pm2 logs mi-api --lines 100

# Ver solo los errores estándar (stderr)
pm2 logs mi-api --err
```

### 5.2 Limpiar historial de logs
```bash
pm2 flush
```

### 5.3 Instalar rotación automática de logs (Evitar llenar el disco)
Por defecto, si una aplicación genera millones de líneas de log, el archivo puede crecer hasta agotar el espacio en disco de tu VPS. Para evitarlo, instala el módulo oficial `pm2-logrotate`:

```bash
pm2 install pm2-logrotate
```

Configuraciones recomendadas para producción:
```bash
# Tamaño máximo por archivo de log antes de rotar (ej: 10MB)
pm2 set pm2-logrotate:max_size 10M

# Guardar un máximo de 7 archivos rotados
pm2 set pm2-logrotate:retain 7

# Comprimir los logs rotados (.gz)
pm2 set pm2-logrotate:compress true
```

---

## 6. Monitoreo en Tiempo Real y Diagnóstico

### 6.1 Dashboard interactivo en consola
PM2 incluye un monitor interactivo completo tipo `htop`:

```bash
pm2 monit
```
*(Presiona `Ctrl + C` para salir).*

Te permite ver:
- Consumo de CPU por proceso en vivo.
- Uso de memoria RAM en Megabytes.
- Logs entrantes en tiempo real.
- Metadatos HTTP e información del entorno.

### 6.2 Información detallada de un proceso
```bash
pm2 show mi-api
# O por ID:
pm2 describe 0
```
Muestra rutas de archivos, argumentos, variables de entorno activas, tiempo activo (*uptime*), número de reinicios por fallas y ubicación exacta de los archivos `.log`.

---

## 7. Archivo de Configuración Profesional (`ecosystem.config.js`)

Para proyectos medianos o grandes, no es recomendable escribir comandos largos con banderas en la terminal. Lo ideal es definir un archivo `ecosystem.config.js` en la raíz de tu proyecto.

### Generar plantilla básica:
```bash
pm2 init simple
```

### Ejemplo completo para Producción y Desarrollo:

Crea o edita `ecosystem.config.js`:

```javascript
module.exports = {
  apps: [
    {
      name: "backend-api",
      script: "./dist/index.js",
      instances: "max",             // Usa todos los núcleos de CPU (modo clúster)
      exec_mode: "cluster",
      autorestart: true,
      max_memory_restart: "500M",   // Reinicia si fuga memoria superando 500MB
      watch: false,                 // En producción siempre en false
      env: {
        NODE_ENV: "development",
        PORT: 3000
      },
      env_production: {
        NODE_ENV: "production",
        PORT: 8080
      },
      log_date_format: "YYYY-MM-DD HH:mm:ss Z",
      error_file: "./logs/err.log",
      out_file: "./logs/out.log",
      merge_logs: true
    }
  ]
};
```

### Comandos con Ecosystem:
```bash
# Iniciar usando entorno de producción
pm2 start ecosystem.config.js --env production

# Reiniciar todas las apps del archivo
pm2 restart ecosystem.config.js

# Recargar sin downtime
pm2 reload ecosystem.config.js --env production
```

---

## 8. Modo Clúster y Recargas con Cero Downtime

Node.js se ejecuta en un solo hilo (*single-thread*). Si tu VPS tiene 4 núcleos de CPU, un `node app.js` común solo aprovechará 1 núcleo (el 25% de la capacidad de procesamiento).

### 8.1 Levantar en clúster
```bash
# Usar todos los núcleos disponibles automáticamente:
pm2 start index.js -i max --name "api-cluster"

# O definir un número exacto de instancias (ej: 2 núcleos):
pm2 start index.js -i 2 --name "api-cluster"
```

PM2 integrará un balanceador de carga interno que distribuirá las peticiones entrantes entre las instancias.

### 8.2 Recargas con Cero Downtime (`pm2 reload`)

- **`pm2 restart`**: Mata el proceso inmediatamente y lo vuelve a levantar. Hay una fracción de segundo donde los usuarios recibirán un error de conexión caída.
- **`pm2 reload`**: En modo clúster, levanta la nueva instancia del código primero y solo cuando está lista apaga la vieja, una por una. **Ninguna petición de usuario se pierde.**

```bash
pm2 reload api-cluster
```

---

## 9. Uso con Otros Lenguajes (Python, PHP, Scripts Bash)

PM2 no está limitado a Node.js; puede vigilar y mantener activo cualquier proceso:

### Scripts de Python:
```bash
pm2 start bot.py --name "telegram-bot" --interpreter python3
```

### Servidor de desarrollo PHP integrado:
```bash
pm2 start "php -S 0.0.0.0:8000 -t public/" --name "php-app"
```

### Scripts de Shell / Tareas en Bash:
```bash
pm2 start worker.sh --name "worker-cola" --interpreter bash
```

---

## 10. Chuleta / Resumen Rápido de Comandos

| Comando | Acción |
| :--- | :--- |
| `pm2 start app.js --name "app"` | Inicia una aplicación con un nombre específico. |
| `pm2 start app.js -i max` | Inicia la app en modo clúster aprovechando todos los núcleos. |
| `pm2 list` o `pm2 ls` | Muestra el estado de todas las aplicaciones. |
| `pm2 stop <nombre/id>` | Detiene una aplicación sin borrarla. |
| `pm2 restart <nombre/id>` | Reinicia una aplicación (reinicia de golpe). |
| `pm2 reload <nombre/id>` | Recarga en caliente sin caída (*Zero Downtime*). |
| `pm2 delete <nombre/id>` | Elimina la aplicación del control de PM2. |
| `pm2 logs [nombre]` | Muestra los registros y mensajes de error en vivo. |
| `pm2 monit` | Abre el panel visual de métricas de CPU y Memoria. |
| `pm2 startup` | Configura el servicio del sistema operativo para arrancar al encender. |
| `pm2 save` | Guarda la lista de procesos actuales para el próximo arranque. |
| `pm2 flush` | Vacía todos los archivos de logs acumulados. |
