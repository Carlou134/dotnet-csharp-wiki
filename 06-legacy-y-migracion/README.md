# Legacy y migración

Este bloque estudia los sistemas .NET que siguen en producción en muchas empresas (.NET Framework, Web Forms, WinForms con DevExpress, WCF, VB.NET, IIS) con un objetivo práctico: poder mantenerlos con seguridad desde el primer día y decidir con criterio cuándo y cómo migrarlos.

-----

## Módulos
| Módulo | Contenido |
| --- | --- |
| [01. Tecnologías heredadas](01-tecnologias-heredadas/README.md) | .NET Framework frente a .NET moderno, ASP.NET Web Forms, Windows Forms y DevExpress, WCF y CoreWCF, VB.NET para desarrolladores de C# |
| [02. IIS y migración](02-iis-y-migracion/README.md) | Despliegue en IIS (in-process, out-of-process, errores 500.3x) y estrategias de migración con strangler fig y YARP |

-----

## Mapa
```text
reconocer (TFM, Web.config, System.Web) → mantener con seguridad → decidir si migrar
                                                                      │
                     en el lugar (bibliotecas, WinForms) · strangler fig (web) · CoreWCF (SOAP con terceros)
```

-----

## Próximos temas
WPF y MVVM, Entity Framework 6 y su migración a EF Core, y servicios de Windows (*Windows Services*) frente a *worker services*.
