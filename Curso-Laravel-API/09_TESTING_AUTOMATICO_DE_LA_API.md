# Módulo 9: Testing Automático de la API (Pest y PHPUnit)

Hasta ahora probábamos **TechStore API** a mano desde Postman: hacer login, copiar el token, pegarlo en el Header, enviar el JSON... Eso funciona con 5 rutas, pero cuando tienes 60 rutas y cambias una línea en un Modelo, **es imposible volver a probarlo todo a mano** cada vez.

Los **tests automáticos** son pequeños programas que hacen esas peticiones por ti en milisegundos y te avisan en rojo si algo se rompió. En este módulo vas a aprender a escribirlos para tu API.

---

## 1. ¿Por qué testear una API? (La analogía del piloto)

Antes de despegar, un piloto no "confía en que todo esté bien": recorre una **lista de comprobación** (combustible ✔, frenos ✔, alerones ✔).
Tus tests son esa lista de comprobación. Antes de subir código a producción ejecutas **un solo comando** y en 3 segundos sabes si:

* ✔ El login sigue devolviendo un token.
* ✔ Un cliente **no** puede borrar productos (`403`).
* ✔ Crear un producto sin precio devuelve `422` con el mensaje correcto.
* ✔ Al pagar un pedido se envía la notificación y se descuenta el stock.

### Tipos de tests en Laravel (carpeta `tests/`)

| Carpeta | ¿Qué prueba? | Ejemplo en TechStore |
| :--- | :--- | :--- |
| **`tests/Feature/`** | Una petición HTTP completa: ruta → middleware → controlador → base de datos → JSON. | `POST /api/products` devuelve `201` y el producto existe en la BD. |
| **`tests/Unit/`** | Una pieza aislada (una función, un cálculo) sin HTTP ni base de datos. | El accessor `precio_con_iva` calcula bien `100 → 121`. |

> 💡 En una API, **el 80% de tus tests serán *Feature tests***, porque lo que te importa es que el cliente (la app móvil) reciba el JSON y el código HTTP correctos.

---

## 2. Pest vs PHPUnit: ¿Cuál uso?

Laravel trae dos herramientas para escribir tests. **Ambas ejecutan exactamente lo mismo por debajo** (Pest está construido encima de PHPUnit); solo cambia la forma de escribirlos:

**Con PHPUnit (estilo clásico, con clases):**
```php
class ProductTest extends TestCase
{
    public function test_el_catalogo_es_publico(): void
    {
        $this->getJson('/api/products')->assertOk();
    }
}
```

**Con Pest (estilo moderno, más corto):**
```php
it('el catálogo es público', function () {
    $this->getJson('/api/products')->assertOk();
});
```

En este curso usaremos **Pest**, porque es la opción por defecto al crear un proyecto nuevo con `laravel new` y se lee casi como español. Si tu proyecto ya usa PHPUnit, todo lo de este módulo funciona igual: solo envuelve cada test en un método `public function test_...()` dentro de una clase.

---

## 3. Preparar el entorno de pruebas (¡No toques tu base de datos real!)

Los tests **crean y borran datos sin parar**. Jamás deben ejecutarse contra tu base de datos de desarrollo (y mucho menos contra la de producción).

### Paso 1: Usar SQLite en memoria (`phpunit.xml`)
Abre el archivo `phpunit.xml` de la raíz y asegúrate de que estas líneas estén **descomentadas**:

```xml
<php>
    <env name="APP_ENV" value="testing"/>
    <env name="DB_CONNECTION" value="sqlite"/>
    <env name="DB_DATABASE" value=":memory:"/>
    <env name="CACHE_STORE" value="array"/>
    <env name="MAIL_MAILER" value="array"/>
    <env name="QUEUE_CONNECTION" value="sync"/>
    <env name="SESSION_DRIVER" value="array"/>
</php>
```
* `:memory:` crea una base de datos **dentro de la RAM** que nace al empezar los tests y desaparece al terminar. Es rapidísima y nunca ensucia tu MySQL.
* `MAIL_MAILER=array` hace que los correos se "envíen" a un array en memoria (no llega nada a nadie).

