# Acceso a datos y transacciones

Este bloque estudia persistencia con criterio: modelado, consultas, cambios de esquema, transacciones, concurrencia y rendimiento. Un CRUD es solo el comienzo; el objetivo es mantener integridad y comportamiento predecible bajo carga.

-----

## Módulos
| Módulo | Contenido |
| --- | --- |
| [01. Entity Framework Core](01-entity-framework-core/README.md) | ADO.NET y ORMs, `DbContext`, modelado, migraciones, relaciones, carga de datos y CRUD |
| [02. ADO.NET y Dapper](02-ado-net-y-dapper/README.md) | ADO.NET a fondo, Dapper, convivencia con EF Core y transacciones |

-----

## Mapa
```text
dominio -> caso de uso -> persistencia -> base de datos
                            |
                            +-> transacción, concurrencia, medición
```

-----

## Próximos temas
Concurrencia optimista en profundidad, rendimiento SQL, índices y planes de ejecución.
