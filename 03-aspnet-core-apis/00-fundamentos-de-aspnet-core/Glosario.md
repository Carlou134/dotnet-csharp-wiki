# Glosario: fundamentos de ASP.NET Core

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Action:** método público de un controlador que el routing puede invocar. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**Blazor:** modelo de UI web interactiva basado en componentes Razor. ([Modelos de aplicaciones](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md))

**Controlador:** clase que agrupa acciones HTTP relacionadas; `ControllerBase` para APIs y `Controller` para vistas. ([Modelos de aplicaciones](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md))

**Endpoint:** destino final que el routing selecciona para atender una petición. ([Host y pipeline](01-Host%20configuracion%20y%20pipeline%20HTTP.md))

**Host:** objeto que reúne configuración, logging, inyección de dependencias, servidor web y ciclo de vida de la aplicación. ([Host y pipeline](01-Host%20configuracion%20y%20pipeline%20HTTP.md))

**Inyección de dependencias (DI):** mecanismo por el que el contenedor crea los objetos que un componente necesita y se los entrega, con un lifetime singleton, scoped o transient. ([Host y pipeline](01-Host%20configuracion%20y%20pipeline%20HTTP.md))

**Middleware:** componente del pipeline que procesa la petición antes y después del siguiente componente. ([Host y pipeline](01-Host%20configuracion%20y%20pipeline%20HTTP.md))

**Minimal API:** endpoints declarados con `MapGet`, `MapPost` y un handler, sin controladores. ([Modelos de aplicaciones](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md))

**Model binding:** construcción de los parámetros de una acción a partir de la ruta, la query string, el formulario o el cuerpo. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**Modo de renderizado:** dónde y cómo se ejecuta un componente Blazor: SSR estático, Interactive Server, Interactive WebAssembly o Auto. ([Modelos de aplicaciones](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md))

**MVC:** patrón de presentación que separa modelo, vista y controlador. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**Options pattern:** lectura de una sección de configuración como una clase tipada, con validación opcional al arrancar. ([Host y pipeline](01-Host%20configuracion%20y%20pipeline%20HTTP.md))

**Razor:** sintaxis que combina HTML con C# para producir UI web; codifica la salida por defecto. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**Razor Pages:** modelo de UI centrado en páginas, cada una con su `PageModel`. ([Modelos de aplicaciones](02-Modelos%20de%20aplicaciones%20ASPNET%20Core.md))

**Routing:** proceso que compara la URL y el método HTTP con los endpoints registrados. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**Ruta convencional:** patrón global, como `{controller}/{action}/{id?}`, que traduce URLs a controlador y acción. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**Tag Helper:** atributo de servidor en una vista Razor (como `asp-action` o `asp-for`) que genera HTML a partir del modelo y las rutas. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**ViewModel:** modelo diseñado para lo que necesita una vista, separado de las entidades persistentes. ([MVC y Razor](03-MVC%20routing%20y%20vistas%20Razor.md))

**wwwroot:** raíz pública convencional de los archivos estáticos; todo lo que contiene se sirve sin autenticación. ([Host y pipeline](01-Host%20configuracion%20y%20pipeline%20HTTP.md))