> ⚠️ **Ojo con SQLite:** Si en tus migraciones usas funciones exclusivas de MySQL (por ejemplo, ciertos índices `fullText` o consultas `DB::raw` con sintaxis MySQL), puede que algún test falle solo en SQLite. En ese caso crea una base de datos MySQL aparte llamada `techstore_testing` y pon `DB_CONNECTION=mysql` y `DB_DATABASE=techstore_testing`.

### Paso 2: El Trait `RefreshDatabase`
Abre `tests/Pest.php` y activa `RefreshDatabase` para todos los Feature tests:

```php
<?php
// Archivo: tests/Pest.php

pest()->extend(Tests\TestCase::class)
    ->use(Illuminate\Foundation\Testing\RefreshDatabase::class)
    ->in('Feature');
```
**`RefreshDatabase`** ejecuta tus migraciones al principio y **deshace todo lo que hizo cada test** al terminarlo. Así cada test empieza con la base de datos vacía y limpia, sin depender del anterior.

### Paso 3: Crear y ejecutar tests
```bash
# Crear un test de tipo Feature (tests/Feature/ProductTest.php)
php artisan make:test ProductTest --pest

# Crear un test Unitario (tests/Unit/PrecioConIvaTest.php)
php artisan make:test PrecioConIvaTest --unit --pest

# Ejecutar TODOS los tests
php artisan test

# Ejecutar solo un archivo o filtrar por nombre
php artisan test tests/Feature/ProductTest.php
php artisan test --filter="catálogo"

# Ejecutar en paralelo (mucho más rápido cuando tienes cientos de tests)
php artisan test --parallel
```

---

## 4. Anatomía de un Feature Test: Arrange, Act, Assert

Todo test sigue siempre la misma receta de 3 pasos (las **3 A**):

1. **Arrange (Preparar):** Creas los datos que necesitas con tus **Factories** (Módulo 2).
2. **Act (Actuar):** Haces la petición HTTP a tu API.
3. **Assert (Afirmar):** Compruebas que la respuesta y la base de datos son las esperadas.

```php
<?php
// Archivo: tests/Feature/ProductTest.php

use App\Models\Product;

it('lista los productos del catálogo de forma pública', function () {
    // 1. ARRANGE: Creamos 3 productos falsos con la Factory
    Product::factory()->count(3)->create();

    // 2. ACT: Pedimos el catálogo SIN iniciar sesión
    $respuesta = $this->getJson('/api/products');

    // 3. ASSERT: Comprobamos el código y la forma del JSON
    $respuesta->assertOk()                 // Código 200
        ->assertJsonCount(3, 'data')       // Hay 3 elementos dentro de "data"
        ->assertJsonStructure([            // Cada producto tiene estas claves (ProductResource)
            'data' => [
                '*' => ['id', 'nombre', 'precios' => ['sin_impuestos', 'con_iva'], 'inventario'],
            ],
            'links',                       // Enlaces de paginación
            'meta' => ['current_page', 'total'],
        ]);
});
```

> 🔑 **¿Por qué `getJson()` y no `get()`?** Porque `getJson()`, `postJson()`, `putJson()` y `deleteJson()` envían automáticamente el Header **`Accept: application/json`** (¿te acuerdas del Módulo 6?). Sin él, un error de validación devolvería una redirección `302` en lugar de un `422` en JSON.

### Las afirmaciones (`assert`) más útiles para APIs

