# 📘 Guía Completa de Git: Desde Cero hasta Producción en VPS

Esta guía está diseñada para que domines Git tanto en tu computadora local (Windows) como en tu servidor VPS, cubriendo el flujo diario, manejo de ramas, exclusión de archivos, despliegues y repositorios divididos (monorepos con `sparse-checkout`).

---

## 📑 Tabla de Contenidos
1. [Conceptos Básicos](#1-conceptos-básicos)
2. [Configuración Inicial](#2-configuración-inicial)
3. [El Ciclo de Trabajo Diario](#3-el-ciclo-de-trabajo-diario)
4. [Ignorar Archivos (.gitignore)](#4-ignorar-archivos-gitignore)
5. [Ramas (Branches): Trabajar sin Miedo](#5-ramas-branches-trabajar-sin-miedo)
6. [Monorepos: Descargar solo una Carpeta Específica (Sparse-Checkout)](#6-monorepos-descargar-solo-una-carpeta-específica-sparse-checkout)
7. [Deshacer Errores y Volver Atrás](#7-deshacer-errores-y-volver-atrás)
8. [Autenticación con GitHub sin Contraseñas (Tokens y Credenciales)](#8-autenticación-con-github-sin-contraseñas-tokens-y-credenciales)
9. [Cambiar de Usuario o URL del Repositorio (Solución a `remote: Repository not found`)](#9-cambiar-de-usuario-o-url-del-repositorio-solución-a-remote-repository-not-found)
10. [Chuleta / Resumen Rápido de Comandos](#10-chuleta--resumen-rápido-de-comandos)

---

## 1. Conceptos Básicos

Piensa en Git como una máquina del tiempo para tu código dividida en 3 zonas:

```text
[ Working Directory ]  ---> git add --->  [ Staging Area ]  ---> git commit --->  [ Repositorio Local ]  ---> git push ---> [ Remoto (GitHub) ]
 (Archivos editándose)                    (Archivos listos)                        (Historial guardado)                      (La nube)
```

- **Working Directory:** Tu carpeta donde editas código.
- **Staging Area:** El "carrito de compras" donde seleccionas qué cambios vas a guardar.
- **Commit:** Una foto instantánea inmutable de tu proyecto con un mensaje descriptivo.
- **Remote:** El servidor central (GitHub, GitLab, Bitbucket).

---

## 2. Configuración Inicial

Al instalar Git (en tu PC o en tu VPS), define tu identidad. Esto asociará tus commits con tu nombre y correo:

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu-correo@ejemplo.com"
```

Verificar configuración actual:
```bash
git config --list
```

---

## 3. El Ciclo de Trabajo Diario

### Paso A: Iniciar un proyecto local
```bash
# Dentro de la carpeta de tu proyecto:
git init

# Vincular con un repositorio remoto en GitHub:
git remote add origin https://github.com/TU_USUARIO/TU_REPOSITORIO.git
```

### Paso B: El flujo repetitivo diario (Add -> Commit -> Push)
Cada vez que avances o termines una función:

```bash
# 1. Ver qué archivos cambiaron
git status

# 2. Agregar todos los cambios al área de preparación (Staging)
git add .

# O agregar solo un archivo en específico:
# git add index.php

# 3. Guardar el cambio con una descripción clara
git commit -m "feat: añadido sistema de login con validación"

# 4. Subir los cambios a GitHub
git push origin main
```

### Paso C: Actualizar el servidor o tu PC con cambios remotos
```bash
# Descarga y fusiona los últimos cambios automáticamente:
git pull origin main
```

---

## 4. Ignorar Archivos (.gitignore)

**Regla de oro:** Nunca subas dependencias (`node_modules`, `vendor`), archivos con contraseñas (`.env`), ni archivos temporales.

Crea un archivo llamado exactamente `.gitignore` en la raíz de tu proyecto:

```text
# Dependencias
node_modules/
vendor/

# Archivos sensibles
.env
*.pem
id_rsa

# Compilados y builds
dist/
build/

# Logs del sistema
*.log
npm-debug.log*

# Archivos del sistema operativo o editores
.DS_Store
Thumbs.db
.vscode/
.idea/
```

> ⚠️ **¿Qué pasa si ya subiste un archivo por error y ahora quieres ignorarlo?**
> Git ya lo está rastreando, por lo que agregarlo a `.gitignore` no lo borrará del historial. Para dejar de rastrearlo sin borrar el archivo de tu disco:
> ```bash
> git rm --cached .env
> git commit -m "fix: dejar de rastrear .env"
> git push
> ```

---

## 5. Ramas (Branches): Trabajar sin Miedo

Las ramas te permiten crear funciones nuevas sin romper el código funcional que está en producción (`main`).

```text
main:     o-------o-----------o (estable / producción)
                   \         /
feature:            o---o---o   (nueva funcionalidad)
```

```bash
# Ver en qué rama estás
git branch

# Crear y moverte a una nueva rama
git checkout -b nueva-caracteristica
# (Equivalente en versiones modernas: git switch -c nueva-caracteristica)

# Trabajar normalmente en esa rama
git add .
git commit -m "Añadida pasarela de pago"

# Volver a la rama principal
git checkout main

# Fusionar los cambios de la rama a main
git merge nueva-caracteristica

# Eliminar la rama una vez fusionada
git branch -d nueva-caracteristica
```

---

## 6. Monorepos: Descargar solo una Carpeta Específica (Sparse-Checkout)

Si tienes un repositorio que contiene tanto el backend como el frontend en carpetas separadas (por ejemplo: `/api` y `/app`), no necesitas clonar todo en cada sitio de CloudPanel.

```text
MiRepositorio/
├── api/   ---> Servidor 1 (Node.js / PHP)
└── app/   ---> Servidor 2 (Vite / React / HTML)
```

### En el VPS para el sitio de la API:
```bash
cd /home/usuario/htdocs/api.misitio.com

# 1. Iniciar repo vacío y conectar el remoto
git init
git remote add origin https://github.com/TU_USUARIO/TU_REPOSITORIO.git

# 2. Habilitar la descarga selectiva
git sparse-checkout init --cone

# 3. Indicar qué carpeta deseas descargar
git sparse-checkout set api

# 4. Descargar solo esa carpeta y vincular la rama (para que luego baste con 'git pull')
git pull -u origin main
# (Si tu repositorio usa 'master' en vez de 'main', usa: git pull -u origin master)
```

### En el VPS para el sitio del Frontend (`app`):
```bash
cd /home/usuario/htdocs/app.misitio.com

git init
git remote add origin https://github.com/TU_USUARIO/TU_REPOSITORIO.git
git sparse-checkout init --cone
git sparse-checkout set app

# Descargar y vincular la rama:
git pull -u origin main
```

> **Para actualizar en el futuro:** Como usamos `-u` la primera vez, tu rama quedó vinculada. Ahora simplemente ejecutas `git pull` a secas dentro de cada carpeta y Git descargará únicamente los cambios de esa ruta.
> 
> *(Si hiciste `git pull` sin `-u` y te sale el mensaje de "Please specify which branch you want to merge with", vincula la rama ejecutando una sola vez: `git branch --set-upstream-to=origin/main main` o para master: `git branch --set-upstream-to=origin/master master`).*

---

## 7. Deshacer Errores y Volver Atrás

| Situación | Comando recomendado |
| :--- | :--- |
| Descartar cambios no guardados en un archivo | `git checkout -- nombre_archivo.php` o `git restore nombre_archivo.php` |
| Sacar un archivo del Staging (deshacer `git add`) | `git restore --staged nombre_archivo.php` |
| Modificar el mensaje del último commit | `git commit --amend -m "Nuevo mensaje corregido"` |
| Ver el historial de commits anteriores | `git log --oneline -n 10` |
| Volver temporalmente a un commit anterior | `git checkout ID_COMMIT` |
| Deshacer el último commit manteniendo tus cambios en los archivos | `git reset --soft HEAD~1` |

---

## 8. Autenticación con GitHub sin Contraseñas (Evitar ponerla siempre)

GitHub ya no acepta tu contraseña habitual de inicio de sesión para operaciones de Git (`push`, `pull`, `clone`). En su lugar, se usan **Tokens de Acceso Personal (PAT)** o **Llaves SSH**.

Aquí tienes cómo configurarlo para **no tener que escribirla nunca más**, tanto en Windows como en tu VPS:

---

### Opción A: Guardar credenciales con HTTPS y Token (El método más fácil)

#### Paso 1: Generar tu Personal Access Token (PAT) en GitHub
1. En GitHub, ve a tu foto de perfil (arriba a la derecha) > **Settings**.
2. Al final del menú izquierdo, haz clic en **Developer settings**.
3. Selecciona **Personal access tokens** > **Tokens (classic)**.
4. Clic en **Generate new token** > **Generate new token (classic)**.
5. Dale un nombre en *Note* (ej: `Mi PC Trabajo` o `VPS CloudPanel`) y marca la casilla **`repo`** (acceso completo a tus repositorios privados).
6. Haz clic en **Generate token** al final.
7. **Copia el token** (comienza por `ghp_...`). *Guárdalo en un lugar seguro porque solo se muestra una vez.*

#### Paso 2: Configurar para que no la vuelva a pedir

* **Si estás en Windows (Tu PC Local):**
  Git para Windows incluye el administrador seguro de Windows de forma automática. Si no está activado, ejecútalo en PowerShell o CMD:
  ```bash
  git config --global credential.helper manager
  ```
  La primera vez que hagas `git push`, se abrirá una ventana para iniciar sesión con tu navegador o te pedirá el token. Una vez puesto, Windows lo guarda de forma segura en el Administrador de Credenciales y **nunca más te lo pedirá**.

* **Si estás en tu VPS / Servidor Linux:**
  Ejecuta en la terminal del VPS:
  ```bash
  git config --global credential.helper store
  ```
  La siguiente vez que hagas un comando (`git pull`, `git push`, `git clone`):
  - **Username:** Tu usuario de GitHub.
  - **Password:** Pega tu token `ghp_...`.
  *(A partir de ese momento queda guardado en texto seguro dentro del servidor y no volverá a solicitarlo).*

  > **Alternativa temporal en VPS (por tiempo):** Si no quieres guardarlo indefinidamente, puedes guardarlo en memoria RAM por tiempo (ejemplo, 8 horas = 28800 segundos):
  > ```bash
  > git config --global credential.helper "cache --timeout=28800"
  > ```

---

### Opción B: Usar Llaves SSH (La opción más recomendada para VPS)

Con SSH no necesitas tokens ni contraseñas temporales; se basa en llaves criptográficas.

1. **Generar la llave en el VPS (o en tu PC):**
   ```bash
   ssh-keygen -t ed25519 -C "tu-correo@ejemplo.com"
   ```
   *(Presiona `Enter` a todo para dejar los valores por defecto).*

2. **Ver y copiar tu llave pública:**
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
   Copia todo el texto que imprima en pantalla (comienza por `ssh-ed25519 ...`).

3. **Agregarla a GitHub:**
   - En GitHub > **Settings** > **SSH and GPG keys** > Clic en **New SSH key**.
   - Pégale un título (ej: `VPS Produccion`) y pega la clave copiada.

4. **Usar el enlace SSH en tu repositorio:**
   Si tu repositorio estaba en HTTPS, cámbialo a SSH con:
   ```bash
   git remote set-url origin git@github.com:TU_USUARIO/TU_REPOSITORIO.git
   ```
   ¡Listo! Cada `git pull` o `git push` se autenticará automáticamente en segundo plano sin pedir nada.

---

## 9. Cambiar de Usuario o URL del Repositorio (Solución a `remote: Repository not found`)

Si cambiaste tu nombre de usuario en GitHub, transferiste el repositorio a otra cuenta/organización, o creaste una cuenta nueva, al hacer `git push` o `git pull` verás el siguiente error:
```text
remote: Repository not found.
fatal: repository 'https://github.com/USUARIO_VIEJO/repo.git/' not found
```

Esto ocurre por dos motivos:
1. Tu repositorio local sigue apuntando a la URL del usuario anterior.
2. Tu computadora o VPS tiene guardadas en caché las credenciales del usuario anterior (el cual ya no tiene permisos sobre el repositorio).

---

### Paso 1: Ver a qué URL apunta tu repositorio
Dentro de la carpeta del proyecto, ejecuta:
```bash
git remote -v
```

---

### Paso 2: Actualizar la URL con el nuevo usuario
Ejecuta `git remote set-url`:

* **Si usas conexión por HTTPS:**
  ```bash
  git remote set-url origin https://github.com/NUEVO_USUARIO/TU_REPOSITORIO.git
  ```
* **Si usas conexión por SSH:**
  ```bash
  git remote set-url origin git@github.com:NUEVO_USUARIO/TU_REPOSITORIO.git
  ```

Confirma que la URL cambió ejecutando nuevamente `git remote -v`.

---

### Paso 3: Actualizar las credenciales guardadas

Si tras cambiar la URL aún te sigue arrojando el error `Repository not found`, tu gestor de credenciales sigue enviando el token o contraseña del usuario anterior.

#### En Windows (Tu computadora local):
1. Presiona la tecla `Windows` y escribe **Administrador de credenciales** (*Credential Manager*).
2. Haz clic en **Credenciales de Windows**.
3. En la sección **Credenciales genéricas**, busca cualquier entrada que comience por:
   - `git:https://github.com`
4. Selecciónala y haz clic en **Quitar** (o en **Editar** para escribir tu nuevo usuario y nuevo Token `ghp_...`).
5. Abre de nuevo tu terminal y corre:
   ```bash
   git pull
   ```
   Git te pedirá que inicies sesión con tu cuenta nueva o que pegues el nuevo token, y lo guardará automáticamente.

#### En tu VPS o Servidor Linux:
Si habías activado `credential.helper store`, las credenciales se almacenan en un archivo de texto en tu carpeta de usuario:

```bash
# Editar el archivo de credenciales de Git
nano ~/.git-credentials
```
Modifica la línea correspondiente con tu nuevo usuario y nuevo token:
```text
https://NUEVO_USUARIO:ghp_TU_NUEVO_TOKEN@github.com
```
*(Guarda con `Ctrl + O`, `Enter` y sal con `Ctrl + X`)*.

---

## 10. Chuleta / Resumen Rápido de Comandos

```bash
# Estado e Inspección
git status                 # Ver qué archivos cambiaron
git diff                   # Ver cambios línea por línea
git log --oneline          # Historial compacto

# Guardar y Enviar
git add .                  # Preparar todos los cambios
git commit -m "mensaje"    # Guardar cambios
git push origin main       # Subir al servidor remoto
git pull origin main       # Descargar cambios del servidor remoto

# Ramas
git branch                 # Listar ramas
git checkout -b feature    # Crear y cambiar a nueva rama
git checkout main          # Volver a main
git merge feature          # Unir cambios de feature a la rama actual
```
