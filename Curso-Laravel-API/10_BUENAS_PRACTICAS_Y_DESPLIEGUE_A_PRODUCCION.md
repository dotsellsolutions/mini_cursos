# Módulo 10: Buenas Prácticas Pro y Despliegue a Producción

**TechStore API** ya funciona, está protegida y tiene tests. En este último módulo la convertimos en una API **lista para el mundo real**: que no rompa las apps viejas cuando cambies algo, que responda siempre con el mismo formato, que esté documentada, que sea rápida y que viva en un servidor de verdad.

---

## 1. Versionado de la API (`/api/v1`): No rompas las apps de tus clientes

Imagina que publicas la app móvil de TechStore y 10.000 personas la instalan. Un mes después cambias el JSON de productos (renombras `precio` a `precios.sin_impuestos`).
**Las 10.000 apps viejas que no se actualizaron se rompen al instante.** Una web la actualizas cuando quieras; una app móvil, no.

La solución es **versionar**: la API vieja (`v1`) sigue viva mientras las apps nuevas usan `v2`.

### Paso 1: Prefijo de versión en `routes/api.php`
```php
<?php
// Archivo: routes/api.php

use Illuminate\Support\Facades\Route;

Route::prefix('v1')->name('v1.')->group(base_path('routes/api_v1.php'));
// El día que haya cambios incompatibles:
// Route::prefix('v2')->name('v2.')->group(base_path('routes/api_v2.php'));
```

```php
<?php
// Archivo: routes/api_v1.php

use App\Http\Controllers\Api\V1\AuthController;
use App\Http\Controllers\Api\V1\ProductController;
use Illuminate\Support\Facades\Route;

Route::post('/login', [AuthController::class, 'login'])->middleware('throttle:intentos-login');
Route::apiResource('products', ProductController::class)->only(['index', 'show']);

Route::middleware(['auth:sanctum', 'rol:admin,vendedor'])->group(function () {
    Route::apiResource('products', ProductController::class)->except(['index', 'show']);
});
```
Ahora tus URLs son `GET /api/v1/products`, `POST /api/v1/login`...

### Paso 2: Organizar Controladores y Resources por versión
```
app/Http/
├── Controllers/Api/V1/ProductController.php
├── Controllers/Api/V2/ProductController.php   ← solo cuando exista la v2
├── Resources/V1/ProductResource.php
└── Resources/V2/ProductResource.php
```
```bash
php artisan make:controller Api/V1/ProductController --api --model=Product
php artisan make:resource V1/ProductResource
```

> 💡 **¿Cuándo creo una `v2`?** Solo cuando hagas un cambio **incompatible** (renombrar o quitar campos, cambiar tipos, cambiar el significado de un endpoint). **Añadir** un campo nuevo al JSON o un endpoint nuevo **no** rompe nada: hazlo en la versión actual.

---

## 2. Un formato de respuesta consistente (y errores predecibles)

Para el frontend es una pesadilla que unos endpoints devuelvan `{"mensaje": ...}`, otros `{"message": ...}` y otros `{"error": ...}`. Elige **un contrato** y respétalo en toda la API:

```json
// ✅ Éxito con un recurso (lo que ya devuelven tus API Resources)
{ "data": { "id": 10, "nombre": "Teclado Mecánico RGB" } }

// ✅ Éxito con lista paginada
{ "data": [ ... ], "links": { ... }, "meta": { "current_page": 1, "total": 87 } }

// ❌ Error (mismo formato que usa Laravel en los 422, así el frontend lee siempre "message")
{ "message": "No tienes permiso para esta acción.", "errors": { } }
```

### Quitar el envoltorio `data` (solo si de verdad lo prefieres)
Por defecto los Resources envuelven todo en `"data"`. Es lo recomendado (te deja espacio para `meta` y `links`), pero si tu frontend no lo quiere:
```php
// En AppServiceProvider::boot()
use Illuminate\Http\Resources\Json\JsonResource;

JsonResource::withoutWrapping();
```

### Añadir datos extra a la respuesta de un Resource
```php
return ProductResource::collection($productos)->additional([
    'filtros_aplicados' => $request->only(['buscar', 'categoria']),
]);
```

### Errores: siempre JSON y sin filtrar secretos
En el Módulo 8 configuraste `bootstrap/app.php` para responder errores en JSON. Recuerda la regla de producción:
* `APP_DEBUG=true` → el JSON de un error `500` muestra la ruta de los archivos, la línea y la *stack trace*. **Útil en tu PC.**
* `APP_DEBUG=false` → solo muestra `"message": "Server Error"`. **Obligatorio en producción**, o regalarás a cualquier atacante la estructura interna de tu servidor.

---

## 3. Paginación profesional: `paginate`, `simplePaginate` y `cursorPaginate`