| Afirmación | ¿Qué comprueba? |
| :--- | :--- |
| `assertOk()` / `assertCreated()` / `assertNoContent()` | Código `200` / `201` / `204` |
| `assertUnauthorized()` / `assertForbidden()` / `assertNotFound()` | Código `401` / `403` / `404` |
| `assertUnprocessable()` | Código `422` (error de validación) |
| `assertStatus(429)` | Cualquier código concreto |
| `assertJson(['nombre' => 'Mouse'])` | Que el JSON **contenga** esos valores |
| `assertJsonPath('data.precios.moneda', 'USD')` | El valor de una clave concreta (notación de punto) |
| `assertJsonStructure([...])` | Que existan esas claves (sin importar los valores) |
| `assertJsonCount(3, 'data')` | Cantidad de elementos de un array |
| `assertJsonValidationErrors(['precio'])` | Que el `422` incluya error en esos campos |
| `assertJsonMissing(['codigo_interno_proveedor' => '...'])` | Que un dato **NO** aparezca (ideal para datos privados) |
| `$this->assertDatabaseHas('products', [...])` | Que exista esa fila en la BD |
| `$this->assertDatabaseMissing(...)` / `assertSoftDeleted($product)` | Que no exista / que esté en la papelera |

---

## 5. Testear rutas protegidas con Sanctum (`Sanctum::actingAs`)

Para probar rutas con `auth:sanctum` **no necesitas hacer login ni copiar tokens**. Sanctum te da un atajo que "inicia sesión" a un usuario solo durante ese test:

```php
<?php
// Archivo: tests/Feature/ProductTest.php

use App\Models\Category;
use App\Models\Product;
use App\Models\User;
use Laravel\Sanctum\Sanctum;

it('rechaza crear productos si no hay sesión iniciada', function () {
    $this->postJson('/api/products', [])->assertUnauthorized(); // 401
});

it('impide que un cliente cree productos', function () {
    Sanctum::actingAs(User::factory()->create(['role' => 'cliente']));

    $this->postJson('/api/products', [])->assertForbidden(); // 403 (middleware rol:admin,vendedor)
});

it('permite a un vendedor publicar un producto', function () {
    // ARRANGE
    $vendedor = User::factory()->create(['role' => 'vendedor']);
    $categoria = Category::factory()->create();

    // Segundo parámetro: las "abilities" del token (Módulo 7)
    Sanctum::actingAs($vendedor, ['producto:crear']);

    // ACT
    $respuesta = $this->postJson('/api/products', [
        'category_id' => $categoria->id,
        'nombre' => 'Teclado Mecánico RGB',
        'precio' => 59.99,
        'stock' => 25,
    ]);

    // ASSERT
    $respuesta->assertCreated()
        ->assertJsonPath('data.nombre', 'Teclado Mecánico RGB');

    $this->assertDatabaseHas('products', [
        'nombre' => 'Teclado Mecánico RGB',
        'user_id' => $vendedor->id, // Se guardó a nombre del vendedor que hizo la petición
    ]);
});
```

### Testear las Policies (un vendedor no toca productos ajenos)
```php
it('impide que un vendedor edite el producto de otro vendedor', function () {
    $carlos = User::factory()->create(['role' => 'vendedor']);
    $productoDePedro = Product::factory()->create(); // La Factory crea otro vendedor (Pedro)

    Sanctum::actingAs($carlos);

    $this->putJson("/api/products/{$productoDePedro->id}", [
        'category_id' => $productoDePedro->category_id,
        'nombre' => 'Intento de hackeo',
        'precio' => 1,
        'stock' => 0,
    ])
        ->assertForbidden()
        ->assertJson(['message' => 'No puedes modificar este producto porque pertenece a otro vendedor.']);

    // Y el nombre original NO cambió en la base de datos:
    expect($productoDePedro->fresh()->nombre)->not->toBe('Intento de hackeo');
});

it('permite al admin borrar cualquier producto (regla before de la Policy)', function () {
    $producto = Product::factory()->create();
    Sanctum::actingAs(User::factory()->create(['role' => 'admin']));

    $this->deleteJson("/api/products/{$producto->id}")->assertSuccessful();

    $this->assertSoftDeleted($producto); // Está en la papelera (SoftDeletes), no borrado del todo
});
```

