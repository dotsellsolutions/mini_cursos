# Guía Paso a Paso: Desplegar App Vite (React/Vue/Svelte) y Compartir la Misma Carpeta entre Múltiples Dominios en CloudPanel

Esta guía explica el método definitivo, probado y 100% libre de errores para desplegar una aplicación frontend moderna creada con **Vite (React + Vite, Vue + Vite, Svelte, etc.)** en un VPS con **CloudPanel**, y hacer que **dos o más dominios completamente diferentes** apunten a los mismos archivos, compartiendo la misma carpeta y con **certificados SSL independientes** en cada uno.

---

## 📋 Tabla de Contenidos
1. [¿Por qué funciona este método y cuáles son sus ventajas?](#1-por-qué-funciona-este-método-y-cuáles-son-sus-ventajas)
2. [Paso 1: Compilar la Aplicación Vite en tu Computadora](#paso-1-compilar-la-aplicación-vite-en-tu-computadora)
3. [Paso 2: Crear el Sitio Principal en CloudPanel y Subir los Archivos](#paso-2-crear-el-sitio-principal-en-cloudpanel-y-subir-los-archivos)
4. [Paso 3: Configuración de DNS para el Segundo Dominio](#paso-3-configuración-de-dns-para-el-segundo-dominio)
5. [Paso 4: Crear el Segundo Sitio en CloudPanel](#paso-4-crear-el-segundo-sitio-en-cloudpanel)
6. [Paso 5: El Truco Clave de Nginx VHost (Reemplazar `{{root}}`)](#paso-5-el-truco-clave-de-nginx-vhost-reemplazar-root)
7. [Paso 6: Emitir el Certificado SSL Independiente (HTTPS)](#paso-6-emitir-el-certificado-ssl-independiente-https)
8. [Paso 7: Cómo Actualizar tu App para Todos los Dominios a la Vez](#paso-7-cómo-actualizar-tu-app-para-todos-los-dominios-a-la-vez)
9. [Solución de Errores Comunes](#solución-de-errores-comunes)

---

## 1. ¿Por qué funciona este método y cuáles son sus ventajas?

- **Rendimiento Máximo:** Las aplicaciones construidas con Vite son SPAs (Single Page Applications) estáticas. Al servirlas como archivos estáticos con Nginx, tu servidor VPS consume prácticamente **0% de CPU y memoria RAM** en comparación con tener un proceso pesado de Node.js corriendo.
- **Sin Errores de SSL:** Muchos métodos de alias fallan cuando son dominios completamente distintos porque el certificado SSL solo valida un dominio. Con este método, **cada dominio tiene su propio certificado Let's Encrypt independiente**.
- **Mantenimiento Cero:** Solo subes tu carpeta compilada (`dist/`) **una sola vez** en el primer dominio. Cualquier cambio o actualización se refleja al instante en todos los dominios configurados.

---

## Paso 1: Compilar la Aplicación Vite en tu Computadora

En la terminal de tu proyecto Vite local, ejecuta:

```bash
npm run build
```

#### 🔍 ¿Qué hace este comando?
- Empaqueta y minifica todo tu código (React/Vue/Svelte, JSX, TypeScript, CSS) generando una carpeta optimizada llamada **`dist/`**.
- La carpeta `dist/` contiene el archivo `index.html`, los assets en `/assets/` y los favicons.

---

## Paso 2: Crear el Sitio Principal en CloudPanel y Subir los Archivos

### 2.1 Crear el sitio en CloudPanel
1. Entra a tu **CloudPanel** (`https://tu-ip:8443`).
2. Ve a **Sites** ➡️ botón **Add Site** ➡️ **Create a Static HTML Site**.
3. Configura:
   - **Domain Name:** `tu-primer-dominio.com` (ej: `brightswimmingrd.com`).
   - **Site User:** Crea un usuario para tu aplicación (ej: `app-user`).
   - **Password:** Contraseña segura.
4. Haz clic en **Create**.

---

### 2.2 Subir los archivos de `dist/`
1. En tu computadora, comprime todo el contenido que está **dentro** de la carpeta `dist/` en un archivo `.zip`.
2. En CloudPanel, entra a tu sitio y ve a la pestaña **File Manager**.
3. Entra a la carpeta del sitio: `/htdocs/tu-primer-dominio.com/`.
4. Haz clic en **Upload**, sube tu archivo `.zip`, dale clic derecho y selecciona **Extract**.
5. Asegúrate de que `index.html` quede directamente dentro de `/htdocs/tu-primer-dominio.com/`.

---

### 2.3 Configurar Nginx para evitar el error 404 al recargar (React / Vue Router)
1. Dentro del sitio en CloudPanel, entra a la pestaña **Vhost**.
2. Busca el bloque `location /` y configúralo así:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

3. Haz clic en **Save**.

---

### 2.4 Emitir Certificado SSL para el Primer Dominio
1. Ve a la pestaña **SSL/TLS** ➡️ **Let's Encrypt**.
2. Haz clic en **Create / Renew Certificate**.

---

## Paso 3: Configuración de DNS para el Segundo Dominio

En el proveedor donde compraste o administras tu segundo dominio (`tu-segundo-dominio.com` o `app.tu-segundo-dominio.com`):

Crea un registro **Tipo A** apuntando a la IP pública de tu VPS:

| Tipo | Nombre / Host | Valor |
| :--- | :--- | :--- |
| **A** | `@` (o el subdominio `app`) | `IP_PUBLICA_DE_TU_VPS` |
| **A** | `www` (si aplica) | `IP_PUBLICA_DE_TU_VPS` |

---

## Paso 4: Crear el Segundo Sitio en CloudPanel

1. En CloudPanel, ve a **Sites** ➡️ **Add Site** ➡️ **Create a Static HTML Site**.
2. En **Domain Name**, coloca tu segundo dominio (ej: `app.brightswimmingrd.com` o `tu-segundo-dominio.com`).
3. ⚠️ **MUY IMPORTANTE (Site User):** En el campo **Site User**, selecciona **el mismo usuario** que usaste en el primer sitio (ej: `app-user`).
4. Haz clic en **Create**.

---

## Paso 5: El Truco Clave de Nginx VHost (Reemplazar `{{root}}`)

CloudPanel usa variables de plantilla como `{{root}}` que por defecto apuntan a la carpeta del propio dominio. La reemplazaremos para que lea la carpeta del primer dominio:

1. Entra al segundo sitio recién creado en CloudPanel.
2. Haz clic en la pestaña **Vhost**.
3. Localiza la línea `{{root}}` justo debajo de `server_name`.

```nginx
# --- ANTES (Como viene por defecto) ---
{{ssl_certificate}}
server_name app.brightswimmingrd.com;
{{root}}
```

4. **Borra `{{root}}`** y escribe la ruta absoluta de la carpeta de tu primer dominio:

```nginx
# --- DESPUÉS (Configuración Correcta) ---
{{ssl_certificate}}
server_name app.brightswimmingrd.com;
root /home/app-user/htdocs/tu-primer-dominio.com;
```

5. Baja hasta el bloque `location /` y asegúrate de incluir la regla de rutas de Vite:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

6. Haz clic en el botón verde **Save**.

---

## Paso 6: Emitir el Certificado SSL Independiente (HTTPS)

1. En la configuración de tu segundo sitio, ve a la pestaña **SSL/TLS**.
2. Entra a la subpestaña **Let's Encrypt**.
3. Haz clic en **Create / Renew Certificate**.

> 🔒 **¿Por qué funciona perfecto?** Let's Encrypt valida únicamente este segundo dominio por separado y genera su propio archivo de certificado, evitando cualquier conflicto con el primero.

---

## Paso 7: Cómo Actualizar tu App para Todos los Dominios a la Vez

¡Esta es la mayor ventaja de esta arquitectura! Cuando hagas cambios en tu código de React/Vue:

1. En tu computadora ejecutas:
   ```bash
   npm run build
   ```
2. Subes el nuevo contenido de `dist/` **únicamente a la carpeta del primer dominio** (`/htdocs/tu-primer-dominio.com/`).
3. **Automáticamente todos los dominios y subdominios mostrarán la nueva versión de inmediato**, sin tener que subir archivos dos veces ni reiniciar ningún servicio.

---

## Solución de Errores Comunes

### ❌ 1. Error `404 Not Found` al recargar una página interna (ej: `/dashboard` o `/login`)
- **Causa:** Nginx busca un archivo físico llamado `/dashboard` en lugar de pasárselo a React/Vue.
- **Solución:** Asegúrate de tener `try_files $uri $uri/ /index.html;` dentro de `location /` en la pestaña **Vhost**.

### ❌ 2. Error `400 Bad Request: Request Header Or Cookie Too Large`
- **Causa:** Se intentó usar un "Reverse Proxy" apuntando a la URL pública externa (`https://...`), creando un bucle infinito de peticiones.
- **Solución:** No uses Reverse Proxy para sitios frontend. Usa sitios tipo **Static HTML** y cambia la directiva `root` en el VHost tal como se explica en el Paso 5.

### ❌ 3. Error `403 Forbidden` al entrar al segundo dominio
- **Causa:** El segundo sitio fue creado con un usuario de sistema diferente que no tiene permisos de lectura sobre la carpeta del primer dominio.
- **Solución:** Asegúrate de que ambos sitios pertenezcan al **mismo Site User** en CloudPanel.
