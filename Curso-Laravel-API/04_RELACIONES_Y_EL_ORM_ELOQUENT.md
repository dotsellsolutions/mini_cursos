# Módulo 4: Relaciones y el ORM Eloquent a Fondo

En el módulo anterior viste al final del modelo `Product.php` funciones como `belongsTo`, `hasMany` y `belongsToMany`. Ahora vamos a dominar el **ORM Eloquent** y todas sus relaciones usando nuestro proyecto **TechStore API**.

---

## 1. ¿Qué es el ORM (Eloquent) y en qué se diferencia de SQL o `DB::table`?

**ORM** significa *Object-Relational Mapping* (Mapeo Objeto-Relacional). Dicho en español sencillo:
* En una base de datos tradicional, las cosas son **tablas frías con filas y columnas**, y para hablar con ellas tienes que escribir frases largas en lenguaje SQL (`SELECT * FROM products WHERE stock > 0 AND category_id = 3...`).
* Un **ORM** traduce cada tabla a un **Objeto vivo de PHP** (tu Modelo). Así, un producto en PHP tiene propiedades (`$product->nombre`), sabe quién es su categoría (`$product->category`) y sabe guardarse o borrarse a sí mismo (`$product->save()`).

### Eloquent ORM vs Query Builder (`DB::table`)
En Laravel tienes dos formas de consultar la base de datos:

| Característica | **Eloquent ORM (`Product::where(...)`)** | **Query Builder (`DB::table('products')`)** |
| :--- | :--- | :--- |
| **¿Cómo funciona?** | Usa tus **Modelos**. Activa `$casts`, `$hidden`, `$appends`, relaciones, eventos y `SoftDeletes`. | Habla directo con la tabla cruda. Ignora el modelo y devuelve datos planos. |
| **¿Cuándo usarlo en TechStore?** | **El 95% de las veces** (crear pedidos, mostrar catálogo, usuarios, perfiles). | Solo cuando necesitas hacer un reporte estadístico gigante de 1 millón de filas a máxima velocidad. |

---

## 2. La Regla de Oro para No Confundir `BelongsTo` con `HasMany` / `HasOne`

Muchos desarrolladores dudan: *"¿En qué modelo pongo `belongsTo` y en cuál pongo `hasMany` o `hasOne`?"*.
Apréndete este truco infalible de 5 segundos:

> 💡 **REGLA DE ORO:** 
> **La tabla que tiene la columna con el `_id` del otro (`user_id`, `category_id`) SIEMPRE lleva `belongsTo` ("Pertenece a").**
> La otra tabla lleva `hasOne` ("Tiene uno") o `hasMany` ("Tiene muchos").

Comprobémoslo en **TechStore API**:
* ¿Quién tiene la columna `category_id`? La tabla `products`.
  * Por tanto, en `Product.php` va **`belongsTo(Category::class)`** (*"El producto pertenece a una categoría"*).
  * Y en `Category.php` va **`hasMany(Product::class)`** (*"La categoría tiene muchos productos"*).

---

## 3. Todas las Relaciones de Laravel Explicadas con TechStore API

### 1. Relación Uno a Uno (`HasOne` y `BelongsTo`)
**Caso en TechStore:** Un Usuario (`User`) tiene un único Perfil (`Profile`), y ese Perfil pertenece a ese Usuario (`profiles` tiene la columna `user_id`).

En `app/Models/User.php`:
```php
use Illuminate\Database\Eloquent\Relations\HasOne;

public function profile(): HasOne
{
    return $this->hasOne(Profile::class);
}
```

En `app/Models/Profile.php`:
```php
use Illuminate\Database\Eloquent\Relations\BelongsTo;

public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}
```

**Cómo se usa:**
```php
$user = User::find(1);
echo $user->profile->telefono; // Accede al teléfono del perfil directamente

// Crear o actualizar el perfil desde el usuario:
$user->profile()->updateOrCreate(
    ['user_id' => $user->id],
    ['telefono' => '+34 699 888 777', 'direccion' => 'Av. Siempre Viva 742']
);
```

---

### 2. Relación Uno a Muchos (`HasMany` y `BelongsTo`)
**Caso en TechStore:** Una Categoría (`Category`) tiene muchos Productos (`Product`). Además, un Usuario (`User`) hace muchos Pedidos (`Order`).

