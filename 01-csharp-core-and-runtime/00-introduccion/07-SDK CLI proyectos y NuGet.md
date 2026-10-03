# SDK, CLI, proyectos y NuGet

## En una frase
El SDK y la CLI convierten una descripción de proyecto en artefactos reproducibles; NuGet resuelve sus dependencias y `global.json` controla qué SDK ejecuta el proceso.

-----

## Antes de empezar
Conviene que ya sepas: [CLR, compilación y sistema de tipos](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md).

Palabras nuevas: **MSBuild:** motor de compilación; **restore:** resolución de dependencias; **PackageReference:** dependencia declarada en el proyecto; **activo transitorio:** archivo intermedio de compilación.

-----

## El problema
Ejecutar comandos de memoria sin entender sus artefactos genera errores difíciles: compilar con otro SDK, versionar `bin/`, instalar un paquete incompatible o creer que `dotnet run` es el comando de producción.

-----

## Cómo funciona
### 1. SDK frente a runtime
El SDK contiene CLI, compiladores, MSBuild y un runtime. El runtime solo permite ejecutar aplicaciones compatibles. Un servidor puede usar una publicación autocontenida sin instalar ninguno de los dos.

### 2. Proyecto SDK-style
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

El `.csproj` declara intención. MSBuild evalúa propiedades, elementos, imports y objetivos para producir la salida.

### 3. Ciclo de comandos
| Comando | Propósito |
| --- | --- |
| `dotnet new console -n Orders` | Crear desde una plantilla |
| `dotnet restore` | Resolver paquetes y generar activos |
| `dotnet build` | Compilar |
| `dotnet run` | Compilar si hace falta y ejecutar en desarrollo |
| `dotnet test` | Ejecutar pruebas |
| `dotnet publish` | Preparar artefactos de despliegue |
| `dotnet clean` | Eliminar salidas conocidas por MSBuild |

`build` ya realiza restore implícito salvo que uses `--no-restore`.

### 4. `bin` y `obj`
`obj/` contiene activos y archivos intermedios; `bin/` contiene salidas por configuración y TFM. Ambos son regenerables y normalmente se excluyen de Git.

### 5. `global.json`
El TFM decide para qué runtime compilas. `global.json` selecciona el SDK que ejecuta la CLI. Son decisiones distintas.

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

### 6. NuGet
En proyectos modernos, una dependencia queda como `PackageReference`. En .NET 10 puedes usar la forma sustantivo-verbo:

```bash
dotnet package add Humanizer --version 2.14.1
```

La forma histórica `dotnet add package` aparece en material anterior y sigue siendo útil al trabajar con SDK previos.

### 7. IDEs
Visual Studio, Rider y VS Code pueden invocar el mismo SDK. El IDE mejora edición y depuración; no reemplaza la comprensión del proyecto ni vuelve diferente el resultado por sí mismo.

-----

## Ejemplo completo
```bash
dotnet new console -n Orders
cd Orders
dotnet build
dotnet run
```

`Program.cs`:
```csharp
Console.WriteLine("Orders listo");
```

Salida de la aplicación:
```text
Orders listo
```

La salida adicional de compilación depende del SDK y no se fija como parte del ejemplo.

-----

## Errores comunes
**1. Confundir `run` con `publish`.** Qué pasa: se despliega desde fuentes o cachés locales. Por qué: `run` es un flujo de desarrollo. Arreglo: genera y despliega la salida de `publish`.

**2. Versionar `bin` y `obj`.** Qué pasa: aparecen conflictos y archivos de otra máquina. Por qué: son derivados. Arreglo: ignóralos y regénéralos.

**3. Creer que `global.json` cambia el TFM.** Qué pasa: el proyecto sigue apuntando a la misma plataforma. Por qué: selecciona SDK, no destino. Arreglo: cambia `TargetFramework` solo cuando corresponda.

**4. Instalar cualquier versión de un paquete.** Qué pasa: restore informa incompatibilidad o llegan cambios no deseados. Por qué: paquete, TFM y política de versiones deben ser compatibles. Arreglo: fija una versión evaluada y revisa dependencias transitivas.

-----

## Según la versión de .NET
- **.NET Core:** introdujo el flujo moderno y multiplataforma de `dotnet`.
- **.NET 6:** consolidó proyectos mínimos y *implicit usings* en plantillas.
- **.NET 10:** agregó el orden sustantivo-verbo, por ejemplo `dotnet package add`; usa la sintaxis que soporte tu SDK.

-----

## Cuándo sí y cuándo no
**Usa la CLI cuando:** necesitas automatización, CI o reproducibilidad. **Usa un IDE cuando:** sus herramientas de navegación y depuración aumentan productividad. No los enfrentes: el IDE suele apoyarse en el mismo SDK.

-----

## Resumen en 5 líneas
1. El SDK sirve para desarrollar; el runtime sirve para ejecutar.
2. El `.csproj` declara cómo se construye el proyecto.
3. `obj` es intermedio y `bin` contiene salidas regenerables.
4. `global.json` selecciona SDK; `TargetFramework` selecciona destino.
5. NuGet registra dependencias como `PackageReference` en proyectos modernos.

-----

## Para profundizar
<details><summary>Restore determinista</summary>Para repetir dependencias puedes usar un archivo de bloqueo y modo bloqueado. Aun así debes controlar fuentes, credenciales y versiones transitivas.</details>

-----

## En entrevista
### Respuesta corta (junior)
El SDK contiene herramientas para crear y compilar. El `.csproj` describe el proyecto, NuGet resuelve paquetes y `dotnet publish` prepara el despliegue.

### Respuesta ampliada (semi-senior)
La CLI conduce MSBuild. Restore genera el grafo de dependencias en `obj`, build produce salidas en `bin` y publish reúne el artefacto desplegable según el modelo elegido. `global.json` controla el SDK sin cambiar el TFM.

### Preguntas frecuentes de seguimiento
**1. ¿`dotnet build` ejecuta restore?** Sí, de forma implícita salvo que se desactive.

**2. ¿NuGet copia paquetes al repositorio?** Normalmente los restaura en la caché global y registra referencias en el proyecto.

-----

## Práctica
**Ejercicio 1.** Explica qué regenerarías después de borrar `bin/` y `obj/`.
<details><summary>Solución</summary>Ejecutaría `dotnet restore` y `dotnet build`; también basta `dotnet build` porque incluye restore implícito.</details>

-----

## Siguiente lección
Terminaste el módulo. Continúa con [Tipos y variables](../01-tipos-y-variables/README.md) o vuelve al [índice](README.md).
