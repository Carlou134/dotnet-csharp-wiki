# Pruebas unitarias y TDD

## En una frase

Una prueba unitaria ejecuta una pieza de lógica aislada y comprueba automáticamente su resultado; un conjunto de pruebas rápidas es la red que te deja cambiar código sin miedo, y TDD es la práctica de escribir esa prueba **antes** del código para que guíe el diseño.

-----

## Antes de empezar

Conviene que ya sepas:

* Clases, métodos y excepciones: [POO](../../01-csharp-core-and-runtime/04-poo/README.md) y [Excepciones](../../01-csharp-core-and-runtime/08-excepciones/README.md).
* Crear proyectos y referencias con la CLI: [Introducción](../../01-csharp-core-and-runtime/00-introduccion/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Prueba unitaria:** prueba automática, rápida y aislada de una unidad de comportamiento.
* **Prueba de integración:** prueba que ejercita varias piezas reales juntas (base de datos, HTTP, archivos).
* **Sistema bajo prueba (SUT, *System Under Test*):** la clase o función que la prueba ejercita.
* **AAA (*Arrange, Act, Assert*):** estructura de una prueba: preparar, ejecutar, verificar.
* **TDD (*Test-Driven Development*):** ciclo de escribir una prueba que falla, el código mínimo que la hace pasar y luego refactorizar.
* **Regresión:** algo que funcionaba y dejó de funcionar por un cambio.
* **Refactorizar:** cambiar la estructura del código sin cambiar su comportamiento.

-----

## El problema

Una tienda calcula el costo de envío:

```csharp
public decimal CalcularEnvio(decimal subtotal, decimal pesoKg)
{
    decimal costo = subtotal >= 100 ? 0 : 10;
    if (pesoKg > 20) costo += 15;
    return costo;
}
```

Para "probarlo", el desarrollador corre la app, arma un carrito y mira el número en pantalla. Meses después, alguien "limpia" el método:

```csharp
public decimal CalcularEnvio(decimal subtotal, decimal pesoKg) =>
    subtotal > 100 ? 0 : pesoKg > 20 ? 25 : 10;
```

Se ve más prolijo. Tiene dos errores:

1. Un subtotal de exactamente 100 ya no tiene envío gratis (`>=` pasó a `>`).
2. Un pedido de 150 que pesa 30 kg ahora tiene envío 0: el recargo por peso desapareció para los pedidos con envío gratis.

Nadie lo nota, porque nadie volvió a armar un carrito de 100 exactos ni uno pesado y caro. Lo descubre contabilidad tres semanas después.

Las consecuencias:

* "probar" a mano no se repite después de cada cambio;
* los casos límite (`100` exacto, `20` kg exactos) nunca se prueban;
* refactorizar da miedo, así que el código se deja como está y se degrada.

-----

## Cómo funciona

### 1. Qué es (y qué no es) una prueba unitaria

```csharp
[Fact]
public void CalcularEnvio_SubtotalDeCienExactos_EsGratis()
{
    // Arrange
    var calculadora = new CalculadoraEnvio();

    // Act
    decimal costo = calculadora.CalcularEnvio(subtotal: 100m, pesoKg: 1m);

    // Assert
    Assert.Equal(0m, costo);
}
```

* **Unidad** es una unidad de **comportamiento** ("el envío es gratis desde 100"), no necesariamente un método. Puede involucrar varias clases propias.
* **Aislada** de lo que es lento o no determinista: base de datos, red, disco, reloj, aleatoriedad.
* **Automática:** pasa o falla sola, sin que nadie mire un número en pantalla.

### 2. La pirámide de pruebas

```text
           /\        E2E / UI: pocas, lentas, frágiles, prueban el sistema completo
          /  \
         /----\      Integración: base real, HTTP real; más lentas, más confianza en el cableado
        /      \
       /--------\    Unitarias: muchas, milisegundos, lógica y reglas de negocio
```

No compiten: las unitarias te dicen rápido **qué regla** se rompió; las de integración, si las piezas **funcionan juntas**. Un proyecto solo con unitarias puede tener toda la lógica verde y un `DbContext` mal configurado.

### 3. AAA: una estructura que se lee en segundos

| Fase | Pregunta | En el ejemplo |
| --- | --- | --- |
| Arrange | ¿Con qué datos y en qué estado? | `new CalculadoraEnvio()` |
| Act | ¿Qué acción pruebo? (una sola) | `CalcularEnvio(100m, 1m)` |
| Assert | ¿Qué debe ser cierto después? | `Assert.Equal(0m, costo)` |

Si el *Act* tiene varias acciones, probablemente son varias pruebas.

### 4. Nombres que explican el fallo

```text
Método_Escenario_ResultadoEsperado
CalcularEnvio_SubtotalDeCienExactos_EsGratis
CalcularEnvio_PedidoPesadoConEnvioGratis_CobraSoloRecargo
```

Cuando una prueba falla en el pipeline, el nombre es lo primero que lees. "Test1" o "CalcularEnvioTest" no dicen qué regla se rompió.

### 5. FIRST: qué hace buena a una prueba

| Letra | Significa | Se rompe cuando... |
| --- | --- | --- |
| **F**ast | Milisegundos | Toca la base, la red o espera con `Thread.Sleep` |
| **I**ndependent | No depende de otras pruebas ni del orden | Una prueba deja datos que otra necesita |
| **R**epeatable | Mismo resultado siempre y en cualquier máquina | Usa `DateTime.Now`, `Random`, la cultura del sistema |
| **S**elf-validating | Pasa o falla sola | Imprime valores para que alguien los mire |
| **T**imely | Se escribe junto con el código (o antes) | Se escribe meses después, si se escribe |

### 6. TDD: rojo, verde, refactor

```text
   ┌──────────► 🔴 ROJO: escribe una prueba que falla (o no compila)
   │                 │
   │                 ▼
   │            🟢 VERDE: el código MÍNIMO para que pase
   │                 │
   │                 ▼
   └──────────── 🔵 REFACTOR: mejora el código con todas las pruebas en verde
```

Aplicado a la calculadora:

1. 🔴 Prueba "subtotal 100 → gratis". No compila: `CalculadoraEnvio` no existe.
2. 🟢 `return 0;`. Pasa. Sí, es trampa a propósito: todavía no hay ninguna prueba que exija otra cosa.
3. 🔴 Prueba "subtotal 50 → 10". Falla.
4. 🟢 `return subtotal >= 100 ? 0 : 10;`. Las dos pasan. Esta segunda prueba **obligó** a generalizar (se llama *triangulación*).
5. 🔴 Prueba "pedido de 150 que pesa 30 kg → 15". Falla.
6. 🟢 Agregas el recargo. Pasan las tres.
7. 🔵 Refactorizas con nombres y constantes. Las tres siguen en verde.

Con TDD, cada regla del negocio quedó escrita como una prueba antes de existir en el código. El refactor del problema habría fallado en dos pruebas al instante.

### 7. Lo que TDD es y lo que no

* Es una herramienta de **diseño**: escribir la prueba primero te obliga a pensar la API desde quien la usa, y empuja hacia clases chicas con dependencias explícitas.
* No es obligatorio para todo: código exploratorio, prototipos o pegamento de infraestructura suelen probarse mejor después, o con pruebas de integración.
* No garantiza ausencia de errores: solo verifica los casos que pensaste. Si nadie pensó en "100 exactos", no hay prueba para ese caso, con o sin TDD.

-----

## Ejemplo completo

Solución con una biblioteca y su proyecto de pruebas xUnit:

```text
dotnet new sln -n Tienda
dotnet new classlib -o src/Tienda
dotnet new xunit -o tests/Tienda.Tests
dotnet sln add src/Tienda tests/Tienda.Tests
dotnet add tests/Tienda.Tests reference src/Tienda
```

`src/Tienda/CalculadoraEnvio.cs`:

```csharp
namespace Tienda;

public sealed class CalculadoraEnvio
{
    public const decimal UmbralEnvioGratis = 100m;
    public const decimal CostoBase = 10m;
    public const decimal PesoMaximoSinRecargoKg = 20m;
    public const decimal RecargoPorPeso = 15m;

    public decimal CalcularEnvio(decimal subtotal, decimal pesoKg)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(subtotal);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(pesoKg);

        decimal costo = subtotal >= UmbralEnvioGratis ? 0m : CostoBase;
        if (pesoKg > PesoMaximoSinRecargoKg)
        {
            costo += RecargoPorPeso;
        }
        return costo;
    }
}
```

`tests/Tienda.Tests/CalculadoraEnvioTests.cs`:

```csharp
using Tienda;

namespace Tienda.Tests;

public class CalculadoraEnvioTests
{
    private readonly CalculadoraEnvio _calculadora = new();

    [Fact]
    public void CalcularEnvio_SubtotalMenorAlUmbral_CobraCostoBase()
    {
        decimal costo = _calculadora.CalcularEnvio(subtotal: 50m, pesoKg: 1m);

        Assert.Equal(10m, costo);
    }

    [Fact]
    public void CalcularEnvio_SubtotalDeCienExactos_EsGratis()
    {
        decimal costo = _calculadora.CalcularEnvio(subtotal: 100m, pesoKg: 1m);

        Assert.Equal(0m, costo);
    }

    [Fact]
    public void CalcularEnvio_PedidoPesadoConEnvioGratis_CobraSoloRecargo()
    {
        decimal costo = _calculadora.CalcularEnvio(subtotal: 150m, pesoKg: 30m);

        Assert.Equal(15m, costo);
    }

    [Fact]
    public void CalcularEnvio_PedidoPesadoYBarato_SumaCostoBaseYRecargo()
    {
        decimal costo = _calculadora.CalcularEnvio(subtotal: 50m, pesoKg: 30m);

        Assert.Equal(25m, costo);
    }

    [Fact]
    public void CalcularEnvio_VeinteKilosExactos_NoTieneRecargo()
    {
        decimal costo = _calculadora.CalcularEnvio(subtotal: 50m, pesoKg: 20m);

        Assert.Equal(10m, costo);
    }

    [Fact]
    public void CalcularEnvio_SubtotalNegativo_LanzaExcepcion()
    {
        Assert.Throws<ArgumentOutOfRangeException>(() => _calculadora.CalcularEnvio(subtotal: -1m, pesoKg: 1m));
    }
}
```

Ejecución:

```text
dotnet test
```

Resultado: **6 pruebas superadas, 0 con error** (el formato exacto del resumen depende de la versión del SDK y del idioma del sistema).

Ahora aplica el "refactor" del problema (`subtotal > 100 ? 0 : pesoKg > 20 ? 25 : 10`, sin las validaciones) y vuelve a ejecutar: fallan **3 pruebas**. `SubtotalDeCienExactos_EsGratis` (devuelve 10 en lugar de 0), `PedidoPesadoConEnvioGratis_CobraSoloRecargo` (devuelve 0 en lugar de 15) y `SubtotalNegativo_LanzaExcepcion` (ya no lanza). Cada nombre dice qué regla se rompió.

Qué observar:

* Las pruebas de los límites (`100` exactos, `20` kg exactos) son las que atrapan el `>=` cambiado por `>`. Los casos "del medio" casi nunca encuentran errores.
* Las constantes con nombre en la clase documentan las reglas; las pruebas usan números literales a propósito, para que un cambio accidental en una constante también se detecte.
* xUnit crea **una instancia nueva** de `CalculadoraEnvioTests` por cada prueba: el campo `_calculadora` no se comparte entre pruebas.
* En .NET 10, `dotnet new sln` crea una solución en formato `.slnx` (XML); en versiones anteriores, `.sln`.

-----

## Errores comunes

**1. Probar solo el caso feliz.**
Qué pasa: los errores en los límites (`>=` frente a `>`) y en los datos inválidos llegan a producción.
Por qué: los valores "típicos" casi nunca están en el borde de una regla.
Arreglo: por cada condición, prueba justo en el límite, justo antes y justo después, más un valor inválido.

**2. Pruebas que dependen del reloj, de la cultura o del orden.**
Qué pasa: pruebas que fallan los lunes, en la máquina de un compañero o solo en CI.
Por qué: `DateTime.Now`, la cultura del sistema y datos compartidos entre pruebas no son repetibles.
Arreglo: inyecta `TimeProvider`, fija la cultura donde importe y no compartas estado mutable entre pruebas.

**3. Pruebas que no verifican nada.**
Qué pasa: todo verde aunque el código esté mal.
Por qué: la prueba llama al método pero no tiene `Assert`, o verifica algo trivial.
Arreglo: cada prueba verifica el resultado o efecto que define la regla. Comprueba que falla cuando rompes el código a propósito.

**4. Copiar ejemplos de NUnit viejos.**
Qué pasa: `CS0117: 'Assert' does not contain a definition for 'AreEqual'` con NUnit 4.
Por qué: NUnit 4 movió el estilo clásico a `ClassicAssert` y recomienda `Assert.That(resultado, Is.EqualTo(esperado))`.
Arreglo: usa la API del framework y versión que tienes instalados; en xUnit, `Assert.Equal(esperado, real)`.

**5. Invertir esperado y real.**
Qué pasa: mensajes de fallo confusos ("esperaba 10, obtuve 0" cuando era al revés).
Por qué: `Assert.Equal(esperado, real)` tiene un orden fijo.
Arreglo: primero el valor esperado, después el calculado. El analizador de xUnit ayuda a detectarlo con constantes.

-----

## Según la versión de .NET

* **2002-2005:** NUnit (puerto de JUnit) y MSTest (con Visual Studio 2005).
* **2007:** xUnit.net, creado por autores de NUnit con un diseño más simple (sin `[SetUp]`, una instancia por prueba).
* **.NET Core:** `dotnet test` y `Microsoft.NET.Test.Sdk` unifican la ejecución desde la CLI.
* **NUnit 4 (2023):** el `Assert` clásico pasa a `ClassicAssert`; el estilo recomendado es `Assert.That`.
* **xUnit v3 (2024):** los proyectos de prueba son ejecutables, `Assert.Skip` dinámico y `TestContext`; la sintaxis de `[Fact]`, `[Theory]` y `Assert` se mantiene.
* **.NET 10:** `dotnet test` puede usar Microsoft.Testing.Platform; las soluciones nuevas usan `.slnx`.

-----

## Cuándo sí y cuándo no

**Escribe pruebas unitarias cuando:** hay reglas de negocio, cálculos, validaciones o decisiones con varios caminos. Es donde más errores se esconden y donde las pruebas son más baratas.

**Usa TDD cuando:** las reglas están claras y quieres que las pruebas guíen el diseño de la API de una clase.

**No insistas en pruebas unitarias cuando:** el código es puro pegamento de infraestructura (configurar EF Core, mapear endpoints): ahí aportan más las pruebas de integración.

-----

## Resumen en 5 líneas

1. Una prueba unitaria verifica automáticamente una unidad de comportamiento, aislada de lo lento y lo no determinista.
2. AAA (preparar, ejecutar, verificar) y nombres `Método_Escenario_Resultado`.
3. FIRST: rápidas, independientes, repetibles, auto-validadas y a tiempo.
4. TDD: rojo, verde, refactor; la triangulación obliga a generalizar.
5. Los errores viven en los límites: pruébalos siempre.

-----

## Para profundizar

<details>
<summary>Escuela clásica frente a escuela de "mocks"</summary>

En la escuela **clásica** (Detroit), una "unidad" es un comportamiento: la prueba usa las clases reales de tu dominio y solo reemplaza dependencias externas (base, red, reloj). En la escuela **londinense** (*mockist*), cada clase se prueba sola y todas sus colaboradoras se reemplazan por mocks. La clásica produce pruebas más resistentes a refactors; la londinense, fallos más localizados pero pruebas acopladas a la implementación. La [lección de dobles de prueba](03-Dobles%20de%20prueba%20con%20Moq.md) vuelve sobre esto.

</details>

<details>
<summary>¿Qué es "código testeable"?</summary>

Código cuyas dependencias se pueden reemplazar y cuyo resultado se puede observar. Las señales de código difícil de probar son: `new` de dependencias externas dentro de los métodos, métodos estáticos con efectos (`DateTime.Now`, `File.ReadAllText`), clases que hacen diez cosas y métodos sin valor de retorno observable. TDD ayuda justamente porque, si escribes la prueba primero, no puedes escribir esas cosas sin sentir el dolor.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una prueba unitaria verifica automáticamente que una parte del código haga lo esperado, sin depender de base de datos ni red. Las escribo con la estructura Arrange-Act-Assert. TDD es escribir primero la prueba que falla, después el código mínimo para pasarla y luego refactorizar.

### Respuesta ampliada (semi-senior)

Pruebo unidades de comportamiento, no métodos sueltos, aisladas de lo lento y no determinista. Sigo AAA y FIRST, nombro las pruebas por escenario y resultado, y priorizo los límites de cada regla, que es donde viven los errores. Combino muchas pruebas unitarias con algunas de integración para el cableado real. Uso TDD cuando las reglas están claras: rojo, verde con el mínimo, refactor, y triangulo con un segundo caso para forzar la generalización. Sé que las pruebas solo cubren los casos que pensé y que la cobertura no reemplaza buenos casos.

### Preguntas frecuentes de seguimiento

**1. ¿Unitaria o de integración?**
Unitaria para reglas y cálculos aislados; integración para comprobar que base, HTTP y configuración funcionan juntas.

**2. ¿Por qué el paso verde es "el mínimo"?**
Para no escribir código que ninguna prueba exige; la siguiente prueba obliga a generalizar.

**3. ¿Cuántos asserts por prueba?**
Los necesarios para verificar **un** comportamiento. Varios asserts del mismo resultado están bien; dos comportamientos distintos son dos pruebas.

**4. ¿TDD siempre?**
No. Es muy útil con reglas claras; en exploración o infraestructura, suele convenir probar después o con integración.

-----

## Práctica

**Ejercicio 1.** Agrega con TDD la regla "los clientes premium nunca pagan el costo base, pero sí el recargo por peso". Escribe primero las pruebas.

<details>
<summary>Solución</summary>

Pruebas (🔴, no compilan porque falta el parámetro):

```csharp
[Fact]
public void CalcularEnvio_ClientePremiumConSubtotalBajo_EsGratis()
{
    decimal costo = _calculadora.CalcularEnvio(subtotal: 50m, pesoKg: 1m, esPremium: true);
    Assert.Equal(0m, costo);
}

[Fact]
public void CalcularEnvio_ClientePremiumConPedidoPesado_CobraSoloRecargo()
{
    decimal costo = _calculadora.CalcularEnvio(subtotal: 50m, pesoKg: 30m, esPremium: true);
    Assert.Equal(15m, costo);
}
```

Código (🟢):

```csharp
public decimal CalcularEnvio(decimal subtotal, decimal pesoKg, bool esPremium = false)
{
    ArgumentOutOfRangeException.ThrowIfNegative(subtotal);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(pesoKg);

    bool envioGratis = esPremium || subtotal >= UmbralEnvioGratis;
    decimal costo = envioGratis ? 0m : CostoBase;
    if (pesoKg > PesoMaximoSinRecargoKg)
    {
        costo += RecargoPorPeso;
    }
    return costo;
}
```

El parámetro opcional mantiene en verde las seis pruebas anteriores. En el 🔵, `envioGratis` como variable con nombre hace legible la regla.

</details>

**Ejercicio 2.** ¿Qué tiene de malo esta prueba? Reescríbela.

```csharp
[Fact]
public void Test1()
{
    var c = new CalculadoraEnvio();
    var r1 = c.CalcularEnvio(50, 1);
    var r2 = c.CalcularEnvio(150, 1);
    Console.WriteLine(r1);
    Assert.True(r2 == 0);
}
```

<details>
<summary>Solución</summary>

* El nombre no dice nada.
* Prueba dos escenarios en una sola prueba, y uno de ellos (`r1`) no se verifica: solo se imprime.
* `Assert.True(r2 == 0)` da un mensaje pobre al fallar ("expected True, got False"); `Assert.Equal` muestra ambos valores.

```csharp
[Fact]
public void CalcularEnvio_SubtotalMenorAlUmbral_CobraCostoBase() =>
    Assert.Equal(10m, new CalculadoraEnvio().CalcularEnvio(50m, 1m));

[Fact]
public void CalcularEnvio_SubtotalMayorAlUmbral_EsGratis() =>
    Assert.Equal(0m, new CalculadoraEnvio().CalcularEnvio(150m, 1m));
```

</details>

-----

## Siguiente lección

[xUnit: Fact, Theory y aserciones](02-xUnit%20Fact%20Theory%20y%20aserciones.md)
