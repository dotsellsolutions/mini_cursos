# Módulo 2: Migraciones, Factories y Seeders (Construyendo la Base de Datos de TechStore)

Antes de que nuestra API pueda vender un solo producto, necesitamos construir el almacén donde se guardará la información. En Laravel jamás creamos tablas a mano abriendo programas como phpMyAdmin: usamos **Migraciones**, **Factories** y **Seeders**.

---

## 1. ¿Qué son las Migraciones? (El control de versiones de tus tablas)

Imagina que trabajas con un compañero y creas la tabla `products` a mano en tu computadora. Cuando tu compañero descarga tu código, a él le dará error porque en su computadora esa tabla no existe.

Las **Migraciones** son archivos PHP en `database/migrations/` que contienen las instrucciones exactas para crear o modificar cada tabla. Cuando cualquiera escribe `php artisan migrate`, Laravel lee esos planos y construye todas las tablas idénticas en 1 segundo.

Cada migración tiene dos funciones:
* **`up()`**: Qué hacer cuando avanzamos (crear la tabla o agregar una columna).
* **`down()`**: Cómo deshacerlo si nos arrepentimos (borrar la tabla o quitar la columna).

---

## 2. Diccionario Completo de Tipos de Columnas y Modificadores

Cuando diseñas una tabla en Laravel, usas la variable `$table` seguida del tipo de dato que quieres guardar:

### A) Tipos de columnas más usados
| Código en la Migración | ¿Qué guarda y para qué sirve en TechStore? |
| :--- | :--- |
| `$table->id();` | Crea el número único autoincremental (`1, 2, 3...`) llamado `id`. |
| `$table->uuid('id')->primary();` | En vez de números `1, 2, 3`, usa códigos largos imposibles de adivinar (`a8f9-4b2c...`). |
| `$table->string('nombre', 150);` | Texto corto (hasta 255 letras por defecto). Ideal para nombres, correos o títulos. |
| `$table->text('descripcion');` | Texto largo sin límite pequeño. Ideal para la descripción detallada de un producto o reseña. |
| `$table->integer('stock');` | Número entero (positivo o negativo). |
| `$table->unsignedInteger('vistas');` | Número entero que **nunca puede ser negativo** (de 0 hacia arriba). |
| `$table->decimal('precio', 10, 2);` | Número con decimales exactos (`10` dígitos en total, de los cuales `2` son centavos). **Obligatorio para dinero.** |
| `$table->boolean('activo');` | Guarda verdadero (`true` / `1`) o falso (`false` / `0`). |
| `$table->enum('rol', ['cliente', 'vendedor', 'admin']);` | Solo permite guardar una de las palabras de esa lista cerrada. |
| `$table->json('especificaciones');` | Guarda un bloque JSON completo (ej. `{"ram": "16GB", "color": "negro"}`). |
| `$table->date('fecha_nacimiento');` | Solo fecha (`2026-11-09`). |
| `$table->timestamp('pagado_en');` | Fecha y hora exacta (`2026-11-09 14:30:00`). |
| `$table->timestamps();` | Crea **2 columnas automáticas**: `created_at` (cuándo se creó) y `updated_at` (cuándo se editó). |
| `$table->softDeletes();` | Crea la columna `deleted_at` para la "Papelera de reciclaje" (borrar sin destruir el registro). |
| `$table->tinyInteger('nivel');` / `$table->bigInteger('visitas');` | Enteros más pequeños (-128 a 127) o más grandes que `integer`. Con `unsigned` delante solo admiten positivos (`unsignedTinyInteger`, `unsignedBigInteger`). |
| `$table->float('peso_kg');` / `$table->double('latitud');` | Decimales **aproximados**. Bien para medidas o coordenadas; **nunca para dinero** (usa `decimal`). |
| `$table->char('codigo_pais', 2);` | Texto de longitud **fija** (ej. `ES`, `MX`). |
| `$table->longText('contenido_html');` | Texto enorme (hasta 4 GB). Para artículos o HTML largo. |
| `$table->dateTime('inicio_oferta');` / `time('hora_apertura');` / `year('anio');` | Fecha y hora, solo hora o solo año. |
| `$table->ulid('id')->primary();` | Como el UUID, pero ordenable por fecha de creación. |
| `$table->foreignId('user_id');` | Entero grande sin signo pensado para llaves foráneas (ver apartado C). |
| `$table->morphs('reviewable');` | Crea `reviewable_id` + `reviewable_type` para relaciones polimórficas. Usa `nullableMorphs()` si pueden quedar vacías. |
| `$table->rememberToken();` | Columna `remember_token` que usa Laravel para el "recordarme" del login. |
| `$table->ipAddress('ip_registro');` | Guarda una dirección IP (v4 o v6). |

