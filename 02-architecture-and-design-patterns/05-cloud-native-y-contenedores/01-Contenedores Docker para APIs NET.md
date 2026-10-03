# Contenedores Docker para APIs .NET

## En una frase

Una imagen empaqueta los artefactos publicados y sus dependencias; un contenedor ejecuta esa imagen como un proceso aislado, pero sigue dependiendo del sistema operativo, arquitectura y configuración disponibles.

-----

## Antes de empezar

Conviene que ya sepas:

* Crear una API: [Minimal APIs](../../03-aspnet-core-apis/02-minimal-apis/README.md).
* Diferenciar compilación y ejecución de .NET.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Image:** plantilla inmutable formada por capas.
* **Container:** proceso aislado creado desde una imagen.
* **Dockerfile:** instrucciones para construir una imagen.
* **Build context:** archivos disponibles durante el build.

-----

## El problema

Copiar el repositorio a una imagen de runtime no crea mágicamente la DLL:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY . .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

Si el contexto contiene solo `.cs` y `.csproj`, `MyApp.dll` no existe porque `aspnet` ejecuta aplicaciones, pero no incluye el SDK para compilarlas. Si el repositorio contiene un `bin/Debug` local, copiarlo mezcla artefactos de otra máquina y configuración.

Un contenedor tampoco garantiza “portabilidad total”: una imagen Linux no corre como contenedor Windows, y una imagen `linux-x64` no es la misma que `linux-arm64`.

-----

## Cómo funciona

### 1. Imagen y contenedor no son sinónimos

```text
Dockerfile ── docker build ──> imagen orders-api:1.0.0
                                      │
                         docker run ──┼──> contenedor A
                                      └──> contenedor B
```

La imagen es contenido de solo lectura más metadatos. Cada contenedor agrega una capa escribible efímera y ejecuta el comando configurado.

### 2. Publica antes de copiar a una imagen de runtime

```text
dotnet publish -c Release -o publish
```

La carpeta contiene la DLL, dependencias y archivos de configuración necesarios. El Dockerfile básico puede copiar **esa salida**, no el repositorio sin compilar.

### 3. El puerto del contenedor es 8080

Las imágenes oficiales de ASP.NET Core usan `8080` por defecto desde .NET 8 para facilitar ejecución sin root:

```text
docker run -p 5000:8080 orders-api:1.0.0
              │     └── puerto dentro del contenedor
              └──────── puerto de la máquina host
```

`EXPOSE 8080` documenta el puerto; no lo publica. La opción `-p` crea el mapeo.

### 4. El contenedor comparte el kernel

No es una máquina virtual completa. Aísla procesos, red, filesystem y recursos mediante mecanismos del host. Esa menor separación importa en el modelo de amenazas: ejecuta sin root, reduce capacidades y mantén kernel/runtime actualizados.

### 5. La configuración llega en ejecución

No hornees secretos ni URLs de cada entorno en la imagen:

```text
docker run --rm -p 5000:8080 \
  -e ConnectionStrings__OrdersDb="<valor del entorno>" \
  orders-api:1.0.0
```

Las variables con `__` representan secciones de configuración de .NET. Para secretos reales usa el mecanismo del entorno u orquestador y evita que aparezcan en historial de shell, inspect o logs.

### 6. El filesystem escribible no es almacenamiento durable

Cuando el contenedor se reemplaza, su capa escribible puede desaparecer. Logs van a stdout/stderr; datos durables van a bases de datos, object storage o volúmenes diseñados y respaldados. “Stateless” significa que una instancia no guarda estado de sesión imprescindible localmente, no que el sistema carezca de estado.

-----

## Ejemplo completo

Proyecto `Orders.Api` creado con `dotnet new web`.

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => TypedResults.Ok(new
{
    Service = "Orders.Api",
    Status = "ready"
}));

app.Run();
```

`Dockerfile` básico, que supone una publicación previa:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0

WORKDIR /app
COPY publish/ .

EXPOSE 8080
USER $APP_UID

ENTRYPOINT ["dotnet", "Orders.Api.dll"]
```

Comandos:

```text
dotnet publish -c Release -o publish
docker build -t orders-api:1.0.0 .
docker run --rm -p 5000:8080 --name orders-api orders-api:1.0.0
```

Petición y respuesta:

```text
GET http://localhost:5000/
→ 200 {"service":"Orders.Api","status":"ready"}
```

Este Dockerfile separa publicación y creación de imagen, pero obliga a tener SDK local. La siguiente lección mueve la compilación al propio Dockerfile.

-----

## Errores comunes

**1. Copiar el código a `aspnet` y esperar una DLL.**
Qué pasa: el entrypoint falla porque la aplicación no fue publicada.
Por qué: la imagen runtime no contiene el SDK.
Arreglo: copia `dotnet publish` o usa un build multi-stage.

**2. Mapear `5000:80` con imágenes .NET 10.**
Qué pasa: el host publica un puerto donde la aplicación no escucha.
Por qué: desde .NET 8 el valor predeterminado es 8080.
Arreglo: usa `5000:8080` o configura explícitamente el puerto interno.

