# Módulo 5: Rutas de API, Controladores, Middlewares (con Parámetros) y CORS

Ya tenemos la base de datos, los modelos y el ORM Eloquent de **TechStore API** listos. Ahora necesitamos abrir las puertas al mundo exterior para que una aplicación móvil o una web en React/Vue pueda pedir productos, iniciar sesión y hacer compras.

---

## 1. Las Rutas de API (`routes/api.php`) y los 5 Verbos HTTP

En una API REST, la misma dirección URL (`/api/products`) sirve para distintas acciones dependiendo del **Verbo HTTP** con el que llames:

| Verbo HTTP | Dirección URL | ¿Qué significa en TechStore API? | Función en `ProductController` |
| :--- | :--- | :--- | :--- |
| **`GET`** | `/api/products` | "Dame la lista de todos los productos" | `index()` |
| **`POST`** | `/api/products` | "Guarda este producto nuevo que te envío" | `store()` |
| **`GET`** | `/api/products/10` | "Dame los detalles del producto #10" | `show()` |
| **`PUT` / `PATCH`** | `/api/products/10` | "Actualiza los datos del producto #10" | `update()` |
| **`DELETE`** | `/api/products/10` | "Envía a la papelera el producto #10" | `destroy()` |

*(Diferencia rápida entre `PUT` y `PATCH`: **`PUT`** se piensa para reemplazar el producto entero enviando todos sus campos; **`PATCH`** para cambiar solo un dato suelto, como subir el stock o cambiar solo el precio).*

### Cómo escribirlas en `routes/api.php`: La forma larga vs `apiResource`

En lugar de escribir 5 líneas por cada tabla de tu proyecto:
```php
// Forma manual (5 líneas)
Route::get('/products', [ProductController::class, 'index']);
Route::post('/products', [ProductController::class, 'store']);
Route::get('/products/{product}', [ProductController::class, 'show']);
Route::put('/products/{product}', [ProductController::class, 'update']);
Route::delete('/products/{product}', [ProductController::class, 'destroy']);
```
Laravel te permite resumir esas 5 rutas en **una sola línea** usando **`Route::apiResource`**:
```php
Route::apiResource('products', ProductController::class);
```
Y si quieres que cualquiera pueda *ver* los productos (`index` y `show`), pero solo los usuarios con sesión iniciada puedan *crear, editar o borrar*:
```php
// Rutas públicas (cualquiera puede ver el catálogo de TechStore)
Route::apiResource('products', ProductController::class)->only(['index', 'show']);

// Rutas protegidas (requieren Token de Sanctum y rol de vendedor o admin)
Route::middleware(['auth:sanctum', 'rol:admin,vendedor'])->group(function () {
    Route::apiResource('products', ProductController::class)->except(['index', 'show']);
});
```

---

## 2. Route Model Binding (La magia de `{product}` en la URL)

Mira lo que harías en otros lenguajes cuando alguien visita `GET /api/products/10`:
```php
// Forma antigua: recibes el número 10 y tú mismo lo buscas
public function show($id)
{
    $product = Product::findOrFail($id);
    return response()->json($product);
}
```
En Laravel, si el parámetro de la ruta se llama `{product}` y en tu controlador pones el tipo `Product $product`, **Laravel hace el `findOrFail($id)` por ti automáticamente antes de entrar a la función**:

```php
// ✅ Forma profesional en Laravel (Route Model Binding):
public function show(Product $product)
{
    // Si el producto 10 no existe, Laravel devuelve Error 404 JSON automáticamente.
    // Si existe, ya lo tienes cargado dentro de $product.
    return response()->json($product->load('category', 'reviews'));
}
```

### Trucos avanzados de Route Model Binding:
* **Buscar por otra columna que no sea el `id` (ej. por `sku` o `slug`):**
  ```php
  // Buscará automáticamente donde la columna 'sku' coincida con la URL (/api/products/TECH-99A8B1)
  Route::get('/products/{product:sku}', [ProductController::class, 'show']);
  ```
* **Incluir productos que están en la papelera (`SoftDeletes`):**
  ```php
  Route::post('/products/{product}/restore', [ProductController::class, 'restore'])->withTrashed();
  ```

---

## 3. Los Códigos de Estado HTTP que toda API debe devolver

En una API profesional no basta con devolver un JSON; debes acompañarlo del **Código de Estado HTTP** correcto para que el frontend (React, Flutter, iOS) sepa al instante si todo salió bien o si hubo un problema:

