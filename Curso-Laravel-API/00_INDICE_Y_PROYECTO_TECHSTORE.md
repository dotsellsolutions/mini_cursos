# Curso Maestro de Laravel para APIs (De Cero a Experto)
## Proyecto Hilo Conductor: **TechStore API**

¡Bienvenido al curso definitivo de Laravel enfocado **100% en la creación de APIs modernas**! 
Aquí no veremos diseño de páginas con HTML/Blade, sino lo que realmente mueve a las aplicaciones actuales: **un Backend de API profesional** capaz de alimentar una página web en React/Vue/Angular, una aplicación móvil en Flutter/iOS/Android, o ambos al mismo tiempo.

---

## 1. ¿Qué es una API (explicado sin tecnicismos)?

Imagina que vas a un restaurante de comida rápida:
1. Tú no entras a la cocina a freír las papas. Tú miras la pantalla y pides: *"Quiero una hamburguesa doble sin cebolla"*.
2. El cajero anota tu pedido, se lo pasa a la cocina, la cocina lo prepara y te entrega una **caja cerrada y ordenada** con tu hamburguesa.

En el mundo de la programación:
* **El Cliente (Frontend / App Móvil):** Es la persona que hace el pedido.
* **La Petición (Request):** Es la orden (*"Dame la lista de productos baratos"* o *"Guarda este nuevo pedido"*).
* **Tu API en Laravel:** Es toda la cocina organizada que revisa si tienes permiso, busca en el almacén (Base de Datos), prepara los datos y los empaqueta.
* **El JSON (Response):** Es esa **caja universal y ordenada** en la que Laravel entrega los datos para que cualquier celular o navegador los entienda al instante.

Un ejemplo de esa "caja" (formato **JSON**) que devuelve nuestra API se ve así de limpio:

```json
{
  "estado": "exito",
  "producto": {
    "id": 10,
    "nombre": "Teclado Mecánico RGB",
    "precio": 59.99,
    "en_oferta": true
  }
}
```

---

## 2. Nuestro Proyecto de principio a fin: **TechStore API**

Para que nunca pierdas el hilo, **todos los ejemplos de todos los capítulos** construyen una misma aplicación real llamada **TechStore API** (una tienda de tecnología donde vendedores publican productos, clientes compran y dejan reseñas).

### ¿Qué tablas y piezas tendrá nuestra TechStore API?

```
┌─────────────────┐       1 a 1 (HasOne)        ┌─────────────────┐
│      User       │ ──────────────────────────► │     Profile     │
│ (Clientes,      │                             │ (Teléfono, Dir, │
│  Vendedores,    │                             │  Datos Bancarios│
│  Admins)        │                             │  Cifrados)      │
└────────┬────────┘                             └─────────────────┘
         │
         │ 1 a Muchos (HasMany)
         ▼
┌─────────────────┐       Muchos a Muchos       ┌─────────────────┐
│      Order      │ ◄─────────────────────────► │     Product     │
│   (Pedidos)     │   (Tabla: order_product)    │   (Productos)   │
└─────────────────┘    guarda: cantidad, precio └────────┬────────┘
                                                         ▲
                                                         │ 1 a Muchos (HasMany)
                                                ┌────────┴────────┐
                                                │    Category     │
                                                │  (Categorías)   │
                                                └─────────────────┘
         
         ✨ Además: Tabla "Review" (Polimórfica: sirve para calificar tanto Productos como Vendedores)
```

---

## 3. Mapa de Módulos del Curso

Cada archivo en esta carpeta es un capítulo completo. Te recomiendo leerlos en orden:

1. **[01_ESTRUCTURA_DE_CARPETAS_Y_CONSOLA_ARTISAN.md](./01_ESTRUCTURA_DE_CARPETAS_Y_CONSOLA_ARTISAN.md)**
   * Qué hay dentro de cada carpeta que crea Laravel (cuáles usarás todos los días y cuáles no debes tocar).
   * La consola **Artisan** explicada comando por comando, `tinker` y cómo crear tus propios comandos automáticos.
2. **[02_MIGRACIONES_FACTORIES_Y_SEEDERS.md](./02_MIGRACIONES_FACTORIES_Y_SEEDERS.md)**
   * Cómo construir las tablas de **TechStore** sin escribir SQL.
   * Todos los tipos de datos, llaves foráneas (`foreignId`) y cómo inventar cientos de productos de prueba con `HasFactory`.
3. **[03_MODELOS_A_FONDO_PROPIEDADES_Y_TRAITS.md](./03_MODELOS_A_FONDO_PROPIEDADES_Y_TRAITS.md)**
   * **Desglose total del Modelo:** Qué significan `HasFactory`, `Notifiable`, `HasApiTokens`, `SoftDeletes`.
   * **Todas las variables `$` del modelo:** `$fillable`, `$guarded`, `$hidden`, `$visible`, `$casts` (fechas, booleanos, JSON, cifrados), `$table`, `$primaryKey`, `$appends`, `$with` y más.
   * *Accessors*, *Mutators*, *Scopes* (filtros) y *Observers* (acciones automáticas).
