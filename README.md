# ⚡ .NET Core & C# Enterprise Architecture Wiki

Repositorio personal de apuntes, guías rápidas (*cheatsheets*) y patrones de diseño para el desarrollo backend enterprise con C#, .NET Core, Clean Architecture y bases de datos relacionales.

---

## 📂 Estructura del Conocimiento

### 1. C# Core & Runtime (`/01-csharp-core-and-runtime`)
- Tipado estático, *Generics*, *Delegates*, *Events* y expresiones lambda.
- Consultas eficientes con **LINQ** en memoria y diferidas (`IEnumerable` vs `IQueryable`).
- Asincronía con `async/await`, `Task`, `CancellationToken` y manejo de hilos.
- Aritmética financiera, precisión decimal y prevención de errores de redondeo.

### 2. Arquitectura & Patrones de Diseño (`/02-architecture-and-design-patterns`)
- **Principios SOLID** con ejemplos prácticos en C#.
- **Patrones GoF** más utilizados (*Factory*, *Strategy*, *Repository*, *Decorator*, *Builder*).
- **Clean Architecture:** Separación de capas (Domain, Application, Infrastructure, API).
- **Patrones de Resiliencia:** *Retry*, *Circuit Breaker* y *Rate Limiting* con Polly.

### 3. ASP.NET Core Web APIs (`/03-aspnet-core-apis`)
- Construcción de APIs RESTful, DTOs y validaciones con **FluentValidation**.
- Inyección de dependencias nativa y ciclo de vida (*Transient*, *Scoped*, *Singleton*).
- Seguridad: Autenticación con **JWT**, autorización basada en roles (RBAC) y OWASP Top 10.
- Auditoría, registros estructurados con **Serilog** y Health Checks.

### 4. Acceso a Datos & Transacciones (`/04-data-access-and-transactions`)
- **Entity Framework Core:** Mapeo, relaciones, *Migrations*, `AsNoTracking` y consultas optimizadas.
- **Transacciones ACID:** Manejo explícito con `IDbContextTransaction` y `TransactionScope`.
- **Concurrencia:** Estrategias de bloqueo optimista (`RowVersion`) y pesimista.
- **Rendimiento SQL:** Solución al problema N+1, indexación y análisis de planes de ejecución.