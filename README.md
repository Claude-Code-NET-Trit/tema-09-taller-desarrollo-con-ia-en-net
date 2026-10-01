# Tema 9 · Taller de Desarrollo con IA y Claude Code en .NET

Noveno y último tema del curso **[Claude Code .NET](https://www.trainingit.es/producto/claude-code-net/)** de TrainingIT.

Recorremos el ciclo profesional completo de una nueva funcionalidad, la gestión de alumnos. La desarrollamos en una rama propia con TDD, con commits redactados por Claude, y abrimos un pull request vinculado a un issue de GitHub. Después revisamos ese pull request desde el papel de revisor, corregimos los comentarios como autor y terminamos con el merge.

## Qué contiene este repositorio

| Carpeta | Descripción |
|---|---|
| `proyecto-video-9.1/` | El proyecto del curso con la funcionalidad de alumnos terminada, tal como queda al final del vídeo 9.1 |

Úsalo para comparar con tu propio proyecto. Los vídeos 9.2 a 9.4 se trabajan sobre **tu propio repositorio**, porque en ellos creas, revisas y resuelves pull requests.

## Cómo ejecutar el proyecto

Como en el tema 8, antes de arrancar la API crea el archivo `src/MasterNet.WebApi/appsettings.Local.json` a partir de `appsettings.Local.example.json`, con los valores que se usan en el curso:

```json
{
  "TokenKey": "c3e6e550-d9cf-4795-8b6a-c40ba0eecf24",
  "ConnectionStrings": {
    "SqlServerDatabase": "Server=localhost,1433;Initial Catalog=MasterNetDB;User ID=sa;Password=Vaxi$2025USPass;Trust Server Certificate=True;"
  }
}
```

## Vídeos del tema

**9.1 Agrega funcionalidad en .NET con Vertical Slice Architecture.** Entidad Student con relación many-to-many, migración y endpoints CRUD desarrollados con TDD; commits hechos con `!` frente a commits redactados por Claude.

**9.2 Crea un pull request en GitHub.** Push de la rama, issue en GitHub y pull request generado por Claude con resumen, plan de pruebas y cierre automático del issue.

**9.3 Evaluando pull requests de otros developers.** Revisión de un pull request clonado aparte y publicación de los hallazgos como comentarios en GitHub, sin modificar código.

**9.4 Solucionando pull requests.** Lectura de los comentarios del pull request, corrección desde la sesión original, respuestas marcadas como resueltas, squash and merge y borrado de la rama.

---

Curso **Claude Code .NET** · [TrainingIT](https://www.trainingit.es/producto/claude-code-net/)
