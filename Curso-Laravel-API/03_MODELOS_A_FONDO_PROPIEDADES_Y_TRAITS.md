# Módulo 3: Modelos a Fondo — Traits, `$fillable` y Todas las Propiedades (`$`)

Cuando abres un Modelo de Laravel (por ejemplo `app/Models/Product.php` o `app/Models/User.php`), ves un montón de palabras clave (`use HasFactory;`, `BelongsTo`, `HasMany`, `$fillable`, `$hidden`, `$casts`...). 

En este módulo vamos a desglosar **absolutamente cada línea, cada `use`, cada propiedad `$` y cada función especial** que puede tener un Modelo en **TechStore API**, explicada en español llano y con ejemplos directos.

---

## 1. El misterio de los dos tipos de `use` en un Modelo (Tu Captura de Pantalla)

En la captura que subiste se veían estas líneas en la parte superior del archivo:
```php
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
```
En PHP y Laravel, la palabra `use` sirve para **dos cosas distintas según dónde esté escrita**:

### A) `use` FUERA de la clase (arriba del todo): Importar herramientas
Es simplemente decirle a tu archivo: *"Oye, voy a usar estas piezas que viven en otras carpetas de Laravel para no tener que escribir su nombre larguísimo cada vez"*.
* `use ...\HasFactory;` 👉 Importa la pieza para fabricar datos de prueba.
* `use ...\BelongsTo;` y `use ...\HasMany;` 👉 Importan los **tipos de relación** para indicar qué devuelve cada función (ayuda a tu editor VS Code a autocompletar el código y detectar errores).

### B) `use` DENTRO de la clase: Activar "Superpoderes" (Traits)
Cuando escribes `use` **adentro** de `class Product extends Model { ... }`, le estás inyectando un **Trait** (un paquete de funciones ya programadas) a tu modelo:

```php
class User extends Authenticatable
{
    // 👇 Estos son TRAITS: le inyectan superpoderes al modelo User
    use HasApiTokens, HasFactory, Notifiable, SoftDeletes;
```

### ¿Qué hace cada uno de los Traits más famosos de Laravel?

| Trait (`use ...;` dentro de la clase) | ¿Qué superpoder le da a tu modelo en TechStore API? |
| :--- | :--- |
| **`HasFactory`** | Te permite usar `Product::factory()->count(10)->create()` para inventar productos de prueba. |
| **`Notifiable`** | Le permite al modelo recibir notificaciones: `$user->notify(new PedidoEnviado($order))`. |
| **`HasApiTokens`** *(de Sanctum)* | Permite que ese usuario cree y gestione Tokens de acceso para la API (`$user->createToken('mi-celular')`). |
| **`SoftDeletes`** | **Papelera de reciclaje:** Cuando haces `$product->delete()`, **NO borra la fila de la base de datos**; solo pone la fecha actual en `deleted_at` y lo oculta de las búsquedas normales. Así puedes restaurarlo con `$product->restore()` o borrarlo para siempre con `$product->forceDelete()`. |
| **`HasUuids`** | Hace que Laravel genere automáticamente un código UUID (`9b1deb4d-3b7d...`) en lugar de usar IDs numéricos `1, 2, 3`. |
| **`Prunable`** | Permite que Laravel limpie automáticamente registros viejos (ej. borrar definitivamente productos en papelera tras 6 meses). |

---

## 2. Todas las Propiedades (`$`) de un Modelo: La Guía Definitiva

Dentro de un modelo puedes configurar muchas variables protegidas (`protected $...`). **Ninguna es obligatoria**, porque Laravel asume valores inteligentes por defecto, pero conocerlas todas te da control total sobre tu API.

---

### 1. `$fillable` vs `$guarded` (Seguridad al guardar datos)

#### ¿Qué problema resuelven?
Imagina que un cliente malintencionado envía desde Postman una petición para actualizar su perfil o crear un producto, y en el JSON mete un campo trampa: `"role": "admin"` o `"precio": 0.01`.
Si en tu controlador haces `User::create($request->all())`, sin protección, el usuario se convertiría en Administrador.

Por eso Laravel bloquea por defecto el guardado masivo (*Mass Assignment*) a menos que declares **`$fillable`** o **`$guarded`**:

#### A) `$fillable` (La Lista Blanca — Recomendada)
Aquí enumeras **únicamente las columnas que permites rellenar de golpe** con `create()` o `update()`:

