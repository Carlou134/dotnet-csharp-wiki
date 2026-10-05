# Glosario: observabilidad con OpenTelemetry

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Activity:** representación de .NET para una operación trazada; normalmente corresponde a un span. ([Trazas](01-Trazas%20distribuidas.md))

**ActivitySource:** fuente con nombre que crea actividades y permite que los listeners decidan cuáles recopilar. ([Trazas](01-Trazas%20distribuidas.md))

**Cardinalidad:** cantidad de combinaciones distintas que pueden tener las dimensiones de una métrica. ([Métricas](02-Metricas%20con%20Meter.md))

**Counter:** instrumento monotónico que acumula cantidades, como pedidos creados. ([Métricas](02-Metricas%20con%20Meter.md))

**Dimensión (atributo o tag):** clave y valor que permiten segmentar telemetría, como `payment.method=card`. ([Métricas](02-Metricas%20con%20Meter.md))

**Exporter:** componente que envía la telemetría recopilada a consola, OTLP o un backend. ([Logs y correlación](03-Logs%20estructurados%20y%20correlacion.md))

**Histogram:** instrumento que registra una distribución de valores, como duración o tamaño. ([Métricas](02-Metricas%20con%20Meter.md))

**Log estructurado:** evento con plantilla y atributos consultables, no una cadena ya concatenada. ([Logs y correlación](03-Logs%20estructurados%20y%20correlacion.md))

**Meter:** fuente con nombre que crea instrumentos de métricas. ([Métricas](02-Metricas%20con%20Meter.md))

**Observabilidad:** capacidad de comprender el estado interno de un sistema a partir de sus señales externas. ([Logs y correlación](03-Logs%20estructurados%20y%20correlacion.md))

**Observable gauge:** instrumento que lee el valor actual mediante un callback en cada ciclo de recolección, como la profundidad de una cola. ([Métricas](02-Metricas%20con%20Meter.md))

**OpenTelemetry (OTel):** estándar y conjunto de APIs, SDKs y herramientas para generar, recopilar y exportar telemetría. ([Trazas](01-Trazas%20distribuidas.md))

**OTLP:** protocolo neutral de OpenTelemetry para transportar trazas, métricas y logs. ([Logs y correlación](03-Logs%20estructurados%20y%20correlacion.md))

**Resource:** atributos que identifican al productor de telemetría, como nombre, versión e instancia del servicio. ([Logs y correlación](03-Logs%20estructurados%20y%20correlacion.md))

**Sampling (muestreo):** decisión de conservar solo una parte de las trazas para controlar costo y volumen. ([Trazas](01-Trazas%20distribuidas.md))

**Span:** unidad de trabajo con nombre, duración, estado, atributos y relación padre-hijo. ([Trazas](01-Trazas%20distribuidas.md))

**Trace:** árbol de spans que representa el recorrido de una operación distribuida. ([Trazas](01-Trazas%20distribuidas.md))
