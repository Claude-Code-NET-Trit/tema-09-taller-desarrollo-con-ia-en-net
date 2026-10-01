# Domain Glossary

Diccionario de datos de soporte para las entidades, objetos de valor, contratos y constantes definidos en `src/MasterNet.Domain`. Los nombres técnicos (clases, propiedades, namespaces) se mantienen en su idioma original (inglés); las definiciones se explican en español.

## Abstractions

Contratos y tipos base compartidos por todas las entidades del dominio. No dependen de ninguna otra capa.

### BaseEntity

`Abstractions/BaseEntity.cs`

Clase base abstracta de la que heredan la mayoría de entidades del dominio. Aporta la propiedad `Id` (`Guid`) como identificador único.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Id` | `Guid` | Identificador único de la entidad. |

### IGenericRepository\<TEntity\>

`Abstractions/IGenericRepository.cs`

Contrato de repositorio genérico "delgado", pensado para operaciones de **escritura** simples. Se usa a través de `IUnitOfWork.Repository<T>()`. Las lecturas complejas (paginación, filtros, proyecciones) no pasan por aquí; inyectan `MasterNetDbContext` directamente desde `Application`.

| Miembro | Descripción |
|---|---|
| `GetByIdAsync(Guid id)` | Obtiene una entidad por su `Id`, o `null` si no existe. |
| `AddAsync(TEntity entity)` | Agrega una nueva entidad al contexto de persistencia. |

### ISpecification\<T\>

`Abstractions/ISpecification.cs`

Contrato del patrón *Specification*, que encapsula un criterio de filtrado como expresión (`Criteria`) reutilizable. Actualmente tiene una única implementación en el proyecto (`DeviceByNameContainsSpec`, en `Persistence`) y se aplica manualmente contra el `DbContext`, sin generalizarse a través del repositorio genérico.

| Miembro | Descripción |
|---|---|
| `Criteria` | Expresión (`Expression<Func<T, bool>>`) que define la condición de filtrado a aplicar sobre `T`. |

### IUnitOfWork

`Abstractions/IUnitOfWork.cs`

Contrato de la Unidad de Trabajo (*Unit of Work*): coordina uno o más repositorios genéricos y confirma o revierte los cambios como una sola transacción.

| Miembro | Descripción |
|---|---|
| `Repository<TEntity>()` | Devuelve el `IGenericRepository<TEntity>` correspondiente a la entidad indicada. |
| `SaveChangesAsync()` | Persiste los cambios pendientes; devuelve `true` si se guardó al menos un cambio. |
| `RollbackAsync()` | Descarta los cambios pendientes sin persistirlos. |

## Courses

Entidades relacionadas con los cursos ofrecidos por la plataforma: el curso en sí y sus tablas de unión con instructores y precios.

### Course

`Courses/Course.cs`

Entidad central del dominio: representa un curso online. Hereda de `BaseEntity`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Title` | `string?` | Título del curso. |
| `Description` | `string?` | Descripción del curso. |
| `PublishedAt` | `DateTime?` | Fecha de publicación del curso. |
| `Ratings` | `ICollection<Rating>?` | Calificaciones (`Rating`) que los estudiantes dejaron sobre el curso. |
| `Prices` | `ICollection<Price>?` | Precios asociados al curso (relación muchos-a-muchos vía `CoursePrice`). |
| `CoursePrices` | `ICollection<CoursePrice>?` | Registros de la tabla de unión curso-precio. |
| `Instructors` | `ICollection<Instructor>?` | Instructores que dictan el curso (relación muchos-a-muchos vía `CourseInstructor`). |
| `CourseInstructors` | `ICollection<CourseInstructor>?` | Registros de la tabla de unión curso-instructor. |
| `Photos` | `ICollection<Photo>?` | Fotos asociadas al curso. |

### CourseInstructor

`Courses/CourseInstructor.cs`

Entidad de unión (tabla intermedia) que representa la relación muchos-a-muchos entre `Course` e `Instructor`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `CourseId` | `Guid?` | Clave foránea hacia el curso. |
| `Course` | `Course?` | Referencia de navegación al curso. |
| `InstructorId` | `Guid?` | Clave foránea hacia el instructor. |
| `Instructor` | `Instructor?` | Referencia de navegación al instructor. |

### CoursePrice

`Courses/CoursePrice.cs`

Entidad de unión (tabla intermedia) que representa la relación muchos-a-muchos entre `Course` y `Price`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `CourseId` | `Guid?` | Clave foránea hacia el curso. |
| `Course` | `Course?` | Referencia de navegación al curso. |
| `PriceId` | `Guid?` | Clave foránea hacia el precio. |
| `Price` | `Price?` | Referencia de navegación al precio. |

## Devices

Modela dispositivos, usados como ejemplo del patrón *Specification* (`ISpecification<T>`) y de objeto de valor (`DeviceName`).

### Device

`Devices/Device.cs`

Entidad sellada (`sealed`) que representa un dispositivo. Hereda de `BaseEntity`. Su constructor es privado por defecto y exige `DeviceName` como parámetro obligatorio, forzando la creación mediante un objeto de valor ya validado.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `DeviceName` | `DeviceName` | Objeto de valor con el nombre del dispositivo. Solo puede asignarse en la construcción (`private set`). |

