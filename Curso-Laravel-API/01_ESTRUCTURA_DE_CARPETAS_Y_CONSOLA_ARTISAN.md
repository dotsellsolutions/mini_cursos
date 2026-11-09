# Módulo 1: Estructura Completa de Carpetas y la Consola Artisan

Cuando instalas Laravel por primera vez para crear **TechStore API**, verás unas 10 carpetas y varios archivos sueltos. Al principio impone respeto, pero en realidad es como los cajones de una cocina bien ordenada: cada cosa tiene su sitio exacto.

---

## Parte 1: Radiografía de Todas las Carpetas de Laravel

Aquí tienes el mapa completo de un proyecto Laravel y qué harás (o no harás) en cada carpeta durante el desarrollo de **TechStore API**:

```
techstore-api/
├── 📁 app/                  ⭐ (Donde pasarás el 80% de tu tiempo)
├── 📁 bootstrap/            ⚙️ (Arranque del sistema y configuración global)
├── 📁 config/               🎛️ (Interruptores y ajustes de todo Laravel)
├── 📁 database/             🗄️ (Planos de tus tablas y datos de prueba)
├── 📁 public/               🚪 (La puerta de entrada pública e imágenes subidas)
├── 📁 resources/            🎨 (Plantillas de correos o vistas; casi no se usa en APIs)
├── 📁 routes/               🗺️ (Las direcciones URL de tu API)
├── 📁 storage/              📦 (Archivos subidos, facturas PDF y registros de errores)
├── 📁 tests/                🧪 (Pruebas automáticas para verificar que todo funcione)
├── 📁 vendor/               🚫 (Código interno de Laravel y librerías — ¡NUNCA TOCAR!)
├── 📄 .env                  🔑 (Tus contraseñas y claves secretas locales)
├── 📄 artisan               🪄 (El asistente mágico de comandos por terminal)
└── 📄 composer.json         📋 (La lista de paquetes instalados en tu proyecto)
```

---

### 1. Carpeta `app/` (El corazón de tu aplicación)
Aquí vive toda la lógica de negocio de **TechStore API**. Dentro de `app/` encontrarás (o irás creando con comandos) estas subcarpetas:

* **`app/Models/`**: Aquí viven tus **Modelos** (`User.php`, `Product.php`, `Category.php`, `Order.php`). Cada archivo representa una tabla de tu base de datos y contiene sus relaciones y configuraciones.
* **`app/Http/`**: Todo lo que tiene que ver con recibir peticiones de internet y devolver respuestas JSON:
  * **`app/Http/Controllers/`**: Tus controladores (`ProductController.php`, `AuthController.php`). Reciben el pedido del usuario, hablan con el Modelo y devuelven el JSON.
  * **`app/Http/Middleware/`**: Los "guardias de seguridad" (`CheckRole.php`) que revisan la petición antes de que llegue al controlador (por ejemplo: ¿tiene sesión iniciada? ¿es vendedor?).
  * **`app/Http/Requests/`**: Tus validadores de formularios (`StoreProductRequest.php`). Separan las reglas de validación para que el controlador no se llene de código.
  * **`app/Http/Resources/`**: Tus maquilladores de JSON (`ProductResource.php`). Le dan forma exacta a los datos antes de enviarlos al cliente.
* **`app/Notifications/`**: Las alertas que envía tu API (`OrderCreatedNotification.php`), ya sea por email, SMS o a la campanita de notificaciones de la base de datos.
* **`app/Jobs/`**: Las tareas pesadas que se ejecutan en segundo plano (`GenerateInvoicePdfJob.php`) para no hacer esperar al usuario.
* **`app/Policies/`**: Las reglas de autorización fina (`ProductPolicy.php`), donde decides cosas como *"Un vendedor solo puede editar un producto si él mismo lo creó"*.
* **`app/Observers/`**: Vigilantes automáticos de tus modelos (`OrderObserver.php`), por ejemplo: *"Cuando se cree un pedido, descuenta el stock del producto automáticamente"*.
* **`app/Console/Commands/`**: Tus propios comandos de terminal (por ejemplo, un robot que cada medianoche cancela los pedidos de TechStore que no fueron pagados).
* **`app/Providers/`**: Contiene `AppServiceProvider.php`. Es el lugar donde configuras comportamientos globales al arrancar tu aplicación.

