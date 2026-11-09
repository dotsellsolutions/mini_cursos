# Módulo 6: Validaciones (`FormRequest`) y Respuestas Limpias (`API Resources`)

Un controlador profesional en Laravel nunca debería tener 100 líneas de código. Su trabajo es simplemente coordinar:
1. Un **`FormRequest`** revisa en la entrada que los datos enviados por el usuario sean válidos.
2. El **Controlador** ejecuta la acción con el Modelo.
3. Un **`API Resource`** empaqueta y maquilla el JSON de salida antes de entregarlo.

---

## 1. Validación en APIs: El detalle clave de `Accept: application/json`

Cuando pruebas tu API en Postman, Insomnia o desde React/Flutter, **siempre debes enviar esta cabecera (Header) en tus peticiones**:
```http
Accept: application/json
```
**¿Por qué es tan importante?**
Porque si no la envías y falla una validación, Laravel pensará que eres un navegador web antiguo e intentará redirigirte a una página anterior. En cambio, con `Accept: application/json`, si un dato está mal, Laravel frena en seco y devuelve automáticamente un **Código `422 Unprocessable Entity`** con un JSON clarísimo indicando qué campo falló:

```json
{
  "message": "El campo precio debe ser al menos 1. (y 1 error más)",
  "errors": {
    "nombre": [
      "El nombre del producto es obligatorio."
    ],
    "precio": [
      "El campo precio debe ser al menos 1."
    ]
  }
}
```

---

## 2. Form Requests: Sacando la validación fuera del Controlador

En lugar de ensuciar tu `ProductController` con bloques gigantes de `$request->validate([...])`, creamos una clase dedicada exclusivamente a validar la creación o edición de un producto en **TechStore API**:

```bash
php artisan make:request StoreProductRequest
```

Esto crea `app/Http/Requests/StoreProductRequest.php`, que tiene **3 funciones clave**:
1. **`authorize()`**: ¿Este usuario tiene permiso siquiera de intentar enviar estos datos? (Devuelve `true` o `false`).
2. **`rules()`**: La lista de reglas para cada campo.
3. **`messages()`** *(Opcional)*:Tus propios mensajes de error personalizados en español.

Veamos cómo queda `StoreProductRequest.php` para **TechStore API**:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreProductRequest extends FormRequest
{
    /**
     * 1. ¿Quién puede hacer esta petición?
     */
    public function authorize(): bool
    {
        // Solo permitimos si el usuario autenticado es vendedor o admin
        return in_array($this->user()?->role, ['vendedor', 'admin']);
    }

    /**
     * 2. Reglas de validación para cada campo del JSON o formulario
     */
    public function rules(): array
    {
        return [
            // Debe existir previamente ese ID en la tabla categories
            'category_id' => ['required', 'integer', 'exists:categories,id'],

            'nombre' => ['required', 'string', 'min:3', 'max:180'],

            // Único en la tabla products. Si estamos editando ($this->product existe), ignora el suyo propio:
            'sku' => [
                'nullable',
                'string',
                'max:50',
                Rule::unique('products', 'sku')->ignore($this->route('product')),
            ],

            'descripcion' => ['nullable', 'string', 'max:2000'],
            'precio' => ['required', 'numeric', 'min:0.50', 'max:99999.99'],
            'stock' => ['required', 'integer', 'min:0'],
            'destacado' => ['sometimes', 'boolean'],

            // Validar un objeto JSON anidado (especificaciones):
            'especificaciones' => ['nullable', 'array'],
            'especificaciones.marca' => ['required_with:especificaciones', 'string'],
            'especificaciones.garantia_meses' => ['nullable', 'integer', 'in:6,12,24,36'],

            // Validar subida de imagen (máximo 2MB = 2048 KB, formatos jpg, png, webp)
            'imagen' => ['nullable', 'image', 'mimes:jpg,jpeg,png,webp', 'max:2048'],
        ];
    }

    /**
     * 3. Mensajes personalizados amigables
     */
    public function messages(): array
    {
        return [
            'category_id.exists' => 'La categoría seleccionada no existe en TechStore.',
            'nombre.required' => 'Por favor, escribe el nombre del producto.',
            'sku.unique' => 'Ya existe otro producto con ese código SKU.',
            'precio.min' => 'El precio mínimo permitido es de 0.50 dólares.',
            'especificaciones.garantia_meses.in' => 'La garantía solo puede ser de 6, 12, 24 o 36 meses.',
        ];
    }
}
```

### Diccionario rápido de las Reglas de Validación más útiles:
* **`required`** vs **`nullable`** vs **`sometimes`**:
  * `required`: El campo tiene que venir sí o sí y no estar vacío.
  * `nullable`: Puede no venir o venir como `null`.
  * `sometimes`: *(Ideal para `PATCH` al editar)* Si el usuario no envía este campo en el JSON, no lo valides; pero si **sí** lo envía, entonces aplica las reglas que siguen.
* **`exists:tabla,columna`**: Comprueba que el ID enviado realmente exista en la base de datos (ej. `exists:categories,id`).
* **`unique:tabla,columna`**: Comprueba que nadie más tenga ese correo o código.
* **`confirmed`**: Obliga a que exista un campo gemelo llamado `password_confirmation` con el mismo valor exacto (ideal para registro o cambio de contraseña).
* **Validar listas (arrays) de cosas (ej. los productos dentro de un Pedido en TechStore):**
  ```php
  'productos' => ['required', 'array', 'min:1'],
  'productos.*.product_id' => ['required', 'exists:products,id'], // El asterisco (*) revisa cada elemento de la lista
  'productos.*.cantidad' => ['required', 'integer', 'min:1'],
  ```

---

## 3. API Resources (`JsonResource`): El Maquillador Profesional de tu JSON

Aunque el modelo `Product` ya tiene `$hidden` y `$appends`, en una API real a menudo quieres:
1. Cambiar los nombres de las propiedades o darles una estructura limpia.
2. Incluir la categoría o las reseñas **solo si el controlador las cargó con `with()`** (para evitar el problema N+1 por accidente).
3. Mostrar ciertos datos sensibles **solo si quien consulta la API es el Administrador**.

Para eso creamos un **API Resource**:
```bash
php artisan make:resource ProductResource
```

Abrimos `app/Http/Resources/ProductResource.php`:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class ProductResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'nombre' => $this->nombre,
            'sku' => $this->sku,
            'descripcion' => $this->descripcion,
            'precios' => [
                'sin_impuestos' => (float) $this->precio,
                'con_iva' => $this->precio_con_iva,
                'moneda' => 'USD',
            ],
            'inventario' => [
                'stock_actual' => $this->stock,
                'disponible' => $this->disponible,
            ],
            'especificaciones' => $this->especificaciones ?? [],
            'imagen_url' => $this->imagen_url ? asset('storage/' . $this->imagen_url) : null,

            // 🔒 DATO CONDICIONAL: Solo aparece en el JSON si el usuario autenticado es 'admin'
            'codigo_interno_proveedor' => $this->when(
                $request->user()?->role === 'admin',
                $this->codigo_interno_proveedor
            ),

            // ⚡ RELACIÓN INTELIGENTE: Solo se incluye en el JSON si en el controlador se hizo ->with('category')
            'categoria' => $this->whenLoaded('category', function () {
                return [
                    'id' => $this->category->id,
                    'nombre' => $this->category->nombre,
                ];
            }),

            // Si usamos withCount('reviews') en el controlador, lo incluye automáticamente:
            'total_resenas' => $this->whenCounted('reviews'),

            'fecha_publicacion' => $this->created_at?->format('Y-m-d'),
        ];
    }
}
```

