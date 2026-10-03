# Ejercicios: cloud native y contenedores

Requisitos previos: [contenedores Docker](01-Contenedores%20Docker%20para%20APIs%20NET.md), [builds multi-stage](02-Builds%20multi-stage%20e%20imagenes%20seguras.md) y [Native AOT](03-Native%20AOT%20y%20diseno%20cloud%20native.md).

-----

## Ejercicios guiados

### Ejercicio 1. Crear un Dockerfile básico

**Objetivo:** empaquetar una API ya publicada en una imagen de runtime.

**Contexto:** la guía copiaba el código fuente directamente a `aspnet:10.0`. Esa imagen no contiene el SDK y no compila el proyecto. También asumía el puerto 80, cuando las imágenes actuales de ASP.NET Core usan 8080 de forma predeterminada.

**Instrucciones:**

1. Crea una Minimal API llamada `Orders.Api`.
2. Publícala en `publish/`.
3. Crea una imagen que copie únicamente esa salida.
4. Ejecuta el proceso con el usuario no privilegiado incluido en la imagen.

<details>
<summary>Solución</summary>

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => Results.Text("Orders API"));

app.Run();
```

`Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY publish/ .

EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

```bash
dotnet publish -c Release -o publish
docker build -t orders-api:runtime .
docker run --rm -p 5000:8080 orders-api:runtime
curl http://localhost:5000/
```

Salida HTTP:

```text
Orders API
```

</details>

**Qué observar:** el Dockerfile básico funciona porque la publicación ocurre antes de construir la imagen. El puerto izquierdo pertenece al host y el derecho al contenedor.

### Ejercicio 2. Construir y ejecutar con configuración externa

**Objetivo:** cambiar el comportamiento sin reconstruir la imagen.

**Contexto:** una imagen debe ser inmutable entre entornos. Incluir configuración específica dentro de ella obliga a crear artefactos distintos y aumenta el riesgo de filtrar secretos.

**Instrucciones:**

1. Lee `StoreName` desde configuración.
2. Construye la imagen una vez.
3. Ejecuta dos contenedores con valores diferentes mediante variables de entorno.

<details>
<summary>Solución</summary>

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var storeName = builder.Configuration["StoreName"] ?? "default";
app.MapGet("/store", () => Results.Text(storeName));

app.Run();
```

Después de publicar y construir con el Dockerfile del ejercicio anterior:

```bash
docker run --rm -p 5001:8080 -e StoreName=Lima orders-api:runtime
docker run --rm -p 5002:8080 -e StoreName=Arequipa orders-api:runtime
```

Respuestas respectivas:

```text
Lima
Arequipa
```

</details>

**Qué observar:** ambas instancias usan la misma imagen. Solo cambia la configuración inyectada al crear cada contenedor.

### Ejercicio 3. Crear un build multi-stage

**Objetivo:** compilar dentro de Docker sin enviar el SDK a producción.

**Contexto:** usar una sola etapa basada en `sdk` produce una imagen final innecesariamente grande y con más herramientas disponibles ante una intrusión.

**Instrucciones:** separa restauración, publicación y ejecución. Ordena las copias para reutilizar la caché de dependencias.

<details>
<summary>Solución</summary>

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY Orders.Api.csproj ./
RUN dotnet restore Orders.Api.csproj

COPY . .
RUN dotnet publish Orders.Api.csproj \
    -c Release \
    --no-restore \
    -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

`.dockerignore`:

```text
**/bin/
**/obj/
.git/
.vs/
publish/
```

```bash
docker build -t orders-api:multi-stage .
docker run --rm -p 5000:8080 orders-api:multi-stage
```

</details>

**Qué observar:** cambiar solo `Program.cs` permite reutilizar la capa de `restore`. La etapa final no hereda el SDK de la etapa de compilación.

### Ejercicio 4. Publicar con Native AOT

**Objetivo:** producir un ejecutable nativo para una plataforma conocida.

**Contexto:** la guía presentaba AOT como una mejora automática. La publicación debe estar libre de advertencias relevantes y compararse con una línea base antes de tomar una decisión.

**Instrucciones:**

1. Activa Native AOT.
2. Usa el modelo reducido de ASP.NET Core.
3. Publica para Linux x64.
4. Registra advertencias y tamaño sin inventar resultados.

<details>
<summary>Solución</summary>

`Orders.Api.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
</Project>
```

`Program.cs`:

```csharp
var builder = WebApplication.CreateSlimBuilder(args);
var app = builder.Build();

app.MapGet("/", () => Results.Text("AOT ready"));

