# Glosario: resiliencia de servicios

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Backoff:** espera creciente entre reintentos para no insistir al mismo ritmo sobre una dependencia degradada. ([Reintentos](01-Reintentos%20y%20backoff.md))

**Circuit breaker:** estrategia que deja de ejecutar temporalmente una operación cuando los fallos superan un umbral. ([Circuit breaker](02-Circuit%20breaker.md))

**Fallo permanente:** error que no desaparecerá por repetir la misma operación, como credenciales inválidas o un `400 Bad Request`. ([Reintentos](01-Reintentos%20y%20backoff.md))

**Fallo transitorio:** error temporal que puede desaparecer al esperar, como una desconexión breve, `408`, `429` o ciertos `5xx`. ([Reintentos](01-Reintentos%20y%20backoff.md))

**Fail fast (fallar rápido):** rechazar una operación sin esperar a una dependencia que se sabe degradada. ([Circuit breaker](02-Circuit%20breaker.md))

**Idempotencia:** propiedad por la que repetir una operación produce el mismo efecto observable que ejecutarla una vez. ([Reintentos](01-Reintentos%20y%20backoff.md))

**Jitter:** variación aleatoria añadida a la espera para que muchos clientes no reintenten al mismo tiempo. ([Reintentos](01-Reintentos%20y%20backoff.md))

**Pipeline de resiliencia:** composición ordenada de estrategias que envuelven una operación. ([Timeouts y pipelines](03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md))

**Timeout por intento:** tiempo máximo permitido para una ejecución individual, incluida cada repetición. ([Timeouts y pipelines](03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md))

**Timeout total:** presupuesto máximo para la operación completa, incluidas esperas y repeticiones. ([Timeouts y pipelines](03-Timeouts%20y%20pipelines%20de%20resiliencia%20HTTP.md))

**Ventana de muestreo:** intervalo reciente en el que el circuit breaker calcula la proporción de fallos. ([Circuit breaker](02-Circuit%20breaker.md))
