# Observabilidad con OpenTelemetry

En esta carpeta aprendes a diagnosticar un sistema desde su telemetría: seguir una petición con trazas distribuidas, medir tendencias con métricas y registrar eventos con logs estructurados. .NET aporta las APIs `ActivitySource`, `Meter` e `ILogger`; OpenTelemetry las recopila y exporta sin atar la aplicación a un proveedor concreto.

-----

## Antes de empezar

Conviene que ya tengas:

* `async`/`await` y aplicaciones ASP.NET Core: [Minimal APIs](../../03-aspnet-core-apis/02-minimal-apis/README.md).
* Fallos, latencia y dependencias remotas: [Resiliencia de servicios](../03-resiliencia-de-servicios/README.md).
* Plantillas de logging en lugar de interpolación: [Texto, char y string](../../01-csharp-core-and-runtime/01-tipos-y-variables/05-Texto%20char%20y%20string.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Trazas distribuidas](01-Trazas%20distribuidas.md) | Trace, span, contexto W3C, instrumentación automática y `ActivitySource` | ASP.NET Core, HTTP |
| [2. Métricas con Meter](02-Metricas%20con%20Meter.md) | Counter, histogram, observable gauge, dimensiones y cardinalidad | Tipos numéricos, DI |
| [3. Logs estructurados y correlación](03-Logs%20estructurados%20y%20correlacion.md) | `ILogger`, scopes, correlación con trazas, recursos, OTLP y límites de la observabilidad | Trazas y métricas |

Para practicar: [Ejercicios de observabilidad](Ejercicios.md), con los cinco ejercicios y los tres retos de la Sesión 13 corregidos.

-----

## El mapa completo en una mirada

```text
Pregunta concreta                           Señal principal
¿Qué ocurrió en esta petición?              trace + spans
¿Con qué frecuencia y cómo evoluciona?      métricas agregadas
¿Qué evento detallado ocurrió?              logs estructurados

.NET API                Instrumenta                 OpenTelemetry SDK          Backend
ActivitySource    ─┐
Meter              ├──> recopila + procesa ───────> OTLP / exporter ────────> consulta, panel, alerta
ILogger           ─┘

Telemetría ≠ observabilidad automática
Faltan preguntas, convenciones, retención, paneles, alertas y una respuesta operativa.
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki. Los ejemplos son minimal APIs para .NET 10 y usan el exportador de consola solo para aprender. En producción se recomienda OTLP hacia un OpenTelemetry Collector o un backend compatible.

-----

## Después de esta carpeta

Continúa con [Cloud native y contenedores](../05-cloud-native-y-contenedores/README.md) para empaquetar y operar APIs .NET. Allí conectarás observabilidad, resiliencia, configuración externa y despliegues reemplazables.