```php
protected $fillable = [
    'category_id',
    'user_id',
    'nombre',
    'sku',
    'descripcion',
    'precio',
    'stock',
    'destacado',
    'especificaciones',
    'imagen_url',
];
```
Si alguien intenta colar un campo que no está en `$fillable`, Laravel simplemente lo ignora en silencio.

#### B) `$guarded` (La Lista Negra — Lo opuesto a `$fillable`)
En lugar de decir cuáles *sí* se pueden llenar, dices cuáles **están prohibidas** (y todas las demás quedan permitidas):

```php
// "Permite rellenar todas las columnas EXCEPTO el id y el rol"
protected $guarded = ['id', 'role'];

// O si quieres desactivar la protección por completo (útil solo si validas todo con FormRequest):
protected $guarded = [];
```
> **Importante:** Usa `$fillable` **O** `$guarded` en un modelo, pero nunca los dos al mismo tiempo.

---

### 2. `$hidden` y `$visible` (Qué se ve y qué se oculta en el JSON de tu API)

Cuando tu API devuelve un usuario o un producto (`return response()->json($product);`), Laravel convierte todas las columnas en JSON. Pero hay datos que **jamás** quieres mostrar al público.

#### A) `$hidden` (Ocultar campos secretos del JSON)
```php
// En User.php: nunca envíes la contraseña en el JSON de la API
protected $hidden = [
    'password',
    'remember_token',
];

// En Product.php: ocultamos el código secreto del proveedor y la fecha de borrado
protected $hidden = [
    'codigo_interno_proveedor',
    'deleted_at',
];
```

#### B) `$visible` (Mostrar SOLO los campos de esta lista en el JSON)
Es lo contrario de `$hidden`. Si defines `$visible`, **solo** esos campos saldrán en el JSON y el resto se ocultará:
```php
protected $visible = ['id', 'nombre', 'precio'];
```
*(Truco: Si en una ruta específica de Administrador sí quieres ver un campo oculto, puedes hacer `$product->makeVisible('codigo_interno_proveedor')` al vuelo).*

---

### 3. `$casts` / `casts()` (El traductor automático de tipos de datos)

En la base de datos casi todo se guarda como texto o números simples (un booleano se guarda como `1` o `0`, un JSON como texto plano `'{"marca":"ASUS"}'`).
Con **`casts`**, Laravel convierte esos valores automáticamente al tipo real de PHP y JSON cuando los lees o los guardas:

```php
protected function casts(): array
{
    return [
        // 1. Convierte el 1 o 0 de la base de datos en true o false real en el JSON
        'destacado' => 'boolean',

        // 2. Asegura que el precio tenga siempre 2 decimales y el stock sea número entero
        'precio' => 'decimal:2',
        'stock' => 'integer',

        // 3. Convierte automáticamente un array de PHP a JSON al guardar, y de JSON a array al leer
        'especificaciones' => 'array',

        // 4. Convierte textos de fecha en objetos Carbon (para sumar días o formatear fechas)
        'email_verified_at' => 'datetime',
        'created_at' => 'datetime:Y-m-d H:i', // Formato limpio en el JSON: "2026-11-09 10:30"

        // 5. En User.php: convierte automáticamente cualquier texto en contraseña hasheada al guardar
        'password' => 'hashed',

        // 6. En Profile.php: CIFRA el dato en la base de datos y lo DESCIFRA al leerlo en PHP
        'cuenta_bancaria_cifrada' => 'encrypted',
    ];
}
```
> *Nota:* En Laravel 10 o anteriores se escribía como propiedad `protected $casts = [...]` y en Laravel 11+ se escribe como método `protected function casts(): array { return [...]; }`. ¡Ambas formas funcionan igual en Laravel actual!

---

### 4. `$appends` (Agregar campos calculados virtuales al JSON de la API)

A veces quieres que el JSON de cada producto en **TechStore API** incluya el **`precio_con_iva`** (sumándole el 21% de impuestos) o un indicador **`disponible`** (`true` si `stock > 0`), pero **no quieres crear columnas reales en la base de datos** para eso.

Con `$appends`, le dices a Laravel que incluya esos campos calculados en el JSON de salida siempre:

```php
protected $appends = [
    'precio_con_iva',
    'disponible',
];
```
*(En la sección 3 de aquí abajo verás cómo se crean las funciones que calculan esos dos valores).*

---

### 5. `$with` (Cargar relaciones automáticamente siempre)