4. **[04_RELACIONES_Y_EL_ORM_ELOQUENT.md](./04_RELACIONES_Y_EL_ORM_ELOQUENT.md)**
   * Qué es el **ORM Eloquent** frente a consultas normales.
   * Todas las relaciones (`HasOne`, `BelongsTo`, `HasMany`, `BelongsToMany` con tablas pivote, `MorphMany`).
   * Consultas avanzadas, el peligroso **problema N+1**, Colecciones y **Transacciones (`DB::transaction`)**.
5. **[05_RUTAS_API_CONTROLADORES_Y_MIDDLEWARES.md](./05_RUTAS_API_CONTROLADORES_Y_MIDDLEWARES.md)**
   * Todas las rutas en `routes/api.php`, códigos de respuesta HTTP (`200`, `201`, `401`, `403`, `404`, `422`).
   * **Middlewares a fondo:** Cómo funcionan y **cómo pasarles parámetros** (ej. `middleware('rol:admin,vendedor')`).
   * Configuración de **CORS** y límite de peticiones por minuto (*Rate Limiting*).
6. **[06_VALIDACIONES_FORM_REQUESTS_Y_API_RESOURCES.md](./06_VALIDACIONES_FORM_REQUESTS_Y_API_RESOURCES.md)**
   * Cómo validar datos entrantes con `FormRequest` y todas las reglas disponibles.
   * Cómo transformar y maquillar tus respuestas JSON usando `API Resources`.
7. **[07_SEGURIDAD_TOTAL_SANCTUM_AUTH_Y_PERMISOS.md](./07_SEGURIDAD_TOTAL_SANCTUM_AUTH_Y_PERMISOS.md)**
   * **Los 6 pilares de Seguridad:**
     1. **Authentication:** **Laravel Sanctum** completo + comparativa clara con **JWT**, **Passport (OAuth2)**, **Socialite** (Login con Google) y **API Keys**.
     2. **Authorization:** *Gates* y *Policies* (quién tiene permiso de editar o borrar cada producto).
     3. **Hashing:** Protección irreversible de contraseñas (`Hash`).
     4. **Encryption:** Cifrado reversible de datos bancarios del vendedor (`Crypt`).
     5. **Email Verification:** Confirmación de cuenta por correo en una API.
     6. **Password Reset:** Recuperación de contraseña olvidada paso a paso.
8. **[08_NOTIFICACIONES_COLAS_ARCHIVOS_Y_TIEMPO_REAL.md](./08_NOTIFICACIONES_COLAS_ARCHIVOS_Y_TIEMPO_REAL.md)**
   * **Notificaciones:** Envío de correos y sistema de **"Campanita de notificaciones"** en Base de Datos para tu API.
   * **Colas (`Jobs`):** Tareas en segundo plano para que tu API sea ultrarrápida.
   * **Subida de fotos (`Storage`):** Guardar imágenes de productos, `storage:link` y devolver su URL.
   * **Consumo de otras APIs (`Http`)**, **Manejo global de errores en JSON** y **Tiempo real (WebSockets)**.
9. **[09_TESTING_AUTOMATICO_DE_LA_API.md](./09_TESTING_AUTOMATICO_DE_LA_API.md)**
   * **Tests automáticos con Pest** (y su equivalente en PHPUnit): *Feature* vs *Unit*, base de datos SQLite en memoria y `RefreshDatabase`.
   * Testear rutas protegidas con `Sanctum::actingAs`, Policies (`403`) y validaciones (`422`) con *datasets*.
   * *Fakes* para correos, notificaciones, colas, archivos y APIs externas (`Notification::fake`, `Storage::fake`, `Http::fake`...) y CI con **GitHub Actions**.
10. **[10_BUENAS_PRACTICAS_Y_DESPLIEGUE_A_PRODUCCION.md](./10_BUENAS_PRACTICAS_Y_DESPLIEGUE_A_PRODUCCION.md)**
   * **Versionado** (`/api/v1`), formato de respuesta consistente y paginación profesional (`cursorPaginate`).
   * **Documentación automática** (Scramble / Scribe) y **rendimiento** (caché, índices, `Model::shouldBeStrict`, Telescope y Pulse).
   * **Despliegue en VPS** paso a paso: Nginx, `php artisan optimize`, Supervisor para colas, cron del Scheduler, script de deploy y checklist de seguridad.
   * 🏋️ **Proyecto final propuesto:** módulo de cupones de descuento para practicar todo lo aprendido.

---

## 4. Requisitos y cómo seguir el curso

* **PHP 8.2+**, **Composer** y **MySQL/MariaDB** (WAMP, XAMPP, Laragon o Laravel Herd sirven perfectamente).
* Los ejemplos están escritos para **Laravel 11 y 12**. Cuando algo cambia respecto a Laravel 10 o anteriores, se indica en el propio módulo.
* Un cliente para probar la API: **Postman**, **Insomnia**, **Bruno** o la extensión *Thunder Client* de VS Code. Recuerda enviar siempre el Header `Accept: application/json`.
* Crea el proyecto con:
  ```bash
  composer create-project laravel/laravel techstore-api
  cd techstore-api
  php artisan install:api
  ```
