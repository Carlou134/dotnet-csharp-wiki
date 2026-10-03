# Cloud native y contenedores con .NET

En esta carpeta aprendes a empaquetar una API .NET 10 como imagen reproducible, separar compilación y ejecución con un Dockerfile multi-stage, ejecutar sin root y decidir con criterio si Native AOT compensa sus restricciones. El contenedor es una unidad de entrega; no convierte por sí solo una aplicación en cloud native.

-----

## Antes de empezar

Conviene que ya tengas:

* Minimal APIs y configuración: [Minimal APIs](../../03-aspnet-core-apis/02-minimal-apis/README.md).
* Resiliencia ante dependencias remotas: [Resiliencia de servicios](../03-resiliencia-de-servicios/README.md).
* Métricas, logs y trazas: [Observabilidad](../04-observabilidad-con-opentelemetry/README.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Contenedores Docker para APIs .NET](01-Contenedores%20Docker%20para%20APIs%20NET.md) | Imagen frente a contenedor, publicación, puerto 8080, filesystem y configuración | Minimal APIs |
| [2. Builds multi-stage e imágenes seguras](02-Builds%20multi-stage%20e%20imagenes%20seguras.md) | Caché de capas, SDK frente a runtime, usuario no root, `.dockerignore` y variantes de imagen | Dockerfile básico |
| [3. Native AOT y diseño cloud native](03-Native%20AOT%20y%20diseno%20cloud%20native.md) | Compilación nativa, trimming, compatibilidad, medición y requisitos operativos cloud native | Multi-stage, observabilidad |

Para practicar: [Ejercicios de contenedores](Ejercicios.md), con los cinco ejercicios y los tres retos de la Sesión 15 corregidos.

-----

## El mapa completo en una mirada

```text
Código + csproj ── SDK/publish ──> artefactos ── runtime image ──> imagen inmutable
                                                           │
docker run -p 5000:8080 <imagen> ──────────────────────────┘
           host : contenedor

Framework-dependent   aspnet:10.0         ejecuta Orders.Api.dll con CoreCLR/JIT
Native AOT            runtime-deps:10.0   ejecuta binario nativo, sin CoreCLR

Contenedor            empaqueta proceso y dependencias; comparte kernel del host
Cloud native          despliegue automatizado + estado externo + resiliencia + observabilidad
No garantiza          portabilidad total · seguridad · escalabilidad · rendimiento
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki. Los Dockerfiles usan imágenes oficiales de .NET 10 y el puerto `8080`. Los comandos se documentan, pero sus tamaños y tiempos deben medirse en el equipo y arquitectura reales.

-----

## Después de esta carpeta

El siguiente paso es diseñar entrega y operación: registry con análisis de vulnerabilidades y firmas, CI/CD, despliegues progresivos, configuración y secretos administrados, health checks y un orquestador solo cuando la complejidad lo justifique.
