# Guía Paso a Paso: Cómo Instalar y Configurar Docker en Windows (Desde Cero)

Esta guía te enseñará desde cero cómo instalar, configurar y dominar **Docker Desktop en Windows (10 u 11)** utilizando el motor moderno **WSL 2 (Windows Subsystem for Linux)**, y cómo dejarlo listo para correr aplicaciones como **n8n**, bases de datos y otros servicios sin que ralentice tu computadora.

---

## 📋 Tabla de Contenidos
1. [¿Qué es Docker y cómo funciona en Windows?](#1-qué-es-docker-y-cómo-funciona-en-windows)
2. [Requisitos Previos y Verificación de Virtualización](#2-requisitos-previos-y-verificación-de-virtualización)
3. [Paso 1: Instalar y Activar WSL 2 (Motor de Linux para Windows)](#paso-1-instalar-y-activar-wsl-2-motor-de-linux-para-windows)
4. [Paso 2: Descargar e Instalar Docker Desktop](#paso-2-descargar-e-instalar-docker-desktop)
5. [Paso 3: Configuración Óptima de Docker Desktop](#paso-3-configuración-óptima-de-docker-desktop)
6. [Paso 4: El Truco Pro — Limitar el Consumo de Memoria RAM de WSL 2](#paso-4-el-truco-pro--limitar-el-consumo-de-memoria-ram-de-wsl-2)
7. [Paso 5: Comprobar que Docker Funciona en tu Terminal](#paso-5-comprobar-que-docker-funciona-en-tu-terminal)
8. [Paso 6: Conceptos Básicos de Docker (Con Analogías Simples)](#paso-6-conceptos-básicos-de-docker-con-analogías-simples)
9. [Paso 7: Correr tu Primer n8n con Docker](#paso-7-correr-tu-primer-n8n-con-docker)
10. [Solución de Errores Comunes en Windows](#solución-de-errores-comunes-en-windows)

---

## 1. ¿Qué es Docker y cómo funciona en Windows?

Imagina que quieres probar n8n, MySQL, Redis o Python. Normalmente tendrías que instalar cada programa en tu Windows, ensuciar tu registro, lidiar con variables de entorno y si algo falla, desinstalarlo es molesto.

**Docker** soluciona esto empaquetando cada aplicación en una "caja cerrada" llamada **Contenedor**:
- El contenedor incluye todo lo necesario para funcionar (Node.js, librerías, dependencias).
- No modifica tu Windows.
- Cuando ya no lo necesitas, lo borras con un solo clic y tu máquina queda 100% limpia.

En Windows, Docker se ejecuta sobre **WSL 2**, lo que le da velocidad casi nativa de Linux.

---

## 2. Requisitos Previos y Verificación de Virtualización

Para que Docker funcione, tu procesador debe tener activada la **Virtualización por Hardware**.

### ¿Cómo comprobarlo en 5 segundos?
1. Presiona `Ctrl + Shift + Esc` para abrir el **Administrador de Tareas** de Windows.
2. Ve a la pestaña **Rendimiento** (Performance) y haz clic en **CPU**.
3. En la parte inferior derecha, busca:
   👉 **Virtualización:** debe decir **Habilitado** (Enabled).

> ⚠️ **Si dice "Deshabilitado":** Debes reiniciar tu PC, entrar a la BIOS/UEFI (presionando `F2`, `F10`, `F12` o `Supr` al encender) y activar **Intel Virtualization Technology (VT-x)** o **AMD-V / SVM Mode**.

---

## Paso 1: Instalar y Activar WSL 2 (Motor de Linux para Windows)

WSL 2 permite a Windows ejecutar un núcleo real de Linux de forma ultrarrápida.

1. Abre **PowerShell** o la **Terminal de Windows** como **Administrador** (Clic derecho ➡️ *Ejecutar como administrador*).
2. Ejecuta el siguiente comando:

```powershell
wsl --install
```

#### 🔍 ¿Qué hace este comando?
- Habilita las características opcionales de Windows para máquinas virtuales.
- Descarga e instala el kernel de Linux más reciente de Microsoft.
- Instala una distribución de Ubuntu por defecto.

3. **Reinicia tu computadora** para que Windows aplique los cambios del sistema operativo.

> 💡 **Nota:** Si ya tenías WSL instalado de antes, asegúrate de tener la versión 2 ejecutando:
> ```powershell
> wsl --update
> wsl --set-default-version 2
> ```

---

## Paso 2: Descargar e Instalar Docker Desktop

1. Ve al sitio oficial de Docker: [docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
2. Haz clic en el botón azul **Download for Windows**.
3. Ejecuta el instalador descargado (`Docker Desktop Installer.exe`).
4. En la pantalla de opciones de instalación, asegúrate de que esté **marcada** la casilla:
   - ✅ **Use WSL 2 instead of Hyper-V (recommended)**
5. Haz clic en **OK** y espera a que termine la instalación (tarda unos 2 a 5 minutos).
6. Al finalizar, haz clic en **Close and restart** (o reinicia tu PC).

---

## Paso 3: Configuración Óptima de Docker Desktop

1. Abre el menú Inicio de Windows y abre **Docker Desktop**.
2. Acepta los términos de servicio del acuerdo de usuario.
3. Puedes hacer clic en **Continue without signing in** (no es obligatorio crear una cuenta para uso local).
4. Verás la ventana principal de Docker con un ícono de engranaje ⚙️ (Settings) arriba a la derecha. Haz clic en ⚙️ **Settings**:

### Configuraciones recomendadas:
- En **General**:
  - ✅ Marca: *Start Docker Desktop when you log in* (si quieres que inicie con Windows, o desmárcalo si solo quieres abrirlo cuando vayas a trabajar).
  - ✅ Marca: *Use the WSL 2 based engine*.
- En **Resources ➡️ WSL Integration**:
  - Asegúrate de que esté habilitada la integración con tu distribución de Linux (`Ubuntu`).
- Haz clic en **Apply & restart** si hiciste algún cambio.

---

## Paso 4: El Truco Pro — Limitar el Consumo de Memoria RAM de WSL 2

Por defecto, WSL 2 puede llegar a consumir hasta el 50% o más de tu memoria RAM si no se limita. Con este truco le pondrás un tope para que tu PC siempre vaya rápida.

1. Presiona `Win + R`, escribe:
   ```text
   %USERPROFILE%
   ```
   y presiona `Enter`. Se abrirá tu carpeta personal de usuario (ej. `C:\Users\TuNombre`).

2. Crea un archivo de texto llamado **`.wslconfig`** (con el punto al inicio).
3. Ábrelo con el Bloc de notas y pega lo siguiente:

```ini
[wsl2]
memory=4GB       # Limita la RAM máxima que puede usar Docker (ej. 4GB o 3GB)
processors=2     # Limita los núcleos de CPU que puede usar
```

4. Guarda el archivo.
5. Abre PowerShell y reinicia WSL para aplicar el límite:
   ```powershell
   wsl --shutdown
   ```
*(La próxima vez que Docker inicie, nunca consumirá más de esos 4 GB de RAM).*

---

## Paso 5: Comprobar que Docker Funciona en tu Terminal

Abre cualquier terminal (PowerShell, CMD, o Git Bash) y ejecuta:

### 1. Comprobar la versión instalada:
```powershell
docker --version
docker compose version
```

### 2. Ejecutar el contenedor de prueba oficial de Docker:
```powershell
docker run hello-world
```

#### 🔍 ¿Qué verás?
Docker descargará una imagen diminuta llamada `hello-world` y te mostrará un mensaje que dice:
> *"Hello from Docker! This message shows that your installation appears to be working correctly."*

Si ves ese mensaje, **¡Docker está 100% instalado y funcionando perfectamente en tu Windows!** 🎉

---

## Paso 6: Conceptos Básicos de Docker (Con Analogías Simples)

Para entender qué haces cuando usas Docker, solo necesitas entender 4 conceptos:

| Concepto | Analogía de la vida real | ¿Qué es en Docker? |
| :--- | :--- | :--- |
| **Imagen (Image)** | La receta de un pastel o el archivo ISO instalador. | La plantilla de solo lectura que contiene el programa (ej. la imagen de `n8n` o de `mysql`). |
| **Contenedor (Container)** | El pastel ya horneado en tu mesa. | La instancia viva y en ejecución de la imagen. Puedes crear 5 contenedores distintos a partir de la misma imagen. |
| **Puerto (Port Mapping)** | El número de puerta de tu casa. | El túnel que conecta un puerto de tu Windows con el puerto dentro del contenedor (ej. `-p 5678:5678`). |
| **Volumen (Volume)** | Un disco duro externo conectado. | Una carpeta en tu disco donde se guardan los datos para que no se borren cuando apagues el contenedor. |

---

## Paso 7: Correr tu Primer n8n con Docker

Ahora que ya tienes Docker listo, poner a correr **n8n** es tan simple como ejecutar este único comando en tu terminal:

```powershell
docker run -d --name n8n_local -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

#### 🔍 ¿Qué hace cada parte de este comando?
- `docker run`: Crea y arranca el contenedor.
- `-d` *(detached)*: Corre en segundo plano para que no te bloquee la terminal.
- `--name n8n_local`: Le pone el nombre `n8n_local` al contenedor.
- `-p 5678:5678`: Abre la puerta para que puedas entrar desde tu Windows en `http://localhost:5678`.
- `-v n8n_data:/home/node/.n8n`: Crea un volumen seguro para guardar todos tus flujos de n8n.
- `docker.n8n.io/n8nio/n8n`: La imagen oficial de n8n.

### ¿Cómo entrar?
Abre tu navegador y entra a:
👉 **`http://localhost:5678`**

---

### Comandos útiles del día a día para gestionar tus contenedores:

```powershell
# Ver qué contenedores están corriendo actualmente
docker ps

# Ver todos los contenedores (incluso los apagados)
docker ps -a

# Detener n8n
docker stop n8n_local

# Volver a encender n8n
docker start n8n_local

# Ver los logs en vivo (por si hay algún error)
docker logs -f n8n_local

# Eliminar el contenedor (tus flujos siguen a salvo en el volumen)
docker rm n8n_local
```

---

## Solución de Errores Comunes en Windows

### ❌ 1. `Docker Desktop - WSL 2 installation is incomplete`
- **Causa:** Falta el paquete de actualización del kernel de Linux.
- **Solución:** Abre PowerShell como Administrador y corre:
  ```powershell
  wsl --update
  ```
  Luego haz clic en "Restart" en el cuadro de diálogo de Docker.

### ❌ 2. `Hardware assisted virtualization and data execution protection must be enabled in the BIOS`
- **Causa:** La virtualización está desactivada en la tarjeta madre de tu PC.
- **Solución:** Entra a la BIOS de tu computadora al encender y activa `Intel VT-x` o `AMD-V`.

### ❌ 3. `Docker Desktop stops responding / starting forever`
- **Causa:** WSL se quedó colgado en memoria.
- **Solución:** Cierra Docker Desktop, abre PowerShell y ejecuta:
  ```powershell
  wsl --shutdown
  ```
  Luego vuelve a abrir Docker Desktop.
