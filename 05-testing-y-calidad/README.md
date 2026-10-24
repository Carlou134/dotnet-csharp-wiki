# Testing y calidad

Este bloque estudia cómo saber que el código hace lo que debe, hoy y después de cada cambio: pruebas unitarias, dobles de prueba, cobertura y pruebas de integración con la API y la base reales, con la mirada puesta en pruebas que detectan errores y no en números que se ven bien.

-----

## Módulos
| Módulo | Contenido |
| --- | --- |
| [01. Pruebas unitarias](01-pruebas-unitarias/README.md) | TDD, xUnit (`Fact`, `Theory`, aserciones), dobles de prueba con Moq y cobertura con Coverlet |
| [02. Pruebas de integración](02-pruebas-de-integracion/README.md) | `WebApplicationFactory`, reemplazo de servicios y autenticación de prueba, Testcontainers, Respawn · [ejercicios](02-pruebas-de-integracion/Ejercicios.md) |

-----

## Mapa
```text
regla de negocio → prueba unitaria (rápida, aislada)
cableado real    → prueba de integración (base, HTTP)
sistema completo → prueba end-to-end (pocas)
calidad de las pruebas → cobertura (qué corrió) + mutación (qué detectan)
```

-----

## Próximos temas
Pruebas de contrato, pruebas de carga y pruebas end-to-end.
