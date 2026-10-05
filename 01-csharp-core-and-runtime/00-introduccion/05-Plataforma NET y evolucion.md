# Plataforma .NET y su evolución

## En una frase

.NET es una plataforma abierta y multiplataforma formada por runtime, bibliotecas, compiladores, SDK y modelos de aplicación; no es lo mismo que .NET Framework (su antecesor solo para Windows) ni que C# (uno de sus lenguajes).

-----

## Antes de empezar

Conviene que ya sepas:

* Qué separa a C# de .NET: [Qué es C# y .NET](01-Que%20es%20CSharp%20y%20.NET.md).
* Instalar el SDK y crear un proyecto: [Preparar el entorno](02-Preparar%20el%20entorno.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Runtime:** lo mínimo necesario para ejecutar una aplicación ya compilada.
* **Workload:** conjunto opcional de herramientas del SDK para un tipo de aplicación (MAUI, WebAssembly).
* **LTS (*Long Term Support*):** versión con 3 años de soporte.
* **STS (*Standard Term Support*):** versión con 2 años de soporte.
* **TFM (*Target Framework Moniker*):** identificador de la plataforma destino, como `net10.0`.

-----

## El problema

Buscas "cómo leer la sesión en .NET" y copias el primer resultado, escrito en 2012:

```csharp
using System.Web;

var usuario = HttpContext.Current.Session["usuario"];
```

En un proyecto `net10.0` no compila:

```text
error CS0234: The type or namespace name 'Web' does not exist in the namespace 'System'
```

`System.Web` era el corazón de ASP.NET sobre .NET Framework. ASP.NET Core no lo incluye: tiene otro pipeline, otra forma de acceder al contexto y otra configuración.

Otro caso. Este código compila en .NET 10, pero el compilador avisa:

```csharp
using Microsoft.Win32;

var ruta = Registry.GetValue(@"HKEY_LOCAL_MACHINE\SOFTWARE\Orders", "DataPath", null);
```

```text
warning CA1416: This call site is reachable on all platforms. 'Registry.GetValue(string, string?, object?)' is only supported on: 'windows'.
```

Si ignoras la advertencia y despliegas en un contenedor Linux, la llamada lanza `PlatformNotSupportedException`.

Las consecuencias de no distinguir las generaciones de .NET son concretas:

* código de tutoriales viejos que no compila;
* paquetes NuGet que solo apuntan a .NET Framework (`net48`) y fallan en ejecución;
* la creencia de que "multiplataforma" significa que cualquier API funciona en cualquier sistema;
* decisiones de migración basadas en el nombre ("es Core, es moderno") y no en las dependencias reales.

-----

## Cómo funciona

### 1. Tres generaciones, un solo producto vigente

```text
2002 ─── .NET Framework 1.0 ──── ... ──── 4.8.1 (2022)    solo Windows, en mantenimiento
                                                          (se actualiza junto con Windows)

2016 ─── .NET Core 1.0 ── 2.x ── 3.1 ─┐
                                      ├─► .NET 5 (2020) ── 6 ── 7 ── 8 ── 9 ── 10 (2025)
Mono / Xamarin ───────────────────────┘     multiplataforma, abierto, versión anual
```

* **.NET Framework** nació orientado a Windows. Sigue recibiendo parches de seguridad, pero no nuevas características.
* **.NET Core** fue la reescritura abierta, modular y multiplataforma.
* **Desde .NET 5**, el producto moderno se llama simplemente **.NET**. Se saltó el número 4 para no confundirlo con .NET Framework 4.x. No existe ".NET Core 8" ni ".NET Core 10".

### 2. Los componentes

| Componente | Responsabilidad | Ejemplo |
| --- | --- | --- |
| Runtime (CLR) | Ejecutar código, administrar memoria, manejar excepciones | CoreCLR |
| Bibliotecas (BCL) | APIs reutilizables | `System.Collections`, `System.Text.Json` |
| Compiladores | Traducir el lenguaje a código intermedio | Roslyn (C#, VB), F# |
| SDK | Crear, compilar, probar y publicar | CLI `dotnet`, MSBuild |
| Stacks (modelos de aplicación) | Resolver un tipo de aplicación | ASP.NET Core, WPF, Windows Forms, MAUI |

Un **stack** se apoya en la plataforma, pero no es la plataforma. ASP.NET Core es un stack web; no es "otro .NET".

### 3. El TFM dice para qué plataforma compilas

```xml
<TargetFramework>net10.0</TargetFramework>
```

| TFM | Plataforma | Comentario |
| --- | --- | --- |
| `net48` | .NET Framework 4.8 | Solo Windows |
| `netstandard2.0` | Contrato compartido | Bibliotecas que deben servir a Framework y a .NET moderno |
| `net10.0` | .NET 10 | Multiplataforma |
| `net10.0-windows` | .NET 10 con APIs de Windows | WPF, Windows Forms |

Un paquete NuGet que solo trae `lib/net48/` no está pensado para `net10.0`. El restore lo acepta por compatibilidad, pero emite la advertencia NU1701: el paquete se compiló para .NET Framework y **puede fallar en ejecución** si usa APIs que .NET moderno no tiene. Trata NU1701 como una señal para buscar una versión compatible, no como ruido.

### 4. Multiplataforma no significa "todas las APIs en todas partes"

El runtime y la mayoría de la BCL funcionan en Windows, Linux y macOS. Algunas APIs dependen del sistema: el Registro, COM, los servicios de Windows o WPF.

El SDK incluye un analizador de compatibilidad de plataforma. Por eso aparece CA1416 en el ejemplo de `Registry`. La advertencia no es ruido: te dice que ese código no es portable.

```csharp
if (OperatingSystem.IsWindows())
{
    var ruta = Registry.GetValue(@"HKEY_LOCAL_MACHINE\SOFTWARE\Orders", "DataPath", null);
}
```

Con la comprobación `OperatingSystem.IsWindows()`, el analizador sabe que esa rama solo corre en Windows y la advertencia desaparece.

### 5. Workloads: herramientas opcionales del SDK

El SDK base trae lo necesario para consola, bibliotecas y ASP.NET Core. Para otros modelos necesitas instalar un *workload*:

```text
dotnet workload list
dotnet workload install maui
dotnet workload install wasm-tools
```

Así el SDK no obliga a todos a descargar las herramientas de Android o iOS.

### 6. Soporte: versión anual, LTS y STS

.NET publica una versión principal cada noviembre.

| Tipo | Versiones | Soporte |
| --- | --- | --- |
| LTS | Pares: 8, 10 | 3 años |
| STS | Impares: 9, 11 | 2 años (desde .NET 9; antes eran 18 meses) |

Ambas reciben parches mensuales. **Estar "en .NET 10" no alcanza: debes estar en el último parche de .NET 10.** Una versión sin parches es una versión con vulnerabilidades conocidas.

### 7. "Uso empresarial" no es una edición

No existe un ".NET Enterprise". "Empresarial" describe exigencias: seguridad, observabilidad, automatización, pruebas, soporte y operación mantenible. Esas propiedades salen del diseño y de las prácticas del equipo, no del producto.

-----

## Ejemplo completo

Aplicación de consola que muestra contra qué compilaste y sobre qué se ejecuta:

```csharp
using System.Runtime.InteropServices;

Console.WriteLine($"TFM:          {AppContext.TargetFrameworkName}");
Console.WriteLine($"Runtime:      {RuntimeInformation.FrameworkDescription}");
Console.WriteLine($"Arquitectura: {RuntimeInformation.ProcessArchitecture}");
Console.WriteLine($"Windows:      {OperatingSystem.IsWindows()}");
Console.WriteLine($"Linux:        {OperatingSystem.IsLinux()}");
```

Salida en Windows x64 con un proyecto `net10.0`:

```text
TFM:          .NETCoreApp,Version=v10.0
Runtime:      .NET 10.0.x
Arquitectura: X64
Windows:      True
Linux:        False
```

Qué observar:

* `Runtime` muestra el parche instalado: con el runtime 10.0.5 verás `.NET 10.0.5`. Por eso el ejemplo usa `10.0.x`.
* El nombre interno del TFM conserva `.NETCoreApp` por compatibilidad. No cambia el nombre comercial: .NET.
* El mismo `.dll` ejecutado en Linux imprimiría `Windows: False` y `Linux: True`. El código intermedio es portable; el sistema donde corre cambia.

-----

## Errores comunes

**1. Llamar ".NET Core" a .NET 10.**
Qué pasa: buscas documentación de ".NET Core" y encuentras material de 2017, con APIs y plantillas que ya no aplican.
Por qué: el nombre cambió en .NET 5; ".NET Core" quedó para las versiones 1.0 a 3.1.
Arreglo: di ".NET 10" y filtra la documentación por versión.

**2. Copiar código de .NET Framework sin revisarlo.**
Qué pasa: CS0234 por `System.Web`, o APIs que compilan pero lanzan `PlatformNotSupportedException`.
Por qué: ASP.NET Core y .NET moderno no incluyen todo lo de .NET Framework.
Arreglo: comprueba a qué plataforma apunta el ejemplo antes de copiarlo.

**3. Ignorar CA1416.**
Qué pasa: la app funciona en tu Windows y falla al desplegarse en Linux.
Por qué: la API solo existe en un sistema operativo.
Arreglo: protege la llamada con `OperatingSystem.IsWindows()` o usa una alternativa portable.

**4. Creer que LTS significa "instalar y olvidar".**
Qué pasa: la aplicación corre con vulnerabilidades ya corregidas.
Por qué: LTS define cuánto tiempo hay parches, no que no los necesites.
Arreglo: actualiza el parche mensual del runtime y de las imágenes base.

**5. Migrar de .NET Framework por moda.**
Qué pasa: el costo supera el beneficio, o la migración se bloquea a mitad de camino.
Por qué: Web Forms, WCF servidor o bibliotecas COM no tienen equivalente directo.
Arreglo: inventaria dependencias y tecnologías antes de decidir; a veces conviene migrar por partes.

-----

## Según la versión de .NET

* **.NET Framework 1.0 (2002):** primera versión pública de la plataforma, solo Windows.
* **.NET Framework 4.8.1 (2022):** última versión de Framework; solo recibe parches.
* **.NET Core 1.0 (2016):** reescritura abierta y multiplataforma.
* **.NET 5 (2020):** unifica el nombre: desaparece "Core" del producto.
* **.NET 6 (2021):** primera LTS unificada; integra MAUI y el hosting mínimo.
* **.NET 9 (2024):** las versiones STS pasan a tener 2 años de soporte.
* **.NET 10 (2025):** versión LTS de referencia de esta wiki, con soporte hasta noviembre de 2028.

-----

## Cuándo sí y cuándo no

**Usa .NET moderno cuando:** creas software nuevo, necesitas ejecutar en Linux o contenedores, o quieres las mejoras de rendimiento y de lenguaje de cada versión.

**Conserva .NET Framework cuando:** una aplicación existente depende de Web Forms, WCF servidor, COM u otra tecnología exclusiva de Windows, y la migración no tiene un beneficio que justifique su costo.

**Elige LTS cuando:** el equipo actualiza pocas veces por año. **Elige STS cuando:** puedes actualizar cada año y quieres las novedades antes.

-----

## Resumen en 5 líneas

1. C# es un lenguaje; .NET es la plataforma que lo compila y ejecuta.
2. .NET Framework es la línea histórica para Windows; desde .NET 5 el producto moderno se llama .NET.
3. El TFM (`net10.0`) decide para qué plataforma compilas y qué paquetes puedes usar.
4. Multiplataforma no significa que toda API funcione en todo sistema: CA1416 te avisa.
5. LTS y STS definen cuánto dura el soporte; los parches mensuales siguen siendo obligatorios.

-----

## Para profundizar

<details>
<summary>.NET Standard: el puente entre generaciones</summary>

.NET Standard es una especificación de APIs, no una implementación. Una biblioteca `netstandard2.0` puede usarse desde .NET Framework 4.6.1+ y desde cualquier .NET moderno. Fue clave durante la transición. Para bibliotecas nuevas que solo apuntan a .NET moderno, conviene compilar directamente para `net8.0` o `net10.0`: tienes más APIs y mejor rendimiento. Mantén `netstandard2.0` solo si todavía tienes consumidores en .NET Framework.

</details>

<details>
<summary>Multi-targeting: una biblioteca para varias plataformas</summary>

Un proyecto puede compilar para varios TFM a la vez:

```xml
<TargetFrameworks>net48;net10.0</TargetFrameworks>
```

El paquete NuGet resultante trae una carpeta `lib/` por cada destino. Dentro del código puedes usar `#if NET48` para las diferencias. Es útil durante una migración gradual, pero cada TFM extra es un entorno más que probar.

</details>

-----

## En entrevista

### Respuesta corta (junior)

.NET es una plataforma para crear y ejecutar aplicaciones; C# es uno de sus lenguajes. .NET Framework es la línea histórica para Windows. Desde .NET 5 la línea moderna, multiplataforma y abierta se llama solo .NET. Las versiones pares son LTS, con 3 años de soporte.

### Respuesta ampliada (semi-senior)

.NET reúne runtime, BCL, compiladores, SDK y stacks como ASP.NET Core. El TFM define el destino de compilación y condiciona qué paquetes puedo usar. Para software nuevo uso .NET moderno, preferentemente LTS, y aplico los parches mensuales. Sé que multiplataforma no es portabilidad total: reviso CA1416 y las dependencias nativas. Para migrar desde .NET Framework primero inventario tecnologías sin equivalente (Web Forms, WCF servidor, COM) y evalúo una migración por partes, con .NET Standard o multi-targeting en las bibliotecas compartidas.

### Preguntas frecuentes de seguimiento

**1. ¿ASP.NET Core es otro runtime?**
No. Es un stack web que se ejecuta sobre .NET.

**2. ¿Por qué no existe .NET 4?**
Se saltó el número para no confundirlo con .NET Framework 4.x.

**3. ¿LTS significa que puedo ignorar parches?**
No. Debes mantenerte en el último parche de la versión.

**4. ¿Qué es .NET Standard?**
Una especificación de APIs que permite compartir bibliotecas entre .NET Framework y .NET moderno.

-----

## Práctica

**Ejercicio 1.** Clasifica cada elemento como lenguaje, plataforma, stack o biblioteca: C#, F#, .NET 10, .NET Framework 4.8, ASP.NET Core, MAUI, Entity Framework Core, `System.Text.Json`.

<details>
<summary>Solución</summary>

| Elemento | Categoría |
| --- | --- |
| C#, F# | Lenguajes |
| .NET 10, .NET Framework 4.8 | Plataformas |
| ASP.NET Core, MAUI | Stacks (modelos de aplicación) |
| Entity Framework Core, `System.Text.Json` | Bibliotecas |

EF Core se distribuye como paquete NuGet; `System.Text.Json` forma parte de la BCL. Ninguno de los dos es una plataforma.

</details>

**Ejercicio 2.** Tu equipo mantiene una aplicación Web Forms en .NET Framework 4.8 y una biblioteca de cálculo de impuestos que usan otras tres aplicaciones. Quieren empezar a migrar. ¿Por dónde empezarías?

<details>
<summary>Solución</summary>

Por la biblioteca. Se puede compilar con `<TargetFrameworks>net48;net10.0</TargetFrameworks>` (o `netstandard2.0`) para que la sigan usando las aplicaciones viejas y las nuevas. Web Forms no existe en .NET moderno: esa interfaz debe reescribirse (Razor Pages, MVC o Blazor), así que es la parte más cara y conviene planificarla por pantallas, no de una sola vez.

</details>

-----

## Siguiente lección

[CLR, compilación y sistema de tipos](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md)