Si cada vez que pides un Producto en **TechStore API** necesitas que venga incluido el nombre de su Categoría, en lugar de escribir `Product::with('category')->get()` en 20 sitios distintos, pones esto en `Product.php`:

```php
// Siempre que se consulte un Producto, traerá su categoría incluida en el JSON
protected $with = ['category'];
```
*(Úsalo con moderación: solo para relaciones que realmente necesites el 95% de las veces).*

---

### 6. Propiedades de Tabla, Clave Primaria y Fechas (Personalización de la Base de Datos)

Laravel asume reglas automáticas:
1. Que el modelo `Product` usa la tabla en plural `products`.
2. Que la clave primaria se llama `id`, es numérica y se autoincrementa.
3. Que la tabla tiene las columnas `created_at` y `updated_at`.

Si alguna tabla tuya (o una base de datos heredada de otra empresa) **no sigue esas reglas**, lo cambias con estas propiedades:

```php
// 1. Si tu tabla en la base de datos se llama "tbl_articulos_tienda" en vez de "products":
protected $table = 'tbl_articulos_tienda';

// 2. Si tu columna de ID único se llama "codigo_producto" en vez de "id":
protected $primaryKey = 'codigo_producto';

// 3. Si tu clave primaria NO es un número que sube solo (1, 2, 3...), sino un código manual como "PROD-001":
public $incrementing = false;
protected $keyType = 'string';

// 4. Si tu tabla NO tiene las columnas created_at ni updated_at:
public $timestamps = false;

// 5. Si sí tiene fechas, pero se llaman "fecha_creacion" y "fecha_modificacion":
const CREATED_AT = 'fecha_creacion';
const UPDATED_AT = 'fecha_modificacion';

// 6. Cuántos productos devolver por página cuando usas Product::paginate() sin número:
protected $perPage = 15;

// 7. Valores por defecto en PHP antes incluso de guardar en la base de datos:
protected $attributes = [
    'stock' => 0,
    'destacado' => false,
];

// 8. Actualizar la columna updated_at de la Categoría cuando se modifica un Producto hijo:
protected $touches = ['category'];

// 9. Si este modelo vive en una segunda base de datos distinta (ej. servidor de contabilidad):
protected $connection = 'mysql_contabilidad';
```

---

## 3. Funciones Especiales dentro del Modelo

Además de las relaciones (que veremos a fondo en el Módulo 4), dentro de un Modelo puedes escribir **3 tipos de funciones súper potentes**:

### A) Accessors y Mutators (Transformar datos al leer o al guardar)
Recuerdas que arriba pusimos `protected $appends = ['precio_con_iva', 'disponible'];`? Aquí es donde los creamos usando la clase `Attribute`:

* **Accessor (`get`)**: Calcula o maquilla un dato **cuando sale hacia el JSON**.
* **Mutator (`set`)**: Limpia o transforma un dato **antes de guardarse en la base de datos**.

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

// 1. Campo virtual: calcula el precio + 21% de IVA al vuelo para el JSON
protected function precioConIva(): Attribute
{
    return Attribute::make(
        get: fn () => round($this->precio * 1.21, 2),
    );
}

// 2. Campo virtual: devuelve true si hay stock disponible
protected function disponible(): Attribute
{
    return Attribute::make(
        get: fn () => $this->stock > 0,
    );
}

// 3. Transformador de columna real (sku):
// Al guardar (set), lo convierte siempre a MAYÚSCULAS y sin espacios.
protected function sku(): Attribute
{
    return Attribute::make(
        set: fn (string $value) => strtoupper(trim($value)),
    );
}
```
> **Ojo al nombre:** Si la función en PHP se escribe en *camelCase* `precioConIva()`, en el JSON y en `$appends` se llama en *snake_case* `'precio_con_iva'`.

---

### B) Query Scopes (Tus propios filtros inteligentes reutilizables)
En lugar de repetir `Product::where('stock', '>', 0)->where('destacado', true)` por todos tus controladores, creas funciones que empiezan por la palabra **`scope`** dentro del modelo:

```php
use Illuminate\Database\Eloquent\Builder;

// Filtro 1: Solo productos con stock mayor a 0
public function scopeConStock(Builder $query): void
{
    $query->where('stock', '>', 0);
}

