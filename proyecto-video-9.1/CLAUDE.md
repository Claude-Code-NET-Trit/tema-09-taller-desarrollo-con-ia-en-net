# CLAUDE.md

## Proyecto

MasterNet es una API backend en ASP.NET Core (.NET 10) para gestión de cursos online (cursos, instructores, precios, ratings, fotos, cuentas de usuario). Usa Clean/Onion Architecture con CQRS en vertical slices y reemplaza MediatR y AutoMapper por implementaciones propias (`libs/Core.MediatOR`, `libs/Core.Mappy`) en vez de los paquetes NuGet, con fines didácticos.

Lee `ARCHITECTURE.md` para el detalle completo (diagrama de capas, secuencia de una petición, decisiones de diseño, deuda técnica). No lo dupliques aquí.

## Comandos

```bash
# Build
dotnet build MasterNet.sln

# Ejecutar la API (Scalar sirve la documentación interactiva en / durante Development)
dotnet run --project src/MasterNet.WebApi

# Migraciones EF Core (ejecutar desde la raíz del repo)
dotnet ef migrations add <Nombre> --project src/MasterNet.Persistence --startup-project src/MasterNet.WebApi
dotnet ef database update --project src/MasterNet.Persistence --startup-project src/MasterNet.WebApi

# SQL Server local
docker compose up -d
```

No existe todavía ningún test en `tests/MasterNet.Application.UnitTests` (solo la carpeta `Cursos/` vacía, con xUnit + Moq referenciados). Si agregas pruebas, colócalas ahí y ejecútalas con `dotnet test MasterNet.sln`.

### Proveedor de base de datos

`MasterNetDbContext` usa `UseSqlServer` con la connection string `SqlServerDatabase` (`src/MasterNet.Persistence/DependencyInjection.cs`). Hay una línea `UseSqlite` comentada y una connection string `SqliteDatabase` como alternativa local rápida — no la actives sin que el usuario lo pida. Los archivos `LocalDatabase.db*` en `src/MasterNet.WebApi/` son restos de cuando se usaba SQLite; SQL Server (vía `docker-compose.yml`) es el proveedor activo.

### Configuración local y secretos

Este repositorio **no usa .NET User Secrets**. La configuración local privada de `MasterNet.WebApi` (p. ej. `TokenKey`, `ConnectionStrings:SqlServerDatabase`) va en `src/MasterNet.WebApi/appsettings.Local.json`, que está en `.gitignore` y solo se carga cuando `ASPNETCORE_ENVIRONMENT=Development` (orden de configuración en `Program.cs`: `appsettings.json` → `appsettings.{Environment}.json` → `appsettings.Local.json` en Development → variables de entorno). Usa `src/MasterNet.WebApi/appsettings.Local.example.json` como plantilla. No añadas `UserSecretsId`, `AddUserSecrets` ni nada relacionado con `secrets.json`.


## Git y commits

- Utiliza **Conventional Commits**.
- Mantén el asunto del commit conciso y descriptivo.
- Agrega un cuerpo cuando la razón del cambio no sea evidente.
- Explica por qué se realizó el cambio, no solamente qué cambió.
- Nunca hagas `push` de los cambios, a menos que se solicite explícitamente.
- No hagas `commit` automáticamente a menos que el usuario lo solicite.
- Antes de crear un commit, revisa los cambios incluidos para evitar archivos no relacionados.
- Escribe los mensajes de commit en español.


## Arquitectura

Las dependencias apuntan hacia adentro (Domain no depende de nada; WebApi depende de todo):

```
MasterNet.Domain          → entidades, IGenericRepository/IUnitOfWork, constantes de seguridad — sin dependencias
MasterNet.Persistence     → MasterNetDbContext (IdentityDbContext<AppUser>), configuraciones/migraciones EF, GenericRepository/UnitOfWork, seed con Bogus+JSON — depende de Domain
MasterNet.Application     → casos de uso CQRS, patrón Result<T>, ValidationBehavior — depende de Domain, Persistence y libs/*
MasterNet.Infrastructure  → Cloudinary (fotos), servicios de token/usuario JWT, reportes CSV — depende de Application
MasterNet.WebApi          → Controllers, Program.cs (composition root), middleware de excepciones — depende de Application + Infrastructure
libs/Core.MediatOR        → mediator propio (IRequest/IRequestHandler/IPipelineBehavior), resuelto vía DI + reflection
libs/Core.Mappy           → mapper propio (IMapper/ProjectTo/IMappingProfile)
```

