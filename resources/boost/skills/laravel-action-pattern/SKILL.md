---
name: laravel-action-pattern
description: Teach and apply the Laravel Action Pattern with first-party Action classes. Use when Codex needs to refactor Laravel controllers, move business logic out of controllers or models, create reusable and composable actions in app/Actions, decide where validation and HTTP concerns belong, use Form Requests with validated data, add database transactions around mutations, reuse logic from controllers, jobs, console commands, API requests, MCP requests, or discuss pragmatic Laravel Action conventions without external packages.
---

# Laravel Action Pattern

## Workflow

1. Inspect the Laravel project before changing code:
   - Identify the Laravel version from `composer.json`, installed packages, namespaces, controller style, Form Request conventions, and whether `app/Actions` or similar classes already exist.
   - If the task depends on current Laravel API behavior, use Context7 to consult the Laravel documentation before answering or editing.
   - Prefer the existing project style unless it conflicts with the Action Pattern boundaries below.

2. Keep the controller responsible for the HTTP layer:
   - Type-hint the `FormRequest`, route model bound models, authenticated user source, and Action.
   - Call `$request->validated()` or `$request->safe()->...` before invoking the Action.
   - Keep redirects, JSON/resources, session invalidation, logout, response codes, and request-only behavior in the controller.

3. Put business mutations in Action classes:
   - Default location for new projects: a flat `app/Actions` directory.
   - Create new actions with `php artisan make:action "{name}" --no-interaction`.
   - Name Actions after what they do, with no suffix, such as `CreateActivity`, `UpdateUser`, or `DestroyAccount`.
   - Give each Action a single `handle(...)` method. Do not make Actions invokable by default.
   - Inject dependencies through the constructor using private properties. Actions without dependencies can omit `__construct` and expose only `handle(...)`.
   - Pass domain objects and already validated scalar/array data. Do not pass `Request`, `FormRequest`, `RedirectResponse`, sessions, or HTTP responses to Actions.

4. Treat validation as complete before entering the Action:
   - The `FormRequest` is responsible for request validation and authorization.
   - The Action may enforce domain invariants, but it must not repeat HTTP validation rules.
   - Use arrays with PHPDoc array shapes by default for simple payloads.
   - Introduce DTOs only when the payload is complex, shared across layers, or needs its own behavior/invariants.

5. Use transactions deliberately:
   - Wrap related mutations in `DB::transaction()` inside Actions so exceptions roll back the unit of work.
   - Use transactions for complex operations when multiple models are involved.
   - Consider retries for deadlocks in contended paths.
   - For emails, events, jobs, notifications, or external APIs, decide whether the effect belongs after commit, inside the transaction, or outside it. Prefer after-commit when the effect should not fire if the transaction rolls back.

6. Prefer explicit Actions over hidden model side effects in central workflows:
   - Use observers/events for truly decoupled integrations or non-critical reactions.
   - Avoid making central flows depend on implicit observers when an Action can show the full sequence directly.

7. Handle queries pragmatically:
   - Leave simple reads in controllers, models, query builders, scopes, or repositories already used by the project.
   - Create query objects or read actions only when the query is complex, reused, or encodes business rules.
   - Do not force every list/show query into an Action just for symmetry.

## Reference

Load `references/action-pattern.md` when creating or reviewing concrete Laravel code examples. It includes examples for controllers, actions, transactions, reuse, DTOs, queries, and account deletion.

## Checklist

Before finishing an Action Pattern change, verify:

- Controllers no longer contain the targeted business mutation.
- Actions have no HTTP-only dependencies.
- Actions receive validated data, models, and services explicitly.
- Multi-step database writes are transactional.
- External effects are intentionally placed relative to the transaction commit.
- Tests cover the Action directly, plus a controller/request path when HTTP behavior changed.