---

### 2. Carpeta `bootstrap/` (El motor de arranque)
* **¿Qué hace?** Prepara a Laravel para encenderse en cada petición.
* **Lo único que tocarás aquí:** En las versiones modernas de Laravel (Laravel 11 y 12), el archivo **`bootstrap/app.php`** es súper importante porque ahí se registran los **Middlewares con alias** y el **manejo global de errores JSON**. *(En Laravel 10 o anteriores, eso se hacía en `app/Http/Kernel.php` y `app/Exceptions/Handler.php`)*.

---

### 3. Carpeta `config/` (El panel de control)
Contiene archivos llenos de opciones para cada parte de Laravel:
* `config/app.php`: Nombre de tu app, zona horaria (`timezone`), idioma (`locale`).
* `config/auth.php`: Configuración de quiénes inician sesión.
* `config/cors.php`: Configuración para permitir que tu frontend en React/Vue/Angular pueda conectarse a tu API sin ser bloqueado por el navegador.
* `config/database.php`: Configuración de MySQL, PostgreSQL, SQLite, etc.
* `config/filesystems.php`: Dónde se guardan las fotos de los productos (en tu servidor local o en la nube como Amazon S3).
* `config/sanctum.php`: Ajustes de los tokens de seguridad de tu API.

> **Regla de oro:** Casi nunca editas los archivos de `config/` directamente; ellos leen los valores desde tu archivo `.env`.

---

### 4. Carpeta `database/` (Tu almacén de datos)
Se divide en tres subcarpetas vitales:
* **`database/migrations/`**: Los planos con fecha y hora que crean o modifican las tablas de TechStore (`create_products_table.php`).
* **`database/factories/`**: "Fábricas" de datos falsos (`ProductFactory.php`). Le dicen a Laravel cómo inventar nombres de productos, precios y descripciones realistas para hacer pruebas.
* **`database/seeders/`**: Los "sembradores" (`DatabaseSeeder.php`). Llenan tu base de datos inicial (por ejemplo, crean el usuario Administrador por defecto y las 5 categorías base de TechStore).

---

### 5. Carpeta `public/` (La única ventana al exterior)
* Por seguridad, ningún visitante puede entrar a tus carpetas `app/` o `database/`. Todo el tráfico de internet entra por un único archivo: **`public/index.php`**.
* Aquí también se crea un acceso directo llamado `public/storage` para que las fotos de los productos de TechStore se puedan ver desde el navegador o la app móvil.

---

### 6. Carpeta `resources/`
* En proyectos web clásicos aquí van las vistas HTML (Blade), hojas de estilo CSS y archivos JS.
* **En una API:** Solo la llegarás a usar si quieres personalizar el diseño del correo electrónico que le llega al cliente cuando compra en TechStore.

---

### 7. Carpeta `routes/` (El mapa de direcciones)
* **`routes/api.php`**: **¡Tu archivo estrella!** Todas las rutas que pongas aquí tendrán automáticamente el prefijo `/api` (ej. `http://localhost:8000/api/productos`) y están pensadas para recibir y devolver JSON. *(Si en Laravel 11+ no ves `api.php` recién instalado, aparece al ejecutar `php artisan install:api`, lo cual además te instala Laravel Sanctum automáticamente)*.
* **`routes/web.php`**: Rutas para páginas web tradicionales con sesiones y vistas.
* **`routes/console.php`**: Donde programas tareas automáticas por horario (ej. *"Ejecuta este comando todos los días a las 03:00 AM"*).

---

### 8. Carpeta `storage/` (El depósito y diario de errores)
* **`storage/app/public/`**: Donde se guardan realmente las fotos de los productos o facturas en PDF que suben o generan los usuarios.
* **`storage/logs/laravel.log`**: **¡Tu mejor amigo cuando algo falla!** Si tu API devuelve un error 500 y no sabes por qué, abres `laravel.log` y al final del archivo verás la línea exacta que falló.

---

### 9. Carpeta `tests/` (El laboratorio de calidad)
* Donde escribes pequeñas pruebas automáticas que simulan a un usuario comprando un producto en TechStore para comprobar que no rompiste nada antes de subir cambios a internet.

---

