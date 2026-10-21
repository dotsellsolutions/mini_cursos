# Guía Paso a Paso: Solucionar Error 404 en Nginx al Recargar Rutas de Vue/React (SPA) y Endpoints de API Slim 4

Esta guía explica detalladamente por qué ocurre el error **404 Not Found** en servidores de producción que corren sobre **Nginx** cuando intentas acceder directamente a rutas virtuales (como `/admin` o `/post/slug`) o al hacer peticiones a un backend API en una subcarpeta (como `/api/...`), y cómo configurarlo en tu servidor (VPS, CloudPanel o servidor dedicado) para solucionarlo de raíz.

---

## 📋 Tabla de Contenidos
1. [¿Por qué ocurre este error 404 en Nginx?](#1-por-qué-ocurre-este-error-404-en-nginx)
2. [Solución para el Frontend (Vue Router / React Router)](#2-solución-para-el-frontend-vue-router--react-router)
3. [Solución para el Backend (Slim 4 en Subcarpeta `/api/`)](#3-solución-para-el-backend-slim-4-en-subcarpeta-api)
4. [Configuración Completa de Nginx VHost (Plantilla Listilla)](#4-configuración-completa-de-nginx-vhost-plantilla-listilla)
5. [Cómo Aplicar los Cambios y Reiniciar Nginx](#5-cómo-aplicar-los-cambios-y-reiniciar-nginx)

---

## 1. ¿Por qué ocurre este error 404 en Nginx?

Cuando utilizas enrutamiento del lado del cliente en un frontend moderno (como Vue o React en modo **History**), las rutas no representan directorios o archivos reales en el disco de tu servidor. 
- Al navegar desde el inicio, Vue Router intercepta los clics en JavaScript y simula el cambio de URL sin consultar a Nginx.
- Al escribir la URL manualmente (ej: `/admin`) o presionar **F5 (Refrescar)**, el navegador le pide a Nginx el recurso `/admin`. Al no haber una carpeta física llamada `admin/`, Nginx retorna **404 Not Found**.

En servidores Apache esto se resuelve de forma automática leyendo el archivo `.htaccess`. Sin embargo, **Nginx no lee archivos `.htaccess`**, por lo que es mandatorio declarar estas reglas en el archivo de configuración del bloque de servidor (`VHost` de Nginx).

---

## 2. Solución para el Frontend (Vue Router / React Router)

Para evitar el error 404 en las rutas de tu SPA y asegurar que el renderizador de SEO (como `index.php` híbrido) capture primero la petición, debes configurar el bloque de ubicación raíz (`location /`) para que intente buscar la URL física y, si no la encuentra, devuelva el control al archivo index principal:

```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

#### 🔍 ¿Cómo funciona?
1. `$uri`: Verifica si el archivo solicitado existe (ej. `css/index.css`). Si existe, lo entrega.
2. `$uri/`: Verifica si la carpeta solicitada existe.
3. `/index.php?$query_string`: Si ninguno de los anteriores existe, enruta internamente a tu archivo `index.php` (o `index.html` en SPAs tradicionales) pasando todos los parámetros query, dejando que tu aplicación resuelva la ruta.

---

## 3. Solución para el Backend (Slim 4 en Subcarpeta `/api/`)

En esta arquitectura, tu API REST está alojada en una subcarpeta llamada `api/`, y su punto de entrada público real es `api/public/index.php`. 

Para que los endpoints funcionen escribiendo `/api/login` o `/api/posts` en Nginx sin tener que incluir `/public/index.php` en la URL del frontend, debes declarar una ubicación específica para `/api`:

```nginx
location /api {
    try_files $uri $uri/ /api/public/index.php?$query_string;
}
```

#### 🔍 ¿Cómo funciona?
Cualquier llamada dirigida a `tu-dominio.com/api/algo` será interceptada. Si no corresponde a una carpeta o archivo físico (como una imagen subida en `uploads/`), se reescribirá automáticamente al despachador frontal seguro de Slim en `/api/public/index.php`, donde Slim procesará la ruta limpiamente.

---

## 4. Configuración Completa de Nginx VHost (Plantilla Listilla)

A continuación se muestra un archivo de configuración del bloque `server` típico para un despliegue de producción unificado (Vite Frontend + Slim 4 API):

```nginx
server {
    listen 80;
    server_name blogs.dotsellsolutions.com;
    root /var/www/dotsellsolutions/blogs; # Ruta absoluta a la carpeta de tu blog
    index index.php index.html;

    # 1. Enrutador del Frontend (Vue SPA / SSR Pre-renderer)
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # 2. Mapa del sitio dinámico
    location = /sitemap.xml {
        rewrite ^/sitemap\.xml$ /sitemap.php last;
    }

    # 3. Enrutador de la API (Slim 4)
    location /api {
        try_files $uri $uri/ /api/public/index.php?$query_string;
    }

    # 4. Procesamiento de archivos PHP (Frontend & Backend)
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock; # Reemplazar con tu socket php-fpm actual
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # 5. Seguridad: Denegar acceso a archivos sensibles
    location ~ /\.(?!well-known) {
        deny all;
    }
    
    location ~* (composer\.json|composer\.lock|package\.json|package-lock\.json|nginx\.conf)$ {
        deny all;
    }
}
```

---

## 5. Cómo Aplicar los Cambios y Reiniciar Nginx

Una vez editado el archivo de host virtual de tu dominio (en CloudPanel ingresando a la pestaña **Vhost** de tu sitio, o mediante consola editando `/etc/nginx/sites-available/tu-sitio`):

1. **Valida la configuración de Nginx** para confirmar que no hay errores de sintaxis (imprescindible para no tumbar otros sitios activos en producción):
   ```bash
   sudo nginx -t
   ```
   *Debería devolver: `nginx: configuration file /etc/nginx/nginx.conf test is successful`*

2. **Recarga o reinicia Nginx** para aplicar los cambios en caliente:
   ```bash
   sudo systemctl reload nginx
   # O alternativamente:
   sudo systemctl restart nginx
   ```

Con estos pasos aplicados, podrás recargar directamente `/admin`, refrescar artículos y realizar peticiones al API de Slim 4 sin volver a ver un error 404 en Nginx.