`Application` referencia `Persistence` directamente, no solo interfaces de Domain. Esto es intencional, no un descuido — habilita el split CQRS pragmático:
- **Escrituras** → `IUnitOfWork.Repository<T>()` (repositorio delgado: `GetByIdAsync`/`AddAsync`).
- **Lecturas complejas** (paginación, filtros, proyecciones) → inyecta `MasterNetDbContext` directo y usa LINQ + `ProjectTo<TDto>()`, sin pasar por el repositorio genérico.

No introduzcas un repositorio genérico para queries de lectura complejas: sigue el patrón existente e inyecta `MasterNetDbContext` directamente en el handler de la query.

### Vertical slices

Cada caso de uso vive en `src/MasterNet.Application/<Feature>/<CasoDeUso>/` en un solo archivo, dentro de una clase contenedora (`public class`) que anida: el record `IRequest<Result<T>>`, su `IRequestHandler<,>` y su `IValidator<T>`. Ejemplo real: `Courses/CourseCreate/CreateCourseCommand.cs` contiene `CreateCourseCommandRequest`, `CreateCourseCommandHandler` y `CreateCourseCommandRequestValidator`.

Sigue este mismo layout de un archivo por caso de uso al agregar features nuevas — no separes Commands, Handlers y Validators en carpetas por tipo técnico.

Los validadores se detectan automáticamente: `ValidationBehavior<,>` en el pipeline del mediator busca un `IValidator<TRequest>` registrado y lo ejecuta antes del handler. No lo invoques manualmente desde el handler ni desde el controller.

Los errores de flujos de negocio esperados (validación, "no encontrado", etc.) se devuelven como `Result<T>.Failure(...)`, nunca como excepciones. `ExceptionMiddleware` solo captura fallos verdaderamente inesperados y devuelve un JSON 500 uniforme. En los controllers, traduce `Result<T>` a HTTP con `this.FromResult(result)` (`WebApi/Extensions/ResultExtensions.cs`) — no construyas respuestas de error a mano en los controllers.

No uses bibliotecas de mapeo automático (AutoMapper, Mapster) ni de mediator (MediatR). Usa `Core.Mappy` (`IMapper`/`ProjectTo`/`IMappingProfile`) y `Core.MediatOR` (`IRequest`/`IRequestHandler`/`IPipelineBehavior`), que son las implementaciones propias del proyecto.

### Autorización

Basada en claims, no en roles: los roles (`ADMIN`, `CLIENT`, en `Domain/Security/CustomRoles.cs`) son solo agrupaciones de claims `POLICIES` granulares (`COURSE_WRITE`, `COURSE_READ`, `RATING_CREATE`, etc., en `Domain/Security/PolicyMaster.cs`) asignados al usuario al emitir el JWT. `[Authorize(Policy = "...")]` en los controllers verifica esos claims directamente (ver `WebApi/Extensions/PoliciesConfiguration.cs`). Al agregar un endpoint protegido, define o reutiliza una constante en `PolicyMaster`, no compares roles directamente. El JWT es HMAC-SHA256 sin estado, expira a los 7 días y no hay refresh tokens.

### Dónde se registran las cosas

- `Program.cs` es el composition root: cada capa expone un `Add<Capa>()` (`AddApplication`, `AddPersistence`, `AddInfrastructure`, `AddIdentityServices`, `AddPoliciesServices`) invocado ahí.
- `MasterNet.Infrastructure.DependencyInjection.AddInfrastructure()` está vacío a propósito — los servicios de infraestructura (`IPhotoService`, `IReportService<>`) se registran directo en `Program.cs`. Sigue ese mismo patrón si agregas un nuevo servicio de infraestructura, no lo muevas a `AddInfrastructure()` sin que te lo pidan.
- OData está habilitado en la ruta `courses` (`/odata`) además de los controllers REST estándar.

## Puntos débiles conocidos (no los "arregles" sin que se pida)

- `TokenKey` (clave de firma JWT) solo está en `appsettings.Development.json`, falta en Production/Staging.
- CORS usa `WithOrigins("*")` + `AllowAnyMethod()`/`AllowAnyHeader()` — abierto a cualquier origen; válido para dev, no para producción.
- Credenciales del admin de seed (`src/MasterNet.Persistence/SeedData` + `SeedDatabase`, invocado vía `UseAsyncSeeding` en `Persistence/DependencyInjection.cs`) y el password de SQL Server (`docker-compose.yml`, connection strings) están hardcodeados en texto plano — es lo esperado en este setup local/didáctico, no lo repliques como patrón en código nuevo.
- `ISpecification<T>` en Domain tiene una única implementación (`DeviceByNameContainsSpec`) y ni siquiera se resuelve a través del repositorio genérico — se aplica a mano contra el `DbContext`. No generalices este patrón sin que el usuario lo pida explícitamente.

# Domain
@docs/domain-glossary.md