En `app/Models/Category.php`:
```php
use Illuminate\Database\Eloquent\Relations\HasMany;

public function products(): HasMany
{
    return $this->hasMany(Product::class);
}
```

En `app/Models/Product.php`:
```php
use Illuminate\Database\Eloquent\Relations\BelongsTo;

public function category(): BelongsTo
{
    return $this->belongsTo(Category::class);
}

// También pertenece a un Vendedor (User), pero como la función se llama "seller"
// y la columna se llama "user_id", le indicamos el nombre de la columna en el 2º parámetro:
public function seller(): BelongsTo
{
    return $this->belongsTo(User::class, 'user_id');
}
```

**Cómo se usa:**
```php
// Obtener todos los productos de la categoría 1 que tengan stock:
$categoria = Category::find(1);
$productosDisponibles = $categoria->products()->where('stock', '>', 0)->get();
```

---

### 3. Relación Muchos a Muchos (`BelongsToMany` y Tablas Pivote)
**Caso en TechStore:** Un Pedido (`Order`) tiene muchos Productos (`Product`), y un Producto puede estar en muchos Pedidos.
Como vimos en el Módulo 2, en medio de los dos existe la tabla intermedia **`order_product`** que guarda además **`cantidad`** y **`precio_unitario`**.

En `app/Models/Order.php`:
```php
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

public function products(): BelongsToMany
{
    return $this->belongsToMany(Product::class, 'order_product')
                ->withPivot('cantidad', 'precio_unitario') // ¡Importante! Trae las columnas extra de la tabla intermedia
                ->withTimestamps(); // Mantiene actualizados created_at y updated_at en la tabla intermedia
}
```

En `app/Models/Product.php`:
```php
public function orders(): BelongsToMany
{
    return $this->belongsToMany(Order::class, 'order_product')
                ->withPivot('cantidad', 'precio_unitario')
                ->withTimestamps();
}
```

**Las 3 funciones mágicas para manejar tablas intermedias (`attach`, `detach`, `sync`):**
```php
$pedido = Order::find(1);

// 1. attach(): Agrega un producto al pedido indicando los datos de la tabla intermedia (pivote)
$pedido->products()->attach($productoId, [
    'cantidad' => 2,
    'precio_unitario' => 59.99,
]);

// 2. detach(): Quita un producto del pedido
$pedido->products()->detach($productoId);

// 3. sync(): Reemplaza todo el carrito del pedido por exactamente esta lista (borra los que no estén y agrega los nuevos)
$pedido->products()->sync([
    10 => ['cantidad' => 1, 'precio_unitario' => 120.00],
    15 => ['cantidad' => 3, 'precio_unitario' => 25.50],
]);

// ¿Cómo leemos esos datos extra ("pivot") cuando recorremos el pedido?
foreach ($pedido->products as $producto) {
    echo $producto->nombre;
    echo $producto->pivot->cantidad;         // 👈 Accede a la columna de la tabla intermedia
    echo $producto->pivot->precio_unitario;  // 👈 Accede al precio histórico guardado
}
```

---

### 4. Relación "A través de" (`HasManyThrough`)
Imagina que quieres saber todos los Pedidos (`Order`) de un País (`Country`), pasando a través de los Usuarios (`User`).
`Country` ➡️ tiene muchos `User` ➡️ tienen muchos `Order`.
```php
// En Country.php:
public function orders(): HasManyThrough
{
    return $this->hasManyThrough(Order::class, User::class);
}
```

---

### 5. Relaciones Polimórficas (`MorphTo` y `MorphMany`)
**Caso en TechStore:** Queremos que la tabla `reviews` (reseñas con estrellas) sirva para calificar tanto un **Producto** (`Product`) como a un **Vendedor** (`User`).

En `app/Models/Review.php`:
```php
use Illuminate\Database\Eloquent\Relations\MorphTo;

class Review extends Model
{
    protected $fillable = ['user_id', 'calificacion', 'comentario'];

    // Esta reseña puede pertenecer a cualquier modelo (Product o User)
    public function reviewable(): MorphTo
    {
        return $this->morphTo();
    }
}
```

Y tanto en `Product.php` como en `User.php` ponemos exactamente lo mismo:
```php
use Illuminate\Database\Eloquent\Relations\MorphMany;

public function reviews(): MorphMany
{
    return $this->morphMany(Review::class, 'reviewable');
}
```

