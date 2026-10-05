# CLR, compilación y sistema de tipos

## En una frase

Roslyn traduce C# a CIL y metadatos dentro de un ensamblado; el CLR carga ese ensamblado, garantiza la seguridad de tipos durante la ejecución, administra la memoria y usa el JIT para convertir cada método en instrucciones de la CPU real.

-----

## Antes de empezar

Conviene que ya sepas:

* El recorrido general del código y qué aporta el CLR: [Qué es C# y .NET](01-Que%20es%20CSharp%20y%20.NET.md).
* Qué es un TFM y por qué .NET es multiplataforma: [Plataforma .NET y su evolución](05-Plataforma%20NET%20y%20evolucion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **CIL (*Common Intermediate Language*):** código intermedio, independiente del procesador, que emiten los compiladores .NET.
* **Metadatos:** descripción de los tipos, métodos y referencias que viaja dentro del ensamblado.
* **CTS (*Common Type System*):** reglas comunes para declarar y usar tipos en .NET.
* **CLS (*Common Language Specification*):** subconjunto del CTS para APIs públicas interoperables entre lenguajes.
* **Código administrado:** código que se ejecuta bajo los servicios del CLR.
* **CoreCLR:** implementación del CLR que usa .NET moderno.

-----

## El problema

Un equipo guarda la clave de un proveedor de pagos en el código:

```csharp
public static class PagosConfig
{
    public const string ApiKey = "sk_live_51H8...";
}
```

El razonamiento es: "el `.dll` está compilado, nadie puede leerlo". Es falso. El ensamblado contiene CIL y **metadatos**: nombres de tipos, métodos, propiedades y constantes. Cualquier descompilador (ILSpy, dotPeek) reconstruye un C# casi idéntico al original, constante incluida.

El mismo malentendido aparece al revés: "compilé en Windows, así que el `.dll` no corre en Linux". Un ensamblado *framework-dependent* contiene CIL, no instrucciones x64 de Windows. Corre en cualquier sistema que tenga el runtime .NET compatible.

Las dos explicaciones habituales están incompletas:

* "C# compila directamente a código máquina": no explica por qué un `.dll` se puede descompilar ni por qué es portable.
* ".NET interpreta C#": no explica por qué el código alcanza velocidad nativa.

Sin entender el recorrido completo, tomas malas decisiones de seguridad, despliegue y rendimiento.

-----

## Cómo funciona

### 1. Del código fuente a la CPU

```text
 Tiempo de compilación                     Tiempo de ejecución
┌──────────────────────────┐             ┌─────────────────────────────────────────┐
│ Pedido.cs                │             │ CLR (CoreCLR)                           │
│     │                    │             │   carga el ensamblado                   │
│     ▼                    │             │   lee los metadatos                     │
│  Roslyn (compilador C#)  │  Orders.dll │   primera llamada a un método:          │
│     │                    │ ──────────► │      JIT: CIL ──► código máquina x64/ARM│
│     ▼                    │             │   llamadas siguientes: código ya nativo │
│ CIL + metadatos          │             │   GC, excepciones, hilos, seguridad de  │
│ + manifiesto             │             │   tipos                                 │
└──────────────────────────┘             └─────────────────────────────────────────┘
```

Hay **dos compiladores**: Roslyn (C# → CIL, al compilar) y el JIT (CIL → código máquina, al ejecutar).

### 2. Roslyn: el compilador de C#

Roslyn analiza sintaxis y semántica, informa los errores CSxxxx y emite CIL y metadatos. También es una plataforma: los analizadores (como CA1416), las refactorizaciones del IDE y los *source generators* usan sus APIs.

### 3. El ensamblado: CIL, metadatos y manifiesto

Para este tipo:

```csharp
public sealed class Pedido
{
    public int Cantidad { get; init; }
    public decimal PrecioUnitario { get; init; }
    public decimal CalcularTotal() => Cantidad * PrecioUnitario;
}
```

El CIL de `CalcularTotal` es, simplificado:

```text
ldarg.0                                   // carga 'this'
call     int32 Pedido::get_Cantidad()
call     decimal Decimal::op_Implicit(int32)   // int → decimal
ldarg.0
call     decimal Pedido::get_PrecioUnitario()
call     decimal Decimal::op_Multiply(decimal, decimal)
ret
```

Observa:

* El CIL es una **máquina de pila**: carga valores, llama, devuelve. No habla de registros de una CPU concreta.
* Los nombres (`Pedido`, `get_Cantidad`) siguen ahí. Eso son los metadatos.
* La conversión implícita `int → decimal` que C# escribe por ti aparece explícita.

El **manifiesto** declara el nombre y la versión del ensamblado y qué otros ensamblados necesita.

### 4. CLR y JIT

El **CLR** es el entorno de ejecución administrada. **CoreCLR** es su implementación en .NET moderno. Proporciona:

| Servicio | Qué hace |
| --- | --- |
| Carga de ensamblados | Encuentra y carga los `.dll` que el manifiesto referencia |
| JIT | Compila cada método a código máquina la primera vez que se llama |
| GC | Libera la memoria de objetos inalcanzables |
| Excepciones | Propaga errores con su pila de llamadas |
| Seguridad de tipos | Lanza `InvalidCastException` o `IndexOutOfRangeException` en vez de corromper memoria |

El JIT de .NET moderno usa **compilación por niveles** (*tiered compilation*):

```text
primera llamada ──► Tier 0: compilación rápida, poco optimizada
         método muy usado (llamado muchas veces)
                ──► Tier 1: recompilación optimizada, con datos reales de ejecución (PGO dinámico)
```

Así una aplicación arranca rápido y, al poco tiempo, sus métodos calientes quedan muy optimizados para la CPU exacta donde corren.

### 5. CTS: un sistema de tipos para todos los lenguajes

El CTS define qué es un tipo en .NET: clases, structs, interfaces, enums, delegados; tipos de valor y de referencia; todos derivan de `System.Object`.

Por eso `int` de C# y `Integer` de Visual Basic son el **mismo** tipo: `System.Int32`. Una biblioteca escrita en F# expone tipos que C# usa sin conversión.

### 6. CLS: el subconjunto para APIs públicas

No todo lenguaje .NET soporta todo el CTS. Visual Basic, históricamente, no tenía enteros sin signo. El CLS define reglas para que una API pública sea consumible desde cualquier lenguaje.

```csharp
[assembly: CLSCompliant(true)]

public sealed class Inventario
{
    public uint Stock { get; set; }   // warning CS3003: Type of 'Inventario.Stock' is not CLS-compliant
}
```

El CLS solo afecta a la **superficie pública**. Un `uint` privado no genera advertencia.

### 7. JIT, ReadyToRun y Native AOT

| Modo | Cuándo se genera código máquina | Ventaja | Costo |
| --- | --- | --- | --- |
| JIT (por defecto) | Al ejecutar cada método | Optimiza para la CPU real; soporta todo el lenguaje | Arranque algo más lento |
| ReadyToRun | Al publicar, con JIT como respaldo | Mejor arranque | Archivos más grandes |
| Native AOT | Todo al publicar | Arranque rápido, sin JIT | Sin carga dinámica de código; reflexión limitada |

Native AOT se estudia a fondo en [Native AOT y diseño cloud native](../../02-architecture-and-design-patterns/05-cloud-native-y-contenedores/03-Native%20AOT%20y%20diseno%20cloud%20native.md).

-----

## Ejemplo completo

Aplicación de consola que inspecciona tipos y metadatos en ejecución:

```csharp
int cantidad = 3;

Console.WriteLine(cantidad.GetType().FullName);
Console.WriteLine(typeof(int) == typeof(System.Int32));
Console.WriteLine(typeof(string).Assembly.GetName().Name);

foreach (var propiedad in typeof(Pedido).GetProperties())
{
    Console.WriteLine($"{propiedad.Name}: {propiedad.PropertyType.Name}");
}

var metodo = typeof(Pedido).GetMethod(nameof(Pedido.CalcularTotal))!;
Console.WriteLine($"{metodo.Name} tiene CIL: {metodo.GetMethodBody()?.GetILAsByteArray()?.Length > 0}");

public sealed class Pedido
{
    public int Cantidad { get; init; }
    public decimal PrecioUnitario { get; init; }
    public decimal CalcularTotal() => Cantidad * PrecioUnitario;
}
```

Salida:

```text
System.Int32
True
System.Private.CoreLib
Cantidad: Int32
PrecioUnitario: Decimal
CalcularTotal tiene CIL: True
```

Qué observar:

* `int` es un alias de C# para el tipo CTS `System.Int32`.
* Los tipos básicos viven en `System.Private.CoreLib`, el ensamblado central de CoreCLR.
* La reflexión lee los **metadatos**: nombres y tipos de las propiedades están dentro del `.dll`. Por eso un descompilador puede reconstruir tu código, y por eso un secreto en una constante no está oculto.

-----

## Errores comunes

**1. Decir que el CLR "es un compilador".**
Qué pasa: se confunden responsabilidades al diagnosticar un problema.
Por qué: el CLR hospeda la ejecución; el JIT es uno de sus componentes, y Roslyn es otro compilador que ni siquiera corre en producción.
Arreglo: separa compilador del lenguaje (Roslyn), runtime (CLR) y JIT.

**2. Guardar secretos en el código porque "está compilado".**
Qué pasa: la clave aparece completa al abrir el `.dll` con ILSpy.
Por qué: el ensamblado guarda constantes y cadenas en metadatos legibles.
Arreglo: lee secretos desde configuración segura (variables de entorno, *user secrets*, un gestor de secretos).

**3. Usar "MIL" como nombre del código intermedio.**
Qué pasa: la búsqueda no encuentra documentación.
Por qué: los nombres correctos son CIL o IL; MSIL es el nombre histórico.
Arreglo: usa CIL o IL.

**4. Confundir CTS con CLS.**
Qué pasa: se cree que un `uint` en una API pública "no es válido en .NET".
Por qué: el CTS permite `uint`; el CLS solo lo desaconseja en APIs públicas que quieran ser interoperables.
Arreglo: piensa "universo de tipos" (CTS) frente a "subconjunto público interoperable" (CLS).

**5. Medir rendimiento con la primera llamada.**
Qué pasa: un *benchmark* casero dice que un método tarda 40 ms, y en producción tarda microsegundos.
Por qué: la primera llamada incluye el JIT, y las siguientes usan código Tier 0 hasta que se recompila en Tier 1.
Arreglo: mide con BenchmarkDotNet, que hace calentamiento y repeticiones.

-----

## Según la versión de .NET

* **.NET Framework:** usa su propio CLR, solo para Windows.
* **.NET Core 1.0:** introduce CoreCLR, multiplataforma.
* **.NET Core 3.0:** la compilación por niveles (*tiered compilation*) queda activada por defecto.
* **.NET 7:** Native AOT tiene soporte oficial para aplicaciones de consola.
* **.NET 8:** el PGO dinámico queda activado por defecto: el JIT usa datos de ejecución reales al recompilar en Tier 1.
* **.NET 10:** el JIT sigue ampliando la devirtualización y la asignación en la pila de objetos que no escapan del método; el modelo Roslyn → CIL → CLR no cambia.

-----

## Cuándo sí y cuándo no

**Profundiza en el pipeline cuando:** diagnosticas errores de carga de ensamblados, mides rendimiento, decides entre JIT y Native AOT, o diseñas una biblioteca pública para varios lenguajes.

**No uses estos términos como decoración:** para explicar una regla de negocio basta el nivel del lenguaje. Hablar de CIL en una revisión de un caso de uso no aporta nada.

-----

## Resumen en 5 líneas

1. Roslyn compila C# a CIL y metadatos; el JIT del CLR compila ese CIL a código máquina al ejecutar.
2. El ensamblado es portable y descompilable: no es lugar para secretos.
3. El CLR aporta GC, excepciones, carga de ensamblados y seguridad de tipos.
4. El CTS hace que todos los lenguajes .NET compartan tipos (`int` es `System.Int32`).
5. El CLS es el subconjunto recomendado para APIs públicas interoperables.

-----

## Para profundizar

<details>
<summary>Ver el CIL de tu propio código</summary>

Puedes inspeccionar cualquier ensamblado con ILSpy (también como extensión de VS Code) o con la herramienta `ildasm`. Compila en Release y abre el `.dll` de `bin/Release/net10.0/`. Verás el CIL de cada método y el C# reconstruido. Es la mejor forma de entender qué genera el compilador por ti: el `async`, las lambdas y los `foreach` se transforman en clases y llamadas que no escribiste.

</details>

<details>
<summary>¿Todo código .NET pasa por el JIT?</summary>

No. Native AOT genera todo el código máquina al publicar, y ReadyToRun precompila parte del código. Algunas plataformas, como iOS, no permiten generar código en ejecución, así que .NET usa compilación anticipada allí. El modelo concreto depende de cómo publiques, no del lenguaje.

</details>

-----

## En entrevista

### Respuesta corta (junior)

C# se compila con Roslyn a CIL, un código intermedio que se guarda en un ensamblado junto con metadatos. Al ejecutar, el CLR carga el ensamblado y el JIT convierte cada método a código máquina la primera vez que se llama. El CLR también administra la memoria con el GC.

### Respuesta ampliada (semi-senior)

El ensamblado incluye CIL, metadatos y manifiesto; por eso es portable entre sistemas y también descompilable. CoreCLR aporta carga de ensamblados, GC, excepciones y seguridad de tipos. El JIT usa compilación por niveles y PGO dinámico: arranca con código rápido de generar y recompila los métodos calientes con optimizaciones basadas en datos reales. El CTS unifica el modelo de tipos entre lenguajes y el CLS define el subconjunto para APIs públicas interoperables. Native AOT cambia el momento de generar código nativo, a costa de limitar la reflexión y la carga dinámica.

### Preguntas frecuentes de seguimiento

**1. ¿`int` es distinto de `System.Int32`?**
No. Es un alias de C# para el mismo tipo.

**2. ¿El CIL es código máquina?**
No. Debe traducirse a la CPU donde se ejecuta, normalmente con el JIT.

**3. ¿Por qué la primera llamada a un método es más lenta?**
Porque incluye la compilación JIT de ese método.

**4. ¿Puedo ocultar código compilando?**
No. Los metadatos permiten descompilarlo. Un ofuscador lo dificulta, pero no protege secretos.

-----

## Práctica

**Ejercicio 1.** Ordena estas piezas según intervienen al ejecutar `dotnet run`: CPU, CIL, Roslyn, JIT, CLR, metadatos.

<details>
<summary>Solución</summary>

```text
Roslyn ──► CIL + metadatos (en el .dll) ──► CLR carga el ensamblado y lee metadatos
       ──► JIT compila cada método al llamarlo ──► CPU ejecuta el código nativo
```

Roslyn trabaja al compilar; el CLR, el JIT y la CPU, al ejecutar.

</details>

**Ejercicio 2.** Agrega `[assembly: CLSCompliant(true)]` a un proyecto y declara este método público. ¿Qué advertencia aparece y cómo la corriges sin cambiar el comportamiento?

```csharp
public static ulong SumarUnidades(ulong a, ulong b) => a + b;
```

<details>
<summary>Solución</summary>

Aparece `CS3001: Argument type 'ulong' is not CLS-compliant` por cada parámetro y `CS3002: Return type of 'SumarUnidades(ulong, ulong)' is not CLS-compliant` por el retorno. Opciones:

* Usar `long` o `decimal` en la API pública, si el rango alcanza.
* Mantener `ulong` y marcar el método con `[CLSCompliant(false)]`, ofreciendo una alternativa conforme para otros lenguajes.

La advertencia no cambia lo que hace el método; avisa que algunos lenguajes .NET podrían no consumirlo.

</details>

-----

## Siguiente lección

[SDK, CLI, proyectos y NuGet](07-SDK%20CLI%20proyectos%20y%20NuGet.md)
