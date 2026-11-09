# Módulo 7: Seguridad Total — Sanctum, Otros Tipos de Auth y los 6 Pilares de Security

En la captura que compartiste del menú oficial de Laravel aparecen los **6 Pilares de Seguridad (Security)**:
1. **Authentication** (Autenticación: *¿Quién eres?*)
2. **Authorization** (Autorización: *¿Qué tienes permiso de tocar?*)
3. **Email Verification** (Verificación de correo electrónico)
4. **Encryption** (Cifrado reversible con llave secreta)
5. **Hashing** (Huella irreversible para contraseñas)
6. **Password Reset** (Recuperación de contraseña olvidada)

Vamos a implementarlos todos paso a paso dentro de **TechStore API**, empezando por dominar **Laravel Sanctum** y todos los tipos de autenticación que existen para APIs.

---

## Pilar 1: Authentication (Autenticación en APIs)

### A) ¿Cómo funciona la Autenticación en una API? (La Pulsera del Hotel)
En una página web tradicional el navegador guarda una cookie automática. Pero en una API (que es usada por apps móviles en iOS/Android, Postman o frontends separados), usamos **Tokens (Bearer Tokens)**:
1. El usuario envía su correo y contraseña a `POST /api/login`.
2. Laravel comprueba que sean correctos y le entrega un **Token** (como la pulsera que te ponen en un hotel Todo Incluido: `1|a8f9k2l0x9...`).
3. A partir de ese momento, en cada petición a rutas privadas, el celular o frontend enseña su pulsera en la cabecera HTTP:
   ```http
   Authorization: Bearer 1|a8f9k2l0x9...
   ```

---

### B) Laravel Sanctum Paso a Paso en TechStore API

**Sanctum** es el sistema oficial de Laravel para autenticar APIs y aplicaciones móviles de forma ligera, rápida y segura.

#### Paso 1: Activarlo en el modelo `User.php`
Si ejecutaste `php artisan install:api`, Sanctum ya está instalado y creó la tabla `personal_access_tokens`. Solo debes asegurarte de que `app/Models/User.php` tenga el Trait **`HasApiTokens`**:

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
```

#### Paso 2: Crear el Controlador Completo de Autenticación (`AuthController.php`)
Creamos `app/Http/Controllers/AuthController.php` con **Registro**, **Login**, **Perfil actual** y **Logout**:

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class AuthController extends Controller
{
    // 1. REGISTRO DE NUEVO USUARIO (POST /api/register)
    public function register(Request $request)
    {
        $datos = $request->validate([
            'name' => ['required', 'string', 'max:100'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'string', 'min:8', 'confirmed'], // Exige enviar password_confirmation
        ]);

        $user = User::create([
            'name' => $datos['name'],
            'email' => $datos['email'],
            'password' => Hash::make($datos['password']),
            'role' => 'cliente',
        ]);

        // Creamos su token (su "pulsera") al instante para que ya quede dentro de la app
        $token = $user->createToken('token-registro')->plainTextToken;

        return response()->json([
            'mensaje' => 'Bienvenido a TechStore',
            'usuario' => $user,
            'token' => $token,
        ], 201);
    }

    // 2. INICIAR SESIÓN (POST /api/login)
    public function login(Request $request)
    {
        $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required', 'string'],
            'dispositivo' => ['nullable', 'string'], // Ej: "iPhone 15 de Juan"
        ]);

        // Buscamos al usuario por su email
        $user = User::where('email', $request->email)->first();

        // Comprobamos si existe y si la contraseña coincide con el Hash guardado
        if (! $user || ! Hash::check($request->password, $user->password)) {
            return response()->json([
                'mensaje' => 'El correo o la contraseña son incorrectos.'
            ], 401);
        }

        // Definimos qué permisos ("abilities") tendrá este token según su rol en TechStore
        $habilidades = match ($user->role) {
            'admin' => ['*'], // El asterisco significa: permiso para TODO
            'vendedor' => ['producto:crear', 'producto:editar', 'pedido:ver'],
            default => ['pedido:crear', 'pedido:ver'],
        };

        $nombreToken = $request->dispositivo ?? 'acceso-general';

        // Generamos el token con sus habilidades y una caducidad opcional de 7 días
        $token = $user->createToken($nombreToken, $habilidades, now()->addDays(7))->plainTextToken;

        return response()->json([
            'mensaje' => 'Sesión iniciada correctamente',
            'usuario' => $user,
            'token' => $token,
        ]);
    }

    // 3. VER MIS DATOS ACTUALES (GET /api/me)
    public function me(Request $request)
    {
        return response()->json($request->user()->load('profile'));
    }

    // 4. CERRAR SESIÓN EN ESTE DISPOSITIVO (POST /api/logout)
    public function logout(Request $request)
    {
        // Borra ÚNICAMENTE el token con el que hizo esta petición (deja abiertos sus otros dispositivos)
        $request->user()->currentAccessToken()->delete();

        return response()->json([
            'mensaje' => 'Sesión cerrada en este dispositivo.'
        ]);
    }

    // 5. CERRAR SESIÓN EN TODOS LOS DISPOSITIVOS A LA VEZ (POST /api/logout-all)
    public function logoutAll(Request $request)
    {
        // Borra TODOS los tokens del usuario (lo saca del celular, tablet y PC)
        $request->user()->tokens()->delete();

        return response()->json([
            'mensaje' => 'Se cerraron todas tus sesiones activas.'
        ]);
    }
}
```

