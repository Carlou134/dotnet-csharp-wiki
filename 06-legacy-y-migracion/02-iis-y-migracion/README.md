# IIS y migración

En esta carpeta aprendes a desplegar y diagnosticar aplicaciones en IIS, tanto de .NET Framework como de ASP.NET Core, y a elegir la estrategia correcta para migrar un sistema heredado sin congelar el negocio: actualización en el lugar, *strangler fig* con YARP o reescritura.

-----

## Antes de empezar

Conviene que ya tengas:

* Las tecnologías heredadas y cómo reconocerlas: [Tecnologías heredadas](../01-tecnologias-heredadas/README.md).
* ASP.NET Core y su pipeline: [Fundamentos de ASP.NET Core](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/README.md).
* Pruebas de integración, para proteger la migración: [Pruebas de integración](../../05-testing-y-calidad/02-pruebas-de-integracion/README.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Despliegue en IIS](01-Despliegue%20en%20IIS.md) | Sitios, *application pools*, ANCM, in-process frente a out-of-process, errores 500.19/500.30/500.31 y despliegues con `app_offline.htm` | ASP.NET Core, .NET Framework |
| [2. Estrategias de migración](02-Estrategias%20de%20migracion.md) | En el lugar, *strangler fig* con YARP, reescritura, inventario, pruebas de caracterización y System.Web adapters | Tecnologías heredadas, pruebas |

-----

## El mapa completo en una mirada

```text
IIS
  sitio + binding ──► application pool (w3wp.exe, identidad, reciclado)
    .NET Framework: CLR dentro de w3wp · Web.config
    ASP.NET Core:   Hosting Bundle + ANCM · in-process (w3wp) o out-of-process (Kestrel) · pool sin código administrado
  500.19 config/módulo · 500.30 falla al arrancar · 500.31 falta runtime · 503 pool detenido
  desplegar: app_offline.htm → copiar → borrar

MIGRACIÓN
  inventario + pruebas de caracterización
  bibliotecas, WinForms ──► en el lugar (netstandard2.0 / net10.0)
  web (Web Forms, MVC 5) ──► strangler fig: app nueva + YARP + System.Web adapters, ruta por ruta
  WCF con terceros       ──► CoreWCF (mismo contrato) + gRPC/REST para nuevos
  big bang               ──► solo piezas chicas
```

-----

## Cómo está armada cada lección

Las dos lecciones siguen las 13 secciones de la wiki con ejemplos en .NET 10. La lección de IIS requiere Windows con IIS para el despliegue; la de migración corre en cualquier sistema.

-----

## Después de esta carpeta

Con esto terminas el bloque de legacy. Para el destino de las migraciones web, sigue con Razor Pages y Blazor (planificados en el bloque de ASP.NET Core); para el despliegue moderno, [Cloud native y contenedores](../../02-architecture-and-design-patterns/05-cloud-native-y-contenedores/README.md).