### B) Modificadores (se encadenan al final de la columna)
* **`->nullable()`**: Permite que el campo quede vacío (`NULL`). Si no lo pones, el campo será obligatorio en la base de datos.
* **`->default('cliente')`**: Si nadie envía un valor, pone este por defecto.
* **`->unique()`**: Prohibido repetir el mismo valor en dos filas (ideal para `email` o `sku` del producto).
* **`->index()`**: Crea un índice de búsqueda rápida (ponlo en columnas por las que busques mucho, como `nombre`).
* **`->unsigned()`**: Solo números positivos (equivale a usar `unsignedInteger`, `unsignedBigInteger`...).
* **`->after('precio')`** / **`->first()`**: Coloca la columna detrás de otra o al principio de la tabla (solo MySQL/MariaDB; muy útil al añadir columnas a tablas existentes).
* **`->comment('Precio sin IVA')`**: Deja un comentario en la columna, visible desde phpMyAdmin o con `php artisan db:table`.
* **`->useCurrent()`**: En un `timestamp`, pone la fecha y hora actual por defecto.
* **`->change()`**: Modifica una columna que ya existe (lo verás en el apartado 3.C).

### C) Claves Foráneas (Conectar una tabla con otra)
Para decir que un Producto pertenece a una Categoría (`category_id`) o a un Vendedor (`user_id`):

```php
// Crea la columna category_id y verifica que exista en la tabla categories
$table->foreignId('category_id')
      ->constrained('categories')
      ->cascadeOnDelete(); // Si se borra la categoría, se borran sus productos automáticamente
```
*(Variante muy útil: `->nullOnDelete()` junto con `->nullable()`, que significa: "Si se borra la categoría, no borres el producto, solo deja su `category_id` vacío")*.

---

## 3. Del Comando Artisan a la Tabla: Crear y Modificar Tablas y Columnas

Artisan **no crea la tabla directamente**: crea un **archivo de migración** (el "plano"). Tú escribes en él las columnas y luego Artisan lo ejecuta contra la base de datos. Siempre son **3 pasos**:

```
1. php artisan make:migration ...   →  Genera el archivo en database/migrations/
2. Editas el archivo                →  Escribes las columnas dentro de up() y cómo deshacerlo en down()
3. php artisan migrate              →  Laravel ejecuta el plano y la tabla/columna aparece en MySQL
```

### A) Crear una tabla nueva desde Artisan

```bash
php artisan make:migration create_coupons_table
```
Se crea un archivo como `database/migrations/2026_11_11_093000_create_coupons_table.php`. La fecha del nombre decide el **orden** en que se ejecutan las migraciones.

> 💡 **El nombre importa:** si sigue el patrón **`create_NOMBRE_table`**, Laravel adivina el nombre de la tabla y te deja el `Schema::create('coupons', ...)` ya escrito. Si tu nombre no sigue el patrón, puedes indicarlo tú: `php artisan make:migration crear_cupones --create=coupons`.

Abres el archivo y añades tus columnas:
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    // Lo que pasa al ejecutar "php artisan migrate"
    public function up(): void
    {
        Schema::create('coupons', function (Blueprint $table) {
            $table->id();
            $table->string('codigo', 30)->unique();          // Ej: "BLACKFRIDAY25"
            $table->unsignedTinyInteger('porcentaje');       // De 0 a 255 (usaremos 1-100)
            $table->unsignedInteger('usos_maximos')->default(1);
            $table->unsignedInteger('usos_actuales')->default(0);
            $table->timestamp('valido_hasta')->nullable();
            $table->timestamps();
        });
    }

    // Lo que pasa al ejecutar "php artisan migrate:rollback"
    public function down(): void
    {
        Schema::dropIfExists('coupons');
    }
};
```
Y la ejecutas:
```bash
php artisan migrate
```

**Atajo:** si la tabla va a tener Modelo (casi siempre), créalos juntos:
```bash
php artisan make:model Coupon -m      # Modelo + migración create_coupons_table
php artisan make:model Coupon -mfs    # + Factory + Seeder
```

---

### B) Añadir columnas a una tabla que YA existe

Imagina que **TechStore** ya está en marcha y ahora quieres un campo `descuento` en `products`.

> ⚠️ **LA REGLA DE ORO:** Si la migración original **ya se ejecutó** (y sobre todo si ya está en producción o la tiene tu compañero), **NUNCA la edites**. Laravel guarda en la tabla `migrations` cuáles ya ejecutó y **no la volverá a leer**: tu cambio simplemente no ocurriría. Siempre se crea una **migración nueva** que modifica la tabla.

```bash
php artisan make:migration add_descuento_to_products_table
```
Con el patrón **`add_ALGO_to_TABLA_table`**, Laravel te prepara `Schema::table('products', ...)` (fíjate: `table`, **no** `create`, porque la tabla ya existe). Equivalente manual: `--table=products`.

```php
public function up(): void
{
    Schema::table('products', function (Blueprint $table) {
        // after(): la coloca justo detrás de 'precio' (solo MySQL/MariaDB)
        $table->decimal('descuento', 5, 2)->default(0)->after('precio');

        // Puedes añadir varias columnas en la misma migración:
        $table->string('marca', 80)->nullable()->after('nombre');
        $table->boolean('envio_gratis')->default(false);
    });
}

