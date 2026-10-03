# Native AOT y diseño cloud native

## En una frase

Native AOT compila una aplicación a código nativo antes de ejecutarla; puede reducir el tiempo de arranque y el tamaño de despliegue, pero exige compatibilidad y mediciones reales.

-----

## Antes de empezar

Conviene que ya sepas: [contenedores Docker para APIs .NET](01-Contenedores%20Docker%20para%20APIs%20NET.md) y [builds multi-stage e imágenes seguras](02-Builds%20multi-stage%20e%20imagenes%20seguras.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

- **Native AOT:** compilación anticipada que genera un ejecutable nativo para una plataforma concreta.
- ***trimming*:** eliminación de código que el compilador considera inaccesible.
- **framework-dependent:** despliegue que necesita el runtime de .NET instalado en el entorno final.
- **cloud native:** enfoque para diseñar sistemas operables, automatizables y escalables en entornos dinámicos.

-----

## El problema

Activar Native AOT no es una optimización universal:

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

Una biblioteca puede depender de reflexión no acotada, carga dinámica o generación de código en tiempo de ejecución. El publicador puede emitir advertencias o producir un ejecutable que falla en una ruta no probada.

Tampoco basta con colocar una API en Docker para llamarla *cloud native*. Si guarda sesiones en memoria o archivos locales, cada réplica observa un estado distinto y una sustitución del contenedor puede perder datos.

Las consecuencias son concretas:

- despliegues que fallan solo en una arquitectura determinada;
- serialización que pierde tipos eliminados por *trimming*;
- imágenes pequeñas que no tienen las capacidades de globalización esperadas;
- réplicas que no pueden intercambiarse sin alterar el comportamiento;
- una optimización aceptada sin medir arranque, memoria, tamaño y rendimiento.

-----

## Cómo funciona

### 1. CoreCLR con JIT frente a Native AOT

| Aspecto | CoreCLR con JIT | Native AOT |
| --- | --- | --- |
| Compilación final | Parte ocurre durante la ejecución | Ocurre al publicar |
| Artefacto | Ensamblados y runtime | Ejecutable nativo autocontenido |
| Arranque | Incluye inicialización y compilación JIT | Puede ser menor |
| Código dinámico | Admite más escenarios | Está restringido |
| Portabilidad del artefacto | Depende del runtime | Depende del sistema y arquitectura publicados |

Native AOT cambia el modelo de despliegue. **No garantiza mayor rendimiento sostenido ni menor memoria para todas las cargas.**

### 2. Publicar para una plataforma concreta

El identificador de runtime define el destino:

```bash
dotnet publish -c Release -r linux-x64
```

El ejecutable resultante no sirve automáticamente para `linux-arm64` ni para Windows. Si produces imágenes multiplataforma, cada arquitectura necesita su propia publicación.

### 3. Tratar las advertencias como problemas de compatibilidad

Native AOT usa *trimming*. Una llamada que obtiene tipos por nombres calculados puede ocultar dependencias al análisis estático:

```csharp
Type? type = Type.GetType(typeName);
```

No suprimas advertencias sin demostrar que el código es seguro. Prefiere APIs analizables, generación de código en compilación y bibliotecas que declaren compatibilidad con AOT.

### 4. Evitar reflexión implícita en JSON

En una API con DTOs, usa metadatos generados en compilación:

```csharp
using System.Text.Json.Serialization;

public sealed record OrderResponse(Guid Id, decimal Total);

[JsonSerializable(typeof(OrderResponse))]
internal partial class ApiJsonContext : JsonSerializerContext;
```

Luego registra el contexto en `HttpJsonOptions`. Así el serializador no necesita descubrir el contrato mediante reflexión en tiempo de ejecución.

### 5. Elegir las imágenes correctas

En .NET 10, la imagen `sdk:10.0-noble-aot` contiene las herramientas nativas necesarias para compilar. El ejecutable final puede correr sobre `runtime-deps:10.0-noble-chiseled`; no necesita la imagen `aspnet` porque ya incluye el runtime.

Las imágenes *chiseled* reducen superficie y no incluyen shell ni administrador de paquetes. La variante básica tampoco incluye todas las capacidades de globalización. Usa una variante `extra` o configura globalización invariante cuando el dominio lo permita.

### 6. Diseñar para operar en la nube

```text
petición -> réplica intercambiable -> dependencia externa
                         |             (base de datos, caché, cola)
                         +-> trazas, métricas y logs
```

Una aplicación preparada para entornos dinámicos suele necesitar:

- estado persistente fuera del contenedor;
- configuración y secretos externos;
- apagado ordenado y cancelación cooperativa;
- señales de salud y disponibilidad;
- observabilidad y políticas de resiliencia;
- automatización de compilación y despliegue;
- escalado horizontal solo cuando el diseño y la carga lo permiten.

Kubernetes puede orquestar estas capacidades, pero **no es un requisito para que el diseño sea cloud native**.

### 7. Medir antes de decidir

Compara alternativas en el mismo hardware, configuración y carga. Registra, como mínimo, tiempo de arranque en frío, memoria residente, tamaño de imagen y rendimiento bajo carga. Ejecuta varias repeticiones y conserva la metodología junto con los resultados.

-----

## Ejemplo completo

Este ejemplo expone texto plano para concentrarse en el proceso de publicación. Está dirigido a Linux x64.

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

app.MapGet("/", () => Results.Text("ready"));

app.Run();
```

`Dockerfile`:

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

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["./Orders.Api"]
```

Construye y ejecuta:

```bash
docker build -t orders-api:aot .
docker run --rm -p 5000:8080 orders-api:aot
curl http://localhost:5000/
```

Salida HTTP:

```text
ready
```

Observa que la etapa final no contiene el SDK ni el runtime de ASP.NET Core. También ejecuta el proceso con un usuario no privilegiado.

-----

## Errores comunes

**1. Suponer que AOT siempre es más rápido.**

Qué pasa: se acepta una optimización sin conocer su efecto sobre la carga real.

Por qué: AOT suele favorecer arranque y tamaño, pero el rendimiento y la memoria dependen de la aplicación.

Arreglo: mide las métricas relevantes con una metodología repetible.

**2. Ignorar las advertencias de AOT y trimming.**

Qué pasa: una ruta basada en reflexión puede fallar después de publicar.

Por qué: el análisis estático no puede conservar código que no logra identificar.

Arreglo: elimina las advertencias mediante APIs compatibles, generación de código o anotaciones justificadas.

**3. Usar la misma publicación en cualquier arquitectura.**

Qué pasa: el contenedor no inicia o informa un formato de ejecutable inválido.

Por qué: un binario Native AOT es específico de sistema operativo y arquitectura.

Arreglo: publica con el RID correspondiente, por ejemplo `linux-x64` o `linux-arm64`.

**4. Buscar una imagen final `runtime-deps:*‑aot`.**

Qué pasa: Docker no encuentra la etiqueta.

Por qué: el sufijo `-aot` corresponde a imágenes SDK de compilación; el resultado se ejecuta sobre `runtime-deps`.

Arreglo: compila con `sdk:10.0-noble-aot` y usa `runtime-deps:10.0-noble-chiseled` como etapa final.

**5. Confundir Docker con diseño cloud native.**

Qué pasa: varias réplicas divergen o pierden estado al reiniciarse.

Por qué: el contenedor empaqueta el proceso, pero no externaliza el estado ni agrega operabilidad.

Arreglo: diseña explícitamente configuración, estado, salud, telemetría, resiliencia y ciclo de vida.

-----

## Según la versión de .NET

- **.NET 7:** Native AOT recibió soporte oficial inicial para aplicaciones de consola.
- **.NET 8:** ASP.NET Core incorporó una plantilla Web API orientada a Native AOT y un conjunto limitado de características compatibles.
- **.NET 9:** continuaron las mejoras de compatibilidad y tamaño para aplicaciones web AOT.
- **.NET 10:** existen imágenes SDK oficiales con sufijo `-aot`; la compatibilidad sigue siendo una propiedad que debes verificar por aplicación y biblioteca.

-----

## Cuándo sí y cuándo no

**Usa Native AOT cuando:** el arranque, la densidad o el tamaño de despliegue importan; tus dependencias son compatibles; y las mediciones justifican el costo.

**No lo uses cuando:** dependes de carga dinámica, reflexión no acotada o bibliotecas incompatibles; necesitas máxima flexibilidad del runtime; o no existe un problema medido.

**Diseña para cloud native cuando:** esperas despliegues automatizados, reemplazo de instancias, variaciones de carga y operación distribuida.

**No introduzcas complejidad distribuida cuando:** una aplicación modular en un único proceso satisface disponibilidad, escala y operación.

-----

## Resumen en 5 líneas

1. Native AOT genera un ejecutable nativo específico de plataforma antes de ejecutar.
2. Sus beneficios más comunes son arranque y tamaño, pero deben medirse.
3. Las advertencias de AOT y *trimming* señalan riesgos reales de compatibilidad.
4. En .NET 10, compila con una imagen SDK `-aot` y ejecuta con `runtime-deps`.
5. Docker empaqueta una aplicación; el diseño cloud native exige además operabilidad y estado externo.

-----

## Para profundizar

<details>
<summary>Globalización en imágenes chiseled</summary>

Las imágenes *chiseled* mínimas no incluyen ICU ni datos de zonas horarias. `InvariantGlobalization=true` evita esa dependencia, pero cambia comparaciones, formatos y reglas culturales. Si el negocio requiere culturas específicas, usa una variante `extra` compatible y prueba esas operaciones.

</details>

<details>
<summary>Despliegues para varias arquitecturas</summary>

Una etiqueta de imagen puede apuntar a un manifiesto con variantes `linux/amd64` y `linux/arm64`. Cada variante debe compilar el ejecutable para su arquitectura. No copies un único binario x64 dentro de ambas imágenes.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Native AOT compila la aplicación .NET a código nativo al publicar. Puede iniciar más rápido y reducir el despliegue, pero limita escenarios dinámicos y produce binarios específicos de plataforma.

### Respuesta ampliada (semi-senior)

La decisión requiere validar dependencias, resolver advertencias de *trimming* y comparar métricas contra una publicación con CoreCLR. En contenedores, la compilación ocurre en una imagen SDK `-aot` y el ejecutable se copia a una imagen `runtime-deps`. Esto es independiente de diseñar una aplicación cloud native, que también exige estado externo, configuración, telemetría, resiliencia y un ciclo de vida operable.

### Preguntas frecuentes de seguimiento

**1. ¿Native AOT elimina toda dependencia del sistema?** No. El binario es autocontenido respecto de .NET, pero todavía depende del sistema operativo, la arquitectura y bibliotecas nativas requeridas.

**2. ¿Por qué no usar la imagen `aspnet` al final?** Porque el ejecutable AOT ya contiene el código necesario de .NET; `runtime-deps` aporta las dependencias nativas mínimas.

**3. ¿Un contenedor debe guardar archivos permanentes?** No en su capa escribible. Usa almacenamiento externo o volúmenes cuando los datos deban sobrevivir al reemplazo.

**4. ¿Cloud native significa microservicios?** No. Un monolito modular también puede automatizarse, observarse y operar con instancias reemplazables.

-----

## Práctica

**Ejercicio 1. Publica para otra arquitectura.** Cambia el ejemplo para producir un ejecutable `linux-arm64` e identifica todas las líneas afectadas.

<details>
<summary>Solución</summary>

Cambia `-r linux-x64` por `-r linux-arm64` tanto en `dotnet restore` como en `dotnet publish`. La etapa final puede conservar `runtime-deps:10.0-noble-chiseled`, pero Docker debe resolver su variante ARM64 durante una compilación para esa plataforma.

</details>

**Ejercicio 2. Define un criterio de decisión.** Escribe qué medirías antes de migrar una API existente a Native AOT.

<details>
<summary>Solución</summary>

Mide arranque en frío, memoria residente, tamaño descargado de la imagen, latencia y rendimiento bajo una carga representativa. Mantén hardware, datos, configuración y duración iguales. Repite las pruebas y agrega como criterio obligatorio que la publicación no emita advertencias AOT o de *trimming* sin resolver.

</details>

-----

## Siguiente lección

[Ejercicios del módulo](Ejercicios.md) · [Volver al índice](README.md)