### Testear el login real (sin `actingAs`)
Para el `AuthController` sí queremos probar el flujo de verdad:
```php
it('devuelve un token con credenciales correctas', function () {
    User::factory()->create([
        'email' => 'ana@techstore.com',
        'password' => 'secreto123', // El cast 'hashed' del modelo lo cifra solo
    ]);

    $this->postJson('/api/login', [
        'email' => 'ana@techstore.com',
        'password' => 'secreto123',
    ])
        ->assertOk()
        ->assertJsonStructure(['mensaje', 'usuario', 'token']);
});

it('rechaza una contraseña incorrecta con 401', function () {
    User::factory()->create(['email' => 'ana@techstore.com']);

    $this->postJson('/api/login', [
        'email' => 'ana@techstore.com',
        'password' => 'incorrecta',
    ])->assertUnauthorized();
});
```

---

## 6. Testear las validaciones (`422`)

Los tests de validación son los más fáciles y los que más bugs atrapan. Con los **datasets** de Pest puedes probar muchos casos inválidos con un solo test:

```php
it('valida los datos del producto', function (array $datosInvalidos, string $campoConError) {
    Sanctum::actingAs(User::factory()->create(['role' => 'vendedor']));

    $datosValidos = [
        'category_id' => Category::factory()->create()->id,
        'nombre' => 'Monitor 27"',
        'precio' => 199.99,
        'stock' => 10,
    ];

    // Mezclamos los datos válidos con el dato roto de cada caso
    $this->postJson('/api/products', array_merge($datosValidos, $datosInvalidos))
        ->assertUnprocessable()
        ->assertJsonValidationErrors([$campoConError]);
})->with([
    'sin nombre'          => [['nombre' => ''], 'nombre'],
    'precio negativo'     => [['precio' => -5], 'precio'],
    'precio de texto'     => [['precio' => 'gratis'], 'precio'],
    'stock decimal'       => [['stock' => 2.5], 'stock'],
    'categoría inventada' => [['category_id' => 9999], 'category_id'],
    'garantía inválida'   => [['especificaciones' => ['marca' => 'ASUS', 'garantia_meses' => 7]], 'especificaciones.garantia_meses'],
]);
```
Pest ejecuta ese test **6 veces**, una por cada caso, y en la terminal ves el nombre de cada uno (`sin nombre`, `precio negativo`...).

---

## 7. "Fakes": Testear correos, colas, archivos y APIs externas sin enviar nada real

Cuando un cliente paga un pedido, tu API envía notificaciones, lanza Jobs, guarda archivos y llama a la pasarela de pagos (Módulo 8). En un test **no quieres** enviar correos reales ni cobrar tarjetas reales.
Laravel trae **"dobles de prueba" (*fakes*)** para cada Facade. Se activan con una línea y después puedes preguntarles *"¿te llamaron?"*:

### A) `Notification::fake()` y `Queue::fake()`
*(Suponemos una ruta `POST /api/orders/{order}/pagar` en `OrderController` que marca el pedido como pagado, ejecuta `$pedido->user->notify(new PedidoPagadoNotification($pedido))` y `GenerarFacturaPdfJob::dispatch($pedido)`, como vimos en el Módulo 8. También necesitas una `OrderFactory`: `php artisan make:factory OrderFactory --model=Order`).*