**3. Confundir EXPOSE con publicación.**
Qué pasa: la API no es accesible desde el host.
Por qué: `EXPOSE` solo aporta metadatos.
Arreglo: usa `docker run -p host:container` o configuración equivalente.

**4. Guardar datos dentro del contenedor.**
Qué pasa: se pierden al reemplazar la instancia.
Por qué: la capa escribible pertenece al ciclo de vida del contenedor.
Arreglo: externaliza datos o usa un volumen con estrategia de respaldo.

**5. Copiar secretos en appsettings de la imagen.**
Qué pasa: cualquiera con acceso a capas o registry puede recuperarlos.
Por qué: borrar el archivo en una capa posterior no lo elimina de capas anteriores.
Arreglo: inyecta secretos durante ejecución.

**6. Afirmar que corre igual en cualquier entorno.**
Qué pasa: fallan arquitectura, kernel, filesystem, certificados o límites de recursos.
Por qué: el contenedor reduce diferencias, no las elimina.
Arreglo: declara plataforma, prueba la imagen exacta y verifica dependencias externas.

-----

## Según la versión de .NET

* **.NET 7 y anteriores:** las imágenes ASP.NET Core solían escuchar en el puerto 80.
* **.NET 8:** el puerto predeterminado cambia a 8080 y las imágenes incorporan el usuario `app`.
* **.NET 10:** se mantienen `8080`, `$APP_UID` y variantes oficiales para distintas distribuciones y arquitecturas.

-----

## Cuándo sí y cuándo no

**Usa contenedores cuando:** necesitas una unidad reproducible para CI/CD, despliegues consistentes, aislamiento de dependencias o plataformas que operan imágenes.

**No los agregues cuando:** el entorno de despliegue no los soporta o una publicación directa es más simple y suficiente. Docker añade builder, registry, parches de imágenes, escaneo y operación.

-----

## Resumen en 5 líneas

1. Una imagen es la plantilla inmutable; un contenedor es su proceso en ejecución.
2. La imagen aspnet ejecuta artefactos publicados, pero no compila el código.
3. ASP.NET Core usa el puerto 8080 por defecto en imágenes .NET 8 o posteriores.
4. EXPOSE documenta; `-p` publica un puerto del contenedor en el host.
5. Configuración, secretos y estado durable no deben quedar horneados en la imagen.

-----

## Para profundizar

<details>
<summary>Tags y digests</summary>

Un tag como `10.0` puede apuntar a una imagen parcheada diferente con el tiempo. Eso facilita recibir actualizaciones al reconstruir, pero reduce reproducibilidad absoluta. Un digest fija contenido exacto, aunque debes actualizarlo deliberadamente para recibir parches. En producción combina actualización automatizada, revisión y trazabilidad del digest desplegado.

</details>

<details>
<summary>Señales de finalización</summary>

Al detener un contenedor, el proceso recibe una señal y dispone de un periodo para terminar. Propaga `CancellationToken`, deja de aceptar trabajo y finaliza operaciones seguras. Si ignoras el cierre, el orquestador terminará el proceso y puede dejar trabajo incompleto.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una imagen contiene la aplicación publicada y sus dependencias. Docker crea contenedores, que son procesos aislados basados en esa imagen. Para una API .NET 10 se suele usar `aspnet:10.0`, puerto 8080 y un usuario no root.

### Respuesta ampliada (semi-senior)

Trato la imagen como artefacto inmutable y la configuro en ejecución. Distingo build context, capas y arquitectura objetivo. Publico 8080, ejecuto como `$APP_UID`, envío logs a stdout y externalizo estado. No prometo portabilidad total: pruebo la misma imagen por digest, considero kernel/arquitectura y mantengo un proceso de actualización y escaneo de bases.

### Preguntas frecuentes de seguimiento

**1. ¿EXPOSE abre el puerto?**
No. El mapeo ocurre con `-p` o el orquestador.

**2. ¿Un contenedor es una VM?**
No. Es un proceso aislado que comparte el kernel del host.

**3. ¿Por qué no guardar secretos en la imagen?**
Porque permanecen recuperables en sus capas y copias del registry.

-----

## Práctica

**Ejercicio 1.** Corrige `docker run -p 5000:80 myapp` para una imagen ASP.NET Core 10 sin configuración adicional.

<details>
<summary>Solución</summary>

```text
docker run --rm -p 5000:8080 myapp
```

El primer puerto es el del host; el segundo es el puerto predeterminado del contenedor.

</details>

**Ejercicio 2.** ¿Por qué `COPY appsettings.Production.json .` con una contraseña es inseguro aunque luego ejecutes `RUN rm appsettings.Production.json`?

<details>
<summary>Solución</summary>

Porque el archivo sigue presente en una capa anterior de la imagen. Los secretos deben proporcionarse en ejecución mediante un mecanismo diseñado para ello.

</details>

-----

## Siguiente lección

[Builds multi-stage e imágenes seguras](02-Builds%20multi-stage%20e%20imagenes%20seguras.md)