| Método | ¿Qué devuelve? | ¿Cuándo usarlo? |
| :--- | :--- | :--- |
| `paginate(15)` | Página actual, **total de resultados y total de páginas**. | Catálogos con numeración "1 2 3 ... 9". Hace un `COUNT(*)` extra. |
| `simplePaginate(15)` | Solo "anterior" / "siguiente" (sin total). | Tablas enormes donde el `COUNT(*)` es lento. |
| `cursorPaginate(15)` | Un "cursor" codificado en vez de número de página. | **Scroll infinito** en apps móviles (feeds, notificaciones). El más rápido y no repite ni salta elementos si se insertan registros nuevos. |

### Dejar que el cliente elija cuántos quiere (con un límite de seguridad)
```php
public function index(Request $request)
{
    // ?por_pagina=50 → máximo 100 para que nadie pida 1.000.000 de filas de golpe
    $porPagina = min((int) $request->integer('por_pagina', 15), 100);

    $productos = Product::with('category')
        ->latest('id')
        ->cursorPaginate($porPagina)
        ->withQueryString(); // Conserva ?buscar=...&categoria=... en los enlaces "siguiente"

    return ProductResource::collection($productos);
}
```

### Ordenamiento seguro por URL (`?ordenar=-precio`)
**Nunca** pases directamente lo que llega por la URL a `orderBy()`. Usa una **lista blanca**:
```php
$permitidos = ['precio', 'nombre', 'created_at'];
$orden = $request->string('ordenar', '-created_at')->toString(); // "-" = descendente
$columna = ltrim($orden, '-');

if (in_array($columna, $permitidos, true)) {
    $query->orderBy($columna, str_starts_with($orden, '-') ? 'desc' : 'asc');
}
```

---

## 4. Documentación automática de tu API (Scramble / Scribe)

Tu compañero de frontend no debería tener que leer tus controladores para saber qué enviar. Hay paquetes que **leen tus rutas, FormRequests y Resources y generan la documentación solos**:

### Opción A: **Scramble** (OpenAPI automático, cero anotaciones)
```bash
composer require dedoc/scramble
```
Sin configurar nada, abre **`http://localhost:8000/docs/api`**: verás todos tus endpoints, los campos que exige cada `FormRequest` (sacados de tus `rules()`) y la forma del JSON de respuesta (sacada de tus `Resources`), con un botón para probarlos.
Además genera un archivo **OpenAPI** (`/docs/api.json`) que puedes importar en **Postman** o **Insomnia** con un clic.

### Opción B: **Scribe** (documentación estática con ejemplos)
```bash
composer require --dev knuckleswtf/scribe
php artisan vendor:publish --tag=scribe-config
php artisan scribe:generate
```
Genera una web de documentación en `public/docs` y una **colección de Postman** lista para importar. Puedes enriquecerla con comentarios en tus controladores:
```php
/**
 * Listar productos
 *
 * Devuelve el catálogo paginado de TechStore.
 *
 * @queryParam buscar string Texto a buscar en el nombre. Example: teclado
 * @queryParam categoria int ID de la categoría. Example: 3
 */
public function index(Request $request) { /* ... */ }
```

> 🔒 En producción, protege la ruta de la documentación (o desactívala) si tu API no es pública.

---

## 5. Rendimiento: Caché, índices y consultas vigiladas

### A) Cachear respuestas que casi nunca cambian (`Cache::remember`)
La lista de categorías de TechStore cambia una vez al mes, pero se pide 50.000 veces al día. ¿Para qué ir a la base de datos cada vez?
```php
use Illuminate\Support\Facades\Cache;

public function index()
{
    // Busca en caché la clave 'categorias-activas'. Si no existe, ejecuta la consulta y la guarda 1 hora.
    $categorias = Cache::remember('categorias-activas', now()->addHour(), function () {
        return Category::where('activa', true)->orderBy('nombre')->get();
    });

    return CategoryResource::collection($categorias);
}
```
Y cuando un admin crea o edita una categoría, **borras la caché** para que se regenere (por ejemplo, en un Observer del Módulo 3):
```php
// app/Observers/CategoryObserver.php
public function saved(Category $category): void
{
    Cache::forget('categorias-activas');
}
```
En producción usa **Redis** como almacén de caché (`CACHE_STORE=redis` en el `.env`): es muchísimo más rápido que la base de datos o los archivos.

### B) Índices en las columnas por las que filtras
Si el buscador filtra constantemente por `category_id`, `estado` o `destacado`, añade índices en la migración:
```php
$table->boolean('destacado')->default(false)->index();
$table->index(['category_id', 'precio']); // Índice compuesto: filtrar por categoría y ordenar por precio
```
*(`foreignId()->constrained()` ya crea un índice en la llave foránea automáticamente).*