#### Paso 3: Comprobar las "Habilidades" (*Abilities*) del Token de Sanctum
¿Viste que en el `login` le dimos permisos internos al token (`['producto:crear', 'producto:editar']`)?
Puedes comprobarlos en cualquier controlador así:
```php
if ($request->user()->tokenCan('producto:crear')) {
    // El token tiene permiso para crear productos
}
```
O directamente como Middleware en `routes/api.php` (registrando en `bootstrap/app.php` los alias `abilities` y `ability` de Sanctum):
```php
Route::post('/products', [ProductController::class, 'store'])
    ->middleware(['auth:sanctum', 'ability:producto:crear']);
```

---

### C) Todos los Demás Tipos de Autenticación en Laravel (Cuándo usar cada uno)

Además de los Tokens de Sanctum, existen otras 4 formas famosas de autenticar en el ecosistema Laravel. Aquí tienes la comparativa definitiva sin tecnicismos:

| Sistema de Auth | ¿Cómo funciona en palabras simples? | ¿Cuándo debes elegirlo? |
| :--- | :--- | :--- |
| **1. Sanctum (API Tokens)** | Guarda los tokens en la tabla `personal_access_tokens`. Puedes revocarlos/borrarlos cuando quieras. | **El mejor en el 90% de los casos:** Apps móviles (Flutter/React Native), APIs para terceros y frontends separados. |
| **2. Sanctum (SPA con Cookies)** | No usa tokens `Bearer`. Usa cookies seguras de sesión entre `techstore.com` y `api.techstore.com`. | Cuando tu API y tu web en React/Vue viven bajo el **mismo dominio** y no tienes app móvil. |
| **3. JWT (JSON Web Tokens)** | El token es un texto largo autoverificable que **no se guarda en la base de datos**. Lleva los datos del usuario codificados dentro. | Cuando tienes una arquitectura gigante de microservicios donde 10 servidores distintos necesitan verificar al usuario sin consultar la tabla de usuarios. |
| **4. Laravel Passport (OAuth2)** | Monta un servidor OAuth2 completo (como cuando una web dice *"Conectar con tu cuenta de Facebook"*). | Solo si quieres que **otras empresas externas** pongan en sus webs un botón *"Iniciar sesión con TechStore"*. |
| **5. Laravel Socialite** | Permite que tus usuarios entren a TechStore usando su cuenta de **Google, Apple, GitHub o Facebook**. | Cuando quieres ahorrarle al usuario el formulario de registro tradicional. |

#### Ejemplo práctico: ¿Cómo se usa Socialite (Login con Google) en una API?
1. El frontend de React o la App móvil abre la ventana de Google y obtiene el `access_token` de Google.
2. El frontend envía ese token a tu ruta `POST /api/auth/google`.
3. En tu controlador de Laravel usas Socialite en 4 líneas y le devuelves su Token de Sanctum de **TechStore**:
   ```php
   use Laravel\Socialite\Facades\Socialite;

   public function loginConGoogle(Request $request)
   {
       // 1. Laravel le pregunta a Google quién es el dueño de ese token
       $googleUser = Socialite::driver('google')->stateless()->userFromToken($request->google_token);

       // 2. Lo buscamos o lo creamos en nuestra tabla users de TechStore
       $user = User::firstOrCreate(
           ['email' => $googleUser->getEmail()],
           [
               'name' => $googleUser->getName(),
               'password' => Hash::make(\Illuminate\Support\Str::random(24)),
               'email_verified_at' => now(), // Google ya verificó su correo
           ]
       );

       // 3. Le entregamos nuestro Token de Sanctum
       return response()->json([
           'token' => $user->createToken('login-google')->plainTextToken,
           'usuario' => $user,
       ]);
   }
   ```

