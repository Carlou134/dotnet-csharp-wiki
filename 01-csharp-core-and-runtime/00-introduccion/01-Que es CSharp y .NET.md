# Qué es C# y .NET

## En una frase

C# es un lenguaje de programación con tipado estático; .NET es la plataforma que compila ese código a un formato intermedio (IL) y lo ejecuta en cualquier sistema operativo mediante un motor llamado CLR.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un programa: una lista de instrucciones que la computadora ejecuta.
* Qué es un sistema operativo (Windows, Linux, macOS) y que cada uno ejecuta programas a su manera.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **C#:** el lenguaje. Lo que escribes en archivos `.cs`.
* **.NET:** la plataforma: compilador, motor de ejecución y librerías.
* **SDK:** el kit para *desarrollar* (compilador + herramientas + runtime).
* **Runtime:** lo mínimo para *ejecutar* una app ya compilada.
* **IL (Intermediate Language):** el código intermedio que genera el compilador de C#.
* **CLR (Common Language Runtime):** el motor que ejecuta el IL.
* **JIT (Just-In-Time):** la parte del CLR que convierte IL en código máquina mientras el programa corre.
* **BCL (Base Class Library):** las librerías estándar de .NET (`Console`, `Math`, `List<T>`...).

-----

## El problema

Una computadora solo entiende código máquina, y ese código es distinto para cada procesador (x64, ARM) y para cada sistema operativo. Si compilas directo a código máquina, como en C, tienes que compilar una versión por cada combinación de plataforma.

Además, escribir directo para la máquina te obliga a manejar la memoria a mano: pedirla, liberarla y no equivocarte nunca. Un solo error es un cuelgue o una vulnerabilidad.

.NET resuelve los dos problemas. Compilas una sola vez a un formato intermedio que no depende de la máquina, y un motor se ocupa de ejecutarlo y de administrar la memoria por ti.

-----

## Cómo funciona

### C# no es .NET

Es la confusión más común. **C#** es el lenguaje: sintaxis, palabras clave y reglas de tipos. **.NET** es la plataforma donde ese lenguaje vive. Sobre .NET corren otros lenguajes (F#, Visual Basic) y todos comparten el mismo runtime y las mismas librerías.

| Pieza | Qué es | Ejemplo |
| --- | --- | --- |
| Lenguaje | Las reglas para escribir código | C# 14, F# |
| Compilador | Traduce C# a IL | Roslyn (`csc`) |
| Runtime | Ejecuta el IL | CLR |
| Librerías | Código ya escrito que reutilizas | `System.Console`, `System.Math` |
| SDK | Todo lo anterior + la CLI `dotnet` | .NET 10 SDK |

### El recorrido del código

```text
Program.cs  ──(compilador Roslyn)──►  MiApp.dll (IL + metadatos)  ──(CLR + JIT)──►  código máquina  ──►  CPU
  C#                                    independiente de la CPU                        específico de tu CPU
```

1. Escribes C# en archivos `.cs`.
2. `dotnet build` invoca al compilador, que valida tipos y genera un **ensamblado** (`.dll`) con IL y metadatos (qué clases y métodos hay).
3. `dotnet run` (o ejecutar la app) arranca el **CLR**, que carga el ensamblado.
4. El **JIT** traduce cada método a código máquina la primera vez que se llama.
5. Mientras el programa corre, el **recolector de basura (GC)** libera la memoria de los objetos que ya no se usan.

### Qué te da el CLR gratis

* **Gestión de memoria automática:** no hay `free()` ni `delete`. El GC limpia.
* **Seguridad de tipos:** no puedes tratar un `string` como si fuera un `int`; el compilador y el runtime lo impiden.
* **Excepciones:** los errores se propagan de forma controlada en lugar de corromper la memoria.
* **Multiplataforma:** el mismo `.dll` corre en Windows, Linux y macOS.

### Qué puedes construir con C#

| Tipo de app | Tecnología |
| --- | --- |
| APIs y backend web | ASP.NET Core |
| Aplicaciones de escritorio | WPF, WinForms (Windows), Avalonia, .NET MAUI |
| Móvil | .NET MAUI |
| Videojuegos | Unity, Godot |
| Servicios en la nube | Azure Functions, workers |
| Herramientas de consola | Aplicación de consola (lo que usarás en estas lecciones) |