public function down(): void
{
    Schema::table('products', function (Blueprint $table) {
        // Deshacer = quitar exactamente lo que añadimos en up()
        $table->dropColumn(['descuento', 'marca', 'envio_gratis']);
    });
}
```
```bash
php artisan migrate
```

> 🧠 **Ojo con las tablas que ya tienen filas:** si añades una columna obligatoria (sin `->nullable()` ni `->default()`), la base de datos no sabe qué poner en los productos que ya existen y puede fallar o rellenar con valores vacíos. **Al añadir columnas a tablas con datos, usa siempre `nullable()` o `default()`.**

#### Añadir una llave foránea a una tabla existente
Por ejemplo, ahora un pedido puede tener un cupón aplicado:
```bash
php artisan make:migration add_coupon_id_to_orders_table
```
```php
public function up(): void
{
    Schema::table('orders', function (Blueprint $table) {
        // nullable: los pedidos antiguos no tienen cupón
        $table->foreignId('coupon_id')->nullable()->after('user_id')
              ->constrained()->nullOnDelete();
    });
}

public function down(): void
{
    Schema::table('orders', function (Blueprint $table) {
        // Borra la llave foránea Y la columna de una sola vez
        $table->dropConstrainedForeignId('coupon_id');
    });
}
```

---

### C) Modificar una columna existente (`->change()`)

Los nombres de producto se quedaron cortos y quieres ampliarlos a 255 caracteres, o quieres que `descripcion` pase a ser opcional:
```bash
php artisan make:migration change_nombre_in_products_table --table=products
```
```php
public function up(): void
{
    Schema::table('products', function (Blueprint $table) {
        $table->string('nombre', 255)->change();
        $table->text('descripcion')->nullable()->change();
    });
}

public function down(): void
{
    Schema::table('products', function (Blueprint $table) {
        $table->string('nombre', 150)->change();
        $table->text('descripcion')->nullable(false)->change();
    });
}
```
> ⚠️ **Muy importante con `->change()`:** escribes la columna **completa, tal como quieres que quede**. Si la columna tenía `->nullable()`, `->default(0)` o era `unsigned` y no lo vuelves a escribir, **se pierde**. Ejemplo: `stock` era `unsignedInteger()->default(0)`; para hacerla nullable escribe `$table->unsignedInteger('stock')->default(0)->nullable()->change();`, no solo `->nullable()->change()`.
>
> *(En Laravel 10 o anteriores, `->change()` necesitaba instalar `composer require doctrine/dbal`. Desde Laravel 11 ya no hace falta).*

---

### D) Renombrar y eliminar columnas

```bash
php artisan make:migration rename_imagen_url_in_products_table --table=products
```
```php
public function up(): void
{
    Schema::table('products', function (Blueprint $table) {
        $table->renameColumn('imagen_url', 'imagen_ruta');   // Renombrar
        $table->dropColumn('codigo_interno_proveedor');      // Eliminar (¡se pierden sus datos!)
    });
}