---

## Pilar 2: Authorization (Autorización: Gates y Policies)

Ya sabemos **quién eres** (*Authentication*) y con nuestro Middleware `rol:admin,vendedor` sabemos que eres un **vendedor**.
Pero surge un problema de seguridad grave en **TechStore**:
> *El Vendedor Carlos (`id: 2`) intenta editar o borrar un producto que pertenece al Vendedor Pedro (`id: 8`).*
> Ambos tienen el rol `'vendedor'`, así que el Middleware los deja pasar. ¿Cómo evitamos que un vendedor toque los productos de otro?

Con **Policies** (Políticas de autorización por modelo) y **Gates** (Puertas simples).

### A) Creando `ProductPolicy` para TechStore
```bash
php artisan make:policy ProductPolicy --model=Product
```
Abrimos `app/Policies/ProductPolicy.php`:

```php
<?php

namespace App\Policies;

use App\Models\Product;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class ProductPolicy
{
    /**
     * Regla VIP opcional: Si el usuario es 'admin', dale permiso a TODO sin preguntar más
     */
    public function before(User $user, string $ability): bool|null
    {
        if ($user->role === 'admin') {
            return true;
        }

        return null; // Si no es admin, pasa a revisar la función específica de abajo
    }

    /**
     * ¿Quién puede actualizar este producto concreto?
     */
    public function update(User $user, Product $product): Response
    {
        return $user->id === $product->user_id
            ? Response::allow()
            : Response::deny('No puedes modificar este producto porque pertenece a otro vendedor.');
    }

    /**
     * ¿Quién puede borrar este producto concreto?
     */
    public function delete(User $user, Product $product): Response
    {
        return $user->id === $product->user_id
            ? Response::allow()
            : Response::deny('Solo el creador del producto o un administrador puede eliminarlo.');
    }
}
```

### ¿Cómo se activa la Policy en el `ProductController`?
Con **una sola línea (`Gate::authorize`)** al principio de tu función:
```php
use Illuminate\Support\Facades\Gate;

public function update(StoreProductRequest $request, Product $product)
{
    // Si el producto no es suyo (ni es admin), frena aquí mismo y devuelve Error 403 JSON con tu mensaje
    Gate::authorize('update', $product);

    $product->update($request->validated());
    return new ProductResource($product);
}
```

---

## Pilar 3 y 4: Hashing vs Encryption (La Diferencia Vital)

Muchos confunden **Hashing** con **Encryption**. Entender la diferencia te evitará fallos de seguridad graves:

| Característica | **Hashing (`Hash`) — Irreversible** | **Encryption (`Crypt`) — Reversible** |
| :--- | :--- | :--- |
| **¿Se puede volver a obtener el texto original?** | **¡JAMÁS!** Es un camino de una sola dirección (como hacer un batido de frutas: no puedes volver a sacar la manzana entera). | **SÍ.** Quien tenga la llave maestra (`APP_KEY` de tu `.env`) puede desencriptarlo y leer el texto original. |
| **¿Para qué se usa en TechStore?** | **Exclusivamente para Contraseñas** (o códigos PIN). | Para guardar **cuentas bancarias (IBAN)** del vendedor, números de documento o claves de API de terceros. |

### Cómo usar Hashing (`Hash`)
```php
use Illuminate\Support\Facades\Hash;

// 1. Convertir contraseña en huella segura al guardar:
$hashSeguro = Hash::make('MiClaveSecreta123');
// Guarda algo como: "$2y$12$QjSH496pcT5CEbzjD/vtVeH03tfHKFy36d4J0Ltp3lRtee9HDxY3K"

// 2. Comprobar en el Login si la contraseña escrita coincide con el Hash guardado:
if (Hash::check($request->password, $user->password)) {
    // ¡Contraseña correcta!
}
```

### Cómo usar Encryption (`Crypt`)
Tienes dos formas en **TechStore**:

