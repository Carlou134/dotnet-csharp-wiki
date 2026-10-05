# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Activos de restore (*assets*).** Archivos que `dotnet restore` genera en `obj/`, como `project.assets.json`, con el grafo de dependencias ya resuelto. Son regenerables. ([SDK, CLI, proyectos y NuGet](07-SDK%20CLI%20proyectos%20y%20NuGet.md))

**AOP (programación orientada a aspectos).** Paradigma que separa la lógica transversal (logging, seguridad, transacciones) de la lógica de negocio.

**Argumento.** El valor que le pasas a un método entre paréntesis. En `Console.WriteLine("Hola")`, el argumento es `"Hola"`.

**Aspecto transversal (*cross-cutting concern*).** Lógica que se repite en muchas partes del sistema y no pertenece a ninguna en particular, como registrar logs o medir tiempos.

**BCL (Base Class Library).** Las librerías estándar de .NET: `Console`, `Math`, `String`, `List<T>` y miles de tipos más.

**C#.** Lenguaje de programación de Microsoft, orientado a objetos, con tipado estático y multiparadigma. Se pronuncia "ci sharp".

**CIL (Common Intermediate Language).** Nombre estándar del IL: código intermedio y portable que emiten los compiladores .NET. ([CLR y compilación](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md))

**CLI `dotnet`.** La herramienta de línea de comandos del SDK: `dotnet new`, `dotnet build`, `dotnet run`, `dotnet publish`.

**CLR (Common Language Runtime).** El motor de .NET que carga el IL, lo compila con el JIT, administra la memoria y maneja excepciones.

**CLS (Common Language Specification).** Subconjunto de reglas del CTS para crear APIs públicas interoperables entre lenguajes .NET. ([CLR y compilación](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md))

**Código administrado.** Código que se ejecuta bajo los servicios del CLR: GC, excepciones, seguridad de tipos y carga de ensamblados. ([CLR y compilación](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md))

**Comentario.** Texto dentro del código que el compilador ignora. En C#: `//`, `/* */` y `///` (documentación XML).

**Compilador.** Programa que traduce tu código a otro formato. El compilador de C# se llama Roslyn y genera IL.

**Consola (terminal).** Ventana de texto donde un programa muestra su salida y recibe la entrada del usuario.

**CoreCLR.** Implementación del CLR que usa .NET moderno. ([CLR y compilación](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md))

**CTS (Common Type System).** Reglas comunes para declarar, usar y administrar tipos en .NET. Por eso `int` de C# es `System.Int32` en cualquier lenguaje .NET. ([CLR y compilación](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md))

**Declarativo.** Estilo que describe qué resultado se quiere, no los pasos para obtenerlo. LINQ es declarativo.

**Efecto secundario.** Cualquier cambio que una función produce fuera de sí misma: modificar una variable externa, escribir en consola, guardar en base de datos.

**Ensamblado (*assembly*).** El archivo `.dll` o `.exe` que produce el compilador: contiene IL, metadatos y un manifiesto.

**Función pura.** Función que con la misma entrada siempre devuelve el mismo resultado y no tiene efectos secundarios.

**GC (Garbage Collector, recolector de basura).** Parte del CLR que libera automáticamente la memoria de los objetos que ya no se usan.

**global.json.** Archivo que fija la versión del SDK (o su política de resolución) que usa la CLI. No cambia el TFM. ([SDK, CLI, proyectos y NuGet](07-SDK%20CLI%20proyectos%20y%20NuGet.md))

**IDE.** Entorno de desarrollo integrado: editor, compilador, depurador y herramientas en un solo programa. Por ejemplo, Visual Studio o Rider.

**IL (Intermediate Language).** Código intermedio, independiente del procesador, que genera el compilador de C#. También se llama CIL o MSIL (nombre histórico).

**Imperativo.** Estilo que describe paso a paso cómo hacer algo: bucles, condiciones, asignaciones.

**Inmutabilidad.** No modificar datos existentes, sino crear datos nuevos a partir de ellos.

**IntelliSense.** El autocompletado inteligente del editor: sugiere nombres, muestra firmas y documentación.

**JIT (Just-In-Time).** Compilador del CLR que convierte el IL de un método en código máquina la primera vez que ese método se ejecuta.

**LTS (Long Term Support).** Versión de .NET con 3 años de soporte. Las versiones pares (8, 10) son LTS.

**Método.** Bloque de código con nombre que realiza una tarea. `WriteLine` es un método de la clase `Console`.

**MSBuild.** Motor que evalúa el `.csproj` y ejecuta los objetivos de restore, compilación y publicación. ([SDK, CLI, proyectos y NuGet](07-SDK%20CLI%20proyectos%20y%20NuGet.md))

**Native AOT.** Modo de compilación que genera código máquina antes de distribuir la app, sin JIT en ejecución.

**.NET.** Plataforma de desarrollo de Microsoft, gratuita, de código abierto y multiplataforma: compilador, runtime y librerías.

**.NET Framework.** La versión original de .NET, solo para Windows. Hoy está en mantenimiento; los proyectos nuevos usan .NET.

**NuGet.** El gestor de paquetes de .NET. Permite agregar librerías de terceros a un proyecto.

**PackageReference.** Elemento del `.csproj` que declara una dependencia NuGet y su versión. ([SDK, CLI, proyectos y NuGet](07-SDK%20CLI%20proyectos%20y%20NuGet.md))

**Paradigma.** Una forma o estilo de organizar y pensar el código.

**POO (programación orientada a objetos).** Paradigma que organiza el código en objetos con estado y comportamiento, basado en clases.

**Punto de entrada.** El lugar donde empieza a ejecutarse un programa: el método `Main` o las top-level statements.

**Restore.** Paso que resuelve y descarga las dependencias declaradas por el proyecto. `dotnet build` lo ejecuta de forma implícita. ([SDK, CLI, proyectos y NuGet](07-SDK%20CLI%20proyectos%20y%20NuGet.md))

**Runtime.** Lo mínimo necesario para ejecutar una app .NET ya compilada. No incluye el compilador.

**SDK (Software Development Kit).** Todo lo necesario para desarrollar en .NET: compilador, MSBuild, runtime y la CLI `dotnet`.

**Secuencia de escape.** Combinación que empieza con `\` dentro de un texto y representa un carácter especial: `\n` (salto de línea), `\"` (comillas).

**Sentencia.** Una instrucción completa. En C# termina con punto y coma `;`.

**String (cadena).** Un texto. En C# se escribe entre comillas dobles: `"Hola"`.

**STS (Standard Term Support).** Versión de .NET con 2 años de soporte. Las versiones impares (9, 11) son STS.

**Target framework (TFM).** La versión de .NET para la que se compila un proyecto, indicada en el `.csproj`. Por ejemplo, `net10.0`.

**Top-level statements.** Sentencias escritas directamente en un archivo, sin declarar `class Program` ni `Main`. El compilador genera esa estructura. Disponibles desde C# 9.

**Workload.** Conjunto opcional de herramientas del SDK para un tipo de aplicación, como MAUI o WebAssembly. Se instala con `dotnet workload install`. ([Plataforma .NET](05-Plataforma%20NET%20y%20evolucion.md))
