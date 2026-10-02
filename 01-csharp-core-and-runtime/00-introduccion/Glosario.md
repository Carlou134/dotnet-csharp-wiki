# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**AOP (programación orientada a aspectos).** Paradigma que separa la lógica transversal (logging, seguridad, transacciones) de la lógica de negocio.

**Argumento.** El valor que le pasas a un método entre paréntesis. En `Console.WriteLine("Hola")`, el argumento es `"Hola"`.

**Aspecto transversal (*cross-cutting concern*).** Lógica que se repite en muchas partes del sistema y no pertenece a ninguna en particular, como registrar logs o medir tiempos.

**BCL (Base Class Library).** Las librerías estándar de .NET: `Console`, `Math`, `String`, `List<T>` y miles de tipos más.

**C#.** Lenguaje de programación de Microsoft, orientado a objetos, con tipado estático y multiparadigma. Se pronuncia "ci sharp".

**CLI `dotnet`.** La herramienta de línea de comandos del SDK: `dotnet new`, `dotnet build`, `dotnet run`, `dotnet publish`.

**CLR (Common Language Runtime).** El motor de .NET que carga el IL, lo compila con el JIT, administra la memoria y maneja excepciones.

**Comentario.** Texto dentro del código que el compilador ignora. En C#: `//`, `/* */` y `///` (documentación XML).

**Compilador.** Programa que traduce tu código a otro formato. El compilador de C# se llama Roslyn y genera IL.

**Consola (terminal).** Ventana de texto donde un programa muestra su salida y recibe la entrada del usuario.

**Declarativo.** Estilo que describe qué resultado se quiere, no los pasos para obtenerlo. LINQ es declarativo.

**Efecto secundario.** Cualquier cambio que una función produce fuera de sí misma: modificar una variable externa, escribir en consola, guardar en base de datos.

**Ensamblado (*assembly*).** El archivo `.dll` o `.exe` que produce el compilador: contiene IL, metadatos y un manifiesto.

**Función pura.** Función que con la misma entrada siempre devuelve el mismo resultado y no tiene efectos secundarios.

**GC (Garbage Collector, recolector de basura).** Parte del CLR que libera automáticamente la memoria de los objetos que ya no se usan.

**IDE.** Entorno de desarrollo integrado: editor, compilador, depurador y herramientas en un solo programa. Por ejemplo, Visual Studio o Rider.

**IL (Intermediate Language).** Código intermedio, independiente del procesador, que genera el compilador de C#. También se llama CIL o MSIL.

**Imperativo.** Estilo que describe paso a paso cómo hacer algo: bucles, condiciones, asignaciones.

**Inmutabilidad.** No modificar datos existentes, sino crear datos nuevos a partir de ellos.

**IntelliSense.** El autocompletado inteligente del editor: sugiere nombres, muestra firmas y documentación.

**JIT (Just-In-Time).** Compilador del CLR que convierte el IL de un método en código máquina la primera vez que ese método se ejecuta.

**LTS (Long Term Support).** Versión de .NET con 3 años de soporte. Las versiones pares (8, 10) son LTS.

**Método.** Bloque de código con nombre que realiza una tarea. `WriteLine` es un método de la clase `Console`.

**Native AOT.** Modo de compilación que genera código máquina antes de distribuir la app, sin JIT en ejecución.

**.NET.** Plataforma de desarrollo de Microsoft, gratuita, de código abierto y multiplataforma: compilador, runtime y librerías.

**.NET Framework.** La versión original de .NET, solo para Windows. Hoy está en mantenimiento; los proyectos nuevos usan .NET.

**NuGet.** El gestor de paquetes de .NET. Permite agregar librerías de terceros a un proyecto.

**Paradigma.** Una forma o estilo de organizar y pensar el código.

**POO (programación orientada a objetos).** Paradigma que organiza el código en objetos con estado y comportamiento, basado en clases.

**Punto de entrada.** El lugar donde empieza a ejecutarse un programa: el método `Main` o las top-level statements.

**Runtime.** Lo mínimo necesario para ejecutar una app .NET ya compilada. No incluye el compilador.

**SDK (Software Development Kit).** Todo lo necesario para desarrollar en .NET: compilador, MSBuild, runtime y la CLI `dotnet`.

**Secuencia de escape.** Combinación que empieza con `\` dentro de un texto y representa un carácter especial: `\n` (salto de línea), `\"` (comillas).

**Sentencia.** Una instrucción completa. En C# termina con punto y coma `;`.

**String (cadena).** Un texto. En C# se escribe entre comillas dobles: `"Hola"`.

**STS (Standard Term Support).** Versión de .NET con 2 años de soporte. Las versiones impares (9, 11) son STS.

**Target framework (TFM).** La versión de .NET para la que se compila un proyecto, indicada en el `.csproj`. Por ejemplo, `net10.0`.

**Top-level statements.** Sentencias escritas directamente en un archivo, sin declarar `class Program` ni `Main`. El compilador genera esa estructura. Disponibles desde C# 9.
