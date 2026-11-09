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

### B) Modificadores (se encadenan al final de la columna)
* **`->nullable()`**: Permite que el campo quede vacío (`NULL`). Si no lo pones, el campo será obligatorio en la base de datos.
* **`->default('cliente')`**: Si nadie envía un valor, pone este por defecto.
* **`->unique()`**: Prohibido repetir el mismo valor en dos filas (ideal para `email` o `sku` del producto).
* **`->index()`**: Crea un índice de búsqueda rápida (ponlo en columnas por las que busques mucho, como `nombre`).

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

## 3. Creando las Migraciones Reales de TechStore API

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

## 4. Factories: La fábrica de datos falsos realistas

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

## 5. Seeders: Sembrando TechStore con un solo comando

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
