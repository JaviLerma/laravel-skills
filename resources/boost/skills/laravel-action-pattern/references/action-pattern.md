# Laravel Action Pattern Reference

Use these examples as defaults when the project does not have a stronger local convention.

## Controller + Form Request + Action

```php
<?php

declare(strict_types=1);

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
        $user = $updateUser->handle($user, $request->validated());

        return UserResource::make($user);
    }
}
```

Controller responsibilities:

- Accept HTTP-layer objects such as `FormRequest`.
- Use route model binding when it fits the route.
- Extract validated payloads with `validated()` or `safe()`.
- Return redirects, JSON responses, resources, or views.

Do not pass the request object to the Action.

## Action With Array Shape

```php
<?php

declare(strict_types=1);

namespace App\Actions;

use App\Models\User;
use Illuminate\Support\Facades\DB;

class UpdateUser
{
    public function __construct(
        private readonly CreateActivity $createActivity,
    ) {
        //
    }

    /**
     * @param array{
     *     name: string,
     *     email: string
     * } $data
     */
    public function handle(User $user, array $data): User
    {
        return DB::transaction(function () use ($user, $data): User {
            $emailChanged = $user->email !== $data['email'];

            $user->forceFill([
                'name' => $data['name'],
                'email' => $data['email'],
            ])->save();

            $this->createActivity->handle(
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

Action responsibilities:

- Execute the business mutation.
- Coordinate other Actions or services through constructor injection with private properties.
- Use transactions around related database changes, especially when multiple models are involved.
- Return domain results, not HTTP responses.

Create new Actions with:

```bash
php artisan make:action "UpdateUser" --no-interaction
```

Actions without dependencies can omit `__construct` and expose only `handle(...)`:

```php
<?php

declare(strict_types=1);

namespace App\Actions;

use App\Models\User;

final class MarkUserEmailAsVerified
{
    public function handle(User $user): bool
    {
        return $user->markEmailAsVerified();
    }
}
```

## Form Request Validation

```php
<?php

declare(strict_types=1);

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

The Action assumes this validation has already happened. Do not duplicate request validation inside the Action unless the rule is a domain invariant that is not specific to HTTP.

## Reuse From Jobs Or Commands

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Actions\UpdateUser;
use App\Models\User;

class SyncUserProfile
{
    public function handle(UpdateUser $updateUser): void
    {
        $user = User::query()->findOrFail($this->userId);

        $updateUser->handle($user, [
            'name' => $this->name,
            'email' => $this->email,
        ]);
    }
}
```

Jobs and commands can reuse Actions because the Action does not depend on the HTTP layer. Validate or build trusted data before invoking the Action from non-HTTP layers.

## Destroy Account: HTTP Boundary

```php
<?php

declare(strict_types=1);

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

        $destroyAccount->handle($user);

        Auth::guard('web')->logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }
}
```

The Action deletes account data. The controller logs out, invalidates session state, and returns the redirect because those responsibilities belong to HTTP/session.

## DTO Rule

Use an array shape first:

```php
/**
 * @param array{name: string, email: string} $data
 */
public function handle(User $user, array $data): User
```

Use a DTO when:

- The same payload is built in several layers.
- The payload has its own transformations or invariants.
- Static analysis needs stronger guarantees than array shapes provide.
- The project already has an established DTO convention.

## Queries

Do not create Actions for every read by default.

Keep this inline or in existing query abstractions:

```php
$users = User::query()
    ->whereBelongsTo($team)
    ->latest()
    ->paginate();
```

Create a query object or read action only when the query is reused, complex, or expresses business language that deserves a name.

## Events, Observers, And Explicitness

Prefer Actions for central business workflows because the sequence stays visible in one place. Use events or observers when the reaction is intentionally decoupled, optional, or cross-cutting. Avoid surprising side effects where a migration, seeder, or maintenance script updates a model and unexpectedly sends notifications.
