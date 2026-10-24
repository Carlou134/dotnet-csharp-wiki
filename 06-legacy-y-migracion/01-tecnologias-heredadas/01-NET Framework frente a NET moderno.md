# .NET Framework frente a .NET moderno

## En una frase

.NET Framework (2002–2019, versión final 4.8.1) es la plataforma original, solo para Windows y en modo mantenimiento; .NET moderno (antes .NET Core, hoy .NET 10) es su reescritura multiplataforma; saber distinguirlos en cinco minutos (formato del `.csproj`, `web.config`, `packages.config`, `System.Web`) es lo primero que necesitas al entrar a un sistema heredado.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué son el SDK, el runtime y un proyecto: [Introducción](../../01-csharp-core-and-runtime/00-introduccion/README.md).
* Cómo arranca una aplicación ASP.NET Core moderna: [Host, configuración y pipeline HTTP](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **.NET Framework:** implementación original de .NET, solo para Windows, que se instala como componente del sistema operativo.
* **.NET moderno:** la plataforma multiplataforma que empezó como .NET Core (2016) y desde .NET 5 se llama solo ".NET".
* **.NET Standard:** especificación de APIs comunes que permitía compartir bibliotecas entre .NET Framework y .NET Core.
* **TFM (*Target Framework Moniker*):** identificador del framework de destino: `net48`, `netstandard2.0`, `net10.0`.
* **Proyecto estilo SDK:** formato moderno y corto del `.csproj` (`<Project Sdk="...">`); el antiguo lista cada archivo y usa `ToolsVersion`.
* **`packages.config`:** archivo de los proyectos antiguos con la lista de paquetes NuGet; reemplazado por `<PackageReference>`.
* **GAC (*Global Assembly Cache*):** carpeta de Windows donde .NET Framework guarda ensamblados compartidos por todo el sistema.

-----

## El problema

Primer día en un trabajo nuevo. Te asignan "agregar un campo al módulo de facturación". Abres la solución:

```text
Facturacion.sln
├── Facturacion.Web/        ← Default.aspx, Global.asax, Web.config, packages.config
├── Facturacion.Servicios/  ← Service1.svc
├── Facturacion.Datos/      ← Model1.edmx
└── Facturacion.Comun/      ← Class1.vb
```

Intentas lo que harías en un proyecto moderno:

1. `dotnet build` desde la terminal: falla con errores sobre `Microsoft.WebApplication.targets` no encontrado. El proyecto web requiere MSBuild de Visual Studio con las herramientas de desarrollo web.
2. Buscas `Program.cs` para ver la configuración: no existe. Está en `Global.asax` y en un `Web.config` de 400 líneas.
3. Agregas un paquete NuGet moderno: no se instala porque solo soporta `net8.0` y el proyecto es `net472`.
4. Copias código de un tutorial de ASP.NET Core: `builder.Services`, `IActionResult` de `Microsoft.AspNetCore.Mvc`, `appsettings.json`... nada de eso existe aquí.
5. Le preguntas a un compañero si "esto es .NET Core" y te responde "es .NET, ¿no?".

Las consecuencias:

* horas perdidas usando herramientas y documentación de la plataforma equivocada;
* paquetes incompatibles y builds que no funcionan fuera de la máquina de alguien;
* decisiones de migración tomadas sin saber qué hay realmente.

-----

## Cómo funciona

### 1. La línea de tiempo

```text
2002  .NET Framework 1.0  ─┐
2005  2.0 (genéricos)      │  Solo Windows, parte del sistema operativo
2010  4.0                  │  ASP.NET Web Forms, MVC 5, WCF, WinForms, WPF
2019  4.8                  │
2022  4.8.1 (final)       ─┘  → mantenimiento: correcciones y seguridad, sin funciones nuevas

2016  .NET Core 1.0 ─┐
2019  .NET Core 3.1  │  Multiplataforma, open source, se instala junto a la app
2020  .NET 5  ───────┤  unificación: desaparece el "Core" del nombre
2023  .NET 8 (LTS)   │
2025  .NET 10 (LTS) ─┘  una versión por año; LTS (3 años) en años impares
```

**.NET Framework no está muerto:** 4.8.1 tiene soporte mientras lo tenga la versión de Windows donde corre. Pero no recibe funciones nuevas, y muchas bibliotecas modernas ya no lo soportan. Las versiones 4.5.2, 4.6 y 4.6.1 perdieron soporte en abril de 2022.

### 2. Diferencias que importan en el día a día

| | .NET Framework 4.x | .NET moderno (5 → 10) |
| --- | --- | --- |
| Sistemas | Solo Windows | Windows, Linux, macOS, contenedores |
| Instalación | Componente de Windows (una versión por máquina) | Runtime por versión, en paralelo, o incluido en la app |
| Web | ASP.NET (`System.Web`): Web Forms, MVC 5, Web API 2 | ASP.NET Core |
| Servicios | WCF (servidor y cliente) | gRPC, REST; WCF solo como cliente o con CoreWCF |
| Escritorio | WinForms, WPF | WinForms, WPF (solo Windows), MAUI |
| Datos | Entity Framework 6, ADO.NET | EF Core, ADO.NET (EF6 también corre) |
| Configuración | `Web.config` / `App.config` (XML) | `appsettings.json`, variables de entorno, opciones tipadas |
| Inyección de dependencias | No integrada (Unity, Autofac, Ninject) | Integrada |
| CLI | MSBuild de Visual Studio | `dotnet build`, `dotnet run`, `dotnet test` |

### 3. Cómo reconocer cada uno en cinco minutos

| Señal | .NET Framework | .NET moderno |
| --- | --- | --- |
| Primera línea del `.csproj` | `<Project ToolsVersion="15.0" xmlns="...msbuild/2003">` | `<Project Sdk="Microsoft.NET.Sdk">` |
| Framework de destino | `<TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>` | `<TargetFramework>net10.0</TargetFramework>` |
| Lista de archivos | `<Compile Include="...">` para cada `.cs` | Implícita (todos los `.cs` de la carpeta) |
| Paquetes | `packages.config` + carpeta `packages/` | `<PackageReference>` en el `.csproj` |
| Arranque web | `Global.asax` + `App_Start/` | `Program.cs` |
| Configuración | `Web.config` con `<system.web>` | `appsettings.json` |
| Atributos del ensamblado | `Properties/AssemblyInfo.cs` | Generados desde el `.csproj` |
| Archivos típicos | `.aspx`, `.ascx`, `.asmx`, `.svc`, `.edmx` | `.razor`, `.cshtml`, `Program.cs` |

Ojo con un caso intermedio: un `.csproj` **estilo SDK** con `<TargetFramework>net48</TargetFramework>` es .NET Framework con el formato de proyecto nuevo. El formato del archivo no dice la plataforma; el TFM sí.

### 4. Los TFM, de un vistazo

| TFM | Plataforma |
| --- | --- |
| `net45`, `net472`, `net48`, `net481` | .NET Framework (sin punto) |
| `netstandard2.0`, `netstandard2.1` | .NET Standard: biblioteca compartible |
| `netcoreapp3.1` | .NET Core (antes de la unificación) |
| `net5.0`, `net8.0`, `net10.0` | .NET moderno (con punto) |
| `net10.0-windows` | .NET moderno con APIs exclusivas de Windows (WinForms, WPF) |

### 5. .NET Standard: el puente

Una biblioteca `netstandard2.0` puede usarse desde .NET Framework 4.6.1+ y desde cualquier .NET moderno. Es la herramienta clave para migrar por partes: mueves la lógica compartida a `netstandard2.0` y la usan a la vez la aplicación vieja y la nueva. Para código nuevo que solo correrá en .NET moderno, no hace falta: apunta directamente a `net10.0`.

### 6. Lo que no existe en .NET moderno

| Tecnología de .NET Framework | Situación en .NET moderno | Reemplazo habitual |
| --- | --- | --- |
| ASP.NET Web Forms | No existe | Blazor, Razor Pages |
| `System.Web` (`HttpContext.Current`, `Session` clásico) | No existe | ASP.NET Core (`IHttpContextAccessor`, sesión de ASP.NET Core) |
| WCF servidor | No incluido | CoreWCF (comunidad), gRPC, REST |
| .NET Remoting, AppDomains múltiples | No existen | gRPC/HTTP, procesos separados, `AssemblyLoadContext` |
| Windows Workflow Foundation | No existe | CoreWF (comunidad), Elsa, Durable Functions |
| `.edmx` (EF6 Database First con diseñador) | EF6 corre, el diseñador no | EF Core con `dbcontext scaffold` |

Para lo que sí existe pero solo en Windows (registro, servicios de Windows, `System.Drawing`), está el **Windows Compatibility Pack** (`Microsoft.Windows.Compatibility`).

-----

## Ejemplo completo

Consola .NET 10 que analiza el contenido de varios `.csproj` y dice qué plataforma usa cada uno. Sirve como lista de verificación para el primer día en un sistema heredado. `Program.cs`:

```csharp
using System.Xml.Linq;

(string Nombre, string Contenido)[] proyectos =
[
    ("Facturacion.Web.csproj", """
        <Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
          <PropertyGroup>
            <TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>
          </PropertyGroup>
          <ItemGroup>
            <Reference Include="System.Web" />
            <Compile Include="Default.aspx.cs" />
          </ItemGroup>
        </Project>
        """),
    ("Tienda.Api.csproj", """
        <Project Sdk="Microsoft.NET.Sdk.Web">
          <PropertyGroup><TargetFramework>net10.0</TargetFramework></PropertyGroup>
        </Project>
        """),
    ("Comun.csproj", """
        <Project Sdk="Microsoft.NET.Sdk">
          <PropertyGroup><TargetFrameworks>netstandard2.0;net8.0</TargetFrameworks></PropertyGroup>
        </Project>
        """),
    ("Escritorio.csproj", """
        <Project Sdk="Microsoft.NET.Sdk">
          <PropertyGroup>
            <TargetFramework>net48</TargetFramework>
            <UseWindowsForms>true</UseWindowsForms>
          </PropertyGroup>
        </Project>
        """)
];

foreach (var (nombre, contenido) in proyectos)
{
    Console.WriteLine($"{nombre,-24} {Analizar(contenido)}");
}

static string Analizar(string contenido)
{
    XElement proyecto = XElement.Parse(contenido);
    var partes = new List<string>();

    // Los proyectos antiguos usan el espacio de nombres de MSBuild: se busca por nombre local
    string? Valor(string elemento) =>
        proyecto.Descendants().FirstOrDefault(e => e.Name.LocalName == elemento)?.Value;

    string? sdk = proyecto.Attribute("Sdk")?.Value;
    partes.Add(sdk is null ? "estilo clásico (pre-SDK)" : $"estilo SDK ({sdk})");

    if (Valor("TargetFrameworkVersion") is { } version)
    {
        partes.Add($".NET Framework {version.TrimStart('v')}");
    }

    string? tfms = Valor("TargetFramework") ?? Valor("TargetFrameworks");
    if (tfms is not null)
    {
        partes.Add(string.Join(", ", tfms.Split(';').Select(Clasificar)));
    }

    if (proyecto.Descendants().Any(e => e.Name.LocalName == "Reference" && (string?)e.Attribute("Include") == "System.Web"))
    {
        partes.Add("usa System.Web (ASP.NET clásico)");
    }
    if (Valor("UseWindowsForms") == "true")
    {
        partes.Add("Windows Forms");
    }

    return string.Join(" | ", partes);
}

static string Clasificar(string tfm) => tfm switch
{
    _ when tfm.StartsWith("netstandard") => $".NET Standard ({tfm})",
    _ when tfm.StartsWith("netcoreapp") => $".NET Core ({tfm})",
    _ when tfm.StartsWith("net") && tfm.Contains('.') => $".NET moderno ({tfm})",
    _ when tfm.StartsWith("net") => $".NET Framework ({tfm})",
    _ => $"desconocido ({tfm})"
};
```

Salida:

```text
Facturacion.Web.csproj   estilo clásico (pre-SDK) | .NET Framework 4.7.2 | usa System.Web (ASP.NET clásico)
Tienda.Api.csproj        estilo SDK (Microsoft.NET.Sdk.Web) | .NET moderno (net10.0)
Comun.csproj             estilo SDK (Microsoft.NET.Sdk) | .NET Standard (netstandard2.0), .NET moderno (net8.0)
Escritorio.csproj        estilo SDK (Microsoft.NET.Sdk) | .NET Framework (net48) | Windows Forms
```

Qué observar:

* `Escritorio.csproj` tiene el formato nuevo pero es **.NET Framework**: lo decide el TFM `net48` (sin punto), no el estilo del archivo.
* `Comun.csproj` apunta a dos frameworks a la vez (*multi-targeting*): la misma biblioteca sirve a la aplicación vieja y a la nueva durante una migración.
* Los proyectos clásicos declaran el espacio de nombres XML de MSBuild; por eso el análisis busca por `LocalName` y no por el nombre completo.
* `System.Web` es la señal más fuerte de ASP.NET clásico: si aparece, ese proyecto **no** se migra cambiando el TFM; hay que reescribir la capa web.

-----

## Errores comunes

**1. Usar la CLI moderna con proyectos clásicos.**
Qué pasa: `dotnet build` falla con `The imported project "...Microsoft.WebApplication.targets" was not found`.
Por qué: los proyectos web clásicos dependen de *targets* que solo instala Visual Studio.
Arreglo: compilar con Visual Studio o con `msbuild` de sus Build Tools (con la carga de trabajo de desarrollo web).

**2. Instalar paquetes sin mirar el TFM.**
Qué pasa: `NU1202: Package X is not compatible with net472`.
Por qué: muchos paquetes modernos ya no publican versiones para .NET Framework.
Arreglo: usar la última versión que soporte `net4x` o `netstandard2.0`, o aislar la funcionalidad en un servicio moderno aparte.

**3. Copiar código de ASP.NET Core en ASP.NET clásico (o al revés).**
Qué pasa: tipos que no existen (`WebApplication`, `IActionResult` de `Microsoft.AspNetCore.Mvc`, `HttpContext.Current`).
Por qué: son frameworks distintos con nombres parecidos.
Arreglo: verificar en la documentación que diga "ASP.NET Core" o "ASP.NET 4.x" según corresponda.

**4. Asumir que "es .NET" significa "se migra fácil".**
Qué pasa: estimaciones de días para migraciones de meses.
Por qué: Web Forms, WCF servidor y `System.Web` no tienen equivalente directo.
Arreglo: inventariar primero qué tecnologías usa cada proyecto (como en el ejemplo) y estimar por tecnología.

**5. Confundir el formato del proyecto con la plataforma.**
Qué pasa: "ya está migrado porque el `.csproj` es corto".
Por qué: un proyecto estilo SDK puede apuntar a `net48`.
Arreglo: mirar el TFM.

-----

## Según la versión de .NET

* **.NET Framework 1.0 (2002) → 4.8.1 (2022):** la plataforma original; 4.8.1 es la versión final, con soporte ligado a Windows.
* **.NET Standard 2.0 (2017):** el puente que más se usa para compartir código entre Framework y Core.
* **.NET Core 1.0 (2016) → 3.1 (2019):** reescritura multiplataforma; en 3.0 llegan WinForms y WPF (solo Windows).
* **.NET 5 (2020):** unificación del nombre; no hay ".NET Framework 5" para no confundir con 4.x.
* **.NET 6, 8 y 10 (LTS):** versiones con soporte de tres años; .NET 10 es la de referencia de esta wiki.

-----

## Cuándo sí y cuándo no

**Mantén .NET Framework cuando:** la aplicación funciona, tiene soporte (4.8.x sobre un Windows soportado), cambia poco y no necesita funciones nuevas de la plataforma. Migrar por migrar es caro.

**Planifica la migración cuando:** necesitas Linux o contenedores, bibliotecas que ya no soportan Framework, rendimiento, o el sistema va a seguir creciendo durante años.

**No empieces nada nuevo en .NET Framework cuando:** puedes elegir. Para código nuevo, .NET moderno siempre.

-----

## Resumen en 5 líneas

1. .NET Framework 4.8.1 es la última versión de la plataforma original: solo Windows y en mantenimiento.
2. .NET moderno (Core → 5 → 10) es otra plataforma, multiplataforma, con una versión por año.
3. Se reconocen por el TFM (`net48` frente a `net10.0`), `Web.config`, `packages.config`, `Global.asax` y `System.Web`.
4. .NET Standard 2.0 permite compartir bibliotecas entre ambos durante una migración.
5. Web Forms, WCF servidor y `System.Web` no existen en .NET moderno: esas partes se reescriben.

-----

## Para profundizar

<details>
<summary>¿Por qué .NET Framework no es open source multiplataforma como .NET Core?</summary>

.NET Framework estaba integrado con Windows (GAC, IIS, el registro, COM) y tenía una regla de compatibilidad muy estricta: una versión nueva se instalaba **encima** de la anterior y tenía que ser compatible con todas las aplicaciones existentes de la máquina. Eso impedía cambios profundos. .NET Core rompió con eso: cada aplicación lleva su versión del runtime (o comparte una de varias instaladas en paralelo), lo que permitió rediseñar, optimizar y llevarlo a Linux sin romper nada en ninguna máquina.

</details>

<details>
<summary>Entity Framework 6 en .NET moderno</summary>

EF 6.4+ corre sobre .NET Core 3.1 y .NET moderno, así que una migración puede mover la aplicación a .NET 10 manteniendo EF6 y migrar a EF Core después, como un paso separado. Lo que no funciona en .NET moderno es el diseñador visual del `.edmx`; el modelo se puede seguir usando, pero conviene pasarlo a Code First o a EF Core con `dbcontext scaffold`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

.NET Framework es la versión original de .NET, solo para Windows, y su última versión es la 4.8.1. .NET moderno, que antes se llamaba .NET Core, es multiplataforma y es la que se usa para proyectos nuevos. Se distinguen por el target framework del proyecto, como `net48` o `net10.0`.

### Respuesta ampliada (semi-senior)

.NET Framework 4.8.1 está en mantenimiento con soporte ligado a Windows; .NET moderno es otra plataforma, multiplataforma, con versiones anuales y LTS cada dos años. En un sistema heredado reviso primero el TFM, el formato del `.csproj`, `packages.config`, `Web.config`, `Global.asax` y referencias a `System.Web`, que indican ASP.NET clásico. Sé qué no existe en .NET moderno (Web Forms, WCF servidor, Remoting, `System.Web`) y que esas partes se reescriben, mientras la lógica de negocio suele poder moverse a bibliotecas `netstandard2.0` compartidas. No migro sin una razón concreta: mantener una aplicación estable en 4.8.1 puede ser la decisión correcta.

### Preguntas frecuentes de seguimiento

**1. ¿.NET Framework sigue teniendo soporte?**
Sí, 4.8.1 mientras esté soportada la versión de Windows donde corre; no recibe funciones nuevas.

**2. ¿Qué es .NET Standard?**
Una especificación de APIs comunes que permite que una biblioteca funcione en .NET Framework y en .NET moderno.

**3. ¿Cómo sabes si un proyecto es Framework?**
Por el TFM: `net48` (sin punto) o `TargetFrameworkVersion v4.x`; además, `Web.config`, `Global.asax` o `System.Web`.

**4. ¿Por qué no existe .NET Framework 5?**
Porque .NET 5 es la continuación de .NET Core; se saltó el 4 para no confundirlo con .NET Framework 4.x.

-----

## Práctica

**Ejercicio 1.** Amplía `Analizar` para detectar `packages.config` como señal clásica cuando, en el proyecto, aparece `<None Include="packages.config" />` o `<Content Include="packages.config" />`.

<details>
<summary>Solución</summary>

```csharp
bool usaPackagesConfig = proyecto.Descendants()
    .Any(e => e.Name.LocalName is "None" or "Content"
              && (string?)e.Attribute("Include") == "packages.config");
if (usaPackagesConfig)
{
    partes.Add("paquetes en packages.config");
}
```

Los proyectos que migran a `<PackageReference>` dejan de necesitar la carpeta `packages/` y resuelven dependencias transitivas automáticamente. Visual Studio tiene la opción "Migrar packages.config a PackageReference" (en el menú contextual de `packages.config`), excepto para proyectos web clásicos de ASP.NET, donde no está soportada.

</details>

**Ejercicio 2.** Tu equipo tiene una biblioteca de cálculo de impuestos en `net472` que usan una aplicación Web Forms y una API nueva en .NET 10. ¿Qué TFM le pondrías y qué condición debe cumplir el código?

<details>
<summary>Solución</summary>

`netstandard2.0` (o *multi-targeting* `net472;net10.0`). Con `netstandard2.0`, la misma DLL funciona en la aplicación Web Forms (que corre en .NET Framework 4.7.2, compatible con .NET Standard 2.0) y en la API .NET 10.

La condición es que el código **no use APIs exclusivas de .NET Framework**: nada de `System.Web`, `HttpContext.Current`, `ConfigurationManager` de `Web.config` ni WCF. La lógica de impuestos debe ser pura: recibe datos y devuelve resultados. Si hoy lee configuración de `Web.config` directamente, primero hay que pasarle esos valores por parámetro o por constructor.

</details>

-----

## Siguiente lección

[ASP.NET Web Forms](02-ASP%20NET%20Web%20Forms.md)
