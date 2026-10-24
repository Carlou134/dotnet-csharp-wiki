# ADO.NET y Dapper

En esta carpeta bajas un nivel por debajo de EF Core: cómo funcionan de verdad las conexiones, los comandos y los readers de ADO.NET; cómo Dapper elimina el mapeo repetitivo sin esconder el SQL; cómo conviven Dapper y EF Core en un mismo proyecto, y cómo las tres herramientas comparten una transacción.

-----

## Antes de empezar

Conviene que ya tengas:

* La introducción a ADO.NET y ORMs: [ADO.NET, ORMs y EF Core](../01-entity-framework-core/01-ADO%20NET%20ORMs%20y%20EF%20Core.md).
* EF Core hasta relaciones y CRUD: [Entity Framework Core](../01-entity-framework-core/README.md). Para las lecciones 1 y 2 alcanza con la lección 1 de ese módulo; la 3 y la 4 usan EF Core.
* SQL básico: `SELECT`, `JOIN`, `INSERT`, `UPDATE`.

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. ADO.NET a fondo](01-ADO%20NET%20a%20fondo.md) | Pool de conexiones, comandos, parámetros tipados, `DBNull`, readers y varios resultados | SQL básico |
| [2. Dapper: consultas y comandos](02-Dapper%20consultas%20y%20comandos.md) | `Query`/`Execute`, parámetros, `IN @Ids`, mapeo por nombre, `DynamicParameters` | ADO.NET |
| [3. Dapper avanzado y convivencia con EF Core](03-Dapper%20avanzado%20y%20convivencia%20con%20EF%20Core.md) | Multi-mapping, `QueryMultiple`, type handlers, conexión compartida y change tracker | Dapper, relaciones en EF Core |
| [4. Transacciones con ADO.NET, Dapper y EF Core](04-Transacciones%20con%20ADO%20NET%20Dapper%20y%20EF%20Core.md) | ACID, commit/rollback, transacción compartida, aislamiento, reintentos y `TransactionScope` | Las tres anteriores |

-----

## El mapa completo en una mirada

```text
                    tu código
       ┌───────────────┼────────────────┐
   EF Core          Dapper           ADO.NET a mano
 (LINQ → SQL,     (tu SQL → objetos)  (tu SQL, tu mapeo)
  tracking,              │                 │
  migraciones)           │                 │
       └───────────────┬─┴─────────────────┘
               DbConnection / DbCommand / DbDataReader
                  (pool de conexiones del proveedor)
                               │
                         base de datos

Transacción compartida = MISMA conexión:
  EF abre  → Dapper usa tx.GetDbTransaction()
  ADO abre → EF usa Database.UseTransaction(tx)
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki. Los ejemplos usan SQLite (`Microsoft.Data.Sqlite` 10, Dapper 2.1, EF Core 10) para correr en cualquier sistema; cuando SQL Server se comporta distinto (tipos, `AddWithValue`, aislamiento), la lección lo indica.

-----

## Después de esta carpeta

Continúa con [Rendimiento y concurrencia](../03-rendimiento-y-concurrencia/README.md): concurrencia optimista a fondo, índices y planes de ejecución. Para organizar dónde va cada herramienta en una aplicación, mira [CQRS y mediator](../../02-architecture-and-design-patterns/06-cqrs-y-mediator/README.md): commands con EF Core, queries que pueden usar Dapper.
