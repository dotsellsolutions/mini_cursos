# Guía Paso a Paso: Cómo Apuntar Dos o Más Dominios a la Misma Carpeta en CloudPanel (VPS)

Esta guía explica en detalle cómo configurar **dos o más dominios (o subdominios)** para que sirvan la misma aplicación web y compartan la misma carpeta raíz en un servidor VPS administrado con **CloudPanel**.

---

## 📋 Tabla de Contenidos
1. [Conceptos Clave: ¿Alias de Dominio o Redirección?](#1-conceptos-clave-alias-de-dominio-o-redirección)
2. [Paso 1: Configuración de Registros DNS (Imprescindible)](#paso-1-configuración-de-registros-dns-imprescindible)
3. [Paso 2: Agregar Dominios Adicionales (Alias) en CloudPanel](#paso-2-agregar-dominios-adicionales-alias-en-cloudpanel)
4. [Paso 3: Emitir Certificado SSL Multi-Dominio (HTTPS)](#paso-3-emitir-certificado-ssl-multi-dominio-https)
5. [Paso 4: Verificación Avanzada del Virtual Host de Nginx](#paso-4-verificación-avanzada-del-virtual-host-de-nginx)
6. [Caso Alternativo: Redirección 301 (Si quieres que el Dominio B redirija al Dominio A)](#caso-alternativo-redirección-301)
7. [Solución de Problemas Comunes](#solución-de-problemas-comunes)

---

## 1. Conceptos Clave: ¿Alias de Dominio o Redirección?

Antes de empezar, identifica qué comportamiento necesitas:

| Tipo | ¿Qué ve el usuario en la barra del navegador? | Caso de uso |
| :--- | :--- | :--- |
| **Alias de Dominio (Domain Alias)** | El usuario entra a `dominio2.com` y la URL **se mantiene** como `dominio2.com`, pero carga los mismos archivos y base de datos que `dominio1.com`. | Plataformas Multi-Tenant / SaaS, landing pages bajo diferentes marcas o dominios regionales (`.es`, `.com`, `.mx`). |
| **Redirección 301 (Redirect)** | El usuario entra a `dominio2.com` y el navegador **cambia automáticamente la URL** a `dominio1.com`. | Evitar contenido duplicado en SEO cuando compras dominios parecidos para proteger tu marca. |

> Esta guía se enfoca principalmente en la opción de **Alias de Dominio** (compartir la misma carpeta).

---

## Paso 1: Configuración de Registros DNS (Imprescindible)

Para que cualquier dominio llegue a tu VPS, ambos dominios deben apuntar a la misma **dirección IP pública del servidor**.

En tu proveedor de dominio o DNS (Cloudflare, Namecheap, GoDaddy, Hostinger, etc.), crea los siguientes registros **Tipo A** para cada dominio:

### Para el Dominio Principal (`dominio-principal.com`):
| Tipo | Nombre (Host) | Valor (Contenido) | TTL |
| :--- | :--- | :--- | :--- |
| **A** | `@` (o la raíz) | `IP_PUBLICA_DE_TU_VPS` | Auto / 1 min |
| **A** | `www` | `IP_PUBLICA_DE_TU_VPS` | Auto / 1 min |

### Para el Segundo Dominio (`segundo-dominio.com`):
| Tipo | Nombre (Host) | Valor (Contenido) | TTL |
| :--- | :--- | :--- | :--- |
| **A** | `@` (o la raíz) | `IP_PUBLICA_DE_TU_VPS` | Auto / 1 min |
| **A** | `www` | `IP_PUBLICA_DE_TU_VPS` | Auto / 1 min |

> ⏳ **Nota sobre la propagación:** Los cambios de DNS pueden tardar entre 2 y 30 minutos en propagarse globalmente. Puedes verificarlo en herramientas como [dnschecker.org](https://dnschecker.org).

---

## Paso 2: Agregar Dominios Adicionales (Alias) en CloudPanel

CloudPanel incluye soporte nativo para múltiples dominios en un mismo sitio sin necesidad de crear archivos duplicados ni tocar la terminal.

### 2.1 Acceder al Sitio en CloudPanel
1. Inicia sesión en tu panel de **CloudPanel** (`https://tu-ip-o-dominio:8443`).
2. En el menú lateral izquierdo, haz clic en **Sites**.
3. Selecciona tu sitio existente (el sitio principal cuya carpeta deseas compartir).

---

### 2.2 Agregar el nuevo nombre de dominio
1. Dentro de la configuración del sitio, ve a la pestaña **Domain Names** (o *Nombres de Dominio*).
2. Haz clic en el botón superior derecho **Add Domain** (o *New Domain*).
3. En el campo **Domain Name**, ingresa tu segundo dominio:
   ```text
   segundo-dominio.com
   ```
4. *(Opcional)* Si también deseas que funcione con `www`, agrega también:
   ```text
   www.segundo-dominio.com
   ```
5. Haz clic en **Save** / **Add Domain**.

---

## Paso 3: Emitir Certificado SSL Multi-Dominio (HTTPS)

Para que tus visitantes puedan entrar mediante `https://` a todos los dominios sin alertas de seguridad de "Sitio no seguro", debes actualizar el certificado SSL de Let's Encrypt.

1. Dentro del mismo sitio en CloudPanel, haz clic en la pestaña **SSL/TLS**.
2. Ve a la subpestaña **Let's Encrypt**.
3. Verás una lista de todos los dominios y alias configurados (ej: `dominio-principal.com`, `www.dominio-principal.com`, `segundo-dominio.com`, `www.segundo-dominio.com`).
4. Marca las casillas de todos los dominios que deseas incluir en el certificado.
5. Haz clic en el botón verde **Create / Renew Certificate**.

CloudPanel validará los dominios contra Let's Encrypt y generará un único certificado **SAN (Subject Alternative Name)** que cubre todos tus dominios automáticamente.

---

## Paso 4: Verificación Avanzada del Virtual Host de Nginx

Si deseas entender qué hizo CloudPanel internamente o necesitas ajustar reglas específicas de Nginx:

1. Ve a la pestaña **Vhost** (Virtual Host) dentro de la configuración del sitio.
2. Observarás que en la directiva `server_name`, CloudPanel agregó automáticamente todos tus dominios separados por espacio:

```nginx
server {
  listen 80;
  listen [::]:80;
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  {{ssl_certificate_key}}
  {{ssl_certificate}}
  
  # Aquí CloudPanel agrupa todos los dominios que apuntan a la misma carpeta:
  server_name dominio-principal.com www.dominio-principal.com segundo-dominio.com www.segundo-dominio.com;
  
  # Ambos dominios leerán exactamente desde esta misma ruta raíz:
  root /home/usuario-app/htdocs/dominio-principal.com;

  # ... resto de la configuración de PHP / Node / Reverse Proxy ...
}
```

---

## Caso Alternativo: Redirección 301

Si **NO** quieres que ambos dominios muestren su propia URL por separado, sino que `segundo-dominio.com` redirija permanentemente a `dominio-principal.com`:

1. En CloudPanel, ve a **Sites** y haz clic en **Add Site**.
2. Selecciona **Create a Reverse Proxy** o **Create a PHP Site** (o usa el tipo **Redirect Site** si tu versión de CloudPanel lo incluye directamente).
3. Configura el nuevo sitio con el nombre `segundo-dominio.com`.
4. Ve a la pestaña **Vhost** de `segundo-dominio.com` y configura la redirección:
   ```nginx
   return 301 https://dominio-principal.com$request_uri;
   ```
5. Guarda y emite el certificado SSL para `segundo-dominio.com`.

---

## Solución de Problemas Comunes

### ❌ 1. Error al generar el Certificado Let's Encrypt (`Challenge failed`)
- **Causa:** El nuevo dominio aún no tiene los registros DNS propagados o no apunta a la IP correcta del VPS.
- **Solución:** Revisa con `nslookup segundo-dominio.com` o `ping segundo-dominio.com` desde tu terminal que la IP devuelta sea exactamente la del VPS antes de solicitar el certificado.

### ❌ 2. En Laravel o WordPress, el segundo dominio me redirige al principal
- **En WordPress:** WordPress fuerza la URL definida en `siteurl` y `home`. Si deseas que un mismo WordPress responda a múltiples dominios sin redirigir, debes usar un plugin de Multi-Domain o definir en `wp-config.php`:
  ```php
  define('WP_SITEURL', 'http://' . $_SERVER['HTTP_HOST']);
  define('WP_HOME', 'http://' . $_SERVER['HTTP_HOST']);
  ```
- **En Laravel / Frameworks:** Revisa en tu archivo `.env` que `APP_URL` no esté forzando redirecciones estrictas en los middlewares de enrutamiento o CORS.

### ❌ 3. Problemas de CORS con fuentes o peticiones AJAX
Si cargas fuentes tipográficas, imágenes o APIs entre ambos dominios, agrega en la pestaña **Vhost** de Nginx:
```nginx
location ~* \.(eot|ttf|woff|woff2|svg)$ {
    add_header Access-Control-Allow-Origin *;
}
```
