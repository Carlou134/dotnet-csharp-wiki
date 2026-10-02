# Preparar el entorno

## En una frase

Para programar en C# necesitas el **.NET SDK** (compilador y herramientas) y un editor: **Visual Studio Code** con la extensión C# Dev Kit, **Visual Studio** (solo Windows) o **JetBrains Rider**.

-----

## Antes de empezar

Conviene que ya sepas:

* Abrir una terminal (PowerShell, Terminal de macOS o la de Linux) y escribir comandos.
* La diferencia entre SDK y runtime, vista en [Qué es C# y .NET](01-Que%20es%20CSharp%20y%20.NET.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **CLI `dotnet`:** la herramienta de línea de comandos que trae el SDK para crear, compilar y ejecutar proyectos.
* **IDE:** entorno de desarrollo integrado: editor, depurador y herramientas en un solo programa.
* **Plantilla:** un esqueleto de proyecto listo para usar (consola, API, librería, pruebas).
* **Archivo `.csproj`:** el archivo del proyecto. Define, entre otras cosas, la versión de .NET que usa.
* **Target framework (TFM):** la versión de .NET para la que se compila el proyecto, por ejemplo `net10.0`.
* **IntelliSense:** el autocompletado inteligente del editor.

-----

## El problema

Puedes escribir C# en el Bloc de notas, pero no tendrías compilador, ni autocompletado, ni depurador, ni detección de errores mientras escribes. Programar así es lento y frustrante.

Por otro lado, un proyecto real tiene varios archivos, dependencias y una versión concreta de .NET. Necesitas una herramienta que los organice y los compile siempre de la misma forma, en tu máquina y en la de tu equipo.

-----

## Cómo funciona

### 1. Instalar el .NET SDK

Descárgalo desde [dotnet.microsoft.com/download](https://dotnet.microsoft.com/download). Elige la versión **LTS** más reciente (hoy, .NET 10).

En Windows también puedes usar `winget`:

```powershell
winget install Microsoft.DotNet.SDK.10
```

Verifica la instalación:

```bash
dotnet --version        # versión del SDK que se usará en esta carpeta
dotnet --list-sdks      # todos los SDK instalados
dotnet --info           # información completa del entorno
```

### 2. Crear y ejecutar un proyecto desde la terminal

```bash
dotnet new console -o MiPrimeraApp   # crea la carpeta MiPrimeraApp con un proyecto de consola
cd MiPrimeraApp
dotnet run                           # compila y ejecuta
```

Verás `Hello, World!` en la terminal. La plantilla generó dos archivos importantes:

```text
MiPrimeraApp/
├── MiPrimeraApp.csproj   ← configuración del proyecto
└── Program.cs            ← tu código
```

### 3. El archivo `.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

</Project>
```

| Propiedad | Qué hace |
| --- | --- |
| `OutputType` | `Exe` genera un ejecutable; sin ella se genera una librería. |
| `TargetFramework` | La versión de .NET. Cámbiala aquí si necesitas otra (por ejemplo `net8.0`). |
| `ImplicitUsings` | Importa automáticamente los espacios de nombres más comunes (`System`, `System.Linq`...). |
| `Nullable` | Activa las advertencias de referencias nulas. Se ve en [Tu primer programa](03-Tu%20primer%20programa.md). |

### 4. Otras plantillas

```bash
dotnet new list                 # muestra todas las plantillas instaladas
dotnet new classlib -o MiLib    # librería de clases
dotnet new webapi -o MiApi      # API web con ASP.NET Core
dotnet new xunit -o MisPruebas  # proyecto de pruebas unitarias
```

### 5. Elegir editor

| Editor | Plataforma | Costo | Cuándo elegirlo |
| --- | --- | --- | --- |
| **VS Code + C# Dev Kit** | Windows, macOS, Linux | Gratis (Dev Kit gratis para uso individual) | Liviano, y si ya lo usas para frontend. |
| **Visual Studio Community** | Solo Windows | Gratis para individuos y equipos chicos | El depurador y las herramientas más completos. |
| **JetBrains Rider** | Windows, macOS, Linux | Gratis para uso no comercial | Refactorizaciones potentes y multiplataforma. |

Visual Studio para Mac fue retirado en 2024; en macOS usa VS Code o Rider.

**VS Code:** instala la extensión **C# Dev Kit** (incluye la extensión C#). Abre la carpeta del proyecto (`code .`) y el editor detecta el `.csproj`.

**Visual Studio:** en el instalador, marca la carga de trabajo **"Desarrollo de ASP.NET y web"** (y "Desarrollo de escritorio de .NET" si vas a hacer apps de escritorio). Luego, *Crear un proyecto → Aplicación de consola*.

### 6. Personalizar VS Code para C#

En `settings.json` (Ctrl + Shift + P → "Preferences: Open User Settings (JSON)"):

```json
{
  "[csharp]": {
    "editor.tabSize": 4,
    "editor.formatOnSave": true
  },
  "editor.semanticHighlighting.enabled": true
}
```

Puedes crear **snippets** propios (Ctrl + Shift + P → "Snippets: Configure Snippets" → `csharp.json`):

```json
{
  "Imprimir en consola": {
    "prefix": "cw",
    "body": ["Console.WriteLine($1);"],
    "description": "Console.WriteLine"
  }
}
```

Al escribir `cw` y presionar Tab, se inserta `Console.WriteLine();` con el cursor entre paréntesis.

### 7. Atajos de teclado útiles

| Acción | Visual Studio | VS Code |
| --- | --- | --- |
| Comentar selección | Ctrl + K, Ctrl + C | Ctrl + K, Ctrl + C |
| Descomentar selección | Ctrl + K, Ctrl + U | Ctrl + K, Ctrl + U |
| Formatear documento | Ctrl + K, Ctrl + D | Shift + Alt + F |
| Renombrar en todo el código | Ctrl + R, Ctrl + R | F2 |
| Acciones rápidas (generar constructor, implementar interfaz, importar `using`) | Ctrl + . | Ctrl + . |
| Buscar y reemplazar | Ctrl + H | Ctrl + H |
| Ir a la definición | F12 | F12 |
| Ejecutar con depurador | F5 | F5 |

Renombrar con F2 o Ctrl + R, R es mejor que buscar y reemplazar: entiende el código y solo cambia ese símbolo, no cualquier texto igual.

-----

## Ejemplo completo

Flujo completo desde cero en la terminal:

```bash
dotnet --version
dotnet new console -o Saludo
cd Saludo
code .                # abre VS Code en la carpeta (opcional)
```

Reemplaza el contenido de `Program.cs`:

```csharp
Console.WriteLine("Mi entorno funciona.");
Console.WriteLine($"Versión de .NET: {Environment.Version}");
```

Ejecuta:

```bash
dotnet run
```

Salida aproximada:

```text
Mi entorno funciona.
Versión de .NET: 10.0.0
```

-----

## Errores comunes

**1. `dotnet` no se reconoce como comando.**
Qué pasa: la terminal no encuentra el programa.
Por qué: la terminal se abrió antes de instalar, o la carpeta de .NET no quedó en el `PATH`.
Arreglo: cierra y vuelve a abrir la terminal; si persiste, reinstala el SDK.

**2. `NETSDK1045: The current .NET SDK does not support targeting .NET X`.**
Qué pasa: el proyecto pide una versión de .NET más nueva que tu SDK.
Por qué: el `TargetFramework` del `.csproj` es mayor que la versión del SDK instalado.
Arreglo: instala el SDK correspondiente o baja el `TargetFramework`.

**3. Ejecutar `dotnet run` fuera de la carpeta del proyecto.**
Qué pasa: error "Couldn't find a project to run".
Por qué: `dotnet run` busca un `.csproj` en la carpeta actual.
Arreglo: `cd` a la carpeta del proyecto o usa `dotnet run --project ruta/al/proyecto`.

**4. VS Code no muestra autocompletado.**
Qué pasa: el código se ve sin colores semánticos ni sugerencias.
Por qué: falta la extensión C# Dev Kit, o abriste un archivo suelto en lugar de la carpeta del proyecto.
Arreglo: instala la extensión y abre la carpeta que contiene el `.csproj`.

-----

## Según la versión de C#

* **.NET 6 (C# 10):** las plantillas de consola pasaron a usar *top-level statements* (sin `class Program` ni `Main` visibles) e `ImplicitUsings`. Si sigues un tutorial antiguo verás la versión larga; las dos son válidas.
* **.NET 10:** puedes ejecutar un archivo `.cs` suelto, sin `.csproj`: `dotnet run app.cs`. Es útil para scripts y pruebas rápidas.
* Para generar la versión con `Main` explícito en un proyecto nuevo: `dotnet new console --use-program-main`.

-----

## Cuándo sí y cuándo no

**Usa la CLI `dotnet` cuando:**

* Quieres entender qué hace el IDE por detrás. Todo lo que hace Visual Studio se puede hacer con la CLI.
* Trabajas en integración continua (CI/CD), donde no hay interfaz gráfica.

**Usa un IDE completo cuando:**

* Necesitas depurar con puntos de interrupción, inspeccionar variables o analizar rendimiento.
* La solución tiene muchos proyectos y quieres navegar entre ellos con facilidad.

-----

## Resumen en 5 líneas

1. Instala el .NET SDK (versión LTS) y verifica con `dotnet --version`.
2. `dotnet new console -o Nombre` crea un proyecto; `dotnet run` lo compila y ejecuta.
3. El `.csproj` define la versión de .NET en `TargetFramework`.
4. Editores: VS Code + C# Dev Kit, Visual Studio (Windows) o Rider.
5. Ctrl + . y F2 (o Ctrl + R, R) son los atajos que más tiempo ahorran.

-----

## Para profundizar

<details>
<summary>global.json: fijar la versión del SDK</summary>

Si tienes varios SDK instalados, `dotnet` usa el más nuevo. Para obligar a un proyecto a usar una versión concreta, crea un `global.json` en la raíz:

```bash
dotnet new globaljson --sdk-version 10.0.100
```

Es útil en equipos para que todos compilen con el mismo SDK.

</details>

<details>
<summary>Qué hace `dotnet run` paso a paso</summary>

`dotnet run` hace `dotnet restore` (descarga paquetes NuGet), luego `dotnet build` (compila a `bin/Debug/net10.0/`) y finalmente ejecuta el resultado. Puedes ejecutar cada paso por separado para entender dónde falla algo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para desarrollar en C# instalo el .NET SDK, que trae el compilador y la CLI `dotnet`. Creo proyectos con `dotnet new`, los ejecuto con `dotnet run` y uso Visual Studio, VS Code o Rider como editor.

### Respuesta ampliada (semi-senior)

El SDK incluye el compilador Roslyn, MSBuild, el runtime y la CLI. El `.csproj` es un archivo de MSBuild que define el `TargetFramework`, las referencias a paquetes NuGet y opciones como `Nullable` e `ImplicitUsings`. `dotnet run` encadena restore, build y ejecución. En equipos se fija el SDK con `global.json` para builds reproducibles, y en CI se usa la CLI directamente, sin IDE.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `dotnet build` y `dotnet publish`?**
`build` compila para desarrollo. `publish` prepara la app para desplegar: incluye dependencias y permite opciones como un solo archivo, AOT o una plataforma de destino concreta.

**2. ¿Para qué sirve `TargetFramework`?**
Indica contra qué versión de .NET se compila, y por lo tanto qué APIs y qué versión de C# están disponibles por defecto.

**3. ¿Qué es NuGet?**
El gestor de paquetes de .NET. Se agrega un paquete con `dotnet add package Nombre`.

-----

## Práctica

**Ejercicio 1.** Crea un proyecto de consola llamado `Practica01`, cambia el mensaje por tu nombre y ejecútalo solo con la terminal.

<details>
<summary>Solución</summary>

```bash
dotnet new console -o Practica01
cd Practica01
# editar Program.cs: Console.WriteLine("Hola, soy Ana");
dotnet run
```

</details>

**Ejercicio 2.** Abre el `.csproj` de ese proyecto y responde: ¿para qué versión de .NET compila? ¿Está activado `Nullable`?

<details>
<summary>Solución</summary>

La versión está en `<TargetFramework>` (por ejemplo `net10.0`). `Nullable` está activado si aparece `<Nullable>enable</Nullable>`, que es el valor por defecto en las plantillas modernas.

</details>

-----

## Siguiente lección

[Tu primer programa](03-Tu%20primer%20programa.md)