### DeviceName

`Devices/DeviceName.cs`

Objeto de valor (*Value Object*) que envuelve el nombre de un dispositivo. Valida que el valor no sea nulo ni vacío al construirse, y define igualdad estructural (`IEquatable<DeviceName>`) case-insensitive.

| Miembro | Descripción |
|---|---|
| `Value` | `string` con el nombre del dispositivo; inmutable tras la construcción. |
| `Equals(DeviceName?)` | Compara dos instancias ignorando mayúsculas/minúsculas. |
| `GetHashCode()` | Hash consistente con la comparación case-insensitive. |

## Instructors

### Instructor

`Instructors/Instructor.cs`

Entidad que representa a un instructor de cursos. Hereda de `BaseEntity`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `LastName` | `string?` | Apellido del instructor. |
| `FirstName` | `string?` | Nombre del instructor. |
| `Degree` | `string?` | Título o grado académico del instructor. |
| `Courses` | `ICollection<Course>?` | Cursos que dicta el instructor (relación muchos-a-muchos vía `CourseInstructor`). |
| `CourseInstructors` | `ICollection<CourseInstructor>?` | Registros de la tabla de unión curso-instructor. |

## Photos

### Photo

`Photos/Photo.cs`

Entidad que representa una foto asociada a un curso, almacenada mediante un proveedor externo (Cloudinary, gestionado en `Infrastructure`). Hereda de `BaseEntity`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Url` | `string?` | URL pública de la foto. |
| `CourseId` | `Guid?` | Clave foránea hacia el curso al que pertenece la foto. |
| `Course` | `Course?` | Referencia de navegación al curso. |
| `PublicId` | `string?` | Identificador público de la imagen en el proveedor externo (Cloudinary), usado para eliminarla o referenciarla. |

## Prices

### Price

`Prices/Price.cs`

Entidad que representa un plan o nivel de precio que puede aplicarse a uno o más cursos. **No** hereda de `BaseEntity`; declara su propio `Id`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Id` | `Guid` | Identificador único del precio. |
| `Name` | `string?` | Nombre del plan de precio (ej. "Básico", "Premium"). |
| `CurrentPrice` | `decimal` | Precio actual/regular. |
| `PromotionPrice` | `decimal` | Precio promocional u oferta. |
| `Courses` | `ICollection<Course>?` | Cursos que usan este precio (relación muchos-a-muchos vía `CoursePrice`). |
| `CoursePrices` | `ICollection<CoursePrice>?` | Registros de la tabla de unión curso-precio. |

## Ratings

### Rating

`Ratings/Rating.cs`

Entidad que representa la calificación que un estudiante deja sobre un curso. Hereda de `BaseEntity`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Student` | `string?` | Nombre del estudiante que califica. |
| `Score` | `int` | Puntaje numérico de la calificación. |
| `Comment` | `string?` | Comentario opcional del estudiante. |
| `CourseId` | `Guid?` | Clave foránea hacia el curso calificado. |
| `Course` | `Course?` | Referencia de navegación al curso. |

## Security

Constantes usadas por el sistema de autorización basado en claims (ver `ARCHITECTURE.md` y `CLAUDE.md` para el detalle del mecanismo). Los roles agrupan políticas (*policies*), y son las políticas las que efectivamente se verifican en los endpoints vía `[Authorize(Policy = "...")]`.

### CustomClaims

`Security/CustomClaims.cs`

Nombres de *claims* personalizados incluidos en el JWT.

| Constante | Valor | Descripción |
|---|---|---|
| `POLICIES` | `"POLICIES"` | Nombre del claim que transporta las políticas (permisos granulares) del usuario dentro del token. |

### CustomRoles

`Security/CustomRoles.cs`

Roles de usuario soportados por la aplicación. Funcionan como agrupaciones de políticas (`PolicyMaster`) al momento de emitir el JWT, no como mecanismo de autorización directo.

| Constante | Valor | Descripción |
|---|---|---|
| `ADMIN` | `"ADMIN"` | Rol administrador; agrupa el conjunto completo de políticas. |
| `CLIENT` | `"CLIENT"` | Rol de usuario cliente/estudiante; agrupa un subconjunto reducido de políticas. |

### PolicyMaster

`Security/PolicyMaster.cs`

Catálogo de políticas (permisos granulares) usadas por `[Authorize(Policy = "...")]` en los controllers. Cada política habilita una operación específica sobre una entidad del dominio.

| Constante | Descripción |
|---|---|
| `COURSE_WRITE` | Permite crear cursos. |
| `COURSE_READ` | Permite leer/consultar cursos. |
| `COURSE_UPDATE` | Permite actualizar cursos existentes. |
| `COURSE_DELETE` | Permite eliminar cursos. |
| `INSTRUCTOR_READ` | Permite leer/consultar instructores. |
| `INSTRUCTOR_UPDATE` | Permite actualizar instructores existentes. |
| `INSTRUCTOR_CREATE` | Permite crear instructores. |
| `RATING_READ` | Permite leer/consultar calificaciones. |
| `RATING_DELETE` | Permite eliminar calificaciones. |
| `RATING_CREATE` | Permite crear calificaciones. |