### C) Detectar el problema N+1 antes de que llegue a producción
En el Módulo 4 viste el problema N+1. Puedes hacer que Laravel **lance un error** en desarrollo si alguna vez olvidas un `with()`:
```php
// En AppServiceProvider::boot()
use Illuminate\Database\Eloquent\Model;

// En local: error si haces lazy loading, si asignas campos fuera de $fillable o accedes a atributos inexistentes.
// En producción: desactivado para no tumbar la API por un descuido.
Model::shouldBeStrict(! app()->isProduction());
```

### D) Ver qué está pasando: Telescope y Pulse
* **Laravel Telescope** (`composer require laravel/telescope --dev`): en desarrollo, te muestra cada petición, cada consulta SQL con su tiempo, cada Job, correo y excepción. Ideal para cazar consultas lentas.
* **Laravel Pulse** (`composer require laravel/pulse`): un panel ligero para **producción** con los endpoints más lentos, consultas lentas, usuarios con más peticiones y uso de colas.

---

## 6. Checklist de Seguridad antes de publicar

* [ ] `APP_ENV=production` y **`APP_DEBUG=false`**.
* [ ] `APP_KEY` generada y **distinta** a la de desarrollo (¡si la cambias después, no podrás descifrar los datos bancarios cifrados con `Crypt` del Módulo 7!).
* [ ] El archivo `.env` **nunca** se sube a Git (comprueba que está en `.gitignore`).
* [ ] HTTPS obligatorio (certificado SSL). Los tokens de Sanctum viajan en un Header: sin HTTPS cualquiera en la misma red Wi-Fi puede robarlos.
* [ ] `config/cors.php` con **solo** tus dominios reales en `allowed_origins` (no `*`).
* [ ] Rate limiting en login, registro y recuperación de contraseña.
* [ ] Caducidad de tokens de Sanctum (`'expiration' => 60 * 24 * 7` en `config/sanctum.php`) y limpieza diaria de tokens vencidos:
  ```php
  // routes/console.php
  use Illuminate\Support\Facades\Schedule;

  Schedule::command('sanctum:prune-expired --hours=24')->daily();
  ```
* [ ] El usuario de base de datos de la API **no** es `root` y solo tiene permisos sobre su propia base de datos.
* [ ] `composer audit` sin vulnerabilidades conocidas en tus dependencias.

---

## 7. Despliegue en un servidor (VPS) paso a paso

> 📚 En esta misma carpeta de tutoriales tienes guías detalladas para la parte de servidor: **[Despliegue con Git en VPS](../desplegar-codigo-con-git-vps.md)**, **[Permisos de carpetas en Linux](../permisos-carpetas-archivos-linux.md)**, **[Fail2ban](../tutorial-fail2ban-seguridad-vps.md)** y **[Comandos de MySQL por SSH](../comandos-comunes-mysql-ssh.md)**.

### Paso 1: Requisitos del servidor
* PHP 8.2 o superior con las extensiones `mbstring`, `xml`, `curl`, `zip`, `bcmath`, `pdo_mysql` (y `redis` si lo usas).
* Composer, MySQL/MariaDB (o PostgreSQL), Nginx o Apache y, opcionalmente, Redis y Supervisor.

### Paso 2: Subir el código e instalar dependencias
```bash
cd /var/www
git clone https://github.com/tu-usuario/techstore-api.git
cd techstore-api

# --no-dev: no instala Pest, Telescope, Faker...  -o: autoloader optimizado
composer install --no-dev --optimize-autoloader

cp .env.example .env
nano .env                    # Rellena DB_*, APP_URL, APP_ENV=production, APP_DEBUG=false...
php artisan key:generate     # ⚠️ SOLO la primera vez

php artisan migrate --force  # --force es obligatorio en producción (Laravel pide confirmación)
php artisan storage:link
```

### Paso 3: Permisos de carpetas
Laravel necesita escribir en `storage/` y `bootstrap/cache/`:
```bash
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
```

### Paso 4: El servidor web debe apuntar a `public/` (¡nunca a la raíz!)
Ejemplo mínimo de Nginx:
```nginx
server {
    listen 80;
    server_name api.mitechstore.com;
    root /var/www/techstore-api/public;   # ← public/, NO la raíz del proyecto

    index index.php;
    client_max_body_size 10M;             # Para permitir subir fotos de productos

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\.(?!well-known).* {
        deny all;                         # Bloquea .env, .git, etc.
    }
}
```
Si la raíz apuntara a la carpeta del proyecto, cualquiera podría descargar tu `.env` con todas tus contraseñas desde `https://api.mitechstore.com/.env`.

### Paso 5: Optimizar (las cachés de producción)
```bash
php artisan optimize
```
Este único comando ejecuta `config:cache`, `route:cache`, `event:cache` y `view:cache`. Tu API arranca bastante más rápido porque no tiene que leer y combinar decenas de archivos en cada petición.