public function down(): void
{
    Schema::table('products', function (Blueprint $table) {
        $table->renameColumn('imagen_ruta', 'imagen_url');
        $table->string('codigo_interno_proveedor')->nullable(); // Se recrea, pero los datos borrados NO vuelven
    });
}
```

**Atajos para quitar columnas "especiales":**

| Para quitar... | Usa |
| :--- | :--- |
| `timestamps()` | `$table->dropTimestamps();` |
| `softDeletes()` | `$table->dropSoftDeletes();` |
| `rememberToken()` | `$table->dropRememberToken();` |
| `morphs('reviewable')` | `$table->dropMorphs('reviewable');` |
| Llave foránea + columna `user_id` | `$table->dropConstrainedForeignId('user_id');` |
| Solo la llave foránea (deja la columna) | `$table->dropForeign(['user_id']);` |

---

### E) Índices en tablas existentes

```php
Schema::table('products', function (Blueprint $table) {
    $table->index('destacado');                      // Índice simple
    $table->index(['category_id', 'precio']);        // Índice compuesto
    $table->unique('sku');                           // Índice único

    // Para borrarlos (en down()) se pasa el mismo array de columnas:
    // $table->dropIndex(['destacado']);
    // $table->dropIndex(['category_id', 'precio']);
    // $table->dropUnique(['sku']);
});
```

---

### F) Renombrar y eliminar tablas completas

```php
// Renombrar la tabla
Schema::rename('coupons', 'cupones');

// Eliminar la tabla (dropIfExists no falla si ya no existe)
Schema::dropIfExists('coupons');
```
> Si renombras una tabla, díselo al Modelo con `protected $table = 'cupones';` (Módulo 3).

### G) Migraciones "a prueba de errores" con `Schema::hasColumn` / `Schema::hasTable`
Útil cuando no estás seguro del estado de una base de datos antigua:
```php
if (! Schema::hasColumn('products', 'descuento')) {
    Schema::table('products', function (Blueprint $table) {
        $table->decimal('descuento', 5, 2)->default(0);
    });
}
```

---

### H) Todos los comandos `migrate` que necesitas

| Comando | ¿Qué hace? | ¿Seguro en producción? |
| :--- | :--- | :--- |
| `php artisan make:migration nombre` | Crea un archivo de migración (vacío o con plantilla según el nombre). | ✅ |
| `php artisan migrate` | Ejecuta **solo** las migraciones pendientes (las que no están en la tabla `migrations`). | ✅ (con `--force`) |
| `php artisan migrate --pretend` | **No toca nada:** te muestra el SQL que ejecutaría. Ideal para revisar antes de producción. | ✅ |
| `php artisan migrate:status` | Lista todas las migraciones y si ya se ejecutaron (`Ran`) o están pendientes (`Pending`). | ✅ |
| `php artisan migrate:rollback` | Deshace el **último lote** (todas las migraciones del último `migrate`) ejecutando sus `down()`. | ⚠️ Con cuidado |
| `php artisan migrate:rollback --step=1` | Deshace solo la **última** migración. | ⚠️ Con cuidado |
| `php artisan migrate:reset` | Deshace **todas** las migraciones. | ❌ |
| `php artisan migrate:refresh` | `reset` + `migrate` (ejecuta todos los `down()` y luego todos los `up()`). | ❌ |
| `php artisan migrate:fresh` | **Borra todas las tablas** directamente (sin usar `down()`) y ejecuta todo desde cero. | ❌ ¡Borra todo! |
| `php artisan migrate:fresh --seed` | Igual, y además ejecuta los Seeders. El favorito en desarrollo. | ❌ ¡Borra todo! |
| `php artisan db:seed` | Ejecuta `DatabaseSeeder` sin tocar la estructura de las tablas. | ⚠️ Según el seeder |
| `php artisan db:seed --class=CategorySeeder` | Ejecuta solo un seeder concreto. | ⚠️ Según el seeder |
| `php artisan db:show` | Resumen de tu base de datos: motor, tamaño, tablas y número de filas. | ✅ |
| `php artisan db:table products` | Muestra las columnas, tipos, índices y llaves foráneas de una tabla. | ✅ |
| `php artisan schema:dump --prune` | Junta todas tus migraciones antiguas en un solo archivo SQL (para proyectos con cientos de migraciones). | ✅ |

> 🔄 **Flujo típico cuando te equivocas en una columna (solo en tu PC):**
> 1. `php artisan migrate:rollback --step=1` (deshace la migración recién ejecutada)
> 2. Corriges el archivo de migración
> 3. `php artisan migrate`
>
> Esto **solo** vale si esa migración aún no la tiene nadie más (no está subida a Git ni ejecutada en producción). Si ya la tiene alguien, crea una migración nueva (apartado B).

---

### I) Recordatorio: tras añadir columnas, actualiza el Modelo
La migración solo cambia la **base de datos**. Para que tu API use la columna nueva:
1. Añádela a **`$fillable`** si vas a guardarla con `create()` / `update()` (si no, se ignora en silencio).
2. Añádela a **`casts()`** si necesita un tipo (`'envio_gratis' => 'boolean'`, `'descuento' => 'decimal:2'`).
3. Añádela a las **reglas del `FormRequest`** (Módulo 6) y al **`API Resource`** si debe aparecer en el JSON.
4. Si la columna es obligatoria, actualiza la **Factory** para que los tests y seeders sigan funcionando.

---

## 4. Creando las Migraciones Reales de TechStore API

Vamos a ver cómo quedan las migraciones de nuestro proyecto **TechStore API** conectadas entre sí:

### 1. Tabla `users` y `profiles` (Relación 1 a 1)
En `database/migrations/0001_01_01_000000_create_users_table.php`:
```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    // Rol del usuario dentro de TechStore
    $table->enum('role', ['cliente', 'vendedor', 'admin'])->default('cliente');
    $table->rememberToken();
    $table->timestamps();
});
```

Y creamos la tabla `profiles` para guardar los datos adicionales y bancarios del usuario:
```php
Schema::create('profiles', function (Blueprint $table) {
    $table->id();
    // Relación 1 a 1 con User (unique garantiza que un usuario solo tenga 1 perfil)
    $table->foreignId('user_id')->unique()->constrained()->cascadeOnDelete();
    $table->string('telefono', 30)->nullable();
    $table->string('direccion')->nullable();
    // Aquí guardaremos el IBAN / Cuenta bancaria CIFRADA (texto largo porque el cifrado ocupa espacio)
    $table->text('cuenta_bancaria_cifrada')->nullable();
    $table->timestamps();
});
```

### 2. Tabla `categories` y `products` (Relación 1 a Muchos)
```php
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('nombre', 100);
    $table->string('slug', 120)->unique(); // Ej: "tarjetas-graficas"
    $table->boolean('activa')->default(true);
    $table->timestamps();
});

