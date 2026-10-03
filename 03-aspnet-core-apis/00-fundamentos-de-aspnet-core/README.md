# Fundamentos de ASP.NET Core

ASP.NET Core es el stack web de .NET. Este módulo explica el host, la configuración, la inyección de dependencias, el pipeline HTTP y cómo elegir entre Minimal APIs, controladores, MVC, Razor Pages y Blazor.

-----

## Antes de empezar
Conviene conocer [la plataforma .NET](../../01-csharp-core-and-runtime/00-introduccion/05-Plataforma%20NET%20y%20evolucion.md) y [SDK, CLI, proyectos y NuGet](../../01-csharp-core-and-runtime/00-introduccion/07-SDK%20CLI%20proyectos%20y%20NuGet.md).

-----

## Orden de lectura
| Lección | Qué aprendes |
| --- | --- |
| [1. Host, configuración y pipeline HTTP](01-Host%20configuracion%20y%20pipeline%20HTTP.md) | `Program.cs`, servicios, middleware, entornos y configuración |
| [2. Modelos de aplicaciones ASP.NET Core](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md) | API, MVC, Razor Pages, Blazor y cuándo elegir cada uno |
| [3. MVC, routing y vistas Razor](03-MVC%20routing%20y%20vistas%20Razor.md) | Responsabilidades, rutas, controladores, vistas y formularios |

-----

## El mapa completo en una mirada
```text
HTTP -> middleware -> routing -> endpoint -> respuesta
                 servicios DI + configuración

Endpoint: Minimal API | controller API | MVC | Razor Pages | Blazor
```

-----

## Cómo está armada cada lección
Las lecciones siguen las 13 secciones de la wiki, con ejemplos para .NET 10 y correcciones explícitas del material antiguo basado en `Startup.cs` o puertos fijos.

-----

## Después de esta carpeta
Continúa con [Diseño de APIs REST](../01-diseno-de-apis-rest/README.md) si construirás servicios HTTP, o profundiza en MVC antes de agregar persistencia con Entity Framework Core.