**Cómo se usa:**
```php
// Dejarle una reseña de 5 estrellas a un Producto:
$producto = Product::find(10);
$producto->reviews()->create([
    'user_id' => auth()->id(),
    'calificacion' => 5,
    'comentario' => '¡Excelente teclado mecánico!',
]);

// Dejarle una reseña de 4 estrellas a un Vendedor:
$vendedor = User::find(2);
$vendedor->reviews()->create([
    'user_id' => auth()->id(),
    'calificacion' => 4,
    'comentario' => 'Envío muy rápido y bien protegido.',
]);
```

---

## 4. El Problema N+1 (El enemigo silencioso de las APIs) y Eager Loading

Imagina que en tu ruta `/api/productos` quieres devolver 50 productos y, dentro de cada uno, el nombre de su categoría.
Si haces esto:
```php
// ❌ MAL (Provoca el problema N+1):
$productos = Product::all(); // 1 consulta para traer los 50 productos
foreach ($productos as $p) {
    echo $p->category->nombre; // ¡Hace 1 consulta extra por cada producto! (50 consultas más)
}
```
¡Tu API acaba de hacer **51 consultas a la base de datos** para una sola pantalla! Con 1.000 productos, el servidor colapsa.

### La Solución: `with()` (Eager Loading / Carga Ansiosa)
Con **`with()`**, Laravel trae los 50 productos en la consulta 1, junta todos los IDs de categorías y los trae todos de golpe en la consulta 2. **Total: solo 2 consultas, haya 10 o 10.000 productos.**

```php
// ✅ BIEN (Solo 2 consultas ultrarrápidas):
$productos = Product::with(['category', 'seller.profile'])->get();

// Y si solo quieres traer un par de columnas de la relación para ahorrar memoria:
$productos = Product::with(['category:id,nombre', 'seller:id,name'])->get();

// ¿Quieres contar cuántas reseñas tiene cada producto y su promedio de estrellas sin traer todas las reseñas?
$productos = Product::withCount('reviews')
                    ->withAvg('reviews', 'calificacion')
                    ->get();
// Esto agrega automáticamente al JSON: "reviews_count": 12 y "reviews_avg_calificacion": 4.8
```

*(Nota: Si ya tenías el modelo cargado en una variable `$producto` y decides cargar la relación después, usas **`$producto->load('category')`**).*

---

## 5. Todas las Consultas Esenciales de Eloquent para tu API

Aquí tienes la chuleta completa de métodos para buscar y guardar en **TechStore API**:

```php
// 1. Buscar por ID o devolver automáticamente Error 404 JSON si no existe:
$producto = Product::findOrFail($id);

// 2. Condiciones combinadas:
$productos = Product::where('destacado', true)
    ->where('stock', '>', 0)
    ->whereBetween('precio', [50, 300])      // Precio entre 50 y 300
    ->whereIn('category_id', [1, 2, 5])      // Que sea de la categoría 1, 2 o 5
    ->whereNotNull('imagen_url')             // Que tenga foto subida
    ->latest()                               // Ordenado del más nuevo al más viejo (created_at DESC)
    ->paginate(15);                          // Paginado de 15 en 15 para la API

// 3. firstOrCreate: Busca si existe la categoría por slug; si no existe, la crea en ese instante
$categoria = Category::firstOrCreate(
    ['slug' => 'monitores-4k'],              // Qué buscar
    ['nombre' => 'Monitores 4K', 'activa' => true] // Con qué datos crearla si no estaba
);

// 4. updateOrCreate: Si el perfil del usuario ya existe lo actualiza; si no, lo crea
$perfil = Profile::updateOrCreate(
    ['user_id' => auth()->id()],
    ['telefono' => '600111222', 'direccion' => 'Calle Nueva 12']
);

// 5. Sumar o restar stock de forma atómica y segura:
$producto->decrement('stock', 2); // Resta 2 al stock directamente en la BD
$producto->increment('stock', 5); // Suma 5 al stock
```

### 🔥 Filtros Dinámicos por URL con `when()` (Buscador Profesional de API)
Cuando el frontend llama a `/api/productos?categoria=2&buscar=teclado&precio_max=100`, en lugar de escribir diez bloques `if ($request->has(...))` desordenados, usas **`when()`**:

```php
public function index(Request $request)
{
    $productos = Product::with('category')
        // Solo filtra por categoría SI enviaron ?categoria=... en la URL
        ->when($request->categoria, function ($query, $categoriaId) {
            $query->where('category_id', $categoriaId);
        })
        // Solo filtra por texto SI enviaron ?buscar=... en la URL
        ->when($request->buscar, function ($query, $texto) {
            $query->where('nombre', 'like', "%{$texto}%");
        })
        // Solo filtra por precio máximo SI enviaron ?precio_max=...
        ->when($request->precio_max, function ($query, $precioMax) {
            $query->where('precio', '<=', $precioMax);
        })
        ->paginate(12);

    return response()->json($productos);
}
```

---

## 6. Colecciones de Laravel (`Collection`)

Cuando haces `$productos = Product::get();`, lo que recibes no es un simple array de PHP, sino una **Colección de Laravel** con docenas de funciones comodísimas:

```php
$productos = Product::all();

// 1. pluck(): Extraer solo una lista con los nombres de todos los productos
$nombres = $productos->pluck('nombre'); // ["Monitor", "Teclado", "Mouse"]

// 2. filter(): Quedarse solo con los que cumplan una condición en memoria
$caros = $productos->filter(fn ($p) => $p->precio > 500);

// 3. map(): Transformar cada elemento en un nuevo formato
$resumen = $productos->map(fn ($p) => [
    'etiqueta' => "{$p->nombre} (\${$p->precio})",
]);

// 4. sum() y avg(): Sumar o promediar cualquier columna
$valorInventario = $productos->sum(fn ($p) => $p->precio * $p->stock);
```

---

## 7. Transacciones (`DB::transaction`): Seguridad Total al Comprar en TechStore

Imagina lo que ocurre cuando un cliente compra en **TechStore API**:
1. Se crea el registro en la tabla `orders`.
2. Se agregan los 3 productos comprados en la tabla intermedia `order_product`.
3. Se descuenta el stock de esos 3 productos en la tabla `products`.

**¿Qué pasa si en el paso 3 el tercer producto ya no tenía stock y lanza un error?**
Sin transacciones, el pedido quedaría creado a medias y el stock de los dos primeros productos se habría descontado por error.

Con **`DB::transaction()`**, Laravel envuelve todo en una caja de seguridad: **o salen bien los 3 pasos, o si falla cualquiera, deshace todos los cambios automáticamente (*Rollback*)** como si nunca hubiera pasado nada:

```php
use Illuminate\Support\Facades\DB;
use Exception;

public function crearPedido(Request $request)
{
    return DB::transaction(function () use ($request) {
        // Paso 1: Crear la cabecera del pedido
        $pedido = Order::create([
            'user_id' => $request->user()->id,
            'estado' => 'pendiente',
            'total' => 0,
        ]);

        $totalCalculado = 0;

        // Paso 2: Recorrer cada producto que el cliente quiere comprar
        foreach ($request->productos as $item) {
            // lockForUpdate() evita que dos clientes compren la última unidad al mismo milisegundo
            $producto = Product::lockForUpdate()->findOrFail($item['product_id']);

            if ($producto->stock < $item['cantidad']) {
                // Al lanzar una excepción dentro de DB::transaction, Laravel DESHACE todo automáticamente
                throw new Exception("No hay suficiente stock para: {$producto->nombre}");
            }

            // Paso 3: Descontar stock
            $producto->decrement('stock', $item['cantidad']);

            // Paso 4: Guardar en la tabla pivote order_product
            $pedido->products()->attach($producto->id, [
                'cantidad' => $item['cantidad'],
                'precio_unitario' => $producto->precio,
            ]);

            $totalCalculado += ($producto->precio * $item['cantidad']);
        }

        // Paso 5: Actualizar el total final del pedido
        $pedido->update(['total' => $totalCalculado]);

        return $pedido->load('products');
    });
}
```

---

👉 **Siguiente paso:** En el **[Módulo 5: Rutas de API, Controladores, Middlewares con Parámetros y CORS](./05_RUTAS_API_CONTROLADORES_Y_MIDDLEWARES.md)** conectaremos este código con internet y crearemos nuestros propios guardias de seguridad (*Middlewares*) capaces de recibir parámetros.
