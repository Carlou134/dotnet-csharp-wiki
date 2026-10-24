# Glosario: IIS y migración

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Actualización en el lugar (in-place):** cambiar el framework de destino de un proyecto y corregir lo que se rompe, sin reescribirlo. ([Migración](02-Estrategias%20de%20migracion.md))

**app_offline.htm:** archivo que, colocado en la carpeta de la aplicación, la detiene y muestra su contenido mientras se despliega. ([IIS](01-Despliegue%20en%20IIS.md))

**Application pool:** grupo de aplicaciones de IIS que comparten un proceso `w3wp.exe`, su identidad y su configuración de reciclado. ([IIS](01-Despliegue%20en%20IIS.md))

**ASP.NET Core Module (ANCM):** módulo nativo de IIS que hospeda o reenvía las peticiones a una aplicación ASP.NET Core. ([IIS](01-Despliegue%20en%20IIS.md))

**Big bang:** reescribir el sistema completo y reemplazarlo de una sola vez. ([Migración](02-Estrategias%20de%20migracion.md))

**Hosting Bundle:** instalador que agrega al servidor el runtime de .NET, el de ASP.NET Core y el ANCM. ([IIS](01-Despliegue%20en%20IIS.md))

**IIS (Internet Information Services):** servidor web de Windows. ([IIS](01-Despliegue%20en%20IIS.md))

**In-process / out-of-process:** la aplicación ASP.NET Core corre dentro de `w3wp.exe`, o en un proceso aparte con Kestrel y el ANCM como proxy. ([IIS](01-Despliegue%20en%20IIS.md))

**Pruebas de caracterización:** pruebas que capturan el comportamiento actual del sistema para detectar cambios no intencionales durante una migración. ([Migración](02-Estrategias%20de%20migracion.md))

**Reciclado:** reinicio periódico o por condiciones del proceso de un *application pool*; borra todo el estado en memoria. ([IIS](01-Despliegue%20en%20IIS.md))

**Sitio (site):** aplicación web publicada en IIS con uno o más *bindings* (protocolo, IP, puerto, nombre de host). ([IIS](01-Despliegue%20en%20IIS.md))

**Strangler fig (higuera estranguladora):** patrón en el que un sistema nuevo crece alrededor del viejo y lo reemplaza ruta por ruta hasta poder apagarlo. ([Migración](02-Estrategias%20de%20migracion.md))

**System.Web adapters:** paquetes de Microsoft para compartir sesión y autenticación, y usar APIs con forma de `System.Web`, entre ASP.NET clásico y ASP.NET Core durante la migración. ([Migración](02-Estrategias%20de%20migracion.md))

**YARP (Yet Another Reverse Proxy):** biblioteca de Microsoft para construir proxies inversos en ASP.NET Core. ([Migración](02-Estrategias%20de%20migracion.md))