Schema::create('products', function (Blueprint $table) {
    $table->id();
    // ¿A qué categoría pertenece?
    $table->foreignId('category_id')->constrained()->cascadeOnDelete();
    // ¿Qué vendedor (User) publicó este producto?
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    
    $table->string('nombre');
    $table->string('sku', 50)->unique(); // Código único de inventario
    $table->text('descripcion')->nullable();
    $table->decimal('precio', 10, 2);
    $table->unsignedInteger('stock')->default(0);
    $table->boolean('destacado')->default(false);
    $table->json('especificaciones')->nullable(); // Ej: {"marca": "ASUS", "garantia_meses": 24}
    $table->string('imagen_url')->nullable();
    $table->string('codigo_interno_proveedor')->nullable(); // Lo ocultaremos en el JSON con $hidden
    
    $table->timestamps();
    $table->softDeletes(); // Activa la papelera de reciclaje (deleted_at)
});
```

### 3. Tabla `orders` y Tabla Pivote `order_product` (Relación Muchos a Muchos)
Un pedido (`Order`) puede incluir varios productos, y un producto puede estar en cientos de pedidos distintos. Por eso necesitamos una **tabla intermedia (pivote)** llamada `order_product`:

```php
Schema::create('orders', function (Blueprint $table) {
    $table->id();
    // Cliente que hizo la compra
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->decimal('total', 10, 2)->default(0);
    $table->enum('estado', ['pendiente', 'pagado', 'enviado', 'cancelado'])->default('pendiente');
    $table->timestamp('pagado_en')->nullable();
    $table->timestamps();
});