| Código | Nombre | ¿Cuándo devolverlo en TechStore API? |
| :--- | :--- | :--- |
| **`200`** | *OK* | Todo salió bien (consultar catálogo, actualizar producto, hacer login). *(Es el valor por defecto).* |
| **`201`** | *Created* | Se acaba de **crear un registro nuevo** en la base de datos (nuevo producto, nuevo pedido, registro de usuario). |
| **`204`** | *No Content* | Todo salió bien (ej. borraste un producto) y no hace falta devolver ningún texto en el cuerpo. |
| **`400`** | *Bad Request* | La petición no tiene sentido (ej. intentar cancelar un pedido que ya fue entregado). |
| **`401`** | *Unauthorized* | **¿Quién eres?** No enviaste Token de sesión o el Token ya caducó. |
| **`403`** | *Forbidden* | **Sé quién eres, pero no tienes permiso.** (Ej. Eres un `'cliente'` intentando entrar a una ruta de `'admin'`). |
| **`404`** | *Not Found* | El producto o pedido que buscas no existe en la base de datos. |
| **`422`** | *Unprocessable Entity* | **Error de validación.** Olvidaste poner el precio o el correo está mal escrito. |
| **`429`** | *Too Many Requests* | **Vas demasiado rápido.** Hiciste 100 peticiones en 10 segundos y el escudo anti-abusos te frenó. |
| **`500`** | *Server Error* | Algo falló en nuestro código PHP o en la base de datos. |

**Cómo devolver cualquier código en tu Controlador:**
```php
// Devolver 201 Created al guardar un producto nuevo:
return response()->json([
    'mensaje' => 'Producto creado con éxito en TechStore',
    'datos' => $product,
], 201);

// Devolver 403 Forbidden si algo no está permitido:
return response()->json([
    'error' => 'Solo los vendedores verificados pueden publicar en esta categoría.'
], 403);
```

---

## 4. Middlewares a Fondo y Cómo Pasarles Parámetros

### ¿Qué es exactamente un Middleware?
Imagina que tu Controlador es la oficina del jefe de **TechStore**, y alrededor de la oficina hay varios **anillos o puertas de seguridad**.
Cada vez que llega una petición desde internet (`Request`), antes de tocar el Controlador tiene que atravesar esos guardias en orden:

```
Petición del Usuario 
   ➡️ [Middleware 1: ¿El servidor está activo?]
      ➡️ [Middleware 2: auth:sanctum (¿Trae un Token válido?)]
         ➡️ [Middleware 3: rol:admin,vendedor (¿Tiene el rol exigido?)]
            ➡️ 🎯 ¡Llega al ProductController!
```
Si cualquiera de los guardias dice *"¡Alto ahí!"*, devuelve un error JSON (`401` o `403`) inmediatamente y el Controlador ni siquiera llega a enterarse.

---

### Tipos de Middleware: *Before* (Antes) vs *After* (Después)

1. **Before Middleware (El 95% de los casos):** Hace su comprobación **antes** de dejar pasar la petición al controlador.
2. **After Middleware:** Deja que el controlador trabaje primero, recoge la respuesta JSON que generó el controlador y le hace algo **antes** de entregársela al usuario (por ejemplo, medir cuántos milisegundos tardó la petición o agregar una cabecera de seguridad al JSON).

```php
// Ejemplo de BEFORE Middleware (Revisa ANTES del controlador):
public function handle(Request $request, Closure $next): Response
{
    if (! $request->user()->activo) {
        return response()->json(['error' => 'Tu cuenta está suspendida'], 403);
    }

    return $next($request); // Deja pasar al siguiente guardia o al controlador
}

// Ejemplo de AFTER Middleware (Actúa DESPUÉS del controlador):
public function handle(Request $request, Closure $next): Response
{
    $ inicio = microtime(true);

    // 1. Primero deja que el controlador haga todo su trabajo y genere la respuesta
    $response = $next($request);

    // 2. Después le añade una cabecera extra a la respuesta JSON antes de enviarla al cliente
    $tiempoMs = round((microtime(true) - $inicio) * 1000, 2);
    $response->headers->set('X-Tiempo-Respuesta-TechStore', "{$tiempoMs}ms");

    return $response;
}
```

---

### 🔥 Cómo Crear un Middleware que Reciba Parámetros (`rol:admin,vendedor`)

En **TechStore API**, nuestros usuarios tienen una columna `role` que puede valer `'cliente'`, `'vendedor'` o `'admin'`.
No queremos crear 3 middlewares distintos (`EsAdmin`, `EsVendedor`, `EsCliente`). Queremos crear **un solo Middleware inteligente** al que podamos pasarle por parámetro qué roles aceptamos en cada ruta, así:
* `->middleware('rol:admin')` (Solo administradores)
* `->middleware('rol:admin,vendedor')` (Administradores **o** vendedores)

#### Paso 1: Crear el Middleware con Artisan
```bash
php artisan make:middleware CheckRole
```