**Forma 1 (Automática en el Modelo — La más cómoda):**
En `app/Models/Profile.php` agregas el cast `'encrypted'`:
```php
protected function casts(): array
{
    return [
        'cuenta_bancaria_cifrada' => 'encrypted',
    ];
}
```
Cuando haces `$profile->cuenta_bancaria_cifrada = 'ES91 2100...';`, en la tabla de MySQL se guarda un texto incomprensible (`eyJpdiI6Ik...`), pero cuando haces `echo $profile->cuenta_bancaria_cifrada;` en PHP, Laravel lo desencripta al vuelo para ti.

**Forma 2 (Manual con la fachada `Crypt`):**
```php
use Illuminate\Support\Facades\Crypt;

$textoCifrado = Crypt::encryptString('Dato ultra secreto');
$textoOriginal = Crypt::decryptString($textoCifrado);
```

---

## Pilar 5: Email Verification en una API

Cuando un cliente se registra en **TechStore API**, queremos comprobar que su correo realmente existe antes de dejarle hacer pedidos grandes.
1. En `app/Models/User.php` implementamos la interfaz `MustVerifyEmail`:
   ```php
   use Illuminate\Contracts\Auth\MustVerifyEmail;

   class User extends Authenticatable implements MustVerifyEmail
   ```
2. Al registrarse, `$user->sendEmailVerificationNotification();` le envía un enlace firmado criptográficamente (o puedes enviarle un código de 6 dígitos).
3. Cuando el usuario pulsa el enlace que apunta a tu ruta de verificación en `routes/api.php`:
   ```php
   Route::get('/email/verify/{id}/{hash}', function (Request $request, $id) {
       $user = User::findOrFail($id);

       if (! $request->hasValidSignature()) {
           return response()->json(['mensaje' => 'Enlace inválido o caducado.'], 403);
       }

       if (! $user->hasVerifiedEmail()) {
           $user->markEmailAsVerified();
       }

       return response()->json(['mensaje' => '¡Correo verificado con éxito!']);
   })->name('verification.verify');
   ```
4. Y para proteger cualquier ruta de tu API exigiendo que tenga el correo verificado, simplemente le añades el middleware **`verified`**:
   ```php
   Route::post('/orders', [OrderController::class, 'store'])
       ->middleware(['auth:sanctum', 'verified']);
   ```

---

## Pilar 6: Password Reset (Recuperar Contraseña por API)

Cuando un usuario de **TechStore** olvida su contraseña en la app móvil, el proceso por API tiene **2 endpoints**:

1. **`POST /api/forgot-password`**: El usuario envía su `email`. Laravel genera un token temporal en la tabla `password_reset_tokens` y se lo envía por correo:
   ```php
   use Illuminate\Support\Facades\Password;

   public function enviarEnlaceRecuperacion(Request $request)
   {
       $request->validate(['email' => 'required|email']);

       Password::sendResetLink($request->only('email'));

       return response()->json([
           'mensaje' => 'Si el correo existe en TechStore, recibirás las instrucciones de recuperación.'
       ]);
   }
   ```
2. **`POST /api/reset-password`**: El usuario envía su `email`, el `token` que recibió en el correo y su nueva `password` + `password_confirmation`:
   ```php
   public function cambiarPassword(Request $request)
   {
       $request->validate([
           'token' => 'required',
           'email' => 'required|email',
           'password' => 'required|min:8|confirmed',
       ]);

       $estado = Password::reset(
           $request->only('email', 'password', 'password_confirmation', 'token'),
           function (User $user, string $password) {
               $user->forceFill([
                   'password' => Hash::make($password)
               ])->save();

               // Por seguridad, cerramos todas las sesiones abiertas anteriormente
               $user->tokens()->delete();
           }
       );

       return $estado === Password::PASSWORD_RESET
           ? response()->json(['mensaje' => 'Contraseña actualizada con éxito.'])
           : response()->json(['error' => 'El token de recuperación es inválido o ha expirado.'], 400);
   }
   ```

---

👉 **Siguiente paso:** En el **[Módulo 8: Notificaciones, Colas (Jobs), Subida de Imágenes, Consumo de APIs y Tiempo Real](./08_NOTIFICACIONES_COLAS_ARCHIVOS_Y_TIEMPO_REAL.md)** cerraremos el curso con todas las herramientas avanzadas que hacen brillar una API en producción.
