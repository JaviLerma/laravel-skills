# Laravel Service Pattern

Usar esta referencia para disenar Services Laravel que representen capacidades cohesionadas, no cajones de metodos ni wrappers superficiales de controllers.

## Decision Principal

Crear un Service cuando la clase puede nombrarse como una capacidad estable:

- `CardReconciliationMatcher`: matchea registros de una conciliacion.
- `DeboSheetSelectionService`: selecciona hojas operativas segun fecha y turno.
- `SqlServerDeboReader`: lee datos desde un boundary tecnico.
- `PaymentGateway`: encapsula comunicacion con un proveedor externo.
- `MoneyNormalizer`: normaliza valores monetarios desde fuentes heterogeneas.

Evitar un Service cuando el nombre natural seria generico, como `UserService`, `OrderService`, `AdminService` o `DataService`. Ese nombre suele indicar que la clase agrupa demasiadas responsabilidades.

## Cuando Usar Service

Usar Service cuando:

- La logica es principalmente de calculo, seleccion, matching, parsing, normalizacion, lectura o comunicacion externa.
- Hay un algoritmo con varias etapas internas que se entiende mejor con helpers privados.
- La capacidad se reutiliza desde controllers, jobs, commands, Filament, seeders, tests u otras clases de aplicacion.
- Existe un boundary tecnico: API client, reader, gateway, connector, file parser, SDK wrapper.
- Queres testear reglas complejas sin depender de HTTP, Livewire, console output ni UI.
- La clase necesita estado inyectado por constructor, como clients, repositories, clocks, parsers o gateways.

## Cuando No Usar Service

No crear un Service cuando:

- Solo estas moviendo una linea de CRUD para que el controller parezca mas chico.
- Cada metodo publico representa una intencion independiente: `createUser()`, `deleteUser()`, `banUser()`, `resetPassword()`.
- La clase necesita `Request`, `FormRequest`, responses, redirects, session, notifications de UI, componentes Livewire o `$this` de una page.
- El metodo principal solo envuelve una unica operacion de aplicacion y no aporta algoritmo, integracion o capacidad reusable.
- El nombre describe un modulo completo del sistema en vez de una capacidad concreta.

Mal indicio:

```php
final class UserService
{
    public function create(array $data): User {}

    public function ban(User $user): void {}

    public function resetPassword(User $user): void {}

    public function exportCsv(): string {}
}
```

Mejor separar por responsabilidad:

- Una clase dedicada para crear usuarios.
- Una clase dedicada para suspender usuarios.
- Una clase dedicada para restablecer credenciales.
- `UserCsvExporter` para la capacidad de exportacion.

## Ejemplos Buenos

```php
final readonly class DeboSheetSelectionService
{
    public function forShift(string $operationalDate, string $shiftClosure): DeboSheetSelectionData
    {
        // Select and validate sheets for this capability.
    }
}
```

```php
interface PaymentGateway
{
    public function charge(Money $amount, Customer $customer): PaymentResult;
}
```

```php
final readonly class MoneyNormalizer
{
    public function decimal(mixed $value): string
    {
        // Parse localized numbers into a canonical decimal string.
    }
}
```

## Metodos Publicos

Un Service puede tener mas de un metodo publico, pero todos deben pertenecer a la misma capacidad.

Aceptable:

```php
final readonly class ExchangeRateService
{
    public function latest(string $currency): ExchangeRate {}

    public function historical(string $currency, CarbonInterface $date): ExchangeRate {}
}
```

Riesgoso:

```php
final readonly class PaymentService
{
    public function authorizePayment(Order $order): Payment {}

    public function refundPayment(Payment $payment): Refund {}

    public function sendReceipt(Payment $payment): void {}

    public function reconcileBatch(Batch $batch): void {}
}
```

En el caso riesgoso, separar capacidades como `PaymentGateway`, `PaymentRefundProcessor`, `PaymentReceiptSender` y `PaymentReconciliationMatcher`.

## Helpers Privados Vs Otro Service

Mantener metodos privados cuando:

- Solo nombran pasos internos.
- No se usan fuera de esa clase.
- No tienen estado ni dependencias propias.
- Ayudan a leer un algoritmo cohesivo.

Extraer otro Service cuando:

- El helper privado se reutiliza en otra clase.
- El helper necesita dependencias propias.
- El helper tiene reglas de negocio suficientemente importantes para testearse aislado.
- El Service principal queda largo y mezcla dos capacidades reales.

## Limites De Capa

Services de dominio o aplicacion no deben recibir concerns del borde de entrada:

- No `Request`, `FormRequest`, `RedirectResponse`, session o response.
- No objetos de Filament, notificaciones de UI, componentes Livewire, schemas, utilidades reactivas ni `$this`.
- No leer `request()` o `auth()` internamente salvo que el proyecto lo tenga como convencion muy clara; preferir pasar usuario/modelos/datos explicitamente.
- No imprimir en consola ni depender de comandos Artisan para devolver resultados.

Controllers, commands, jobs y Filament preparan datos y llaman al Service. El Service devuelve resultados de dominio, DTOs, modelos o valores simples.

## Transacciones

Si el Service es el duenio de una capacidad de escritura de varios pasos, usar transaccion adentro del Service.

Si una clase externa coordina varias capacidades y define la unidad transaccional completa, dejar la transaccion en esa clase externa y mantener los Services internos sin transacciones anidadas innecesarias.

```php
final readonly class ImportDeboSheetService
{
    /**
     * @param list<array<string, mixed>> $coupons
     */
    public function import(DeboSqlServerSheetData $sheet, array $coupons): DeboSheet
    {
        return DB::transaction(function () use ($sheet, $coupons): DeboSheet {
            // Persist sheet and coupons atomically.
        });
    }
}
```

## Integraciones Externas

Para APIs, SDKs, archivos, bases externas o servicios remotos:

- Preferir contratos cuando aportan testabilidad, reemplazo o mas de una implementacion.
- Mantener detalles de transporte dentro del gateway/client/reader.
- Devolver resultados propios del dominio o DTOs, no respuestas crudas del proveedor salvo que el proyecto ya lo haga.
- Centralizar retries, timeouts, autenticacion y mapeo de errores en el boundary tecnico.

## Testing

Testear Services directamente cuando contienen reglas propias:

```php
it('selects night shift sheets for the previous operational date', function (): void {
    $selection = resolve(DeboSheetSelectionService::class)
        ->forShift('2026-05-18', 'NOCHE');

    expect($selection->sheetNumbersText())->toBe('18063, 18064, 18065');
});
```

Cuando un Service participa en un flujo de entrada:

- Testear el Service para reglas detalladas del algoritmo o integracion.
- Testear el borde de entrada solo para conversion de datos, autorizacion, validacion, notificaciones, redirects o efectos visibles.
- Fakear clientes externos, HTTP y colas cuando el Service cruce boundaries tecnicos.

## Checklist De Diseno

Antes de crear o aprobar un Service:

- Puedo describirlo como una capacidad concreta.
- El nombre no es generico.
- Los metodos publicos tienen alta cohesion.
- La clase no depende de HTTP, Filament, Livewire ni console output.
- Las dependencias entran por constructor.
- Si cruza un boundary externo, hay contrato cuando aporta testabilidad o reemplazo.
- Si escribe multiples registros relacionados, la transaccion esta en el nivel correcto.
- Si crece en varias responsabilidades, se separa por capacidad.