### 10. Carpeta `vendor/` y archivos raíz
* **`vendor/`**: Contiene el código fuente del propio Laravel y de todas las librerías descargadas. **Jamás modifiques nada dentro de `vendor/`**, porque se sobrescribe cuando actualizas paquetes.
* **`.env`**: Archivo secreto con la conexión a tu base de datos (`DB_DATABASE=techstore`), las contraseñas de correo y la llave maestra de cifrado (`APP_KEY`). **Nunca se sube a GitHub.**
* **`composer.json`**: La lista de ingredientes e instalaciones de tu proyecto PHP.

---

## Parte 2: La Consola Artisan Bien Explicada

**Artisan** es el robot ayudante de Laravel. En lugar de crear carpetas y archivos a mano (y cometer errores al escribir los nombres o los `namespace`), le pides a Artisan que los cree por ti con la estructura perfecta.

Todos los comandos se escriben en tu terminal situándote dentro de la carpeta del proyecto:

```bash
php artisan nombre-del-comando
```

### 1. Los comandos del día a día que usarás en TechStore API

#### A) Comandos para arrancar y explorar
* **`php artisan serve`**: Enciende un servidor local rápido (normalmente en `http://127.0.0.1:8000`) para probar tu API.
* **`php artisan install:api`**: (En Laravel 11+) Prepara tu proyecto para ser una API pura: crea `routes/api.php` e instala **Laravel Sanctum**.
* **`php artisan route:list`**: Te muestra una tabla preciosa en la terminal con **todas las rutas de tu API**, qué método usan (`GET`, `POST`, `DELETE`), a qué controlador van y qué middlewares las protegen.
  * *Truco pro:* `php artisan route:list --path=api` te muestra solo las rutas de tu API.

#### B) Comandos `make:` (Crear piezas de código al instante)
* **`php artisan make:model Product`**: Crea el modelo `app/Models/Product.php`.
* **🔥 El comando "Todo en Uno" más usado por profesionales:**
  ```bash
  php artisan make:model Product -mfs --api
  ```
  ¿Qué significan esas letras (`-mfs --api`)? De un solo golpe te crea:
  * El Modelo (`Product.php`)
  * **`-m`** (Migration): La migración para crear la tabla `products`.
  * **`-f`** (Factory): La fábrica de productos falsos `ProductFactory.php`.
  * **`-s`** (Seeder): El sembrador `ProductSeeder.php`.
  * **`--api`**: El controlador `ProductController.php` limpio, listo para API (sin funciones de vistas HTML).
* **`php artisan make:controller Api/AuthController`**: Crea un controlador dentro de una subcarpeta `Api`.
* **`php artisan make:middleware CheckRole`**: Crea un guardia de seguridad en `app/Http/Middleware/CheckRole.php`.
* **`php artisan make:request StoreProductRequest`**: Crea un validador de datos en `app/Http/Requests/`.
* **`php artisan make:resource ProductResource`**: Crea un formateador de JSON en `app/Http/Resources/`.
* **`php artisan make:notification NewOrderNotification`**: Crea una notificación en `app/Notifications/`.
* **`php artisan make:policy ProductPolicy --model=Product`**: Crea las reglas de permisos para el modelo `Product`.
* **`php artisan make:observer OrderObserver --model=Order`**: Crea un vigilante automático para el modelo `Order`.
* **`php artisan make:job ProcessInvoiceJob`**: Crea una tarea en segundo plano.

#### C) Comandos de Base de Datos (`migrate`)
* **`php artisan migrate`**: Lee tus archivos de migraciones nuevos y crea las tablas en la base de datos.
* **`php artisan migrate:rollback`**: Deshace el último grupo de migraciones (por si te equivocaste en una columna).
* **`php artisan migrate:fresh --seed`**: **¡El favorito mientras estás desarrollando!** Borra todas las tablas, las vuelve a crear desde cero limpias y ejecuta los *Seeders* para llenarla con tus datos de prueba. *(¡Nunca lo uses en producción con clientes reales porque borra todo!)*.

#### D) El comando salvavidas cuando Laravel "no hace caso"
A veces cambias algo en el archivo `.env` o agregas una ruta y parece que Laravel no se entera porque guardó la versión anterior en memoria caché. Ejecuta esto y limpia todo de golpe:
```bash
php artisan optimize:clear
```

---

### 2. `php artisan tinker`: Tu laboratorio en vivo

