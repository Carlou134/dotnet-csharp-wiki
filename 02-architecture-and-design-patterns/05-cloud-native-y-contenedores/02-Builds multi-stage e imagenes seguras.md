# Builds multi-stage e imágenes seguras

## En una frase

Un build multi-stage usa el SDK para restaurar y publicar, pero copia a producción solo el resultado sobre una imagen runtime, reduciendo herramientas, capas innecesarias y superficie de ataque.

-----

## Antes de empezar

Conviene que ya sepas:

* Diferenciar imagen, contenedor y publicación: [Contenedores Docker](01-Contenedores%20Docker%20para%20APIs%20NET.md).
* Qué contiene un `.csproj` y cómo funciona `dotnet restore`.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Multi-stage build:** varias etapas dentro de un Dockerfile.
* **Runtime image:** base sin herramientas de compilación.
* **Image digest:** identidad del contenido exacto de la imagen.

-----

## El problema

Esta imagen funciona, pero lleva el SDK completo a producción:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o out
ENTRYPOINT ["dotnet", "out/Orders.Api.dll"]
```

Incluye compiladores, NuGet y archivos fuente que el proceso no necesita. Además, cualquier cambio en un `.cs` invalida la capa que restauró paquetes, porque el repositorio entero se copió antes de `dotnet publish`.

“Funciona” no alcanza. Una imagen de producción debe ser reproducible, actualizable, mínima **sin romper capacidades necesarias** y ejecutarse con privilegios reducidos.

-----

## Cómo funciona

### 1. Separa build y runtime

```text
stage build: sdk:10.0
  csproj → restore → source → publish
                              │
                              ▼ COPY --from
stage final: aspnet:10.0
  solo /app/publish → dotnet Orders.Api.dll
```

La etapa final no hereda las capas del SDK. Solo recibe los artefactos copiados explícitamente.

### 2. Ordena COPY para reutilizar caché

```dockerfile
COPY Orders.Api.csproj ./
RUN dotnet restore Orders.Api.csproj
COPY . .
RUN dotnet publish Orders.Api.csproj --no-restore ...
```

Mientras no cambie el proyecto ni sus dependencias, Docker puede reutilizar el restore aunque cambie código fuente. En soluciones con varios proyectos debes copiar todos los `.csproj`, props y archivos que influyen en restore antes de ejecutarlo.

### 3. Reduce el build context con .dockerignore

```dockerignore
**/bin/
**/obj/
.git/
.vs/
.idea/
.vscode/
publish/
*.user
*.suo
.env
```

Esto evita enviar artefactos, historial y secretos accidentales al builder. `.dockerignore` no reemplaza un escáner de secretos ni corrige un secreto ya confirmado en Git.

### 4. Ejecuta con el usuario no root de la imagen

```dockerfile
USER $APP_UID
```

Reduce el impacto de una vulnerabilidad, pero no vuelve segura la aplicación. También puedes usar filesystem de solo lectura, eliminar capacidades Linux y limitar recursos según la plataforma.

### 5. Elige la variante mínima que conserve tus requisitos

* `aspnet:10.0`: runtime de ASP.NET Core con una distribución Linux general.
* `aspnet:10.0-alpine`: menor, usa musl y puede afectar dependencias nativas.
* `aspnet:10.0-noble-chiseled`: sin shell ni package manager; menor superficie.
* variantes `extra`: agregan ICU y datos de zonas horarias para globalización completa.

Una imagen más pequeña puede dificultar diagnóstico o romper culturas, Kerberos, LDAP o librerías nativas. Mide y prueba; no optimices por una cifra aislada.

### 6. Multi-stage ayuda, pero no garantiza seguridad

Debes reconstruir periódicamente para recibir parches, escanear dependencias e imagen base, producir SBOM/provenance cuando corresponda y controlar quién puede publicar en el registry. Quitar el SDK elimina herramientas innecesarias; no corrige una biblioteca vulnerable dentro de la aplicación.

-----

## Ejemplo completo

Dockerfile para un proyecto `Orders.Api.csproj` ubicado en el mismo directorio:

```dockerfile
# syntax=docker/dockerfile:1

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY Orders.Api.csproj ./
RUN dotnet restore Orders.Api.csproj

