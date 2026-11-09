# Módulo 8: Notificaciones, Colas (`Jobs`), Archivos (`Storage`), Cliente HTTP y Tiempo Real

En este último módulo vamos a dotar a **TechStore API** de las funciones que diferencian a un proyecto escolar de una plataforma comercial lista para el mundo real.

---

## 1. Las Notificaciones a Fondo (Correo + Campanita de Base de Datos)

Cuando un cliente paga un pedido en **TechStore API**, queremos avisarle de **dos formas al mismo tiempo**:
1. Enviándole un **Correo Electrónico** con el resumen de su compra.
2. Guardando una alerta en la **Campanita de Notificaciones** dentro de la app móvil o web para que le salga un circulito rojo con un `1` de *"Tienes una notificación sin leer"*.

Con las **Notificaciones de Laravel**, programas ambas cosas en un solo archivo.

### Paso 1: Crear la tabla de notificaciones y la clase
Ejecutas en tu terminal:
```bash
php artisan make:notifications-table
php artisan migrate
php artisan make:notification PedidoPagadoNotification
```

### Paso 2: Configurar `app/Notifications/PedidoPagadoNotification.php`
Abrimos la notificación creada y le decimos por qué canales debe enviarse (`mail` y `database`):

```php
<?php

namespace App\Notifications;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

// Al agregar "implements ShouldQueue", la notificación se envía en segundo plano sin hacer esperar al usuario
class PedidoPagadoNotification extends Notification implements ShouldQueue
{
    use Queueable;

    // Recibimos el pedido recién pagado
    public function __construct(
        public Order $order
    ) {}

    /**
     * 1. ¿Por qué canales queremos enviar esta notificación?
     * 'mail' = Correo electrónico
     * 'database' = Guarda un JSON en la tabla 'notifications' para la campanita de la API
     */
    public function via(object $notifiable): array
    {
        return ['mail', 'database'];
    }

    /**
     * 2. Diseño del Correo Electrónico que recibe el cliente
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject("¡Confirmación de tu Pedido #{$this->order->id} en TechStore!")
            ->greeting("¡Hola, {$notifiable->name}!")
            ->line("Hemos recibido correctamente tu pago por un total de \${$this->order->total}.")
            ->line('Ya estamos preparando tus productos para el envío.')
            ->action('Ver estado de mi pedido', url("https://mitechstore.com/pedidos/{$this->order->id}"))
            ->salutation('¡Gracias por confiar en TechStore!');
    }

    /**
     * 3. Datos que se guardan en la columna 'data' de la tabla 'notifications' (Para tu API)
     */
    public function toArray(object $notifiable): array
    {
        return [
            'order_id' => $this->order->id,
            'titulo' => 'Pago confirmado',
            'mensaje' => "Tu pedido #{$this->order->id} por \${$this->order->total} está en preparación.",
            'icono' => 'check-circle',
        ];
    }
}
```

### Paso 3: Cómo disparar la notificación desde tu Controlador
Gracias al Trait **`use Notifiable;`** que tiene el modelo `User`, enviarla cuesta **una sola línea**:
```php
// En OrderController después de confirmar el pago:
$pedido->user->notify(new PedidoPagadoNotification($pedido));
```

---

### Paso 4: Crear los Endpoints de la "Campanita de Notificaciones" en tu API
¿Cómo hace el frontend (React, Vue, Flutter) para mostrarle al usuario sus notificaciones guardadas en la base de datos y marcarlas como leídas cuando hace clic en ellas?

Creamos 3 rutas súper sencillas en `NotificationController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class NotificationController extends Controller
{
    // 1. GET /api/notifications (Ver mis notificaciones y cuántas tengo sin leer)
    public function index(Request $request)
    {
        $user = $request->user();

        return response()->json([
            'sin_leer_total' => $user->unreadNotifications()->count(),
            'no_leidas' => $user->unreadNotifications,
            'ultimas_todas' => $user->notifications()->take(20)->get(),
        ]);
    }

    // 2. PATCH /api/notifications/{id}/read (Marcar 1 notificación como leída al tocarla)
    public function marcarComoLeida(Request $request, string $id)
    {
        $notificacion = $request->user()->notifications()->findOrFail($id);

        // Pone la fecha actual en la columna 'read_at'
        $notificacion->markAsRead();

        return response()->json(['mensaje' => 'Notificación marcada como leída.']);
    }

    // 3. POST /api/notifications/read-all (Botón "Marcar todas como leídas")
    public function marcarTodasComoLeidas(Request $request)
    {
        $request->user()->unreadNotifications->markAsRead();

        return response()->json(['mensaje' => 'Todas las notificaciones fueron leídas.']);
    }
}
```

---

## 2. Colas de Trabajo (`Jobs` & `Queues`): APIs Ultrarrápidas