// TABLA INTERMEDIA (PIVOTE): Conecta orders con products
Schema::create('order_product', function (Blueprint $table) {
    $table->id();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->foreignId('product_id')->constrained()->cascadeOnDelete();
    
    // Datos extra de esa compra en particular:
    $table->unsignedInteger('cantidad');
    $table->decimal('precio_unitario', 10, 2); // Guardamos cuánto costaba el día que lo compró
    $table->timestamps();
});
```

### 4. Tabla `reviews` (Relación Polimórfica)
En TechStore queremos que los clientes puedan dejar reseñas con estrellas tanto a un **Producto** (`Product`) como a un **Vendedor** (`User`). En lugar de crear dos tablas de reseñas distintas, usamos `morphs('reviewable')`, que crea dos columnas mágicas (`reviewable_type` y `reviewable_id`):

```php
Schema::create('reviews', function (Blueprint $table) {
    $table->id();
    // Autor de la reseña
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    
    // Crea reviewable_id (ej: 5) y reviewable_type (ej: "App\Models\Product" o "App\Models\User")
    $table->morphs('reviewable');
    
    $table->unsignedTinyInteger('calificacion'); // De 1 a 5 estrellas
    $table->text('comentario')->nullable();
    $table->timestamps();
});
```

---

## 5. Factories: La fábrica de datos falsos realistas

Probar una API con la base de datos vacía es aburridísimo. Para eso existen las **Factories** (`database/factories/`) junto con una librería incluida llamada **Faker**, que inventa nombres, correos, precios y textos de mentira pero muy realistas.

Recuerdas que en tu captura aparecía `use HasFactory;` dentro del Modelo? Esa línea le da al modelo el superpoder `Product::factory()` para usar esta fábrica.

Veamos `database/factories/ProductFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Category;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class ProductFactory extends Factory
{
    public function definition(): array
    {
        return [
            // Si no le pasamos categoría o vendedor, crea uno automáticamente
            'category_id' => Category::factory(),
            'user_id' => User::factory(),
            
            'nombre' => fake()->words(3, true), // Ej: "Monitor Curvo Ultra"
            'sku' => strtoupper(Str::random(8)), // Ej: "X9F2K8L1"
            'descripcion' => fake()->paragraph(),
            'precio' => fake()->randomFloat(2, 10, 1500), // Precio entre 10.00 y 1500.00
            'stock' => fake()->numberBetween(0, 100),
            'destacado' => fake()->boolean(20), // 20% de probabilidad de ser destacado
            'especificaciones' => [
                'marca' => fake()->randomElement(['Sony', 'Samsung', 'Logitech', 'ASUS']),
                'garantia_meses' => fake()->randomElement([12, 24, 36]),
            ],
            'codigo_interno_proveedor' => 'PROV-' . fake()->numerify('####'),
        ];
    }
}
```

---

## 6. Seeders: Sembrando TechStore con un solo comando

El archivo `database/seeders/DatabaseSeeder.php` es el encargado de organizar la creación de tus datos iniciales usando tus Modelos y tus Factories:

```php
<?php

namespace Database\Seeders;

use App\Models\Category;
use App\Models\Product;
use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // 1. Creamos nuestro usuario Administrador fijo para poder entrar siempre desde Postman
        $admin = User::create([
            'name' => 'Admin TechStore',
            'email' => 'admin@techstore.com',
            'password' => Hash::make('password123'),
            'role' => 'admin',
        ]);

        // 2. Creamos un Vendedor de prueba con su Perfil
        $vendedor = User::create([
            'name' => 'Carlos Vendedor',
            'email' => 'vendedor@techstore.com',
            'password' => Hash::make('password123'),
            'role' => 'vendedor',
        ]);

        $vendedor->profile()->create([
            'telefono' => '+34 600 123 456',
            'direccion' => 'Calle Mayor 10, Madrid',
            'cuenta_bancaria_cifrada' => 'ES91 2100 0418 4502 0005 1332',
        ]);

        // 3. Creamos 3 Categorías reales y, por cada una, fabricamos 10 Productos del vendedor
        $categorias = ['Portátiles', 'Periféricos', 'Componentes PC'];

        foreach ($categorias as $nombreCat) {
            $categoria = Category::create([
                'nombre' => $nombreCat,
                'slug' => \ Illuminate\Support\Str::slug($nombreCat),
                'activa' => true,
            ]);

            // ¡La magia de HasFactory! Crea 10 productos para esta categoría y este vendedor
            Product::factory()->count(10)->create([
                'category_id' => $categoria->id,
                'user_id' => $vendedor->id,
            ]);
        }
    }
}
```

Cuando ejecutas en tu terminal:
```bash
php artisan migrate:fresh --seed
```
En menos de 2 segundos tendrás tus tablas creadas, 1 administrador, 1 vendedor con perfil, 3 categorías y **30 productos completos** listos para ser consultados en tu API.

---

👉 **Siguiente paso:** Ahora que las tablas existen, pasemos al plato fuerte: **[Módulo 3: Modelos a Fondo, Propiedades ($fillable, $casts, $hidden...) y Traits](./03_MODELOS_A_FONDO_PROPIEDADES_Y_TRAITS.md)**, donde destriparemos línea por línea todo lo que va dentro de un Modelo.
