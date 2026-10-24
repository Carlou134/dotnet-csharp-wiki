# Glosario: rendimiento y concurrencia

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Actualización atómica:** una sola sentencia que lee y escribe en la base (`SET Stock = Stock - 1 WHERE Stock > 0`), sin ventana entre lectura y escritura. ([Concurrencia](01-Concurrencia%20optimista%20a%20fondo.md))

**Actualización perdida (lost update):** cambio sobrescrito sin aviso porque otro guardó encima con datos leídos antes. ([Concurrencia](01-Concurrencia%20optimista%20a%20fondo.md))

**Concurrencia pesimista:** bloquear una fila al leerla para que nadie más la modifique hasta terminar la transacción. ([Concurrencia](01-Concurrencia%20optimista%20a%20fondo.md))

**Consulta compilada:** consulta LINQ traducida una vez y reutilizada con `EF.CompileAsyncQuery`. ([Paginación](04-Paginacion%20y%20consultas%20eficientes.md))

**Cursor:** valor, normalmente opaco, que identifica dónde terminó la página anterior. ([Paginación](04-Paginacion%20y%20consultas%20eficientes.md))

**Etiqueta de consulta (query tag):** comentario que EF Core agrega al SQL con `TagWith` para identificar su origen. ([Consultas lentas](03-Detectar%20consultas%20lentas%20en%20produccion.md))

**ETag:** encabezado HTTP con la versión de un recurso; con `If-Match`, el cliente pide modificarlo solo si no cambió. ([Concurrencia](01-Concurrencia%20optimista%20a%20fondo.md))

**Índice:** estructura ordenada, normalmente un árbol B, con los valores de una o más columnas y la ubicación de cada fila. ([Índices](02-Indices%20y%20planes%20de%20ejecucion.md))

**Índice compuesto:** índice sobre varias columnas, ordenado primero por la primera, después por la segunda, etc. ([Índices](02-Indices%20y%20planes%20de%20ejecucion.md))

**Índice que cubre (covering index):** índice que contiene todas las columnas que la consulta necesita, así que no hace falta leer la tabla. ([Índices](02-Indices%20y%20planes%20de%20ejecucion.md))

**Interceptor de comandos:** clase `DbCommandInterceptor` de EF Core que se ejecuta antes y después de cada comando SQL. ([Consultas lentas](03-Detectar%20consultas%20lentas%20en%20produccion.md))

**Orden total:** ordenamiento sin empates, logrado agregando una columna única al final. ([Paginación](04-Paginacion%20y%20consultas%20eficientes.md))

**Paginación por cursor (keyset):** pedir las filas que siguen a la última vista, filtrando por su clave de ordenamiento. ([Paginación](04-Paginacion%20y%20consultas%20eficientes.md))

**Paginación por offset:** pedir una página saltando N filas (`Skip`/`Take`, `OFFSET`/`LIMIT`). ([Paginación](04-Paginacion%20y%20consultas%20eficientes.md))

**Plan de ejecución:** estrategia que elige el motor para resolver una consulta: índices, orden de uniones, ordenamientos. ([Índices](02-Indices%20y%20planes%20de%20ejecucion.md))

**Pooling de contextos:** reutilización de instancias de `DbContext` entre peticiones con `AddDbContextPool`. ([Paginación](04-Paginacion%20y%20consultas%20eficientes.md))

**Presupuesto de consultas:** máximo de consultas aceptable para una petición; superarlo señala un N+1. ([Consultas lentas](03-Detectar%20consultas%20lentas%20en%20produccion.md))

**Query Store / pg_stat_statements:** registros del motor (SQL Server / PostgreSQL) con estadísticas acumuladas por consulta. ([Consultas lentas](03-Detectar%20consultas%20lentas%20en%20produccion.md))

**rowversion / xmin:** columnas que la base actualiza sola en cada cambio, usadas como token de concurrencia (SQL Server / PostgreSQL). ([Concurrencia](01-Concurrencia%20optimista%20a%20fondo.md))

**Sargable:** condición que el motor puede resolver con un índice (Search ARGument ABLE). ([Índices](02-Indices%20y%20planes%20de%20ejecucion.md))

**Scan frente a seek / search:** recorrer toda la tabla o índice frente a ir directo a las filas que cumplen la condición. ([Índices](02-Indices%20y%20planes%20de%20ejecucion.md))

**Selectividad:** proporción de filas que una condición deja pasar; cuanto menor, más útil es el índice. ([Índices](02-Indices%20y%20planes%20de%20ejecucion.md))

**Token de concurrencia:** columna que cambia en cada actualización y se usa en el `WHERE` para detectar si la fila cambió. ([Concurrencia](01-Concurrencia%20optimista%20a%20fondo.md))

**Umbral de lentitud:** duración a partir de la cual una consulta se considera lenta y se registra. ([Consultas lentas](03-Detectar%20consultas%20lentas%20en%20produccion.md))
