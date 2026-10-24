# Tecnologías heredadas

En esta carpeta aprendes a reconocer, mantener y planificar la migración de las tecnologías de .NET Framework que todavía vas a encontrar en empresas: el propio .NET Framework frente a .NET moderno, ASP.NET Web Forms, Windows Forms con DevExpress, WCF y VB.NET. El objetivo no es dominarlas para proyectos nuevos, sino **leerlas, corregirlas con seguridad y saber qué camino tiene cada una**.

-----

## Antes de empezar

Conviene que ya tengas:

* C# y la plataforma .NET: [C# Core & Runtime](../../README.md).
* ASP.NET Core y APIs: [Fundamentos de ASP.NET Core](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/README.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Entorno del ejemplo |
| --- | --- | --- |
| [1. .NET Framework frente a .NET moderno](01-NET%20Framework%20frente%20a%20NET%20moderno.md) | Línea de tiempo, TFM, cómo reconocer cada plataforma en cinco minutos, .NET Standard y lo que no existe en .NET moderno | .NET 10 (cualquier sistema) |
| [2. ASP.NET Web Forms](02-ASP%20NET%20Web%20Forms.md) | Postback, ciclo de vida, ViewState, controles de servidor, seguridad y destinos de migración | .NET Framework 4.8 + Visual Studio (Windows) |
| [3. Windows Forms y DevExpress](03-Windows%20Forms%20y%20DevExpress.md) | Hilo de interfaz, `async void` en eventos, enlace de datos, MVP y lo que implica depender de DevExpress | .NET 10 (Windows) |
| [4. WCF y sus reemplazos](04-WCF%20y%20sus%20reemplazos.md) | ABC de WCF, bindings, SOAP, CoreWCF, cliente en .NET moderno, gRPC y REST | .NET 10 con CoreWCF (cualquier sistema) |
| [5. VB.NET para desarrolladores de C#](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md) | Equivalencias de sintaxis, `AndAlso`, `Option Strict`, eventos con `Handles`, estado del lenguaje y conversión a C# | .NET 10 (cualquier sistema) |

-----

## El mapa completo en una mirada

```text
.NET Framework 4.8.1 (Windows, mantenimiento)          .NET 10 (multiplataforma, LTS)
──────────────────────────────────────────            ───────────────────────────────
Web Forms (.aspx, ViewState, postback)    ──reescribir──► Blazor / Razor Pages
WinForms (net48)                          ──en el lugar──► WinForms (net10.0-windows)
WCF servidor (SOAP)                       ──CoreWCF────► mismo contrato en .NET 10 · nuevos: gRPC / REST
VB.NET                                    ──convertir──► C# (proyecto por proyecto) · o mantener
Bibliotecas de lógica (net4x)             ──retarget───► netstandard2.0 / net10.0 (puente de la migración)

Señales: TFM net48 · Web.config · packages.config · Global.asax · System.Web · .svc · .vb
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki. Cuando el ejemplo necesita .NET Framework o Windows, la lección lo indica; el resto corre en .NET 10. La sección "Según la versión de .NET" recorre las versiones de la tecnología heredada.

-----

## Después de esta carpeta

Sigue con [IIS y migración](../02-iis-y-migracion/README.md): cómo se despliegan estas aplicaciones (y las modernas) en IIS y qué estrategia elegir para migrarlas sin congelar el negocio.
