# Acceso a datos y transacciones

Este bloque estudia persistencia con criterio: modelado, consultas, cambios de esquema, transacciones, concurrencia y rendimiento. Un CRUD es solo el comienzo; el objetivo es mantener integridad y comportamiento predecible bajo carga.

-----

## Módulos
| Módulo | Contenido |
| --- | --- |
| [01. Entity Framework Core](01-entity-framework-core/README.md) | `DbContext`, configuración, migraciones y consultas |

-----

## Mapa
```text
dominio -> caso de uso -> persistencia -> base de datos
                            |
                            +-> transacción, concurrencia, medición
```

-----

## Próximos temas
Relaciones, transacciones ACID, concurrencia optimista, rendimiento SQL, índices y planes de ejecución.
