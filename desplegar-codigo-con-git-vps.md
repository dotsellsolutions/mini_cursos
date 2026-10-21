# 🚀 Cómo Transferir y Desplegar Código en tu VPS usando Git

Usar Git para transferir archivos a tu servidor es el estándar de la industria. No solo es más seguro y rápido que usar FTP o comandos como `scp`, sino que te permite tener un control exacto de **qué archivos omitir** (como carpetas pesadas o archivos con contraseñas) y **qué cambios se hicieron**.

## 📌 Ventajas de usar Git
1. **Ignora archivos fácilmente:** Usando `.gitignore` evitas subir `node_modules`, `.env`, etc.
2. **Transferencias ultra rápidas:** Al actualizar, Git solo transfiere las líneas de código que modificaste, no todo el archivo de nuevo.
3. **Historial de cambios:** Si algo se rompe en el servidor, puedes regresar a una versión anterior rápidamente.

---

## 🛠️ Paso 1: Configurar qué archivos omitir (En tu Computadora)

Antes de subir cualquier código, debes decirle a Git qué cosas **NO** quieres que se suban. Para esto usamos un archivo especial.

1. En la raíz de tu proyecto, crea un archivo de texto llamado exactamente **`.gitignore`** (asegúrate de que tenga el punto al principio).
2. Ábrelo en tu editor de código (como VS Code) y añade lo que deseas ignorar. Ejemplo:

```text
# Dependencias de Node.js
node_modules/

# Archivos de variables de entorno (¡Nunca subas contraseñas a Git!)
.env

# Carpetas de compilación
dist/
build/

# Archivos de registro de errores
*.log
```

---

## 💻 Paso 2: Subir tu código a la nube (En tu Computadora)

Necesitas usar una plataforma gratuita como **GitHub**, **GitLab** o **Bitbucket** como puente entre tu computadora y tu servidor.

1. Crea una cuenta en GitHub y crea un **Nuevo Repositorio (New Repository)**. Mantenlo como **Privado** para que nadie más vea tu código.
2. Abre la terminal (CMD o Git Bash) en la carpeta de tu proyecto local y ejecuta:

```bash
# Iniciar el sistema Git en tu proyecto
git init

# Añadir todos los archivos (Git leerá el .gitignore y omitirá lo que pusiste ahí)
git add .

# Crear una "foto" de tu código actual (Commit)
git commit -m "Primera versión de mi proyecto"

# Conectar tu computadora con el repositorio de GitHub (Reemplaza con tu URL real)
git remote add origin https://github.com/TU_USUARIO/TU_REPOSITORIO.git

# Subir tu código a la nube
git push -u origin main
```

*(Nota: Si es la primera vez, GitHub te pedirá que inicies sesión en una ventana emergente).*

---

## 🌐 Paso 3: Descargar el código en tu Servidor (En tu VPS)

Ahora que tu código está seguro en la nube (sin los archivos pesados ni contraseñas), vamos a descargarlo en el servidor.

1. Conéctate a tu VPS por SSH:
   ```bash
   ssh usuario@IP_DEL_VPS
   ```

2. Navega a la carpeta pública donde servirá tu web (Ejemplo de CloudPanel):
   ```bash
   cd /home/nombre-sitio/htdocs/misitio.com
   ```

3. Descarga (Clona) el repositorio:
   ```bash
   # El punto al final es MUY IMPORTANTE. Le dice a Git que vuelque los archivos 
   # en esta carpeta exacta, en lugar de crear una subcarpeta nueva.
   git clone https://github.com/TU_USUARIO/TU_REPOSITORIO.git .
   ```

4. **¡Importante! Reconstruir lo que ignoramos.**
   Como Git no subió ni el `.env` ni la carpeta `node_modules`, debes generarlos en el servidor:
   
   ```bash
   # 1. Instalar las dependencias de Node
   npm install --production
   
   # 2. Crear tu archivo .env
   nano .env
   # (Pega aquí tus variables de entorno, contraseñas de BD del servidor, etc. y guarda con Ctrl+O, Enter, Ctrl+X)
   ```

---

## 🔄 Paso 4: El flujo de trabajo diario (Cómo actualizar)

Aquí es donde Git brilla. Cuando modifiques cosas en tu computadora y quieras subirlas al servidor, **ya no tienes que volver a hacer todo lo anterior**. Solo sigues este flujo de dos pasos:

### 1. Desde tu Computadora (Subir la actualización):
```bash
git add .
git commit -m "Corregido error visual en el menú y actualizada BD"
git push
```

### 2. Desde tu VPS (Aplicar la actualización):
Entras por SSH, vas a la carpeta de tu web y simplemente descargas los cambios:
```bash
cd /home/nombre-sitio/htdocs/misitio.com
git pull
```

**Git descargará únicamente las diferencias en segundos.**

*(Si cambiaste algo de Node.js como instalar un nuevo paquete, recuerda correr `npm install` de nuevo en el servidor, o si es un proyecto de Vite, correr `npm run build` y reiniciar el servicio pm2 si aplica).*

---

## 💡 Pro-Tips Adicionales

- **Tokens de Acceso:** Si en el VPS GitHub te pide contraseña al hacer `git clone`, recuerda que GitHub ya no usa contraseñas. Debes generar un **Personal Access Token (PAT)** en los ajustes de tu cuenta de GitHub (Settings > Developer settings > Personal access tokens) y usar ese token como contraseña.
- **Evitar que te pida contraseña en el VPS:** Para que tu VPS no te pida el Token cada vez que haces `git pull`, puedes correr este comando en el servidor:
  `git config --global credential.helper store`
  La próxima vez que pongas tu usuario y Token, se guardarán para siempre.
