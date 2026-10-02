# Resiliencia de servicios con Polly

En esta carpeta aprendes a tratar los fallos remotos como parte normal de un sistema distribuido: reintentar solo errores transitorios, dejar de llamar temporalmente a una dependencia dañada, limitar cada intento y construir un *pipeline* HTTP con Polly 8 y `Microsoft.Extensions.Http.Resilience`.

-----

## Antes de empezar

Conviene que ya tengas:

* `async`/`await`, cancelación y excepciones: [Asincronía y archivos](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).
* Códigos de estado y métodos HTTP: [Diseño de APIs REST](../../03-aspnet-core-apis/01-diseno-de-apis-rest/README.md).
* Reintentos e idempotencia entre consumidores: [Entrega confiable](../02-event-driven-architecture/03-Entrega%20confiable%20e%20idempotencia.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Reintentos y backoff](01-Reintentos%20y%20backoff.md) | Fallos transitorios, límite de intentos, espera exponencial, jitter e idempotencia | Excepciones, HTTP |
| [2. Circuit breaker](02-Circuit%20breaker.md) | Estados cerrado, abierto y medio abierto; ventana de muestreo y estado compartido | Reintentos |
| [3. Timeouts y pipelines de resiliencia HTTP](03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md) | Cancelación cooperativa, timeout por intento y total, orden de estrategias e integración con `HttpClientFactory` | Retry, circuit breaker |

Para practicar: [Ejercicios de resiliencia](Ejercicios.md), con los cinco ejercicios y los tres retos de la Sesión 12 corregidos.

-----

## El mapa completo en una mirada

```text
Fallo transitorio       puede desaparecer pronto             → retry con límite + backoff + jitter
Fallo permanente        repetir no cambia el resultado       → fallar y corregir la causa
Dependencia degradada   muchas llamadas siguen fallando      → circuit breaker, fail fast
Operación demasiado lenta                                    → timeout + CancellationToken

Pipeline HTTP estándar (de afuera hacia adentro):
rate limiter → timeout total → retry → circuit breaker → timeout por intento → llamada

Retry aumenta carga y latencia · circuit breaker no repara nada · timeout necesita cooperación
POST/PUT/PATCH/DELETE solo se reintentan si la operación es idempotente o usa clave de idempotencia
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki: problema antes que solución, ejemplo completo, errores con causa técnica, diferencias por versión, criterios de uso, entrevista y práctica.

Los ejemplos usan Polly 8. Para HTTP se emplea `Microsoft.Extensions.Http.Resilience`, no la API heredada `Policy`/`Policy.WrapAsync` de Polly 7.

-----

## Después de esta carpeta

La resiliencia evita que un fallo remoto se propague sin control, pero no reemplaza la observabilidad ni corrige una dependencia. Continúa con *health checks*, métricas, trazas distribuidas y pruebas de caos; para proteger la entrada de una API, estudia *rate limiting* en ASP.NET Core.
