# SDK, CLI, proyectos y NuGet

## En una frase

El SDK y la CLI `dotnet` convierten la descripción de un proyecto (`.csproj`) en artefactos reproducibles: MSBuild orquesta el proceso, NuGet resuelve las dependencias y `global.json` fija qué SDK hace el trabajo.

-----

## Antes de empezar

Conviene que ya sepas:

* Crear y ejecutar un proyecto, y leer un `.csproj` básico: [Preparar el entorno](02-Preparar%20el%20entorno.md).
* Qué es un TFM: [Plataforma .NET y su evolución](05-Plataforma%20NET%20y%20evolucion.md).
* Qué produce el compilador: [CLR, compilación y sistema de tipos](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **MSBuild:** motor que evalúa el `.csproj` y ejecuta los pasos de restore, compilación y publicación.
* **Restore:** paso que resuelve y descarga las dependencias declaradas.
* **PackageReference:** dependencia NuGet declarada en el proyecto.
* **Activos de restore (*assets*):** grafo de dependencias resuelto que se guarda en `obj/`.
* **global.json:** archivo que fija el SDK que usa la CLI.

-----

## El problema

En tu máquina todo compila. En el servidor de integración continua (CI), el mismo commit falla:

```text
error NETSDK1045: The current .NET SDK does not support targeting .NET 10.0.
Either target .NET 9.0 or lower, or use a version of the .NET SDK that supports .NET 10.0.
```

El agente de CI tenía instalado el SDK 9. Tu proyecto apunta a `net10.0`. Nadie había declarado qué SDK necesita el repositorio.

Al día siguiente otro compañero clona el repositorio y aparecen conflictos en `bin/Debug/net10.0/Orders.dll` y `obj/project.assets.json`: alguien los subió a Git. Y una semana después, producción se despliega con `dotnet run` desde el código fuente, porque "en local funciona así".

Las consecuencias:

* compilaciones que dependen de qué SDK hay instalado en cada máquina;
* conflictos en archivos generados, con rutas de otra computadora;
* despliegues lentos y no reproducibles que compilan en el servidor de producción;
* dependencias que cambian de versión sin que nadie lo decida.

-----

## Cómo funciona

### 1. SDK frente a runtime

```text
dotnet --list-sdks        SDK instalados (para desarrollar)
dotnet --list-runtimes    runtimes instalados (para ejecutar)
dotnet --info             resumen: SDK activo, runtimes, sistema operativo
```

| | SDK | Runtime |
| --- | --- | --- |
| Contiene | CLI, Roslyn, MSBuild, NuGet, plantillas y un runtime | Solo el runtime y la BCL |
| Sirve para | Crear, compilar, probar, publicar | Ejecutar aplicaciones ya compiladas |
| Se versiona como | `10.0.100`, `10.0.101`, `10.0.200` | `10.0.0`, `10.0.1`, `10.0.2` |

Un servidor de producción solo necesita el runtime. Con una publicación *self-contained* ni siquiera eso: el runtime viaja dentro de la aplicación.

### 2. `global.json`: qué SDK usa este repositorio

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

* `version`: SDK mínimo.
* `rollForward: latestFeature`: acepta cualquier SDK `10.0.x` igual o posterior, pero no salta a 11.

`global.json` y el TFM resuelven preguntas distintas:

```text
global.json      ¿QUÉ SDK compila?          10.0.100 o superior dentro de 10.0
<TargetFramework> ¿PARA QUÉ runtime compilo?  net10.0
```

Un SDK 10 puede compilar para `net8.0`. Un SDK 9 no puede compilar para `net10.0`: eso era el NETSDK1045.

### 3. MSBuild evalúa el proyecto

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Humanizer" Version="2.14.1" />
  </ItemGroup>
</Project>
```

El `.csproj` es corto porque `Sdk="Microsoft.NET.Sdk"` importa cientos de propiedades y objetivos por defecto. MSBuild trabaja con tres conceptos:

| Concepto | Ejemplo | Qué es |
| --- | --- | --- |
| Propiedad | `TargetFramework` | Un valor con nombre |
| Ítem | `PackageReference`, los `.cs` | Una lista de entradas |
| Objetivo (*target*) | `Restore`, `Build`, `Publish` | Una tarea que se ejecuta |

Los archivos `.cs` de la carpeta se incluyen solos: no hace falta listarlos.

### 4. El ciclo de comandos

| Comando | Qué hace | Cuándo |
| --- | --- | --- |
| `dotnet restore` | Resuelve paquetes y escribe `obj/project.assets.json` | Implícito en build, test, run y publish |
| `dotnet build` | Compila a `bin/<Configuración>/<TFM>/` | Desarrollo y CI |
| `dotnet run` | Compila si hace falta y ejecuta | Solo desarrollo |
| `dotnet test` | Compila y ejecuta las pruebas | Desarrollo y CI |
| `dotnet publish` | Produce la salida desplegable (Release por defecto desde .NET 8) | CI, despliegue |
| `dotnet clean` | Borra las salidas conocidas por MSBuild | Diagnóstico |

```text
            restore           build                 publish
.csproj ──────────► obj/ ──────────► bin/Debug/... ──────────► bin/Release/net10.0/publish/
          (assets)         (compilación)               (solo lo que se despliega)
```

### 5. `bin/` y `obj/`: derivados, nunca en Git

* `obj/` contiene los activos del restore y archivos intermedios.
* `bin/` contiene las salidas por configuración (`Debug`, `Release`) y TFM.

Ambos se regeneran desde el `.csproj` y el código. `dotnet new gitignore` crea un `.gitignore` que los excluye.

### 6. NuGet: dependencias directas y transitivas

En .NET 10 la CLI admite la forma sustantivo-verbo:

```text
dotnet package add Humanizer --version 2.14.1
dotnet package list --include-transitive
```

La forma anterior, `dotnet add package`, sigue funcionando y es la que verás en material previo.

Una **dependencia directa** es la que declaras. Una **transitiva** llega porque tu dependencia la necesita. Ambas terminan en tu aplicación, y ambas pueden traer vulnerabilidades:

```text
dotnet package list --vulnerable --include-transitive
```

Los paquetes se descargan a la caché global (`~/.nuget/packages`), no al repositorio. El proyecto solo guarda la referencia.

### 7. IDE y CLI usan el mismo SDK

Visual Studio, Rider y VS Code invocan el mismo SDK y MSBuild. Si algo compila en el IDE y falla en la CLI, la diferencia casi siempre está en el SDK seleccionado o en la configuración (`Debug`/`Release`), no en "el IDE". La CLI es la referencia: es lo que ejecuta tu CI.

-----

## Ejemplo completo

```text
dotnet new console -n Orders
cd Orders
dotnet new globaljson --sdk-version 10.0.100 --roll-forward latestFeature
dotnet new gitignore
dotnet package add Humanizer --version 2.14.1
```

Reemplaza el contenido de `Program.cs`:

```csharp
using Humanizer;

string evento = "PedidoEnviadoAlCliente";
Console.WriteLine(evento.Humanize());
```

```text
dotnet run
```

Salida de la aplicación:

```text
Pedido enviado al cliente
```

Ahora publica:

```text
dotnet publish -o publish
```

Qué observar:

* `global.json` quedó en la carpeta del proyecto; en un repositorio real va en la raíz, para que aplique a todos los proyectos.
* El `.csproj` ganó una línea `<PackageReference Include="Humanizer" Version="2.14.1" />`. La DLL del paquete no está en el repositorio: está en la caché global y se copia a `bin/`.
* `dotnet run` compiló en Debug antes de ejecutar. `dotnet publish` compiló en Release y dejó en `publish/` solo lo que se despliega: `Orders.dll`, el ejecutable lanzador (`Orders.exe` en Windows), `Humanizer.dll`, los archivos `.json` de configuración del runtime y una carpeta por idioma (`es/`, `fr/`...) con las traducciones que trae el paquete. Una dependencia "pequeña" también pesa en el despliegue.
* La salida de compilación (advertencias, tiempos, rutas) depende del SDK y no forma parte del ejemplo.

-----

## Errores comunes

**1. No fijar el SDK.**
Qué pasa: NETSDK1045 en CI, o comportamientos distintos entre máquinas.
Por qué: la CLI usa el SDK más nuevo instalado si no hay `global.json`.
Arreglo: agrega `global.json` en la raíz del repositorio con una política de `rollForward` explícita.

**2. Creer que `global.json` cambia el TFM.**
Qué pasa: subes el SDK a 10 y el proyecto sigue apuntando a `net8.0`.
Por qué: `global.json` elige la herramienta, no el destino.
Arreglo: cambia `<TargetFramework>` cuando decidas migrar el runtime.

**3. Versionar `bin/` y `obj/`.**
Qué pasa: conflictos, repositorio pesado y rutas absolutas de otra máquina en `project.assets.json`.
Por qué: son archivos derivados.
Arreglo: `dotnet new gitignore`, y bórralos del repositorio con `git rm -r --cached bin obj`.

**4. Desplegar con `dotnet run`.**
Qué pasa: el servidor necesita el SDK, compila en cada arranque y ejecuta en Debug.
Por qué: `run` es un comando de desarrollo.
Arreglo: despliega la salida de `dotnet publish`.

**5. Usar versiones flotantes (`Version="*"`) o no revisar transitivas.**
Qué pasa: dos compilaciones del mismo commit usan versiones distintas, o una transitiva vulnerable llega a producción.
Por qué: NuGet resuelve la versión disponible en el momento del restore.
Arreglo: fija versiones evaluadas y ejecuta `dotnet package list --vulnerable --include-transitive` en CI.

-----

## Según la versión de .NET

* **.NET Core 1.0 (2016):** aparece la CLI `dotnet` multiplataforma, con el formato `project.json`.
* **SDK de .NET Core 1.0 (2017):** el `.csproj` estilo SDK con `PackageReference` reemplaza a `project.json`.
* **.NET 6:** las plantillas usan *top-level statements* e *implicit usings*.
* **.NET 8:** `dotnet publish` usa Release por defecto.
* **.NET 10:** la CLI agrega la forma sustantivo-verbo (`dotnet package add`, `dotnet package list`) y permite ejecutar un único archivo `.cs` con `dotnet run app.cs`.

-----

## Cuándo sí y cuándo no

**Usa la CLI cuando:** automatizas, configuras CI, reproduces un error de compilación o necesitas saber exactamente qué se ejecuta.

**Usa el IDE cuando:** sus herramientas de navegación, depuración y refactorización te hacen más productivo. No los enfrentes: el IDE usa el mismo SDK.

**No uses `dotnet run` cuando:** el destino es un servidor, un contenedor o cualquier entorno que no sea tu máquina de desarrollo.

-----

## Resumen en 5 líneas

1. El SDK sirve para desarrollar; el runtime, para ejecutar.
2. `global.json` fija el SDK; `TargetFramework` fija el destino.
3. MSBuild evalúa el `.csproj` y ejecuta restore, build y publish.
4. `bin/` y `obj/` son derivados: se regeneran y no van a Git.
5. NuGet registra `PackageReference` en el proyecto y descarga los paquetes a una caché global; las transitivas también cuentan.

-----

## Para profundizar

<details>
<summary>Restore reproducible con lock files</summary>

Agrega `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>` al proyecto. NuGet genera `packages.lock.json` con la versión exacta de cada dependencia, incluidas las transitivas. En CI usa `dotnet restore --locked-mode`: si el lock file no coincide, el restore falla en vez de resolver otra versión en silencio.

</details>

<details>
<summary>Gestión central de versiones</summary>

En soluciones con muchos proyectos, un archivo `Directory.Packages.props` en la raíz con `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` centraliza las versiones con elementos `<PackageVersion>`. Cada `.csproj` declara `<PackageReference Include="Humanizer" />` sin versión. Así evitas que dos proyectos usen versiones distintas del mismo paquete.

</details>

<details>
<summary>Aplicaciones de un solo archivo en .NET 10</summary>

`dotnet run app.cs` compila y ejecuta un archivo sin `.csproj`. Las dependencias se declaran con directivas al inicio:

```csharp
#:package Humanizer@2.14.1
using Humanizer;

Console.WriteLine("PedidoEnviadoAlCliente".Humanize());
```

Sirve para scripts y pruebas rápidas. Cuando el código crece, `dotnet project convert app.cs` lo convierte en un proyecto normal.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El SDK contiene las herramientas para crear y compilar; el runtime, lo necesario para ejecutar. El `.csproj` describe el proyecto, NuGet resuelve paquetes y `dotnet publish` prepara lo que se despliega. `bin/` y `obj/` no se suben a Git.

### Respuesta ampliada (semi-senior)

La CLI conduce MSBuild. Restore genera el grafo de dependencias en `obj/project.assets.json`, build produce salidas en `bin/` y publish reúne el artefacto desplegable. Fijo el SDK con `global.json` y una política de `rollForward`, sin confundirlo con el TFM. Controlo dependencias con versiones explícitas, lock files en CI y auditoría de transitivas vulnerables; en soluciones grandes uso gestión central de versiones. Despliego siempre la salida de `publish`, nunca `dotnet run`.

### Preguntas frecuentes de seguimiento

**1. ¿`dotnet build` ejecuta restore?**
Sí, de forma implícita, salvo que pases `--no-restore`.

**2. ¿NuGet copia los paquetes al repositorio?**
No. Los descarga a la caché global y el proyecto solo guarda la referencia.

**3. ¿Qué pasa si no hay `global.json`?**
La CLI usa el SDK más nuevo instalado en esa máquina.

**4. ¿Qué es una dependencia transitiva?**
Una que llega porque la necesita otra dependencia tuya. También se despliega y también puede ser vulnerable.

-----

## Práctica

**Ejercicio 1.** Borras `bin/` y `obj/` de un proyecto. ¿Qué comando los regenera y qué se pierde?

<details>
<summary>Solución</summary>

`dotnet build` basta: ejecuta restore (regenera `obj/` con los activos) y compila (regenera `bin/`). No se pierde nada, porque todo sale del `.csproj`, el código y la caché de NuGet. Si algo se pierde al borrarlos, alguien guardó a mano un archivo donde no correspondía.

</details>

**Ejercicio 2.** Tu repositorio tiene este `global.json` y la máquina de CI tiene instalados los SDK `10.0.100` y `10.0.200`. ¿Cuál usa la CLI? ¿Y si solo tuviera `11.0.100`?

```json
{ "sdk": { "version": "10.0.100", "rollForward": "latestFeature" } }
```

<details>
<summary>Solución</summary>

Usa `10.0.200`: `latestFeature` acepta la banda de características más nueva dentro de 10.0. Si solo existiera `11.0.100`, la CLI fallaría con un error de SDK no encontrado, porque `latestFeature` no cambia de versión principal. Para permitirlo haría falta `latestMajor`, una decisión que conviene tomar a conciencia.

</details>

-----

## Siguiente lección

Terminaste el módulo. Continúa con [Tipos y variables](../01-tipos-y-variables/README.md) o vuelve al [índice](README.md).
