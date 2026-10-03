# Glosario: cloud native y contenedores

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Build context:** conjunto de archivos que Docker puede enviar al builder durante `docker build`. ([Contenedores](01-Contenedores%20Docker%20para%20APIs%20NET.md))

**Cloud native:** enfoque para construir y operar sistemas que aprovechan automatización, elasticidad, resiliencia y servicios administrados de la nube. ([Native AOT y cloud native](03-Native%20AOT%20y%20diseno%20cloud%20native.md))

**Container (contenedor):** proceso aislado iniciado desde una imagen y que comparte el kernel del host. ([Contenedores](01-Contenedores%20Docker%20para%20APIs%20NET.md))

**Dockerfile:** receta declarativa para construir las capas y configuración de una imagen. ([Contenedores](01-Contenedores%20Docker%20para%20APIs%20NET.md))

**Framework-dependent:** publicación que necesita un runtime .NET compatible en la imagen final. ([Multi-stage](02-Builds%20multi-stage%20e%20imagenes%20seguras.md))

**Image (imagen):** conjunto inmutable de capas y metadatos usado para crear contenedores. ([Contenedores](01-Contenedores%20Docker%20para%20APIs%20NET.md))

**Image digest:** identificador criptográfico del contenido exacto de una imagen. ([Multi-stage](02-Builds%20multi-stage%20e%20imagenes%20seguras.md))

**Multi-stage build:** Dockerfile con etapas separadas para compilar y producir una imagen final mínima. ([Multi-stage](02-Builds%20multi-stage%20e%20imagenes%20seguras.md))

**Native AOT:** publicación que compila IL a código nativo antes de ejecutar y no usa CoreCLR/JIT en producción. ([Native AOT](03-Native%20AOT%20y%20diseno%20cloud%20native.md))

**Registry:** servicio que almacena y distribuye imágenes de contenedor. ([Native AOT](03-Native%20AOT%20y%20diseno%20cloud%20native.md))

**Runtime image:** imagen que contiene lo necesario para ejecutar, pero no el SDK de compilación. ([Multi-stage](02-Builds%20multi-stage%20e%20imagenes%20seguras.md))

**Trimming:** análisis que elimina código considerado no utilizado; puede romper reflexión o carga dinámica no declarada. ([Native AOT](03-Native%20AOT%20y%20diseno%20cloud%20native.md))