Imagina que cuando el cliente compra en **TechStore**, tu servidor tiene que:
1. Generar una factura en PDF con código QR.
2. Avisar al sistema de logística del almacén.
3. Enviar el correo de confirmación.

Hacer todo eso tarda unos **4 segundos**. Si el usuario tiene que esperar 4 segundos mirando una rueda de carga en su celular, la experiencia es mala.

### La Solución: Crear un `Job` (Trabajo en Segundo Plano)
```bash
php artisan make:job GenerarFacturaPdfJob
```

En `app/Jobs/GenerarFacturaPdfJob.php`:
```php
<?php

namespace App\Jobs;

use App\Models\Order;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Support\Facades\Log;

class GenerarFacturaPdfJob implements ShouldQueue
{
    use Queueable;

    // Si falla (ej. se cortó internet un segundo), reinténtalo hasta 3 veces automáticamente
    public int $tries = 3;

    public function __construct(
        public Order $order
    ) {}

    public function handle(): void
    {
        // Aquí va la tarea pesada que tarda varios segundos
        Log::info("Generando factura PDF en segundo plano para el pedido #{$this->order->id}...");
    }
}
```

**Cómo lo lanzas en el Controlador:**
```php
// Lo pone en la fila de tareas y continúa en 0.001 segundos:
GenerarFacturaPdfJob::dispatch($pedido);

return response()->json(['mensaje' => '¡Compra realizada al instante!'], 201);
```
Y para que tu servidor procese esa fila de tareas de fondo continuamente, ejecutas en una terminal:
```bash
php artisan queue:work
```

---

## 3. Subida de Imágenes y Archivos por API (`Storage`)

Cuando un vendedor crea un producto en **TechStore API**, quiere subir la foto del producto.
> **Recuerda:** En Postman o en tu frontend, cuando envías archivos no usas JSON puro, sino **`multipart/form-data`**.

### Paso 1: Crear el puente público (`storage:link`)
Por seguridad, los archivos que se suben van a `storage/app/public`, que está oculta de internet. Para que las fotos se puedan ver desde una URL pública, ejecutas **una sola vez en la vida del proyecto**:
```bash
php artisan storage:link
```
Esto conecta `public/storage` con `storage/app/public`.

### Paso 2: Guardar, actualizar y borrar la foto en `ProductController.php`
```php
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\Storage;

public function store(StoreProductRequest $request)
{
    // Quitamos 'imagen' (es el archivo subido, no una columna) y asignamos el vendedor dueño del producto
    $datos = $request->safe()->except('imagen');
    $datos['user_id'] = $request->user()->id;

    // ¿El vendedor adjuntó un archivo en el campo 'imagen'?
    if ($request->hasFile('imagen')) {
        // Guarda la foto con un nombre único en storage/app/public/productos
        // Devuelve la ruta: "productos/a8f9k2l1.webp"
        $rutaImagen = $request->file('imagen')->store('productos', 'public');
        $datos['imagen_url'] = $rutaImagen;
    }

    $producto = Product::create($datos);

    return (new ProductResource($producto))->response()->setStatusCode(201);
}

// ¿Y si al editar el producto sube una foto nueva? Borramos la foto vieja para no llenar el disco:
public function update(StoreProductRequest $request, Product $product)
{
    Gate::authorize('update', $product); // Solo el dueño o un admin (Policy del Módulo 7)

    $datos = $request->safe()->except('imagen');

    if ($request->hasFile('imagen')) {
        // 1. Si ya tenía una foto anterior, la borramos del disco
        if ($product->imagen_url) {
            Storage::disk('public')->delete($product->imagen_url);
        }

        // 2. Guardamos la foto nueva
        $datos['imagen_url'] = $request->file('imagen')->store('productos', 'public');
    }

    $product->update($datos);
    return new ProductResource($product);
}
```
*(Lo mejor de usar `Storage::disk(...)`: si mañana tu tienda crece y quieres guardar las fotos en **Amazon S3** o **Cloudflare R2**, no cambias tu código, solo cambias `FILESYSTEM_DISK=s3` en tu archivo `.env`).*

---

## 4. Cliente HTTP de Laravel (`Http`): Conectando TechStore con otras APIs

A menudo tu API de Laravel también tiene que pedirle datos a **otras APIs externas** (como cobrar una tarjeta en Stripe/PayPal o consultar el precio del dólar en tiempo real).
Laravel incluye la fachada **`Http`** para hacerlo súper fácil:

```php
use Illuminate\Support\Facades\Http;

public function procesarPagoExterno(Order $pedido, string $tokenTarjeta)
{
    // Llamamos a una pasarela de pagos externa enviando Token y un JSON:
    $respuesta = Http::withToken(config('services.pasarela.secret_key'))
        ->timeout(10) // Si tarda más de 10 segundos, cancela para no colgar nuestra API
        ->post('https://api.pasareladepagos.com/v1/charges', [
            'amount' => $pedido->total,
            'currency' => 'USD',
            'source' => $tokenTarjeta,
            'description' => "Pedido #{$pedido->id} en TechStore",
        ]);

    // Comprobamos si la API externa respondió con éxito (código 200-299)
    if ($respuesta->successful()) {
        $datosPago = $respuesta->json(); // Convierte la respuesta externa en array de PHP
        $pedido->update(['estado' => 'pagado', 'pagado_en' => now()]);
        return true;
    }

    return false;
}
```

---

## 5. Manejo Global de Errores en JSON (`bootstrap/app.php`)

Nada queda peor en una API que buscar un producto que no existe (`/api/products/999999`) y que Laravel devuelva un mensaje técnico en inglés o una página HTML si el cliente olvidó enviar la cabecera `Accept: application/json`.

En `bootstrap/app.php` puedes personalizar cómo responde tu API ante cualquier error en todo el proyecto:

```php
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Illuminate\Http\Request;

->withExceptions(function (Exceptions $exceptions) {
    // 1. Forzar que cualquier ruta que empiece por api/* SIEMPRE devuelva errores en JSON (nunca HTML)
    $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
        return $request->is('api/*') || $request->expectsJson();
    });

    // 2. Personalizar el mensaje cuando no se encuentra un Producto, Pedido o Ruta (Error 404)
    $exceptions->render(function (NotFoundHttpException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'exito' => false,
                'mensaje' => 'El recurso o registro que buscas no existe en TechStore.',
            ], 404);
        }
    });
})
```

---

## 6. Tiempo Real (WebSockets / Broadcasting con Laravel Reverb)

¿Cómo hace Uber Eats o un panel de administración moderno para que, en el mismo instante en que un cliente paga un pedido, **suene una campanita en la pantalla del vendedor sin recargar la página**?

Eso se llama **Broadcasting (Eventos en Tiempo Real por WebSockets)** y Laravel tiene su propio servidor oficial llamado **Laravel Reverb** (`php artisan install:broadcasting`).

Funciona añadiendo `implements ShouldBroadcast` a un Evento de Laravel:

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class NuevoPedidoEnVivoEvent implements ShouldBroadcast
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public Order $order
    ) {}

    /**
     * Canal privado seguro donde transmitimos el aviso en tiempo real
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('panel-ventas-techstore'),
        ];
    }
}
```
Cuando en tu controlador ejecutas `NuevoPedidoEnVivoEvent::dispatch($pedido);`, el panel del vendedor recibe el pedido en vivo en menos de 50 milisegundos.

---

## 🎓 Resumen Final: El Flujo Completo de una Petición en TechStore API

Para cerrar este bloque, mira cómo encajan **todas las piezas que has aprendido** desde que el cliente pulsa "Comprar" en su celular hasta que recibe el JSON de confirmación:

```
1. 📱 El Celular envía: POST /api/orders (con Header "Authorization: Bearer 1|xyz..." y "Accept: application/json")
       ⬇
2. 🌐 CORS (config/cors.php) verifica que el origen esté permitido.
       ⬇
3. 🛡️ Rate Limiter (throttle) comprueba que no esté haciendo 1.000 compras por segundo.
       ⬇
4. 🔑 Middleware "auth:sanctum" verifica el Token e identifica al usuario ($request->user()).
       ⬇
5. 👮 Middleware con parámetros "rol:cliente,admin" comprueba su rol.
       ⬇
6. 📋 FormRequest (StoreOrderRequest) valida que los productos y cantidades sean reales (si no, devuelve 422 JSON).
       ⬇
7. 🧠 Controlador (OrderController) abre una Transacción (DB::transaction):
      ├── Usa el ORM Eloquent (Order, Product) con $fillable y $casts.
      ├── Guarda en la tabla pivote (attach) con BelongsToMany.
      └── El Observer o el Controlador descuenta el stock.
       ⬇
8. 🚀 Lanza en segundo plano (Queue) el Job de la factura y la Notificación (Correo + Campanita DB).
       ⬇
9. 🎨 API Resource (OrderResource) maquilla los datos ocultando lo privado ($hidden) y sumando calculados ($appends).
       ⬇
10. 📦 Devuelve al celular un JSON impecable con código "201 Created" en menos de 50 milisegundos.
```

👉 **Siguiente paso:** En el **[Módulo 9: Testing Automático de la API](./09_TESTING_AUTOMATICO_DE_LA_API.md)** aprenderás a escribir tests que comprueben todo este flujo en segundos cada vez que cambies el código, y en el **[Módulo 10](./10_BUENAS_PRACTICAS_Y_DESPLIEGUE_A_PRODUCCION.md)** a publicar TechStore API en un servidor real.
