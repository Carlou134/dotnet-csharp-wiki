# Glosario: ADO.NET y Dapper

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**ACID:** atomicidad, consistencia, aislamiento y durabilidad, las cuatro garantías de una transacción. ([Transacciones](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md))

**Commit / rollback:** confirmar o deshacer todo lo hecho dentro de una transacción. ([Transacciones](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md))

**Conjunto de resultados (result set):** cada bloque de filas que devuelve un comando; se avanza al siguiente con `NextResult`. ([ADO.NET](01-ADO%20NET%20a%20fondo.md))

**Consulta con buffer:** comportamiento por defecto de Dapper, que lee todas las filas a una lista antes de devolverla. ([Dapper](02-Dapper%20consultas%20y%20comandos.md))

**Dapper:** micro-ORM que mapea resultados de SQL a objetos con métodos de extensión sobre `DbConnection`. ([Dapper](02-Dapper%20consultas%20y%20comandos.md))

**DbCommand:** objeto con el texto SQL, los parámetros y la transacción con la que se ejecuta. ([ADO.NET](01-ADO%20NET%20a%20fondo.md))

**DbDataReader:** cursor de solo avance y solo lectura sobre las filas de un comando. ([ADO.NET](01-ADO%20NET%20a%20fondo.md))

**DBNull.Value:** representación del `NULL` de SQL en ADO.NET; distinto de `null` en C#. ([ADO.NET](01-ADO%20NET%20a%20fondo.md))

**Deadlock (interbloqueo):** dos transacciones que esperan cada una un bloqueo que tiene la otra; la base cancela una. ([Transacciones](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md))

**DynamicParameters:** colección de parámetros de Dapper que se arma en tiempo de ejecución. ([Dapper](02-Dapper%20consultas%20y%20comandos.md))

**Estrategia de ejecución:** mecanismo de EF Core que reintenta operaciones ante errores transitorios y exige envolver las transacciones manuales. ([Transacciones](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md))

**GridReader:** lector que devuelve `QueryMultiple` para recorrer varios resultados en orden. ([Dapper avanzado](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md))

**Materialización:** conversión de una fila del resultado en un objeto de C#. ([Dapper](02-Dapper%20consultas%20y%20comandos.md))

**Modelo de lectura:** clases pensadas solo para mostrar datos, distintas de las entidades que se modifican. ([Dapper avanzado](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md))

**Multi-mapping:** mapeo de cada fila de un `JOIN` a varios objetos unidos por una función. ([Dapper avanzado](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md))

**Nivel de aislamiento:** cuánto ve una transacción de los cambios no confirmados de otras. ([Transacciones](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md))

**Objeto de parámetros:** objeto cuyas propiedades Dapper convierte en parámetros SQL con el mismo nombre. ([Dapper](02-Dapper%20consultas%20y%20comandos.md))

**Ordinal:** posición numérica de una columna dentro del resultado. ([ADO.NET](01-ADO%20NET%20a%20fondo.md))

**Pool de conexiones:** conjunto de conexiones físicas que se reutilizan; `Open` toma una y `Dispose` la devuelve. ([ADO.NET](01-ADO%20NET%20a%20fondo.md))

**Savepoint:** punto intermedio dentro de una transacción al que se puede volver sin deshacerla entera. ([Transacciones](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md))

**splitOn:** nombre de la columna donde empieza el siguiente objeto en un multi-mapping. ([Dapper avanzado](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md))

**Transacción:** grupo de operaciones que la base confirma o descarta como una unidad. ([Transacciones](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md))

**Type handler:** clase que enseña a Dapper a convertir un tipo propio desde y hacia la base. ([Dapper avanzado](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md))