app.Run();
```

```bash
dotnet publish -c Release -r linux-x64 -o publish-aot
```

La salida exacta del publicador y el tamaño dependen del SDK, el sistema y el proyecto. Regístralos desde tu ejecución; no uses una cifra prefijada.

</details>

**Qué observar:** el directorio contiene un ejecutable nativo para Linux x64. Una publicación correcta no convierte ese archivo en portable a ARM64 o Windows.

### Ejercicio 5. Contenerizar una publicación AOT

**Objetivo:** combinar una etapa de compilación AOT con una imagen final mínima.

**Contexto:** una imagen SDK genérica puede carecer de la cadena nativa requerida. Además, el sufijo `-aot` corresponde a la imagen de compilación; no existe una imagen final `runtime-deps:10.0-noble-aot`.

**Instrucciones:** usa la imagen SDK AOT de .NET 10 y una imagen final *chiseled*.

<details>
<summary>Solución</summary>

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-noble-aot AS build
WORKDIR /src

COPY Orders.Api.csproj ./
RUN dotnet restore Orders.Api.csproj -r linux-x64

COPY . .
RUN dotnet publish Orders.Api.csproj \
    -c Release \
    -r linux-x64 \
    --no-restore \
    -o /app/publish \
    && rm -f /app/publish/*.dbg

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["./Orders.Api"]
```

```bash
docker build -t orders-api:aot .
docker run --rm -p 5000:8080 orders-api:aot
curl http://localhost:5000/
```

Salida HTTP:

```text
AOT ready
```

</details>

**Qué observar:** la etapa final no ejecuta `dotnet Orders.Api.dll`; inicia directamente el binario nativo. Tampoco contiene una shell para depuración interactiva.

-----

## Retos

### Reto 1. API reemplazable

**Misión:** crea una API de pedidos con `GET /orders/{id}` que pueda ejecutar dos réplicas sin depender de memoria o archivos locales. Explica dónde vivirían los datos y la configuración en producción.

**Pista:** separa el contrato HTTP del almacenamiento. Para el ejercicio puedes usar un repositorio falso sin estado y documentar el adaptador externo real.

<details>
<summary>Solución</summary>

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IOrderReader, DemoOrderReader>();

var app = builder.Build();

app.MapGet("/orders/{id:guid}", async (Guid id, IOrderReader reader, CancellationToken ct) =>
{
    OrderResponse? order = await reader.FindAsync(id, ct);
    return order is null ? Results.NotFound() : Results.Ok(order);
});

app.Run();

public sealed record OrderResponse(Guid Id, decimal Total);

public interface IOrderReader
{
    Task<OrderResponse?> FindAsync(Guid id, CancellationToken cancellationToken);
}

public sealed class DemoOrderReader : IOrderReader
{
    public Task<OrderResponse?> FindAsync(Guid id, CancellationToken cancellationToken)
    {
        cancellationToken.ThrowIfCancellationRequested();
        OrderResponse order = new(id, 125.50m);
        return Task.FromResult<OrderResponse?>(order);
    }
}
```

En producción, implementa `IOrderReader` contra una base de datos compartida. Inyecta cadenas de conexión y opciones mediante el entorno o un almacén de secretos. No escribas estado permanente en la capa del contenedor.

</details>

### Reto 2. Optimizar con evidencia

**Misión:** compara las imágenes runtime, multi-stage y AOT sin afirmar de antemano cuál es mejor.

**Pista:** conserva Dockerfile, comando, arquitectura, fecha, digest de las imágenes base y tamaño reportado por Docker.

<details>
<summary>Solución</summary>

Crea las tres imágenes con etiquetas distintas y registra sus valores reales:

```bash
docker image ls orders-api
docker image inspect orders-api:runtime
docker image inspect orders-api:multi-stage
docker image inspect orders-api:aot
```

Compara tamaño, presencia de SDK/runtime, usuario efectivo y compatibilidad funcional. Una imagen más pequeña no compensa una biblioteca incompatible ni una operación imposible de diagnosticar.

</details>

### Reto 3. Diseñar una prueba AOT

**Misión:** define una prueba reproducible para decidir entre CoreCLR y Native AOT.

**Pista:** separa arranque en frío de rendimiento estable y evita comparar ejecuciones con distinta configuración.

<details>
<summary>Solución</summary>

1. Construye ambas variantes desde el mismo commit y en modo Release.
2. Usa el mismo equipo, límites de CPU y memoria, datos y variables de entorno.
3. Mide varias veces el tiempo desde la creación del contenedor hasta la primera respuesta correcta.
4. Ejecuta una carga estable y registra latencia, solicitudes por segundo y memoria residente.
5. Registra tamaño comprimido y sin comprimir de la imagen.
6. Verifica primero exactitud funcional y ausencia de advertencias AOT no resueltas.
7. Publica mediana, dispersión, herramientas y comandos; no solo la mejor ejecución.

</details>

-----

## Checkpoint

1. ¿Por qué `COPY . .` dentro de una imagen `aspnet` no compila una API?
2. ¿Qué significan los dos puertos de `-p 5000:8080`?
3. ¿Qué evita que el SDK llegue a la imagen final de un build multi-stage?
4. ¿Por qué un ejecutable AOT para `linux-x64` no funciona necesariamente en ARM64?
5. ¿Qué riesgos indican las advertencias de *trimming*?
6. ¿Qué capacidades necesita una aplicación además de Docker para operar como cloud native?

-----

[Volver al índice del módulo](README.md)
