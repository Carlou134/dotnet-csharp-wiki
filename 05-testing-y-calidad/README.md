# Testing y calidad

Este bloque estudia cómo saber que el código hace lo que debe, hoy y después de cada cambio: pruebas unitarias, dobles de prueba y cobertura, con la mirada puesta en pruebas que detectan errores y no en números que se ven bien.

-----

## Módulos
| Módulo | Contenido |
| --- | --- |
| [01. Pruebas unitarias](01-pruebas-unitarias/README.md) | TDD, xUnit (`Fact`, `Theory`, aserciones), dobles de prueba con Moq y cobertura con Coverlet |

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
Pruebas de integración con `WebApplicationFactory`, Testcontainers, pruebas de contrato y pruebas de carga.