// Filtro 2: Productos más baratos que un monto recibido por parámetro
public function scopeMasBaratosQue(Builder $query, float $precioMaximo): void
{
    $query->where('precio', '<=', $precioMaximo);
}
```
**¿Cómo se usan en el controlador?** Llamándolos sin la palabra `scope` y encadenados como si fueran propios de Laravel:
```php
$ofertas = Product::conStock()->masBaratosQue(100)->get();
```

---

### C) Eventos del Modelo y Observers (Acciones Automáticas)
¿Qué pasa si en **TechStore API** queremos que cada vez que se borre un Producto, se borre también su foto del disco duro para no dejar basura? ¿O que al crear un Producto sin código `sku`, se invente uno automáticamente?

Podemos usar un **Observer** (`app/Observers/ProductObserver.php`) o el método `booted()` dentro del mismo modelo:

```php
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

protected static function booted(): void
{
    // Antes de crear el producto en la base de datos:
    static::creating(function (Product $product) {
        if (empty($product->sku)) {
            $product->sku = 'TECH-' . strtoupper(Str::random(6));
        }
    });

    // Cuando el producto se borra definitivamente:
    static::forceDeleted(function (Product $product) {
        if ($product->imagen_url) {
            Storage::disk('public')->delete($product->imagen_url);
        }
    });
}
```

---

## 4. El Modelo `Product.php` Completo de TechStore API (Todo unido)

Mira qué claro se lee ahora un modelo profesional cuando entiendes qué hace cada línea:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\MorphMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Product extends Model
{
    // 1. TRAITS (Superpoderes activados en este modelo)
    use HasFactory, SoftDeletes;

    // 2. COLUMNAS PERMITIDAS PARA GUARDADO MASIVO
    protected $fillable = [
        'category_id',
        'user_id',
        'nombre',
        'sku',
        'descripcion',
        'precio',
        'stock',
        'destacado',
        'especificaciones',
        'imagen_url',
        'codigo_interno_proveedor',
    ];

    // 3. COLUMNAS OCULTAS EN EL JSON DE LA API
    protected $hidden = [
        'codigo_interno_proveedor',
        'deleted_at',
    ];

    // 4. CAMPOS VIRTUALES CALCULADOS QUE SE AGREGAN AL JSON
    protected $appends = [
        'precio_con_iva',
        'disponible',
    ];

    // 5. CONVERSIÓN AUTOMÁTICA DE TIPOS DE DATOS
    protected function casts(): array
    {
        return [
            'precio' => 'decimal:2',
            'stock' => 'integer',
            'destacado' => 'boolean',
            'especificaciones' => 'array',
            'created_at' => 'datetime:Y-m-d H:i',
        ];
    }

    // 6. ACCESSORS Y MUTATORS (Transformadores de datos)
    protected function precioConIva(): Attribute
    {
        return Attribute::make(
            get: fn () => round($this->precio * 1.21, 2),
        );
    }

    protected function disponible(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->stock > 0,
        );
    }

    // 7. SCOPES (Filtros reutilizables)
    public function scopeConStock(Builder $query): void
    {
        $query->where('stock', '>', 0);
    }

    // 8. RELACIONES (Las veremos a fondo en el Módulo 4)
    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function seller(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function orders(): BelongsToMany
    {
        return $this->belongsToMany(Order::class)
                    ->withPivot('cantidad', 'precio_unitario')
                    ->withTimestamps();
    }

    public function reviews(): MorphMany
    {
        return $this->morphMany(Review::class, 'reviewable');
    }
}
```

Quando tu API devuelve este producto en JSON, el cliente recibe automáticamente esta maravilla sin esfuerzo:
```json
{
  "id": 1,
  "category_id": 2,
  "nombre": "Monitor Gamer 27 Pulgadas",
  "sku": "TECH-99A8B1",
  "precio": "200.00",
  "precio_con_iva": 242,
  "stock": 15,
  "disponible": true,
  "destacado": true,
  "especificaciones": {
    "marca": "ASUS",
    "garantia_meses": 24
  },
  "created_at": "2026-11-09 10:00"
}
```
*(Observa cómo `codigo_interno_proveedor` desapareció gracias a `$hidden`, `especificaciones` salió como objeto JSON gracias a `$casts`, y `precio_con_iva` junto con `disponible` aparecieron gracias a `$appends`).*

---

👉 **Siguiente paso:** En el **[Módulo 4: Relaciones y el ORM Eloquent a Fondo](./04_RELACIONES_Y_EL_ORM_ELOQUENT.md)** entenderás todas las relaciones (`BelongsTo`, `HasMany`, `BelongsToMany`, polimórficas), todas las consultas del ORM, el problema N+1 y las Transacciones.