COPY . .
RUN dotnet publish Orders.Api.csproj \
    -c Release \
    --no-restore \
    -o /app/publish \
    -p:UseAppHost=false

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
.idea/
.vscode/
publish/
.env
```

Comandos:

```text
docker build --pull -t orders-api:1.0.0 .
docker run --rm -p 5000:8080 orders-api:1.0.0
```

Para el endpoint fijo de la lección anterior:

```text
GET http://localhost:5000/
→ 200 {"service":"Orders.Api","status":"ready"}
```

`--pull` consulta una base actualizada para el tag. En CI también registra el digest final para saber exactamente qué se desplegó.

-----

## Errores comunes

**1. Usar sdk como imagen final.**
Qué pasa: se distribuyen compiladores, cachés y una superficie mayor.
Por qué: no se separaron responsabilidades de build y runtime.
Arreglo: usa etapas y termina en `aspnet`.

**2. Copiar todo antes de restore.**
Qué pasa: cualquier cambio de código vuelve a descargar/restaurar dependencias.
Por qué: se invalida la capa previa.
Arreglo: copia primero archivos de proyecto y restore; luego el código.

**3. Omitir .dockerignore.**
Qué pasa: el contexto incluye `bin`, `.git`, secretos locales y archivos enormes.
Por qué: Docker envía todo lo no excluido.
Arreglo: crea y revisa `.dockerignore`.

**4. Ejecutar como root.**
Qué pasa: una explotación obtiene privilegios mayores dentro del contenedor.
Por qué: no se seleccionó el usuario no privilegiado.
Arreglo: `USER $APP_UID` y puerto 8080.

**5. Elegir chiseled sin revisar globalización.**
Qué pasa: fallan culturas, zonas horarias o diagnóstico basado en shell.
Por qué: la variante elimina componentes deliberadamente.
Arreglo: usa `extra` o una base compatible y pruébala.

**6. Usar latest o confiar solo en un tag mutable.**
Qué pasa: dos builds con el mismo Dockerfile reciben bases distintas sin trazabilidad.
Por qué: el tag puede moverse.
Arreglo: registra digests y automatiza actualizaciones controladas; no te congeles para siempre en una base vulnerable.

-----

## Según la versión de .NET

* **.NET 8:** imágenes oficiales adoptan puerto 8080 y usuario `app` con `$APP_UID`.
* **.NET 8 en adelante:** Ubuntu chiseled y otras variantes reducidas amplían opciones de hardening.
* **.NET 10:** las muestras oficiales usan SDK 10.0, runtime 10.0, usuario no root y Dockerfiles preparados para distintas arquitecturas.

-----

## Cuándo sí y cuándo no

**Usa multi-stage cuando:** Docker construye la aplicación y quieres una imagen final sin SDK, reproducible en CI y con capas cacheables.

**No necesitas compilar dentro de Docker cuando:** CI ya produce artefactos firmados y controlados que otra fase empaqueta. Separar build y packaging puede ser válido, pero debes conservar trazabilidad y compatibilidad de plataforma.

-----

## Resumen en 5 líneas

1. Multi-stage compila con sdk y ejecuta con aspnet.
2. Copiar csproj antes del código permite reutilizar la capa de restore.
3. .dockerignore reduce contexto y evita copiar basura o secretos accidentales.
4. USER $APP_UID aplica defensa en profundidad sin privilegios root.
5. Una imagen mínima exige parches, escaneo y pruebas de compatibilidad.

-----

## Para profundizar

<details>
<summary>BuildKit y cachés</summary>

BuildKit permite montajes de caché para NuGet sin convertir el caché en una capa de la imagen. Mejora tiempos de CI, pero la caché no debe ocultar restores no deterministas. Usa lock files cuando el contexto lo requiera y conserva registros de dependencias resueltas.

</details>

<details>
<summary>Multi-arquitectura</summary>

Una imagen manifest puede apuntar a variantes `linux/amd64` y `linux/arm64`. El build debe publicar para cada arquitectura y probar dependencias nativas. Que el código C# sea portable no garantiza que una biblioteca nativa incluida lo sea.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un Dockerfile multi-stage usa una imagen SDK para restaurar y publicar, y una imagen ASP.NET más pequeña para ejecutar. Así la imagen final no contiene compiladores ni código fuente.

### Respuesta ampliada (semi-senior)

Optimizo capas copiando archivos de proyecto antes del código, uso `--no-restore`, `.dockerignore`, usuario no root y una base compatible con globalización y dependencias nativas. Registro el digest desplegado, reconstruyo por parches y escaneo la cadena de suministro. No confundo tamaño con seguridad: una imagen mínima también necesita actualizaciones, autorización del registry y pruebas.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué la etapa SDK no agranda la imagen final?**
Porque la etapa final parte de otra base y solo copia `/app/publish`.

**2. ¿Chiseled siempre es mejor?**
No. Reduce componentes, pero puede faltar globalización, shell o dependencias necesarias.

**3. ¿Multi-stage elimina vulnerabilidades?**
Solo evita herramientas innecesarias; debes escanear y actualizar aplicación y base.

-----

## Práctica

**Ejercicio 1.** ¿Qué instrucciones deben ocurrir antes para aprovechar caché: `COPY . .` o copiar csproj y `dotnet restore`?

<details>
<summary>Solución</summary>

Primero se copian los archivos que determinan dependencias y se ejecuta restore. Después se copia el código. Así un cambio en `.cs` no invalida restore.

</details>

**Ejercicio 2.** ¿Quitar el SDK vuelve segura la imagen?

<details>
<summary>Solución</summary>

No. Reduce superficie y tamaño, pero pueden quedar vulnerabilidades en runtime, sistema base o dependencias de la aplicación. También importan usuario, capacidades, secretos, registry y parches.

</details>

-----

## Siguiente lección

[Native AOT y diseño cloud native](03-Native%20AOT%20y%20diseno%20cloud%20native.md)
