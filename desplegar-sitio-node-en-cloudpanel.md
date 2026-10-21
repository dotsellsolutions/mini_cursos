# Guía Paso a Paso: Cómo Subir y Desplegar un Sitio Node.js en CloudPanel (VPS)

Esta guía explica desde cero cómo subir, configurar y poner en producción una aplicación **Node.js (Express, NestJS, Next.js, Fastify, etc.)** en un servidor VPS administrado con **CloudPanel**.

---

## 📋 Tabla de Contenidos
1. [Paso 1: Crear el Sitio Node.js en CloudPanel](#paso-1-crear-el-sitio-nodejs-en-cloudpanel)
2. [Paso 2: Subir los Archivos de tu Aplicación](#paso-2-subir-los-archivos-de-tu-aplicación)
   - [Opción A: Vía Git / SSH (Recomendado)](#opción-a-vía-git--ssh-recomendado)
   - [Opción B: Vía File Manager o SFTP (FileZilla)](#opción-b-vía-file-manager-o-sftp-filezilla)
3. [Paso 3: Instalar Dependencias (`npm install`)](#paso-3-instalar-dependencias-npm-install)
4. [Paso 4: Variables de Entorno y Puerto (`.env`)](#paso-4-variables-de-entorno-y-puerto-env)
5. [Paso 5: Configurar el Script de Inicio y Arrancar la App](#paso-5-configurar-el-script-de-inicio-y-arrancar-la-app)
6. [Paso 6: Activar Certificado SSL Gratuito (HTTPS)](#paso-6-activar-certificado-ssl-gratuito-https)
7. [Manejo de Errores Comunes (502 Bad Gateway)](#manejo-de-errores-comunes-502-bad-gateway)

---

## Paso 1: Crear el Sitio Node.js en CloudPanel

CloudPanel incluye soporte nativo para Node.js con gestión de procesos y reverse proxy automático.

1. Entra a tu **CloudPanel** (`https://tu-ip:8443`).
2. En el menú de la izquierda, haz clic en **Sites** ➡️ botón **Add Site**.
3. Selecciona **Create a Node.js Site**.
4. Completa el formulario:
   - **Domain Name:** Tu dominio (ej. `mi-app.com`).
   - **Node.js Version:** Selecciona la versión requerida por tu proyecto (ej. Node 18, 20 o 22 LTS).
   - **App Port:** Puerto interno donde escuchará tu aplicación (por defecto: `3000`).
   - **Site User:** Usuario del sistema para esta app (ej. `app-user`).
   - **Password:** Contraseña segura para el usuario SSH/SFTP.
5. Haz clic en **Create**.

> ⚡ **¿Qué hace CloudPanel internamente?** Configura Nginx para recibir tráfico en los puertos 80/443 y redirigirlo internamente al puerto `3000`, y crea la carpeta del proyecto en `/home/app-user/htdocs/mi-app.com/`.

---

## Paso 2: Subir los Archivos de tu Aplicación

### Opción A: Vía Git / SSH (Recomendado)
1. Conéctate por SSH con el usuario de la app o como root:
   ```bash
   ssh app-user@IP_DE_TU_VPS
   ```
2. Entra a la carpeta del sitio:
   ```bash
   cd ~/htdocs/mi-app.com/
   ```
3. Clona tu repositorio de GitHub / GitLab:
   ```bash
   # Si la carpeta está vacía:
   git clone https://github.com/tu-usuario/tu-repo.git .
   ```

---

### Opción B: Vía File Manager de CloudPanel o SFTP (FileZilla)
1. En tu computadora, comprime tu proyecto en un archivo **`.zip`** (⚠️ **Importante:** excluye la carpeta `node_modules` antes de comprimir).
2. En CloudPanel, entra a tu sitio y haz clic en la pestaña **File Manager**.
3. Navega dentro de `/htdocs/mi-app.com/`.
4. Haz clic en **Upload**, selecciona tu archivo `.zip` y luego dale clic derecho ➡️ **Extract**.

---

## Paso 3: Instalar Dependencias (`npm install`)

Conéctate por SSH al VPS para instalar los módulos necesarios:

```bash
# 1. Entrar a la carpeta del proyecto
cd /home/app-user/htdocs/mi-app.com

# 2. Instalar dependencias de producción
npm install --production

# 3. Si tu proyecto usa TypeScript / Next.js / Build step, compílalo:
npm run build
```

---

## Paso 4: Variables de Entorno y Puerto (`.env`)

Crea el archivo `.env` en la raíz de tu proyecto:

```bash
nano .env
```

Pega tus variables de entorno, asegurándote de que el puerto coincida con el **App Port** de CloudPanel:

```ini
PORT=3000
NODE_ENV=production
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=tu_usuario_bd
DB_PASSWORD=tu_clave_bd
DB_NAME=tu_base_datos
```

> 💾 Guarda con `Ctrl + O`, `Enter` y sal con `Ctrl + X`.

---

### ⚠️ Regla de Oro en tu código Node.js:
Asegúrate de que tu archivo de inicio (`app.js`, `server.js` o `index.js`) escuche en `process.env.PORT` y en `0.0.0.0` o `127.0.0.1`:

```javascript
const PORT = process.env.PORT || 3000;

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Servidor corriendo en el puerto ${PORT}`);
});
```

---

## Paso 5: Configurar el Script de Inicio y Arrancar la App

1. En CloudPanel, entra a tu sitio Node.js.
2. Ve a la pestaña **Node.js Settings** (o **Settings**).
3. Configura:
   - **Entry Point / Script:** El archivo principal que arranca tu app (ej: `app.js`, `dist/main.js`, `server.js`) o el comando npm (ej: `npm start`).
4. Haz clic en **Save & Restart**.

CloudPanel iniciará tu aplicación con su gestor de procesos en segundo plano y la mantendrá viva 24/7.

---

## Paso 6: Activar Certificado SSL Gratuito (HTTPS)

1. En CloudPanel, ve a la pestaña **SSL/TLS** de tu sitio.
2. Entra a la subpestaña **Let's Encrypt**.
3. Haz clic en **Create / Renew Certificate**.

---

## Manejo de Errores Comunes

### ❌ Error `502 Bad Gateway`
- **Causa:** Nginx está intentando conectarse al puerto configurado (ej. 3000), pero la aplicación Node.js está apagada o falló al iniciar.
- **Cómo solucionarlo:**
  1. En CloudPanel, ve a la pestaña **Logs** de tu sitio y revisa los logs de error de Node.
  2. O por SSH, corre `node app.js` manualmente para ver si arroja algún error de sintaxis o de conexión a base de datos.
  3. Comprueba que el puerto en `.env` sea exactamente el mismo que definiste en CloudPanel (`3000`).
