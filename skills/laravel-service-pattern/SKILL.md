---
name: laravel-service-pattern
description: Guia y aplica el patron Service en aplicaciones Laravel para capacidades cohesionadas, algoritmos e integraciones reutilizables. Usar cuando Codex necesite crear o refactorizar service classes, modelar gateways/readers/clients/connectors, encapsular matchers/selectors/normalizers/calculators/importers/exporters/motores de negocio, definir pocos metodos publicos cohesionados con helpers privados, mantener dependencias externas por contrato, aislar Services de HTTP/UI/Filament/Livewire/console, decidir transacciones dentro de una capacidad compleja, o evitar Services genericos tipo cajon de sastre.
---

# Laravel Service Pattern

## Flujo

1. Inspeccionar el proyecto Laravel antes de cambiar codigo:
   - Identificar version de Laravel, paquetes instalados, namespaces, convenciones de `app/Services`, contracts, readers, support classes y tests.
   - Revisar clases hermanas antes de proponer una convencion nueva.
   - Si la tarea depende de una API concreta de Laravel o de un paquete, consultar documentacion vigente antes de responder o editar.

2. Decidir primero si la responsabilidad es una capacidad estable:
   - Usar Service para una capacidad, algoritmo, integracion o motor con varias etapas internas.
   - Evitar Service cuando solo seria un wrapper de CRUD o una intencion puntual sin algoritmo propio.
   - Mantener controllers, jobs, commands y Filament como bordes de entrada que preparan datos y delegan la capacidad al Service.

3. Disenar Services cohesionados:
   - Nombrar por capacidad especifica, como `CardReconciliationMatcher`, `DeboSheetSelectionService`, `SqlServerDeboReader`, `PaymentGateway` o `MoneyNormalizer`.
   - Evitar nombres genericos como `UserService`, `PaymentService` o `AdminService` salvo que el proyecto ya tenga una convencion clara y acotada.
   - Preferir uno o pocos metodos publicos que pertenezcan a la misma capacidad. Si aparecen responsabilidades independientes, separar en clases distintas.
   - Mantener helpers privados cuando solo explican pasos internos del algoritmo. Extraer otro Service cuando esos helpers son reutilizables o tienen una responsabilidad propia.

4. Mantener limites de capa:
   - No pasar `Request`, `FormRequest`, responses, redirects, sesiones, objetos de Filament, componentes Livewire ni helpers de UI a Services de dominio.
   - Inyectar dependencias por constructor. Evitar `app()` o `resolve()` dentro del Service.
   - En integraciones externas, considerar contratos/interfaces para gateways, readers o clients.

5. Usar transacciones y efectos secundarios deliberadamente:
   - Si el Service ejecuta una unidad de escritura de base de datos de varios pasos, envolverla en `DB::transaction()`.
   - Si el Service solo calcula, selecciona, normaliza o lee datos, mantenerlo libre de mutaciones.
   - Ubicar jobs, events, notifications, mails y APIs externas respecto del commit de forma explicita.

## Referencia

Cargar `references/service-pattern.md` cuando haya que crear o revisar ejemplos concretos de Services, modelar una capacidad, disenar limites publicos/privados, o refactorizar una clase existente.

## Checklist

Antes de terminar un cambio con Service Pattern, verificar:

- El Service representa una capacidad cohesionada, no un cajon de metodos.
- El nombre describe la capacidad real y no un area generica de la aplicacion.
- Los metodos publicos son pocos y pertenecen al mismo concepto.
- Los helpers privados no esconden responsabilidades independientes.
- Controllers, pages, jobs y commands no pasan concerns HTTP/UI al Service.
- Las dependencias externas entran por constructor y, cuando corresponde, por contratos.
- Las escrituras relacionadas son transaccionales y los efectos externos estan ubicados intencionalmente.
- Los tests cubren la capacidad del Service directamente y el borde de entrada que cambio.