> ⚠️ **Recuerda la regla de oro del Módulo 1:** tras `config:cache`, Laravel **deja de leer el `.env`**. Por eso `env()` solo debe aparecer dentro de `config/`. Y cada vez que cambies el `.env` en el servidor, vuelve a ejecutar `php artisan optimize`.

### Paso 6: Mantener las colas siempre funcionando (Supervisor)
`php artisan queue:work` (Módulo 8) se detiene si cierras la terminal o el proceso falla. En producción lo vigila **Supervisor**, que lo reinicia automáticamente:
```ini
; /etc/supervisor/conf.d/techstore-worker.conf
[program:techstore-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/techstore-api/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/techstore-api/storage/logs/worker.log
stopwaitsecs=3600
```
```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start "techstore-worker:*"
```

### Paso 7: Activar las tareas programadas (Scheduler)
Los comandos programados con `->daily()` o `->dailyAt('08:00')` (Módulo 1) **no se ejecutan solos**. Necesitan **una sola línea** en el cron del servidor (`sudo crontab -u www-data -e`):
```cron
* * * * * cd /var/www/techstore-api && php artisan schedule:run >> /dev/null 2>&1
```
Cada minuto, Laravel revisa qué tareas tocan y las ejecuta.

### Paso 8: Script para cada nueva actualización (deploy)
Cada vez que subas cambios, ejecuta siempre la misma secuencia. Guárdala en `deploy.sh`:
```bash
#!/bin/bash
set -e   # Si algún comando falla, el script se detiene

cd /var/www/techstore-api

php artisan down --retry=15            # Modo mantenimiento: la API responde 503 temporalmente
git pull origin main
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan optimize                    # Regenera las cachés con el código nuevo
php artisan queue:restart               # Los workers cargan el código nuevo al terminar su Job actual
php artisan up

echo "✅ TechStore API actualizada"
```

> 🚀 **¿Quieres algo más automático?** Servicios como **Laravel Forge**, **Laravel Cloud** o paneles como **CloudPanel** hacen todos estos pasos (Nginx, SSL, Supervisor, cron, despliegue con un clic) por ti.

---

## 8. Verificar que todo está vivo: Health Check

Laravel 11+ trae de serie la ruta **`/up`** (configurada en `bootstrap/app.php` con `health: '/up'`). Devuelve `200` si la aplicación arranca correctamente.
Conéctala a un servicio de monitoreo gratuito (UptimeRobot, Better Stack...) para que te avise por correo o Telegram **si tu API se cae**, antes de que te lo digan tus clientes.

---

## 🎓 Cierre del Curso: Lo que ya sabes hacer

| Módulo | Lo que dominas ahora |
| :--- | :--- |
| 1 | Estructura de Laravel, Artisan, configuración propia y comandos personalizados |
| 2 | Migraciones, Factories y Seeders |
| 3 | Modelos: Traits, propiedades, casts, accessors, scopes y observers |
| 4 | Relaciones, Eloquent, N+1, colecciones y transacciones |
| 5 | Rutas, controladores, middlewares con parámetros, CORS y rate limiting |
| 6 | Validación con FormRequests y respuestas con API Resources |
| 7 | Sanctum, Policies, hashing, cifrado, verificación de correo y reseteo de contraseña |
| 8 | Notificaciones, colas, archivos, cliente HTTP, errores JSON y tiempo real |
| 9 | Tests automáticos con Pest, fakes y CI en GitHub |
| 10 | Versionado, contrato de respuestas, documentación, rendimiento y despliegue |

### 🏋️ Proyecto final propuesto (para practicar sin mirar)
Añade a **TechStore API** un módulo de **Cupones de descuento** aplicando todo lo aprendido:
1. Migración `coupons` (`codigo` único, `porcentaje`, `valido_hasta`, `usos_maximos`, `usos_actuales`) y su Factory.
2. Modelo con `casts()` y un scope `vigentes()`.
3. Endpoint `POST /api/v1/orders/{order}/aplicar-cupon` con un `FormRequest` que valide que el cupón existe y está vigente.
4. Una `CouponPolicy`: solo el dueño del pedido puede aplicarle cupones, y solo si está `pendiente`.
5. Descontar el uso dentro de una `DB::transaction` con `lockForUpdate()` para que dos clientes no gasten el último uso a la vez.
6. Un `CouponResource` y rate limiting de 10 intentos por minuto (para que nadie adivine cupones por fuerza bruta).
7. Tests con Pest: cupón válido, vencido, agotado, de otro usuario (`403`) y código inventado (`422`).

¡Felicidades! 🎉 Ya tienes todo lo necesario para diseñar, proteger, probar y publicar cualquier API profesional con Laravel.
