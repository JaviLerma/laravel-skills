---
name: laravel-action-pattern
description: Enseña y aplica Action Pattern en aplicaciones Laravel usando clases Action propias. Usar cuando Codex necesite refactorizar controladores Laravel, mover business logic fuera de controllers o models, crear acciones reutilizables, decidir donde van validacion y HTTP concerns, usar Form Requests con datos validados, agregar transacciones de base de datos alrededor de mutaciones, reutilizar logica desde controllers, jobs o console commands, o discutir convenciones pragmaticas de Laravel Actions sin paquetes externos.
---

# Laravel Action Pattern

## Flujo

1. Inspeccionar el proyecto Laravel antes de cambiar codigo:
   - Identificar la version de Laravel desde `composer.json`, paquetes instalados, namespaces, estilo de controllers, convenciones de Form Requests, y si ya existen `app/Actions` o clases similares.
   - Si la tarea depende del comportamiento actual de una API de Laravel, usar Context7 para consultar la documentacion de Laravel antes de responder o editar.
   - Preferir el estilo existente del proyecto salvo que contradiga los limites del Action Pattern definidos abajo.

2. Mantener el controller a cargo de la capa HTTP:
   - Type-hintear el `FormRequest`, modelos por route model binding, fuente del usuario autenticado y la Action.
   - Llamar `$request->validated()` o `$request->safe()->...` antes de invocar la Action.
   - Mantener redirects, JSON/resources, session invalidation, logout, response codes y comportamiento exclusivo del request en el controller.

3. Poner mutaciones de negocio en clases Action:
   - Ubicacion default para proyectos nuevos: carpeta plana `app/Actions`.
   - Nombrar Actions con verbo + objeto, como `CreateActivity`, `UpdateUser` o `DestroyAccount`.
   - Preferir Actions invocables con `__invoke(...)`; usar `handle(...)` solo si el proyecto ya usa ese estilo.
   - Pasar objetos de dominio y datos escalares/arrays ya validados. No pasar `Request`, `FormRequest`, `RedirectResponse`, sesiones ni HTTP responses a las Actions.

4. Tratar la validacion como completa antes de entrar a la Action:
   - El `FormRequest` es responsable de validacion y autorizacion del request.
   - La Action puede hacer cumplir invariantes de dominio, pero no debe repetir reglas de validacion HTTP.
   - Usar arrays con PHPDoc array shapes por default para payloads simples.
   - Introducir DTOs solo cuando el payload sea complejo, compartido entre capas, o necesite comportamiento/invariantes propias.

5. Usar transacciones deliberadamente:
   - Envolver mutaciones relacionadas en `DB::transaction()` para que las excepciones hagan rollback de la unidad de trabajo.
   - Considerar retries ante deadlocks en caminos con contencion.
   - Para emails, events, jobs, notifications o APIs externas, decidir si el efecto debe ocurrir after commit, dentro de la transaccion o fuera de ella. Preferir after-commit cuando el efecto no deberia dispararse si la transaccion hace rollback.

6. Preferir Actions explicitas sobre side effects ocultos del modelo en workflows centrales:
   - Usar observers/events para integraciones realmente desacopladas o reacciones no criticas.
   - Evitar que flujos centrales dependan de observers implicitos cuando una Action puede mostrar la secuencia completa directamente.

7. Manejar queries de forma pragmatica:
   - Dejar lecturas simples en controllers, models, query builders, scopes o repositories ya usados por el proyecto.
   - Crear query objects o read actions solo cuando la query sea compleja, reutilizada o codifique reglas de negocio.
   - No forzar cada list/show query a una Action solo por simetria.

## Referencia

Cargar `references/action-pattern.md` cuando haya que crear o revisar ejemplos concretos de codigo Laravel. Incluye ejemplos de controller, action, transaccion, reutilizacion, DTOs, queries y destruccion de cuenta.

## Checklist

Antes de terminar un cambio con Action Pattern, verificar:

- Los controllers ya no contienen la mutacion de negocio objetivo.
- Las Actions no tienen dependencias exclusivas de HTTP.
- Las Actions reciben datos validados, modelos y servicios de forma explicita.
- Las escrituras de base de datos en varios pasos son transaccionales.
- Los efectos externos estan ubicados intencionalmente respecto del commit de la transaccion.
- Los tests cubren la Action directamente, mas un camino controller/request cuando cambio comportamiento HTTP.