#### Paso 2: Escribir el código recibiendo los parámetros en `app/Http/Middleware/CheckRole.php`
Fíjate en el tercer parámetro de la función `handle`: **`string ...$roles`**. Los tres puntos (`...`) en PHP significan: *"Recoge todos los parámetros que me pasen separados por coma después de los dos puntos `:` y mételos en un array llamado `$roles`"*.

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckRole
{
    /**
     * Si en la ruta escribes: ->middleware('rol:admin,vendedor')
     * Laravel llama a esta función y dentro de $roles tendrás: ['admin', 'vendedor']
     */
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        // 1. Verificamos si hay un usuario con sesión iniciada
        $usuario = $request->user();

        if (! $usuario) {
            return response()->json([
                'mensaje' => 'Debes iniciar sesión para acceder a esta sección.'
            ], 401);
        }

        // 2. Comprobamos si el rol del usuario está dentro de los roles permitidos en la ruta
        if (! in_array($usuario->role, $roles)) {
            return response()->json([
                'mensaje' => 'Acceso denegado. No tienes el rango necesario.',
                'tu_rol_actual' => $usuario->role,
                'roles_permitidos' => $roles,
            ], 403);
        }

        // 3. ¡Todo en orden! Puede pasar al controlador
        return $next($request);
    }
}
```

#### Paso 3: Registrar el alias `'rol'` en Laravel
Para que Laravel entienda qué clase debe ejecutar cuando escribes `'rol:admin,vendedor'`, registramos su apodo (alias):

* **En Laravel 11 y 12 (`bootstrap/app.php`):**
  ```php
  ->withMiddleware(function (Middleware $middleware) {
      $middleware->alias([
          'rol' => \App\Http\Middleware\CheckRole::class,
      ]);
  })
  ```
* *(En Laravel 10 o anteriores se ponía en `app/Http/Kernel.php` dentro del array `$middlewareAliases`)*:
  ```php
  protected $middlewareAliases = [
      // ...
      'rol' => \App\Http\Middleware\CheckRole::class,
  ];
  ```

#### Paso 4: ¡Usarlo en `routes/api.php` con total libertad!
```php
// Ruta exclusiva para el super administrador (1 parámetro):
Route::delete('/users/{user}', [UserController::class, 'destroy'])
    ->middleware(['auth:sanctum', 'rol:admin']);

// Ruta compartida para administradores y vendedores (2 parámetros separados por coma):
Route::post('/products', [ProductController::class, 'store'])
    ->middleware(['auth:sanctum', 'rol:admin,vendedor']);
```
> **¿Y si quieres pasar dos tipos distintos de parámetros?** (Por ejemplo: el rol y si exigimos cuenta verificada).
> Simplemente los defines en orden en tu función:
> `public function handle(Request $request, Closure $next, string $rol, string $estadoCuenta)`
> Y en la ruta los separas por coma: `->middleware('verificar:vendedor,activa')`.

---

## 5. CORS (`config/cors.php`): Evitando el bloqueo del navegador

Cuando terminas tu API en `http://localhost:8000` y creas tu página web en React/Vue/Angular en `http://localhost:5173`, el navegador web tiene una medida de seguridad llamada **CORS** (*Cross-Origin Resource Sharing*). Si tu API no dice explícitamente *"Confío en `localhost:5173`"*, el navegador bloquea la respuesta con un error rojo enorme.

En Laravel es facilísimo solucionarlo:
1. Si no ves el archivo `config/cors.php`, publícalo con:
   ```bash
   php artisan config:publish cors
   ```
2. Abre `config/cors.php` y pon la dirección de tu frontend en `allowed_origins`:
   ```php
   return [
       'paths' => ['api/*', 'sanctum/csrf-cookie'],
       'allowed_methods' => ['*'], // Permite GET, POST, PUT, DELETE...
       'allowed_origins' => [
           'http://localhost:5173',      // Tu frontend local en React / Vue
           'https://mitechstore.com',    // Tu dominio real en producción
       ],
       'allowed_headers' => ['*'],
       'supports_credentials' => true,
   ];
   ```

---

## 6. Rate Limiting (Escudo contra ataques y abusos)

¿Qué pasa si un robot malicioso intenta probar 10.000 contraseñas por minuto en la ruta `/api/login` de **TechStore**?
Usamos el middleware **`throttle`** (basado en `RateLimiter`) para limitar cuántas peticiones por minuto puede hacer una misma persona:

En `App\Providers\AppServiceProvider.php` (función `boot`):
```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    // Escudo estricto para el Login: máximo 5 intentos por minuto por cada IP o correo
    RateLimiter::for('intentos-login', function (Request $request) {
        return Limit::perMinute(5)->by($request->ip())->response(function () {
            return response()->json([
                'mensaje' => 'Demasiados intentos de inicio de sesión. Espera 1 minuto.'
            ], 429);
        });
    });
}
```
Y en `routes/api.php` proteges la ruta con su nombre:
```php
Route::post('/login', [AuthController::class, 'login'])->middleware('throttle:intentos-login');
```

---

👉 **Siguiente paso:** En el **[Módulo 6: Validaciones (Form Requests) y Transformación JSON (API Resources)](./06_VALIDACIONES_FORM_REQUESTS_Y_API_RESOURCES.md)** aprenderás cómo validar los datos antes de que entren al controlador y cómo diseñar el JSON perfecto de salida.