### Por qué C# te hace mejor programador

* **Tipado estático:** cada variable tiene un tipo conocido al compilar, así que muchos errores que en JavaScript aparecen en producción aquí aparecen antes de ejecutar.
* **Multiparadigma:** es orientado a objetos, pero también tiene herramientas funcionales (LINQ, lambdas, records). Se ve en [Paradigmas de programación](04-Paradigmas%20de%20programacion.md).
* **Rendimiento:** el JIT genera código máquina optimizado. ASP.NET Core está entre los frameworks web más rápidos en los benchmarks públicos de TechEmpower.

-----

## Ejemplo completo

Puedes ver el IL que genera el compilador. Con este programa:

```csharp
int a = 2;
int b = 3;
Console.WriteLine(a + b);
```

el compilador produce, de forma simplificada, algo así (lo puedes inspeccionar con herramientas como ILSpy o [sharplab.io](https://sharplab.io)):

```text
ldc.i4.2        // apila la constante 2
ldc.i4.3        // apila la constante 3
add             // suma los dos valores de la pila
call void [System.Console]System.Console::WriteLine(int32)
ret
```

No necesitas leer IL para programar en C#. Lo importante es la idea: **tu código C# no se ejecuta tal cual; primero se convierte en IL, y el CLR convierte ese IL en código máquina.**

-----

## Errores comunes

**1. Decir "programo en .NET" cuando quieres decir "programo en C#".**
Qué pasa: en una entrevista suena a que no distingues lenguaje de plataforma.
Por qué: son cosas distintas: puedes usar .NET con F# y C# con Unity (que usa su propio runtime).
Arreglo: "Programo en C# sobre .NET 10".

**2. Confundir .NET Framework con .NET.**
Qué pasa: buscas documentación de .NET Framework 4.8 y no aplica a tu proyecto moderno (o al revés).
Por qué: .NET Framework es la versión antigua, solo para Windows, y está en mantenimiento. .NET (antes ".NET Core") es la plataforma moderna y multiplataforma.
Arreglo: fíjate en la versión de la documentación (`net-10.0` frente a `netframework-4.8`).

**3. Instalar solo el runtime y querer compilar.**
Qué pasa: `dotnet build` falla con "No .NET SDKs were found".
Por qué: el runtime ejecuta apps, pero no trae compilador.
Arreglo: instala el **SDK**. Se ve en [Preparar el entorno](02-Preparar%20el%20entorno.md).

-----

## Según la versión de C#

C# y .NET avanzan juntos: cada versión de .NET trae una versión de C# por defecto.

| .NET | C# | Año | Soporte |
| --- | --- | --- | --- |
| .NET Framework 4.8 | 7.3 | 2019 | Solo Windows, mantenimiento |
| .NET 6 | 10 | 2021 | LTS (finalizado) |
| .NET 8 | 12 | 2023 | LTS |
| .NET 9 | 13 | 2024 | STS |
| .NET 10 | 14 | 2025 | LTS (la recomendada hoy) |

* **LTS** (Long Term Support): 3 años de soporte. Es lo que eligen las empresas.
* **STS** (Standard Term Support): 2 años de soporte (antes eran 18 meses).

En estas lecciones se usa **.NET 10 / C# 14**. Cuando una característica es nueva, la sección "Según la versión de C#" de cada lección lo indica, porque en el trabajo vas a encontrar código escrito con versiones anteriores.

-----

## Cuándo sí y cuándo no

**Usa C# / .NET cuando:**

* Construyes backend, APIs o servicios empresariales: es uno de los ecosistemas más maduros para eso.
* Quieres tipado estático fuerte con buen rendimiento sin manejar memoria a mano.
* Desarrollas videojuegos con Unity o Godot.

**Ten en cuenta:**

* Para scripts muy cortos, Python o Bash pueden ser más rápidos de escribir (aunque C# ya permite ejecutar un solo archivo con `dotnet run app.cs` desde .NET 10).
* Para sistemas embebidos con muy poca memoria o código de sistema operativo, C, C++ o Rust siguen siendo la opción habitual.

-----

## Resumen en 5 líneas

1. C# es el lenguaje; .NET es la plataforma (compilador, runtime y librerías).
2. El compilador convierte C# en IL, que se guarda en un ensamblado `.dll`.
3. El CLR carga el IL y el JIT lo convierte en código máquina mientras el programa corre.
4. El GC libera la memoria automáticamente; tú no llamas a `free()`.
5. Para desarrollar necesitas el SDK; para ejecutar basta el runtime.

-----

## Para profundizar

<details>
<summary>JIT, AOT y compilación por niveles</summary>

El JIT no optimiza todo de entrada. Con la **compilación por niveles** (tiered compilation), primero genera código rápido de compilar y poco optimizado, y si un método se llama muchas veces lo recompila con más optimizaciones. Por eso una app .NET "calienta" en los primeros segundos.

Existe también **Native AOT** (Ahead-Of-Time): compila todo a código máquina antes de distribuir, sin JIT en ejecución. Arranca más rápido y ocupa menos memoria, pero pierde algunas capacidades dinámicas (como parte de la reflexión). Se usa en microservicios y herramientas de línea de comandos.

</details>

<details>
<summary>CTS y CLS: por qué varios lenguajes conviven</summary>

El **CTS** (Common Type System) define los tipos que todos los lenguajes .NET comparten: el `int` de C# y el `Integer` de VB son ambos `System.Int32`. El **CLS** (Common Language Specification) es el subconjunto de reglas que una librería debe respetar para poder usarse desde cualquier lenguaje .NET. Por eso una librería escrita en F# se puede usar desde C# sin traducción.

</details>

-----

## En entrevista

### Respuesta corta (junior)

C# es un lenguaje orientado a objetos y con tipado estático. .NET es la plataforma que lo ejecuta: el compilador convierte el código C# en un lenguaje intermedio (IL) y el CLR lo ejecuta, convirtiéndolo en código máquina con el JIT. Además, el recolector de basura administra la memoria automáticamente.

### Respuesta ampliada (semi-senior)

C# compila a IL mediante Roslyn y genera ensamblados con IL y metadatos. En ejecución, el CLR carga el ensamblado, verifica tipos y el JIT compila cada método a código nativo la primera vez que se invoca, con compilación por niveles para optimizar los métodos calientes. El GC es generacional (generaciones 0, 1 y 2, más el LOH para objetos grandes) y libera la memoria de los objetos inalcanzables. Gracias al CTS, todos los lenguajes .NET comparten tipos. Para escenarios de arranque rápido existe Native AOT, que elimina el JIT a cambio de restricciones en la reflexión. Hoy se trabaja con .NET 10 (LTS) y C# 14; .NET Framework 4.8 queda para mantenimiento de sistemas Windows heredados.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre .NET Framework, .NET Core y .NET?**
.NET Framework es la plataforma original, solo para Windows. .NET Core fue la reescritura multiplataforma y de código abierto. Desde la versión 5 se llama simplemente ".NET" y es la plataforma actual.

**2. ¿Qué es un ensamblado?**
La unidad de despliegue de .NET: un `.dll` o `.exe` que contiene IL, metadatos de tipos y un manifiesto con su versión y dependencias.

**3. ¿C# es compilado o interpretado?**
Compilado dos veces: de C# a IL al compilar, y de IL a código máquina con el JIT al ejecutar (o todo antes, con AOT). No es interpretado línea por línea.

**4. ¿Qué es el JIT?**
El compilador Just-In-Time del CLR. Traduce el IL de cada método a código máquina la primera vez que se llama y guarda el resultado para las siguientes llamadas.

-----

## Práctica

**Ejercicio 1.** Explica con tus palabras, en tres pasos, qué pasa desde que escribes `Console.WriteLine("Hola");` hasta que el texto aparece en pantalla.

<details>
<summary>Solución</summary>

1. El compilador (Roslyn) valida el código y lo traduce a IL dentro de un `.dll`.
2. Al ejecutar, el CLR carga ese `.dll` y el JIT convierte el método a código máquina.
3. La CPU ejecuta ese código, que llama a `Console.WriteLine`, y el sistema operativo muestra el texto en la consola.

</details>

**Ejercicio 2.** Un compañero dice: "Instalé .NET pero no puedo compilar". ¿Qué le preguntas primero?

<details>
<summary>Solución</summary>

Si instaló el **SDK** o solo el **runtime**. Con `dotnet --list-sdks` lo verifica: si la lista sale vacía, no tiene compilador.

</details>

-----

## Siguiente lección

[Preparar el entorno](02-Preparar%20el%20entorno.md)
