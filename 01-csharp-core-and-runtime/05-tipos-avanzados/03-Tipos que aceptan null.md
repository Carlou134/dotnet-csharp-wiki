# Tipos que aceptan null

## En una frase

`null` representa "ausencia de valor": los tipos de valor solo lo aceptan si los marcas con `?` (`int?`, que es `Nullable<int>`), y los tipos de referencia, con los **tipos de referencia que aceptan null** activados, se dividen en `string` (nunca null) y `string?` (puede ser null), para que el compilador te avise antes de un `NullReferenceException`.

-----

## Antes de empezar

Conviene que ya sepas:

* Tipos de valor frente a referencia y qué es `null`, de [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md).
* Que viste el operador `!` y la advertencia CS8600 en [Tu primer programa](../00-introduccion/03-Tu%20primer%20programa.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`null`:** valor que indica que no hay valor (o que una referencia no apunta a ningún objeto).
* **Tipo de valor que acepta null:** `int?`, `bool?`, `DateTime?`... Es el struct genérico `Nullable<T>`.
* **Tipo de referencia que acepta null (NRT):** anotación `string?` que indica que una referencia puede ser `null`.
* **Contexto nullable:** la configuración (`<Nullable>enable</Nullable>`) que activa las advertencias de nulos.
* **Operador de fusión de null (`??`):** devuelve el lado derecho si el izquierdo es `null`.
* **Operador condicional de null (`?.`, `?[]`):** accede a un miembro solo si el objeto no es `null`.
* **Operador que perdona null (`!`):** silencia la advertencia de nulos; no cambia nada en ejecución.
* **Operadores elevados (*lifted*):** operadores normales que funcionan con `T?` y propagan el `null`.

-----

## El problema

Un formulario pide email (obligatorio) y edad (opcional). ¿Cómo guardas la edad de alguien que no la indicó?

```csharp
int edad = 0;   // ¿0 significa "no la dijo" o "es un bebé recién nacido"?
```

Un `int` no puede representar "no hay dato". Usar un valor especial (`0`, `-1`) mezcla datos con señales y alguien, tarde o temprano, va a calcular un promedio incluyendo esos `-1`.

Con las referencias pasa lo contrario: **cualquier** variable de tipo referencia podía ser `null`, y el compilador no avisaba:

```csharp
string nombre = ObtenerNombre();   // ¿puede devolver null? Nadie lo sabe
Console.WriteLine(nombre.Length);  // NullReferenceException... a veces, en producción
```

Tony Hoare, quien inventó la referencia nula en 1965, la llamó su "error de mil millones de dólares". C# ofrece herramientas para que `null` sea **explícito** en ambos casos.

-----

## Cómo funciona

### Parte 1: tipos de valor que aceptan null (`int?`)

Agregar `?` a un tipo de valor permite que también guarde `null`:

```csharp
int? edad = null;          // sin dato
int? otraEdad = 25;        // con dato

DateTime? fechaBaja = null;
bool? aceptoTerminos = null;   // true, false o "todavía no respondió"
```

`int?` es una forma corta de `Nullable<int>`, un struct que contiene un valor y un indicador de si existe:

```csharp
Nullable<int> edad2 = 25;   // idéntico a int? edad2 = 25;
```

### Leer el valor: `HasValue`, `Value` y alternativas

```csharp
int? edad = 25;

if (edad.HasValue)
{
    Console.WriteLine(edad.Value);   // 25
}

if (edad is int e)                   // pattern matching: comprobar y extraer en un paso
{
    Console.WriteLine(e + 1);        // 26
}
```

Acceder a `.Value` cuando no hay valor lanza `InvalidOperationException: Nullable object must have a value.` Por eso siempre se comprueba antes, o se usa un valor por defecto:

```csharp
int? sinEdad = null;

int a = sinEdad.GetValueOrDefault();      // 0 (el default de int)
int b = sinEdad.GetValueOrDefault(18);    // 18
int c = sinEdad ?? 18;                    // 18 (el operador ??, la forma más común)
```

### Convertir de `int?` a `int`

`int` → `int?` es implícito (siempre seguro). `int?` → `int` **no**:

```csharp
int? quizas = 5;
int seguro = quizas;          // error CS0266: falta una conversión explícita
int seguro2 = (int)quizas;    // compila; si quizas fuera null → InvalidOperationException
int seguro3 = quizas ?? 0;    // la forma recomendada
```

### Operadores con tipos que aceptan null

Los operadores aritméticos funcionan con `T?` y **propagan** el `null`:

```csharp
int? a = 5;
int? b = null;

Console.WriteLine(a + 3);     // 8
Console.WriteLine(a + b);     // (vacío): el resultado es null
Console.WriteLine(a * b ?? 0);// 0
```

Las comparaciones tienen reglas propias:

| Expresión | Resultado |
| --- | --- |
| `5 == null` | `false` |
| `5 != null` | `true` |
| `null == null` (ambos `int?`) | `true` |
| `5 > null`, `5 < null`, `null <= null` | **`false`** (cualquier comparación de orden con `null`) |

Cuidado con la última fila: con `int? x = null`, tanto `x > 0` como `x <= 0` son `false`. Si negas una comparación pensando que cubres "el otro caso", te olvidas del `null`.

Con `bool?`, `&&` y `||` no compilan; `&` y `|` aplican lógica de tres valores (ver [Lógica booleana](../02-control-de-flujo/01-Logica%20booleana.md)).

### Parte 2: los operadores de null (sirven para todo)

```csharp
string? nombre = ObtenerNombre();

// ?? : valor alternativo si es null
string mostrar = nombre ?? "Anónimo";

// ??= : asigna solo si es null
nombre ??= "Invitado";

// ?. : accede al miembro solo si no es null; si lo es, todo da null
int? largo = nombre?.Length;
string? ciudad = cliente?.Direccion?.Ciudad;     // se encadena

// ?[] : lo mismo con índices
char? inicial = nombre?[0];

// ?. con métodos y delegados
evento?.Invoke(this, EventArgs.Empty);
```

Fíjate en que `nombre?.Length` es `int?`, no `int`: si `nombre` es `null`, el resultado no puede ser un número. Por eso suele combinarse con `??`:

```csharp
int largoSeguro = nombre?.Length ?? 0;
```

### Comprobar null: `is null` e `is not null`

```csharp
if (cliente is null) return;
if (cliente is not null) { /* ... */ }
```

Se prefiere `is null` a `== null` porque no puede ser alterado por una sobrecarga del operador `==`.

### Null frente a "sin asignar"

Son dos cosas distintas:

```csharp
string? a = null;              // asignada, con el valor null: se puede leer
Console.WriteLine(a is null);  // True

string b;                      // declarada pero sin asignar
Console.WriteLine(b is null);  // error CS0165: uso de la variable local no asignada 'b'

string?[] nombres = new string?[3];   // los elementos de un array SÍ empiezan en null
Console.WriteLine(nombres[0] is null); // True
```

`null` es un valor; "sin asignar" significa que todavía no tiene ninguno, y el compilador no te deja leerla. Los campos de una clase y los elementos de un array, en cambio, empiezan siempre con su valor por defecto (`null` para referencias).

### Parte 3: tipos de referencia que aceptan null (NRT)

Con el contexto nullable activado, `string` y `string?` dejan de significar lo mismo **para el compilador**:

* `string`: "esta referencia **nunca** debería ser `null`".
* `string?`: "esta referencia **puede** ser `null`; compruébalo antes de usarla".

Se activa en el `.csproj` (las plantillas modernas ya lo traen):

```xml
<Nullable>enable</Nullable>
```

O por archivo, con `#nullable enable` al principio.

```csharp
string nombre = null;          // warning CS8600: convertir un literal null en un tipo que no acepta null
string? apodo = null;          // ✅

Console.WriteLine(apodo.Length);    // warning CS8602: desreferencia de una referencia posiblemente nula

if (apodo is not null)
{
    Console.WriteLine(apodo.Length);   // ✅ sin advertencia: el compilador sabe que no es null aquí
}
```

El compilador hace **análisis de flujo**: sigue los `if`, los `return` y las asignaciones para saber en cada punto si una variable puede ser `null`.

Claves que suelen confundirse:

* Son **advertencias**, no errores. El código compila igual (salvo que configures `<WarningsAsErrors>nullable</WarningsAsErrors>`, algo recomendable).
* **No cambian nada en ejecución**: `string?` y `string` son el mismo tipo `System.String`. No existe `HasValue` ni `Value` en un `string?`; eso es solo para `Nullable<T>`.
* Una propiedad `string` sin inicializar en una clase genera `warning CS8618`: el compilador te pide garantizar que nunca será `null` (con un valor inicial, el constructor o `required`).

### El operador que perdona null `!`

```csharp
string? posible = ObtenerResultado();
Console.WriteLine(posible!.Length);   // sin advertencia
```

`!` le dice al compilador "confía en mí, no es `null`". **No hace ninguna comprobación**: si te equivocas, el `NullReferenceException` ocurre igual. Úsalo solo cuando sabes algo que el compilador no puede deducir (por ejemplo, que otro método ya lo validó), y prefiere siempre una comprobación explícita.

### Validar argumentos

```csharp
public void Registrar(string email)
{
    ArgumentNullException.ThrowIfNull(email);            // lanza si es null
    ArgumentException.ThrowIfNullOrWhiteSpace(email);    // lanza si es null, vacío o espacios
    // ...
}
```

Aunque el parámetro sea `string` (no anulable), quien llama puede ignorar las advertencias o venir de código sin NRT. En los métodos públicos, valida.

-----

## Ejemplo completo

```csharp
var registros = new[]
{
    new Registro("ana@mail.com", 30, "Lima"),
    new Registro("luis@mail.com", null, null),
    new Registro("eva@mail.com", 25, "Cusco")
};

foreach (var r in registros)
{
    string edadTexto = r.Edad is int e ? $"{e} años" : "edad no indicada";
    string ciudad = r.Ciudad ?? "sin ciudad";
    int largoCiudad = r.Ciudad?.Length ?? 0;

    Console.WriteLine($"{r.Email,-15} {edadTexto,-17} {ciudad} ({largoCiudad} letras)");
}

int?[] edades = registros.Select(r => r.Edad).ToArray();
double? promedio = edades.Average();                     // Average ignora los null
Console.WriteLine($"Promedio de edad (solo quienes la indicaron): {promedio:F1}");

int? sumaConNull = edades[0] + edades[1];
Console.WriteLine($"30 + null = {sumaConNull?.ToString() ?? "null"}");
Console.WriteLine($"¿null > 0? {edades[1] > 0}  ¿null <= 0? {edades[1] <= 0}");

Cliente? buscado = Buscar("eva@mail.com");
Console.WriteLine(buscado?.Nombre.ToUpper() ?? "No encontrado");

Cliente? noExiste = Buscar("x@mail.com");
Console.WriteLine(noExiste?.Nombre.ToUpper() ?? "No encontrado");

static Cliente? Buscar(string email) =>
    email == "eva@mail.com" ? new Cliente { Nombre = "Eva" } : null;

record Registro(string Email, int? Edad, string? Ciudad);

class Cliente
{
    public required string Nombre { get; init; }   // nunca null: required lo garantiza
    public string? Telefono { get; init; }          // opcional: puede ser null
}
```

Salida:

```text
ana@mail.com    30 años           Lima (4 letras)
luis@mail.com   edad no indicada  sin ciudad (0 letras)
eva@mail.com    25 años           Cusco (5 letras)
Promedio de edad (solo quienes la indicaron): 27.5
30 + null = null
¿null > 0? False  ¿null <= 0? False
EVA
No encontrado
```

-----

## Errores comunes

**1. Asignar `int?` a `int` sin conversión.**
Qué pasa: `error CS0266: Cannot implicitly convert type 'int?' to 'int'. An explicit conversion exists (are you missing a cast?)`.
Por qué: el valor podría ser `null`, y un `int` no lo admite.
Arreglo: `int x = quizas ?? 0;` o comprobar con `is int x`.

**2. Leer `.Value` sin valor.**
Qué pasa: `System.InvalidOperationException: Nullable object must have a value.` (lo mismo con el cast `(int)quizas`).
Por qué: no hay un `int` que devolver.
Arreglo: comprueba `HasValue` o usa `??` / `GetValueOrDefault`.

**3. Usar `?.` y esperar un tipo no anulable.**
Qué pasa: `int largo = nombre?.Length;` da `error CS0266: Cannot implicitly convert type 'int?' to 'int'`.
Por qué: si `nombre` es `null`, el resultado es `null`.
Arreglo: `int largo = nombre?.Length ?? 0;`.

**4. Desreferenciar un posible null.**
Qué pasa: `warning CS8602: Dereference of a possibly null reference.` y, si de verdad es `null`, `NullReferenceException`.
Por qué: el valor es `string?` y no se comprobó.
Arreglo: `if (x is not null)`, `?.` o `??`.

**5. Propiedad no anulable sin inicializar.**
Qué pasa: `warning CS8618: Non-nullable property 'Nombre' must contain a non-null value when exiting constructor.`
Por qué: nada garantiza que `Nombre` reciba un valor.
Arreglo: inicialízala (`= "";`), asígnala en el constructor, márcala `required` o hazla `string?` si de verdad es opcional.

**6. Abusar de `!`.**
Qué pasa: desaparecen las advertencias y vuelven los `NullReferenceException` en producción.
Por qué: `!` no comprueba nada.
Arreglo: comprobaciones explícitas; reserva `!` para los casos en que puedes justificar por qué no es `null`.

-----

## Según la versión de C#

* **C# 2:** `Nullable<T>` y la sintaxis `int?`, y el operador `??`.
* **C# 6:** operadores condicionales de null `?.` y `?[]`.
* **C# 7:** `is null` y pattern matching (`x is int n`).
* **C# 8:** tipos de referencia que aceptan null (`string?`), el operador `!` y `??=`.
* **C# 9:** `is not null`.
* **.NET 6 / .NET 8:** `ArgumentNullException.ThrowIfNull` y `ArgumentException.ThrowIfNullOrWhiteSpace`.
* **C# 14:** asignación condicional de null: `cliente?.Direccion = nueva;` (solo asigna si `cliente` no es `null`).

-----

## Cuándo sí y cuándo no

**Usa `T?` (tipos de valor) cuando:**

* El dato es genuinamente opcional: fecha de baja, edad no indicada, campo opcional de un formulario o columna `NULL` de una base de datos.

**Usa `string?` (y anota tus referencias) cuando:**

* La ausencia es un caso válido (un segundo nombre, un resultado de búsqueda que puede no existir). Deja `string` sin `?` para todo lo que nunca debería ser `null`.

**Evita:**

* Valores mágicos (`-1`, `""`) para representar "no hay dato".
* Devolver `null` en colecciones: devuelve una colección vacía (`[]`), así quien llama puede recorrerla sin comprobar.
* Desactivar el contexto nullable para "quitar advertencias": las advertencias son bugs encontrados gratis.

-----

## Resumen en 5 líneas

1. `int?` (`Nullable<int>`) permite `null` en tipos de valor: `HasValue`, `Value`, `GetValueOrDefault()`.
2. `??` da un valor alternativo, `??=` asigna si es `null` y `?.`/`?[]` acceden solo si no es `null`.
3. Con `<Nullable>enable</Nullable>`, `string` no admite `null` y `string?` sí; el compilador avisa (CS8600, CS8602, CS8618).
4. Las anotaciones de referencia son solo para el compilador: en ejecución, `string?` y `string` son lo mismo.
5. `!` silencia advertencias sin comprobar nada; `null` (un valor) no es lo mismo que "sin asignar" (CS0165).

-----

## Para profundizar

<details>
<summary>Atributos de análisis de nulos</summary>

A veces el compilador no puede deducir lo que tú sabes. Los atributos de `System.Diagnostics.CodeAnalysis` le dan esa información:

```csharp
using System.Diagnostics.CodeAnalysis;

static bool TryObtener(string clave, [NotNullWhen(true)] out string? valor)
{
    valor = clave == "a" ? "hola" : null;
    return valor is not null;
}

if (TryObtener("a", out var v))
{
    Console.WriteLine(v.Length);   // sin advertencia: si devolvió true, v no es null
}
```

Otros útiles: `[MaybeNull]`, `[NotNull]`, `[MemberNotNull]`, `[DoesNotReturn]`. Así funciona `string.IsNullOrEmpty`, que después de devolver `false` permite usar el string sin advertencia.

</details>

<details>
<summary>Patrón Null Object y Option</summary>

En lugar de devolver `null`, algunos diseños devuelven un objeto "vacío" que no hace nada (*Null Object*), por ejemplo un `ILogger` que descarta los mensajes. Otros usan un tipo `Option<T>` o `Result<T>` que obliga a quien llama a manejar el caso "sin valor". Son alternativas para evitar que `null` se propague por todo el sistema.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los tipos de valor como `int` no aceptan `null`; para eso existe `int?`, que es `Nullable<int>` y tiene `HasValue` y `Value`. Para los tipos de referencia, desde C# 8 se puede activar el contexto nullable: `string` indica que nunca es `null` y `string?` que puede serlo, y el compilador avisa si usas algo que podría ser `null`. Los operadores `??`, `?.` y `??=` ayudan a manejar los nulos.

### Respuesta ampliada (semi-senior)

`Nullable<T>` es un struct con `HasValue` y `Value`; los operadores aritméticos se elevan y propagan el `null`, mientras que las comparaciones de orden con `null` dan siempre `false`, lo que rompe la lógica de complementos. Los NRT son una característica del compilador: las anotaciones se emiten como metadatos (`NullableAttribute`) y el análisis de flujo genera advertencias, sin efecto en ejecución, así que en las APIs públicas se sigue validando con `ArgumentNullException.ThrowIfNull`. Conviene tratar las advertencias de nulos como errores, usar `required` para propiedades obligatorias, atributos como `[NotNullWhen]` para patrones `TryX`, y evitar `!` salvo casos justificados. En colecciones, se devuelve una colección vacía en lugar de `null`.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `int?` y `string?`?**
`int?` es otro tipo (`Nullable<int>`) con `HasValue` y `Value`. `string?` es el mismo `System.String` con una anotación para el compilador.

**2. ¿Qué hace el operador `!`?**
Suprime la advertencia de posible `null`. No comprueba ni convierte nada en ejecución.

**3. ¿Por qué `null > 0` y `null <= 0` son ambos `false`?**
Porque cualquier comparación de orden con un operando `null` da `false` en los tipos de valor anulables.

**4. ¿Las advertencias de nulos impiden compilar?**
No, salvo que se configuren como errores (`<WarningsAsErrors>nullable</WarningsAsErrors>`), algo recomendable en proyectos nuevos.

-----

## Práctica

**Ejercicio 1.** Sin ejecutar, ¿qué imprime cada línea?

```csharp
int? a = null;
int? b = 10;
Console.WriteLine(a ?? b ?? 0);
Console.WriteLine((a + b).HasValue);
Console.WriteLine(a < b || a >= b);
Console.WriteLine(b.GetValueOrDefault(5));
string? s = null;
Console.WriteLine(s?.ToUpper() ?? "vacío");
```

<details>
<summary>Solución</summary>

```text
10
False
False
10
vacío
```

`a + b` es `null`. `a < b` y `a >= b` son ambos `false` porque `a` es `null`.

</details>

**Ejercicio 2.** Esta clase genera advertencias de nulos. Corrígela: `Nombre` es obligatorio, `SegundoNombre` es opcional, y el método `NombreCompleto` debe funcionar con o sin segundo nombre.

```csharp
class Persona
{
    public string Nombre { get; set; }
    public string SegundoNombre { get; set; }
    public string NombreCompleto() => Nombre + " " + SegundoNombre.Trim();
}
```

<details>
<summary>Solución</summary>

```csharp
var p1 = new Persona { Nombre = "Ana" };
var p2 = new Persona { Nombre = "Luis", SegundoNombre = " Alberto " };
Console.WriteLine(p1.NombreCompleto());   // Ana
Console.WriteLine(p2.NombreCompleto());   // Luis Alberto

class Persona
{
    public required string Nombre { get; init; }
    public string? SegundoNombre { get; init; }

    public string NombreCompleto() =>
        string.IsNullOrWhiteSpace(SegundoNombre) ? Nombre : $"{Nombre} {SegundoNombre.Trim()}";
}
```

Después de `string.IsNullOrWhiteSpace` devolver `false`, el compilador sabe que `SegundoNombre` no es `null` (gracias a sus atributos de análisis), así que `.Trim()` no genera advertencia.

</details>

-----

## Siguiente lección

[Genéricos](04-Genericos.md)
