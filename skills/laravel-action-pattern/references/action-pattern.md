# Referencia de Laravel Action Pattern

Usar estos ejemplos como defaults cuando el proyecto no tenga una convencion local mas fuerte.

## Controller + Form Request + Action

```php
<?php

namespace App\Http\Controllers;

use App\Actions\UpdateUser;
use App\Http\Requests\UpdateUserRequest;
use App\Http\Resources\UserResource;
use App\Models\User;

class UserController
{
    public function update(
        UpdateUserRequest $request,
        User $user,
        UpdateUser $updateUser,
    ): UserResource {
        $user = $updateUser($user, $request->validated());

        return UserResource::make($user);
    }
}
```

Responsabilidades del controller:

- Aceptar objetos de la capa HTTP, como `FormRequest`.
- Usar route model binding cuando encaje con la ruta.
- Extraer payloads validados con `validated()` o `safe()`.
- Devolver redirects, JSON responses, resources o views.

No pasar el request object a la Action.

## Action Invocable Con Array Shape

```php
<?php

namespace App\Actions;

use App\Models\User;
use Illuminate\Support\Facades\DB;

class UpdateUser
{
    public function __construct(
        private readonly CreateActivity $createActivity,
    ) {
    }

    /**
     * @param array{
     *     name: string,
     *     email: string
     * } $data
     */
    public function __invoke(User $user, array $data): User
    {
        return DB::transaction(function () use ($user, $data): User {
            $emailChanged = $user->email !== $data['email'];

            $user->forceFill([
                'name' => $data['name'],
                'email' => $data['email'],
            ])->save();

            ($this->createActivity)(
                user: $user,
                description: 'Updated profile',
            );

            if ($emailChanged) {
                // Prefer an after-commit notification/job when rollback would make the effect invalid.
                $user->sendEmailVerificationNotification();
            }

            return $user->refresh();
        });
    }
}
```

Responsabilidades de la Action:

- Ejecutar la mutacion de negocio.
- Coordinar otras Actions o servicios mediante constructor injection.
- Usar transacciones alrededor de cambios de base de datos relacionados.
- Devolver resultados de dominio, no HTTP responses.

## Form Request Validation

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('user'));
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => [
                'required',
                'string',
                'lowercase',
                'email',
                'max:255',
                Rule::unique('users', 'email')->ignore($this->route('user')),
            ],
        ];
    }
}
```

La Action asume que esta validacion ya ocurrio. No duplicar validacion de request dentro de la Action salvo que se trate de una invariante de dominio no especifica de HTTP.

## Reutilizacion Desde Jobs O Commands

```php
<?php

namespace App\Jobs;

use App\Actions\UpdateUser;
use App\Models\User;

class SyncUserProfile
{
    public function handle(UpdateUser $updateUser): void
    {
        $user = User::query()->findOrFail($this->userId);

        $updateUser($user, [
            'name' => $this->name,
            'email' => $this->email,
        ]);
    }
}
```

Jobs y commands pueden reutilizar Actions porque la Action no depende de la capa HTTP. Validar o construir datos confiables antes de invocar la Action desde capas no HTTP.

## Destroy Account: Limite HTTP

```php
<?php

namespace App\Http\Controllers;

use App\Actions\DestroyAccount;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AccountController
{
    public function destroy(Request $request, DestroyAccount $destroyAccount): RedirectResponse
    {
        $user = $request->user();

        $destroyAccount($user);

        Auth::guard('web')->logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }
}
```

La Action borra datos de cuenta. El controller hace logout, invalida estado de sesion y devuelve el redirect porque eso pertenece a HTTP/session.

## Regla Para DTOs

Usar array shape primero:

```php
/**
 * @param array{name: string, email: string} $data
 */
public function __invoke(User $user, array $data): User
```

Usar DTO cuando:

- El mismo payload se construye en varias capas.
- El payload tiene transformaciones o invariantes propias.
- El analisis estatico necesita garantias mas fuertes que array shapes.
- El proyecto ya tiene una convencion establecida de DTOs.

## Queries

No crear Actions para cada lectura por default.

Mantener esto inline o en abstracciones de query existentes:

```php
$users = User::query()
    ->whereBelongsTo($team)
    ->latest()
    ->paginate();
```

Crear un query object o read action solo cuando la query sea reutilizada, compleja o exprese lenguaje de negocio que merezca un nombre.

## Events, Observers Y Explicitud

Preferir Actions para workflows centrales de negocio porque la secuencia queda visible en un lugar. Usar events u observers cuando la reaccion sea intencionalmente desacoplada, opcional o transversal. Evitar side effects sorpresivos donde una migration, seeder o script de mantenimiento actualiza un model y termina enviando notifications inesperadas.