```php
use App\Jobs\GenerarFacturaPdfJob;
use App\Models\Order;
use App\Notifications\PedidoPagadoNotification;
use Illuminate\Support\Facades\Notification;
use Illuminate\Support\Facades\Queue;

it('al pagar un pedido notifica al cliente y encola la factura', function () {
    Notification::fake(); // Las notificaciones NO se envían, solo se registran
    Queue::fake();        // Los Jobs NO se ejecutan, solo se registran

    $cliente = User::factory()->create();
    $pedido = Order::factory()->for($cliente)->create(['estado' => 'pendiente']);
    Sanctum::actingAs($cliente);

    $this->postJson("/api/orders/{$pedido->id}/pagar")->assertOk();

    Notification::assertSentTo($cliente, PedidoPagadoNotification::class);
    Queue::assertPushed(GenerarFacturaPdfJob::class, fn ($job) => $job->order->is($pedido));
});
```

### B) `Storage::fake()`: Subida de imágenes sin llenar tu disco
```php
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

it('guarda la foto del producto en el disco público', function () {
    Storage::fake('public'); // Disco temporal falso que se borra solo

    Sanctum::actingAs(User::factory()->create(['role' => 'vendedor']));

    $respuesta = $this->postJson('/api/products', [
        'category_id' => Category::factory()->create()->id,
        'nombre' => 'Auriculares Bluetooth',
        'precio' => 39.90,
        'stock' => 5,
        // Crea una imagen falsa de 600x600 px sin necesitar un archivo real
        'imagen' => UploadedFile::fake()->image('auriculares.jpg', 600, 600),
    ])->assertCreated();

    $ruta = Product::first()->imagen_url; // Ej: "productos/a8f9k2l1.jpg"
    Storage::disk('public')->assertExists($ruta);
});

it('rechaza archivos que no son imágenes', function () {
    Storage::fake('public');
    Sanctum::actingAs(User::factory()->create(['role' => 'vendedor']));

    $this->postJson('/api/products', [
        'imagen' => UploadedFile::fake()->create('virus.exe', 100),
    ])->assertJsonValidationErrors(['imagen']);
});
```

### C) `Http::fake()`: Simular la pasarela de pagos externa
```php
use Illuminate\Support\Facades\Http;

it('marca el pedido como pagado si la pasarela aprueba el cobro', function () {
    // Cualquier petición a esa URL devolverá esta respuesta inventada (no sale nada a internet)
    Http::fake([
        'api.pasareladepagos.com/*' => Http::response(['status' => 'approved', 'id' => 'ch_123'], 200),
    ]);

    // ... haces la petición a tu endpoint de pago ...

    Http::assertSent(fn ($request) => str_contains($request->url(), '/v1/charges'));
});

it('responde con error si la pasarela está caída', function () {
    Http::fake(['api.pasareladepagos.com/*' => Http::response(null, 500)]);
    // ... y compruebas que tu API devuelve un error controlado y el pedido sigue "pendiente"
});
```

> 🛡️ **Truco de seguridad:** Añade `Http::preventStrayRequests();` al principio de tus tests (o en `tests/Pest.php` con `beforeEach`). Si algún código intenta llamar a una URL que no hayas "fakeado", el test falla en vez de llamar a internet de verdad.

| Facade | Fake | Afirmaciones típicas |
| :--- | :--- | :--- |
| `Notification` | `Notification::fake()` | `assertSentTo($user, Clase::class)`, `assertNothingSent()` |
| `Mail` | `Mail::fake()` | `assertSent(Clase::class)`, `assertQueued(...)` |
| `Queue` / `Bus` | `Queue::fake()` | `assertPushed(Job::class)`, `assertNotPushed(...)` |
| `Event` | `Event::fake()` | `assertDispatched(NuevoPedidoEnVivoEvent::class)` |
| `Storage` | `Storage::fake('public')` | `assertExists($ruta)`, `assertMissing($ruta)` |
| `Http` | `Http::fake([...])` | `assertSent(fn ...)`, `assertSentCount(1)` |

---

## 8. Factories "con estados" para tests más legibles

En los tests anteriores repetimos mucho `User::factory()->create(['role' => 'vendedor'])`. Puedes crear **estados** con nombre dentro de `database/factories/UserFactory.php`:

```php
// Dentro de la clase UserFactory
public function vendedor(): static
{
    return $this->state(fn () => ['role' => 'vendedor']);
}

public function admin(): static
{
    return $this->state(fn () => ['role' => 'admin']);
}
```
Ahora tus tests se leen como frases:
```php
Sanctum::actingAs(User::factory()->vendedor()->create());
Sanctum::actingAs(User::factory()->admin()->create());

// Y en ProductFactory podrías tener ->agotado() o ->destacado():
Product::factory()->agotado()->count(5)->create();
```

---

## 9. Un test Unitario: el accessor `precio_con_iva`

Los tests unitarios no usan HTTP. Son perfectos para lógica de cálculo:

```php
<?php
// Archivo: tests/Unit/PrecioConIvaTest.php

use App\Models\Product;

it('calcula el precio con IVA usando la configuración de TechStore', function () {
    config(['techstore.impuestos.iva' => 0.21]); // Fijamos el IVA solo para este test

    $producto = new Product(['precio' => 100]); // Sin guardar en la BD

    expect($producto->precio_con_iva)->toBe(121.0);
});
```
*(Fíjate en la ventaja de haber puesto el IVA en `config/techstore.php` en el Módulo 1: en el test lo cambias con `config([...])` sin tocar el `.env`).*

> ⚠️ Si tu test unitario necesita `config()` u otras funciones de Laravel, debe extender `Tests\TestCase`. Añade en `tests/Pest.php`: `pest()->extend(Tests\TestCase::class)->in('Unit');`

---

## 10. Cobertura y ejecución automática en GitHub (CI)

### ¿Cuánto de mi código está probado? (`--coverage`)
```bash
php artisan test --coverage --min=70
```
Muestra el porcentaje de líneas cubiertas por tests en cada archivo y **falla si baja del 70%**. *(Requiere tener instalada la extensión **Xdebug** o **PCOV** de PHP).*

### Ejecutar los tests en cada `git push` (GitHub Actions)
Crea el archivo `.github/workflows/tests.yml` en tu proyecto:

```yaml
name: Tests TechStore API

on: [push, pull_request]

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: sqlite3, pdo_sqlite
          coverage: none

      - name: Instalar dependencias
        run: composer install --no-interaction --prefer-dist

      - name: Preparar .env
        run: |
          cp .env.example .env
          php artisan key:generate

      - name: Ejecutar tests
        run: php artisan test --parallel
```
A partir de ahora, cada vez que tú o tu equipo hagan `git push`, GitHub ejecuta todos los tests y marca el commit con ✅ o ❌. **Nunca más subirás a producción algo que rompa el login sin enterarte.**

---

## ✅ Checklist: ¿Qué debería tener testeado TechStore API como mínimo?

* [ ] **Auth:** registro (`201`), login correcto (token), login incorrecto (`401`), logout invalida el token.
* [ ] **Permisos:** sin token → `401`; rol incorrecto → `403`; dueño vs no dueño (Policy) → `200` / `403`.
* [ ] **Validación:** al menos un caso inválido por cada campo obligatorio (`422`).
* [ ] **CRUD de productos:** listar, ver, crear, editar, borrar (y que el borrado sea *soft delete*).
* [ ] **Privacidad:** `codigo_interno_proveedor` **no** aparece para clientes (`assertJsonMissingPath`).
* [ ] **Compra:** la transacción descuenta stock y no permite comprar más unidades de las disponibles.
* [ ] **Efectos secundarios:** notificaciones, Jobs y archivos verificados con *fakes*.

---

👉 **Siguiente paso:** En el **[Módulo 10: Buenas Prácticas Pro y Despliegue a Producción](./10_BUENAS_PRACTICAS_Y_DESPLIEGUE_A_PRODUCCION.md)** aprenderás a versionar tu API, documentarla automáticamente, hacerla más rápida con caché y subirla a un servidor real.