Imagina que acabas de crear el modelo `Product` y sus relaciones, pero todavía no has creado ni rutas ni controladores. ¿Cómo pruebas si funciona sin abrir Postman ni escribir rutas?

Ejecutas en tu terminal:
```bash
php artisan tinker
```
Se abrirá una consola interactiva donde puedes hablar directamente con tu proyecto de Laravel en vivo:

```php
// Crear un producto en vivo desde la terminal:
> App\Models\Product::create(['nombre' => 'Mouse Gamer', 'precio' => 29.99, 'stock' => 15]);

// Consultar cuántos productos hay en TechStore:
> App\Models\Product::count();
= 1

// Ver el primer usuario y sus pedidos:
> $user = App\Models\User::with('orders')->first();
```
Para salir de Tinker solo escribes `exit`.

---

### 3. Cómo crear tu propio Comando Artisan personalizado (con parámetros)

En **TechStore API**, imagina que cuando un cliente hace un pedido pero pasan **24 horas sin pagarlo**, queremos cancelar ese pedido automáticamente y devolver el stock a la tienda.

#### Paso 1: Crear el comando
```bash
php artisan make:command CancelarPedidosVencidos
```
Esto crea el archivo `app/Console/Commands/CancelarPedidosVencidos.php`.

#### Paso 2: Programar qué hace y cómo recibe parámetros
Abrimos ese archivo y lo dejamos así de claro:

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\Order;

class CancelarPedidosVencidos extends Command
{
    /**
     * 1. El nombre con el que llamaremos al comando en la terminal.
     * {horas=24} es un argumento con valor por defecto 24.
     * {--notificar} es una opción opcional (bandera).
     */
    protected $signature = 'techstore:limpiar-pedidos {horas=24} {--notificar}';

    /**
     * 2. La descripción que sale cuando escribes "php artisan list"
     */
    protected $description = 'Cancela los pedidos pendientes de pago que superen las horas indicadas';

    /**
     * 3. Lo que se ejecuta cuando lanzas el comando
     */
    public function handle()
    {
        // Leemos el parámetro {horas} que escribió el usuario (o 24 por defecto)
        $horasLimite = $this->argument('horas');

        // Leemos si activaron la opción --notificar (true o false)
        $debeNotificar = $this->option('notificar');

        $this->info("Buscando pedidos sin pagar con más de {$horasLimite} horas...");

        // Buscamos con Eloquent los pedidos pendientes viejos
        $pedidosVencidos = Order::where('estado', 'pendiente')
            ->where('created_at', '<=', now()->subHours($horasLimite))
            ->get();

        if ($pedidosVencidos->isEmpty()) {
            $this->comment('No hay pedidos vencidos hoy. ¡Todo limpio!');
            return;
        }

        foreach ($pedidosVencidos as $pedido) {
            $pedido->update(['estado' => 'cancelado']);

            if ($debeNotificar) {
                // Aquí podríamos enviar una notificación al cliente
                $this->line("Pedido #{$pedido->id} cancelado y cliente notificado.");
            } else {
                $this->line("Pedido #{$pedido->id} cancelado.");
            }
        }

        $this->info("¡Proceso terminado! Se cancelaron {$pedidosVencidos->count()} pedidos.");
    }
}
```

#### Paso 3: Cómo usarlo manualmente o dejarlo en piloto automático
* **Para ejecutarlo tú mismo en la terminal:**
  * Con las 24 horas por defecto: `php artisan techstore:limpiar-pedidos`
  * Pasándole 48 horas y pidiendo que notifique: `php artisan techstore:limpiar-pedidos 48 --notificar`

* **Para que se ejecute solo todas las noches (Scheduler):**
  Abres `routes/console.php` y agregas una sola línea:
  ```php
  use Illuminate\Support\Facades\Schedule;

  // Se ejecutará automáticamente todos los días a la medianoche (00:00)
  Schedule::command('techstore:limpiar-pedidos 24 --notificar')->daily();
  ```

---

👉 **Siguiente paso:** En el **[Módulo 2: Migraciones, Factories y Seeders](./02_MIGRACIONES_FACTORIES_Y_SEEDERS.md)** vamos a construir todas las tablas de **TechStore API** y llenaremos nuestra tienda de datos de prueba en segundos.
