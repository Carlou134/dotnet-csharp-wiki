# Host, configuración y pipeline HTTP

## En una frase
Una aplicación ASP.NET Core configura servicios y fuentes de configuración, construye un host y procesa cada solicitud mediante un pipeline ordenado de middleware y endpoints.

-----

## Antes de empezar
Conviene que ya sepas: [SDK, CLI, proyectos y NuGet](../../01-csharp-core-and-runtime/00-introduccion/07-SDK%20CLI%20proyectos%20y%20NuGet.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)): **host:** proceso y ciclo de vida; **middleware:** componente encadenado; **endpoint:** destino seleccionado por routing.

-----

## El problema
Copiar un `Program.cs` antiguo con `Startup.cs`, puertos `5000/5001` fijos y middleware en cualquier orden puede producir seguridad rota o una app que funciona solo en una máquina. La estructura de carpetas tampoco activa características por sí sola.

-----

## Cómo funciona
### 1. Crear y construir
`WebApplication.CreateBuilder(args)` prepara configuración, logging, DI y servidor. `builder.Build()` cierra el registro normal y crea la aplicación.

### 2. Registrar servicios
```csharp
builder.Services.AddSingleton<Clock>();
```
Registrar no ejecuta el servicio. Describe cómo se resolverá cuando un endpoint lo necesite.

### 3. Fuentes de configuración
La configuración puede combinar `appsettings.json`, archivos por entorno, secretos de desarrollo, variables de entorno y argumentos. La fuente posterior puede sobrescribir una clave anterior.

### 4. Entornos
`Development`, `Staging` y `Production` permiten variar comportamiento operativo. No son una frontera de seguridad: un secreto sigue sin pertenecer al repositorio.

### 5. Pipeline
```text
solicitud -> excepción -> HTTPS -> estáticos -> auth -> routing/endpoint
respuesta <-           <-       <-          <-      <-
```
El orden importa porque cada middleware decide si continúa, modifica la respuesta o corta el pipeline.

### 6. `launchSettings.json`
Define perfiles locales para herramientas. No se publica como configuración de producción y no garantiza los puertos del despliegue.

### 7. `wwwroot`
Contiene activos públicos servidos por la aplicación. No coloques secretos allí; cualquier archivo expuesto por el middleware estático se trata como contenido público.

-----

## Ejemplo completo
`Program.cs` de `dotnet new web`:
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<Clock>();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.MapGet("/time", (Clock clock) => TypedResults.Ok(clock.UtcNow));
app.MapGet("/error", () => Results.Problem(statusCode: 500));
app.Run();

public sealed class Clock
{
    public DateTimeOffset UtcNow => DateTimeOffset.UtcNow;
}
```

La respuesta contiene la hora real de la ejecución, por lo que no se fija una salida inventada.

-----

## Errores comunes
**1. Poner autorización antes de autenticación.** Qué pasa: no existe identidad validada. Por qué: el orden del pipeline cambia el contexto disponible. Arreglo: autentica antes de autorizar.

**2. Guardar secretos en `appsettings.json`.** Qué pasa: llegan al repositorio o artefacto. Por qué: es configuración, no almacén secreto. Arreglo: usa variables o un gestor de secretos.

**3. Tratar `launchSettings.json` como producción.** Qué pasa: el despliegue ignora sus perfiles. Por qué: es una ayuda local. Arreglo: configura el entorno real desde su plataforma.

-----

## Según la versión de .NET
- **ASP.NET Core 1-5:** era habitual separar arranque entre `Program.cs` y `Startup.cs`.
- **.NET 6+:** el modelo de hosting mínimo concentra la configuración en `Program.cs`.
- **.NET 10:** las plantillas modernas usan `WebApplication`; `Startup.cs` sigue siendo código histórico válido, no el modelo recomendado para material nuevo.

-----

## Cuándo sí y cuándo no
**Crea middleware cuando:** una política HTTP debe envolver muchas rutas. **No lo uses cuando:** la lógica pertenece a un único endpoint o al dominio; colócala en la capa correspondiente.

-----

## Resumen en 5 líneas
1. El builder configura servicios, logging, servidor y configuración.
2. El contenedor DI resuelve dependencias registradas.
3. Los middleware procesan solicitudes en orden y respuestas en orden inverso.
4. `launchSettings.json` solo describe perfiles locales.
5. Los secretos deben venir de una fuente diseñada para protegerlos.

-----

## Para profundizar
<details><summary>Short-circuit</summary>Un middleware puede responder sin invocar al siguiente. Es útil para archivos estáticos o errores, pero un orden incorrecto puede saltarse políticas necesarias.</details>

-----

## En entrevista
### Respuesta corta (junior)
ASP.NET Core registra servicios, construye una aplicación y procesa HTTP mediante middleware ordenado hasta seleccionar un endpoint.

### Respuesta ampliada (semi-senior)
El host integra configuración, DI, logging y ciclo de vida. El pipeline es bidireccional y su orden afecta seguridad y funcionalidad. Routing selecciona endpoints; DI resuelve sus dependencias.

### Preguntas frecuentes de seguimiento
**1. ¿Una carpeta `Controllers` registra controladores?** No; debes agregar y mapear los servicios correspondientes.

**2. ¿El puerto siempre es 5000?** No; depende del perfil, variable, servidor o plataforma de despliegue.

-----

## Práctica
**Ejercicio 1.** Agrega un middleware que incluya `X-App: Orders` en cada respuesta.
<details><summary>Solución</summary>

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers["X-App"] = "Orders";
    await next(context);
});
```
</details>

-----

## Siguiente lección
[Modelos de aplicaciones ASP.NET Core](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md)