### ¿Cómo se usa un `Resource` en el Controlador?
* **Para devolver 1 solo producto:**
  ```php
  return new ProductResource($product);
  ```
* **Para devolver una lista o paginación de productos** (conserva automáticamente los enlaces de página siguiente, total de páginas, etc.):
  ```php
  $productos = Product::with('category')->withCount('reviews')->paginate(10);
  return ProductResource::collection($productos);
  ```

---

## 4. El Controlador `ProductController.php` Completo y Limpio

Mira qué elegante, corto y fácil de leer queda tu controlador de **TechStore API** cuando unes **Route Model Binding + FormRequest + Eloquent + API Resource**:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreProductRequest;
use App\Http\Resources\ProductResource;
use App\Models\Product;
use Illuminate\Http\Request;

class ProductController extends Controller
{
    // GET /api/products
    public function index(Request $request)
    {
        $productos = Product::with('category')
            ->withCount('reviews')
            ->when($request->buscar, fn ($q, $texto) => $q->where('nombre', 'like', "%{$texto}%"))
            ->when($request->categoria, fn ($q, $cat) => $q->where('category_id', $cat))
            ->latest()
            ->paginate(12);

        return ProductResource::collection($productos);
    }

    // POST /api/products
    public function store(StoreProductRequest $request)
    {
        // Si el código llega a esta línea, los datos YA están 100% validados por StoreProductRequest
        $datosValidados = $request->validated();
        $datosValidados['user_id'] = $request->user()->id;

        $producto = Product::create($datosValidados);

        return (new ProductResource($producto->load('category')))
            ->response()
            ->setStatusCode(201);
    }

    // GET /api/products/{product}
    public function show(Product $product)
    {
        return new ProductResource($product->load(['category', 'reviews']));
    }

    // PUT/PATCH /api/products/{product}
    public function update(StoreProductRequest $request, Product $product)
    {
        $product->update($request->validated());

        return new ProductResource($product->load('category'));
    }

    // DELETE /api/products/{product}
    public function destroy(Product $product)
    {
        $product->delete(); // Lo envía a la papelera gracias a SoftDeletes

        return response()->json([
            'mensaje' => 'Producto enviado a la papelera correctamente.'
        ], 200);
    }
}
```

---

👉 **Siguiente paso:** En el **[Módulo 7: Seguridad Total — Sanctum, Otros Tipos de Auth y los 6 Pilares de Security](./07_SEGURIDAD_TOTAL_SANCTUM_AUTH_Y_PERMISOS.md)** abordaremos de principio a fin toda la seguridad de tu API.
