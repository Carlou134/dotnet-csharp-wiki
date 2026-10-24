# Rendimiento y concurrencia

En esta carpeta pasas de un CRUD que funciona a un acceso a datos que aguanta carga: concurrencia optimista a fondo (y cuándo una actualización atómica es mejor), índices y planes de ejecución, cómo detectar consultas lentas y N+1 en producción, y paginación por cursor junto con el resto de las técnicas de consultas eficientes.

-----

## Antes de empezar

Conviene que ya tengas:

* EF Core completo, en especial relaciones y CRUD: [Entity Framework Core](../01-entity-framework-core/README.md).
* Transacciones y SQL propio: [ADO.NET y Dapper](../02-ado-net-y-dapper/README.md).
* Logging con `ILogger`: [Logging con ILogger](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/04-Logging%20con%20ILogger.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Concurrencia optimista a fondo](01-Concurrencia%20optimista%20a%20fondo.md) | *Lost update*, tokens (`rowversion`, `xmin`), estrategias de resolución, ETag/`If-Match`, actualizaciones atómicas y bloqueo pesimista | Change tracker, transacciones |
| [2. Índices y planes de ejecución](02-Indices%20y%20planes%20de%20ejecucion.md) | Árboles B, scan frente a seek, sargabilidad, índices compuestos, cubrientes, filtrados y su costo | SQL básico, Fluent API |
| [3. Detectar consultas lentas en producción](03-Detectar%20consultas%20lentas%20en%20produccion.md) | Logs de comandos, interceptores, `TagWith`, presupuesto de consultas, OpenTelemetry, Query Store y `pg_stat_statements` | Logging, N+1 |
| [4. Paginación y consultas eficientes](04-Paginacion%20y%20consultas%20eficientes.md) | Offset frente a cursor, orden total, cursores opacos, consultas compiladas, pooling y streaming | Índices |

Para practicar: [Ejercicios de rendimiento y concurrencia](Ejercicios.md), con tres ejercicios guiados, dos retos y un checkpoint.

-----

## El mapa completo en una mirada

```text
CONCURRENCIA
  edición de usuarios  → token (rowversion / xmin / Guid) + ETag/If-Match → 409/412 + resolver por negocio
  contadores y stock   → UPDATE ... SET x = x - n WHERE x >= n  (atómico, sin ventana)
  alta contención      → bloqueo pesimista corto, nunca abarcando al usuario

ÍNDICES
  columna sola a un lado (sargable) · compuesto: igualdad → rango/orden · cubrir = no leer la tabla
  plan: SEARCH/Seek ✔   SCAN ✗ (en tablas grandes)   · cada índice encarece las escrituras

PRODUCCIÓN
  log de comandos en Warning · interceptor de lentas · TagWith · contar consultas por petición
  trazas (por qué tardó esta petición) · Query Store / pg_stat_statements (qué consume más en total)

PAGINACIÓN
  offset: simple, lento en profundidad, repite/omite filas · cursor: "después de X", estable, con índice
  orden total (desempate por Id) · Take(n+1) en vez de COUNT · cursor opaco
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki. Los ejemplos usan SQLite para ser reproducibles; cuando SQL Server o PostgreSQL se comportan distinto (tokens, planes, índices incluidos), la lección lo indica.

-----

## Después de esta carpeta

El siguiente paso del roadmap es la caché: `IMemoryCache`, `HybridCache` y Redis, con estrategias de invalidación (planificado). Recuerda el orden: primero se arregla la consulta (este módulo), después se cachea.
