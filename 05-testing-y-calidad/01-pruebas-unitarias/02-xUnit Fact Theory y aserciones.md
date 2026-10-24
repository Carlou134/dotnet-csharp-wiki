# xUnit: Fact, Theory y aserciones

## En una frase

En xUnit, `[Fact]` marca una prueba sin parámetros y `[Theory]` una prueba que se ejecuta una vez por cada conjunto de datos (`[InlineData]`, `[MemberData]`); las aserciones de `Assert` dan mensajes claros cuando eliges la adecuada, y cada prueba corre en una instancia nueva de su clase.

-----

## Antes de empezar

Conviene que ya sepas:

* AAA, FIRST y casos límite: [Pruebas unitarias y TDD](01-Pruebas%20unitarias%20y%20TDD.md).
* Excepciones y `async`/`await`: [Excepciones](../../01-csharp-core-and-runtime/08-excepciones/README.md) y [Asincronía](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Fact:** prueba sin parámetros que siempre debe pasar.
* **Theory:** prueba parametrizada que debe pasar para todos los datos que recibe.
* **InlineData / MemberData:** atributos que suministran datos a una `Theory`, escritos en línea o desde un miembro de la clase.
* **TheoryData&lt;T&gt;:** colección tipada de casos para `[MemberData]`.
* **Fixture:** objeto compartido entre pruebas para preparar algo costoso una sola vez (`IClassFixture<T>`).
* **Prueba omitida (*skipped*):** prueba que existe pero no se ejecuta; no es un éxito.

-----

## El problema

Pruebas de una clase de utilidades de texto, escritas copiando y pegando:

```csharp
[Fact]
public void RemoveWhitespace1() => Assert.Equal("holamundo", _ops.RemoveWhitespace("hola mundo"));

[Fact]
public void RemoveWhitespace2() => Assert.Equal("hola", _ops.RemoveWhitespace("  hola  "));

[Fact]
public void ConcatenateStrings()
{
    var result = _ops.ConcatenateStrings("hello", "plexi");
    Assert.NotNull(result);
    Assert.NotEmpty(result);
    Assert.Equal("hello plexi", result);
}

[Fact]
public void IsPalindrome() => Assert.Equal(true, _ops.IsPalindrome("ama"));

[Fact(Skip = "falla desde la actualización")]
public void GetStringLength_Null() { ... }
```

Lo que pasa:

1. Diez métodos casi idénticos para `RemoveWhitespace`, y aun así nadie probó `""`, `null` ni un tabulador.
2. `NotNull` y `NotEmpty` antes de `Equal` no agregan nada: si el resultado es `"hello plexi"`, ya no es nulo ni vacío. Solo agregan ruido.
3. `Assert.Equal(true, ...)` funciona, pero el analizador de xUnit lo marca (**xUnit2004**) y el mensaje de fallo es peor que el de `Assert.True`.
4. Nadie prueba que `IsPalindrome("hola")` devuelva `false`: la función podría devolver siempre `true` y la suite seguiría verde.
5. La prueba omitida lleva seis meses así. El pipeline está verde, pero esa regla ya no se verifica.

Las consecuencias:

* mucho código de prueba que cubre pocos casos;
* asserts redundantes que esconden lo importante;
* pruebas apagadas que dan una falsa sensación de seguridad.

-----

## Cómo funciona

### 1. `[Fact]` y `[Theory]`

```csharp
[Fact]
public void ConcatenateStrings_DosPalabras_LasUneConUnEspacio() =>
    Assert.Equal("hello plexi", _ops.ConcatenateStrings("hello", "plexi"));

[Theory]
[InlineData("hola mundo", "holamundo")]
[InlineData("  hola  ", "hola")]
[InlineData("h\to\nla", "hola")]
[InlineData("", "")]
public void RemoveWhitespace_QuitaTodosLosEspacios(string entrada, string esperado) =>
    Assert.Equal(esperado, _ops.RemoveWhitespace(entrada));
```

* Cada `[InlineData]` es **una prueba distinta** en el resultado: si falla el tercer caso, el reporte dice cuál y con qué valores.
* Los valores de `[InlineData]` deben ser constantes de compilación (números, strings, `null`, `typeof`, enums). Para objetos, usa `[MemberData]`.

### 2. `[MemberData]` con `TheoryData<T>`

```csharp
public static TheoryData<int, string> Romanos => new()
{
    { 1, "I" },
    { 4, "IV" },
    { 9, "IX" },
    { 1994, "MCMXCIV" }
};

[Theory]
[MemberData(nameof(Romanos))]
public void ARomano_ConvierteCorrectamente(int numero, string esperado) =>
    Assert.Equal(esperado, Conversor.ARomano(numero));
```

`TheoryData<...>` es tipado: si el caso no coincide con los parámetros del método, no compila.

### 3. Elegir la aserción correcta

| Quieres verificar | Usa | Evita |
| --- | --- | --- |
| Igualdad | `Assert.Equal(esperado, real)` | `Assert.True(a == b)` (mensaje pobre) |
| Booleano | `Assert.True(x)` / `Assert.False(x)` | `Assert.Equal(true, x)` (xUnit2004) |
| Nulo | `Assert.Null(x)` / `Assert.NotNull(x)` | `Assert.Equal(null, x)` |
| Excepción | `Assert.Throws<T>(() => ...)` | `try`/`catch` a mano |
| Excepción async | `await Assert.ThrowsAsync<T>(() => ...)` | `.Result` o `.Wait()` |
| Texto parcial | `Assert.StartsWith`, `Assert.Contains` | `Assert.True(s.Contains(...))` |
| Colección | `Assert.Equal(new[] { 1, 2 }, lista)`, `Assert.Single`, `Assert.Empty`, `Assert.Contains` | Recorrerla con `foreach` + asserts |
| Objeto por valor (estructura) | `Assert.Equivalent(esperado, real)` | Comparar propiedad por propiedad |

`Assert.Throws<T>` exige el tipo **exacto**: si se lanza `ArgumentNullException` y esperas `ArgumentException`, falla. Para aceptar derivados, `Assert.ThrowsAny<T>`. Además, devuelve la excepción para verificar sus datos:

```csharp
var ex = Assert.Throws<ArgumentNullException>(() => _ops.GetStringLength(null!));
Assert.Equal("texto", ex.ParamName);
```

### 4. Booleanos: los dos lados

```csharp
[Theory]
[InlineData("ama", true)]
[InlineData("Anita lava la tina", true)]
[InlineData("hola", false)]
[InlineData("", true)]
public void IsPalindrome_DistintosTextos(string texto, bool esperado) =>
    Assert.Equal(esperado, _ops.IsPalindrome(texto));
```

Aquí `Assert.Equal` con un `bool` está bien: el valor esperado viene de los datos. Si solo pruebas casos `true`, una implementación que siempre devuelve `true` pasa.

### 5. Casos límite de texto

Para cada función que recibe un `string`, piensa en:

```text
normal · vacío "" · null · solo espacios "   " · un carácter
mayúsculas/minúsculas · tildes y ñ · tabuladores y saltos de línea · muy largo
```

No todos aplican siempre, pero cada uno debería ser una decisión, no un olvido.

### 6. Ciclo de vida de una clase de pruebas

```text
Por CADA prueba:
  new StringOperationsTests()   ← el constructor es el "setup"
  ejecuta la prueba
  Dispose()                     ← si implementa IDisposable, es el "teardown"
```

* No hay `[SetUp]` ni `[TearDown]` como en NUnit: se usa el constructor y `IDisposable`/`IAsyncLifetime`.
* Como cada prueba tiene su propia instancia, los campos **no** se comparten: no hay forma accidental de que una prueba dependa de otra.
* Para algo costoso que sí quieres crear una sola vez por clase (por ejemplo, un contenedor de base de datos), `IClassFixture<T>`.

### 7. `Skip`: temporal, con motivo y con fecha de vuelta

```csharp
[Fact(Skip = "Bloqueada por el bug #482 de la biblioteca X; reactivar al actualizar a 3.1")]
```

Una prueba omitida no falla, así que es fácil olvidarla. Úsala con un motivo concreto y un ticket. Si la prueba falla por un error **tuyo**, arréglalo o bórrala: omitirla es esconder el problema.

### 8. Pruebas asíncronas

```csharp
[Fact]
public async Task BuscarAsync_TextoVacio_LanzaArgumentException()
{
    await Assert.ThrowsAsync<ArgumentException>(() => _servicio.BuscarAsync(""));
}
```

La prueba devuelve `Task` y usa `await`. Nunca `async void` (xUnit no podría esperar el resultado) ni `.Result`.

-----

## Ejemplo completo

Misma estructura de solución que la [lección 1](01-Pruebas%20unitarias%20y%20TDD.md). `src/Tienda/StringOperations.cs`:

```csharp
namespace Tienda;

public sealed class StringOperations
{
    public string ConcatenateStrings(string a, string b) => $"{a} {b}";

    public string RemoveWhitespace(string texto)
    {
        ArgumentNullException.ThrowIfNull(texto);
        return string.Concat(texto.Where(c => !char.IsWhiteSpace(c)));
    }

    public bool IsPalindrome(string texto)
    {
        ArgumentNullException.ThrowIfNull(texto);
        var letras = texto.Where(char.IsLetterOrDigit).Select(char.ToLowerInvariant).ToArray();
        return letras.SequenceEqual(letras.Reverse());
    }

    public int GetStringLength(string texto)
    {
        ArgumentNullException.ThrowIfNull(texto);
        return texto.Length;
    }
}
```

`tests/Tienda.Tests/StringOperationsTests.cs`:

```csharp
using Tienda;

namespace Tienda.Tests;

public class StringOperationsTests
{
    private readonly StringOperations _ops = new();

    [Fact]
    public void ConcatenateStrings_DosPalabras_LasUneConUnEspacio() =>
        Assert.Equal("hello plexi", _ops.ConcatenateStrings("hello", "plexi"));

    [Theory]
    [InlineData("hola mundo", "holamundo")]
    [InlineData("  hola  ", "hola")]
    [InlineData("h\to\nla", "hola")]
    [InlineData("", "")]
    [InlineData("   ", "")]
    public void RemoveWhitespace_QuitaTodosLosEspacios(string entrada, string esperado) =>
        Assert.Equal(esperado, _ops.RemoveWhitespace(entrada));

    [Theory]
    [InlineData("ama", true)]
    [InlineData("Anita lava la tina", true)]
    [InlineData("hola", false)]
    public void IsPalindrome_DistintosTextos(string texto, bool esperado) =>
        Assert.Equal(esperado, _ops.IsPalindrome(texto));

    [Fact]
    public void GetStringLength_Null_LanzaArgumentNullExceptionConElParametro()
    {
        var ex = Assert.Throws<ArgumentNullException>(() => _ops.GetStringLength(null!));

        Assert.Equal("texto", ex.ParamName);
    }

    [Fact(Skip = "Ejemplo de prueba omitida: pendiente de definir si se recortan espacios (ticket #12)")]
    public void ConcatenateStrings_ConEspaciosAlBorde_LosRecorta() =>
        Assert.Equal("hola mundo", _ops.ConcatenateStrings(" hola ", " mundo "));
}
```

Ejecución con `dotnet test`. Resultado: **10 pruebas superadas, 0 con error, 1 omitida** (11 en total: 1 `Fact` + 5 casos + 3 casos + 1 `Fact` + 1 omitida). El formato del resumen depende de la versión del SDK y del idioma.

Qué observar:

* Las dos `Theory` producen ocho resultados independientes; en Test Explorer aparecen con sus valores (`RemoveWhitespace_QuitaTodosLosEspacios(entrada: "  hola  ", esperado: "hola")`).
* `"h\to\nla"` prueba que se quitan tabuladores y saltos de línea, no solo espacios: `char.IsWhiteSpace` cubre todos.
* `ArgumentNullException.ThrowIfNull(texto)` pone automáticamente el nombre del parámetro (`"texto"`) en la excepción, y la prueba lo verifica.
* La prueba omitida figura en el resumen. Omitida no es superada.

-----

## Errores comunes

**1. Copiar y pegar `[Fact]` con datos distintos.**
Qué pasa: mucho código, pocos casos y nombres como `Test2`.
Por qué: falta la parametrización.
Arreglo: `[Theory]` con `[InlineData]` o `[MemberData]`.

**2. Asserts redundantes.**
Qué pasa: `NotNull` + `NotEmpty` + `Equal` sobre el mismo valor; ruido sin información.
Por qué: `Equal("hello plexi", r)` ya implica que no es nulo ni vacío.
Arreglo: el assert más específico que define el comportamiento.

**3. `Assert.Equal(true, x)` o `Assert.True(a == b)`.**
Qué pasa: advertencia xUnit2004, o mensajes como "Expected: True, Actual: False" sin los valores reales.
Por qué: la aserción genérica no sabe qué comparabas.
Arreglo: `Assert.True(x)` para booleanos; `Assert.Equal(a, b)` para valores.

**4. `[InlineData]` con objetos.**
Qué pasa: error de compilación `CS0182: An attribute argument must be a constant expression...`.
Por qué: los argumentos de atributos tienen que ser constantes.
Arreglo: `[MemberData]` con `TheoryData<...>`.

**5. Pruebas `async void`.**
Qué pasa: la prueba "pasa" aunque el código falle después del primer `await`.
Por qué: xUnit no puede esperar un `async void`.
Arreglo: `public async Task NombreDeLaPrueba()`.

**6. `Skip` como forma de silenciar fallos.**
Qué pasa: reglas que dejan de verificarse sin que nadie lo note.
Por qué: omitida no rompe el pipeline.
Arreglo: motivo + ticket + plan de vuelta; si es un error propio, arréglalo.

-----

## Según la versión de .NET

* **xUnit 1.x (2007):** `[Fact]`, una instancia por prueba, sin `[SetUp]`.
* **xUnit 2.x (2015):** `[Theory]` con `InlineData`/`MemberData`, ejecución en paralelo entre clases y analizadores.
* **xUnit 2.4+:** `TheoryData<...>` tipado y `Assert.Equivalent` (2.4.2).
* **xUnit v3 (2024):** proyectos de prueba ejecutables, `Assert.Skip(...)` dinámico, `TestContext.Current` (con su `CancellationToken`) y mejoras de `MemberData`.
* **C# 11:** literales `"""` cómodos para textos esperados largos.

-----

## Cuándo sí y cuándo no

**Usa `[Theory]` cuando:** la misma regla se verifica con varios datos (límites, equivalencias, tablas de conversión).

**Usa `[Fact]` cuando:** el escenario es único o la preparación cambia mucho de un caso a otro: forzarlo en una `Theory` con diez parámetros la vuelve ilegible.

**No uses `Skip` cuando:** el fallo es tuyo y se puede arreglar hoy.

-----

## Resumen en 5 líneas

1. `[Fact]` para un escenario; `[Theory]` + `[InlineData]`/`[MemberData]` para varios datos de la misma regla.
2. Cada caso de una `Theory` es una prueba independiente en el reporte.
3. La aserción más específica: `Equal`, `True`/`False`, `Throws`, `Contains`, `Equivalent`.
4. Booleanos y validaciones: prueba ambos lados y los límites (`""`, `null`, espacios).
5. Una instancia por prueba: constructor como setup, `IDisposable` como teardown, `IClassFixture` para lo costoso.

-----

## Para profundizar

<details>
<summary>Paralelismo en xUnit</summary>

Por defecto, xUnit ejecuta en paralelo las **clases** de prueba (cada clase es una "colección") y en secuencia las pruebas dentro de una misma clase. Si dos clases comparten un recurso global (un archivo, una base de datos, una variable estática), pueden pisarse. Agrúpalas en la misma colección con `[Collection("Base de datos")]` para que no corran a la vez, o mejor, elimina el estado compartido.

</details>

<details>
<summary>FluentAssertions, Shouldly y la licencia</summary>

Bibliotecas como FluentAssertions (`resultado.Should().Be("hola")`) o Shouldly (`resultado.ShouldBe("hola")`) ofrecen una sintaxis más legible y mensajes más ricos. FluentAssertions pasó a licencia comercial para uso empresarial desde su versión 8 (2025); la 7.x sigue siendo libre, y Shouldly o las aserciones de xUnit son alternativas sin costo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

En xUnit, `[Fact]` es una prueba normal y `[Theory]` una prueba con parámetros que se ejecuta una vez por cada `[InlineData]`. Verifico con `Assert.Equal`, `Assert.True`, `Assert.Throws` y similares. `Skip` desactiva temporalmente una prueba.

### Respuesta ampliada (semi-senior)

Parametrizo con `[Theory]`, usando `[InlineData]` para constantes y `[MemberData]` con `TheoryData<...>` para datos complejos; cada caso es un resultado independiente. Elijo la aserción más específica para tener buenos mensajes y evito redundancias. Pruebo ambos lados de los booleanos y los límites de entrada. Sé que xUnit crea una instancia por prueba (constructor como setup, `IDisposable` como teardown), que paraleliza por clase y que `IClassFixture` comparte recursos costosos. Las pruebas async devuelven `Task`, y `Skip` va con motivo y ticket.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `Fact` y `Theory`?**
`Fact` no recibe datos; `Theory` se ejecuta por cada conjunto de datos y debe pasar con todos.

**2. ¿Cómo comparto un objeto costoso entre pruebas?**
Con `IClassFixture<T>` (por clase) o `ICollectionFixture<T>` (entre clases).

**3. ¿Por qué `Assert.True` y no `Assert.Equal(true, ...)`?**
Por claridad y por el mensaje de error; el analizador lo marca como xUnit2004.

**4. ¿Una prueba omitida cuenta como aprobada?**
No. Aparece como omitida y no verifica nada.

-----

## Práctica

**Ejercicio 1.** Escribe una `Theory` para `GetStringLength` que cubra texto normal, vacío, con espacios y con emojis. ¿Qué devuelve `"👍".Length` y por qué?

<details>
<summary>Solución</summary>

```csharp
[Theory]
[InlineData("hola", 4)]
[InlineData("", 0)]
[InlineData("  ", 2)]
[InlineData("👍", 2)]
public void GetStringLength_DistintosTextos(string texto, int esperado) =>
    Assert.Equal(esperado, _ops.GetStringLength(texto));
```

`"👍".Length` es **2**: `string.Length` cuenta unidades UTF-16, y ese emoji ocupa dos (un par sustituto). Si la regla de negocio es "cantidad de caracteres visibles", la implementación debería usar `StringInfo.LengthInTextElements`. La prueba deja documentada la decisión.

</details>

**Ejercicio 2.** Pasa los casos de `RemoveWhitespace` a `[MemberData]` con `TheoryData` y agrega uno con un espacio de no separación (`' '`).

<details>
<summary>Solución</summary>

```csharp
public static TheoryData<string, string> TextosConEspacios => new()
{
    { "hola mundo", "holamundo" },
    { "  hola  ", "hola" },
    { "h\to\nla", "hola" },
    { "", "" },
    { "hola mundo", "holamundo" }
};

[Theory]
[MemberData(nameof(TextosConEspacios))]
public void RemoveWhitespace_QuitaTodosLosEspacios(string entrada, string esperado) =>
    Assert.Equal(esperado, _ops.RemoveWhitespace(entrada));
```

El caso con ` ` pasa porque `char.IsWhiteSpace` reconoce el espacio de no separación. Si la implementación hubiera sido `texto.Replace(" ", "")`, este caso (y el del tabulador) habrían fallado: por eso vale la pena incluirlos.

</details>

-----

## Siguiente lección

[Dobles de prueba con Moq](03-Dobles%20de%20prueba%20con%20Moq.md)
