# Sintaxis moderna de C#

## En una frase

Desde C# 6 (2015) hasta C# 14 (2025), el lenguaje sumó características que eliminan código repetitivo (interpolación, miembros con cuerpo de expresión, records, constructores primarios, `global using`...) y conocerlas te permite **escribir menos ruido** y **leer** tanto código moderno como código heredado.

-----

## Antes de empezar

Conviene que ya sepas:

* Todo lo anterior del curso: esta lección es un **mapa de repaso** que reúne las mejoras de sintaxis vistas en las secciones "Según la versión de C#" de cada lección.
* Principios de código limpio, de [Nombres, comentarios y code smells](01-Nombres%20comentarios%20y%20code%20smells.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Azúcar sintáctico:** sintaxis más corta que el compilador traduce a otra más larga, sin cambiar lo que hace.
* **Código ceremonial (*boilerplate*):** código repetitivo que no expresa lógica de negocio.
* **`LangVersion`:** propiedad del proyecto que indica qué versión de C# usa el compilador.
* **`global using`:** importación de un espacio de nombres que aplica a todos los archivos del proyecto.
* **Espacio de nombres por archivo:** `namespace MiApp;` sin llaves, para todo el archivo.

-----

## El problema

Este código es C# perfectamente válido... de 2010:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace Tienda
{
    public class Producto
    {
        private readonly string _nombre;
        private readonly decimal _precio;

        public Producto(string nombre, decimal precio)
        {
            if (nombre == null) throw new ArgumentNullException("nombre");
            _nombre = nombre;
            _precio = precio;
        }

        public string Nombre { get { return _nombre; } }
        public decimal Precio { get { return _precio; } }

        public override string ToString()
        {
            return string.Format("{0}: {1}", Nombre, Precio);
        }
    }
}
```

Y esto es lo mismo, en C# moderno:

```csharp
namespace Tienda;

public record Producto(string Nombre, decimal Precio)
{
    public string Nombre { get; } = Nombre ?? throw new ArgumentNullException(nameof(Nombre));
    public override string ToString() => $"{Nombre}: {Precio}";
}
```

Menos líneas significa menos que leer, menos lugares donde equivocarse y más espacio para la lógica que importa. Pero vas a trabajar con **ambos** estilos: los proyectos heredados siguen escritos en la forma vieja.

-----

## Cómo funciona

Cada versión, con lo más usado y un "antes / después". Cada tema se estudia a fondo en la lección enlazada.

### C# 6 (2015, .NET Framework 4.6)

**Interpolación de strings** ([Texto: char y string](../01-tipos-y-variables/05-Texto%20char%20y%20string.md)):

```csharp
string.Format("Hola {0}, tienes {1} años", nombre, edad);   // antes
$"Hola {nombre}, tienes {edad} años";                       // después
```

**Miembros con cuerpo de expresión** ([Expresiones lambda](../03-metodos/04-Expresiones%20lambda.md)):

```csharp
public int Sumar(int x, int y) { return x + y; }   // antes
public int Sumar(int x, int y) => x + y;            // después
```

**Inicializadores de propiedades automáticas** ([Propiedades](../04-poo/03-Propiedades.md)):

```csharp
public string Saludo { get; set; } = "Hola mundo";
public List<string> Items { get; } = new();         // de solo lectura
```

**Operador condicional de null** y **`nameof`** ([Tipos que aceptan null](../05-tipos-avanzados/03-Tipos%20que%20aceptan%20null.md)):

```csharp
int? anio = fecha?.Year;
string ciudad = cliente?.Direccion?.Ciudad ?? "Sin ciudad";
throw new ArgumentNullException(nameof(cliente));    // en lugar de "cliente" a mano
```

Además: `using static`, filtros de excepción (`catch ... when`) e inicializadores de índice (`["clave"] = valor`).

### C# 7.x (2017-2018)

**Tuplas con nombre y deconstrucción** ([Valores de retorno y parámetros out](../03-metodos/03-Valores%20de%20retorno%20y%20parametros%20out.md)):

```csharp
var persona = (Nombre: "Ana", Edad: 30);
Console.WriteLine(persona.Nombre);

(int Min, int Max) Extremos(int[] datos) => (datos.Min(), datos.Max());
var (min, max) = Extremos(new[] { 4, 1, 9 });
```

**`out var` y descartes**:

```csharp
int numero;                                  // antes
int.TryParse(texto, out numero);

int.TryParse(texto, out int n);              // después
int.TryParse(texto, out _);                  // descarte: no me interesa el valor
```

**Pattern matching con `is`** ([Polimorfismo y casting](../04-poo/09-Polimorfismo%20y%20casting.md)):

```csharp
var perro = animal as Perro;                 // antes
if (perro != null) perro.Ladrar();

if (animal is Perro p) p.Ladrar();           // después
```

**Funciones locales** ([Definir y llamar métodos](../03-metodos/01-Definir%20y%20llamar%20metodos.md)), **separador de dígitos** (`1_000_000`), **expresiones `throw`** (`x ?? throw new ...`), `default` literal, `private protected` y `readonly struct`.

### C# 8 (2019, .NET Core 3.0)

**Expresiones `switch`** ([Condicionales](../02-control-de-flujo/02-Condicionales.md)):

```csharp
string estado;                               // antes
switch (codigo)
{
    case 200: estado = "OK"; break;
    case 404: estado = "No encontrado"; break;
    default: estado = "Desconocido"; break;
}

string estado2 = codigo switch               // después
{
    200 => "OK",
    404 => "No encontrado",
    _ => "Desconocido"
};
```

**Tipos de referencia que aceptan null** (`string?`), **`??=`**, **índices y rangos** (`arr[^1]`, `arr[1..3]`), **declaraciones `using` sin llaves**, **flujos asíncronos** (`IAsyncEnumerable`, `await foreach`) y **métodos por defecto en interfaces**.

```csharp
using var archivo = new StreamReader("datos.txt");   // se cierra al final del bloque
string ultimo = palabras[^1];
lista ??= new List<int>();
```

### C# 9 (2020, .NET 5)

**Records** ([Structs y records](../05-tipos-avanzados/02-Structs%20y%20records.md)) e **`init`**:

```csharp
public record Persona(string Nombre, int Edad);
var mayor = persona with { Edad = 31 };
```

**Top-level statements** ([Tu primer programa](../00-introduccion/03-Tu%20primer%20programa.md)):

```csharp
// antes: namespace + class Program + static void Main(string[] args) { ... }
Console.WriteLine("Hola mundo");             // después: el archivo entero
```

**`new()` con tipo de destino** y **patrones relacionales y lógicos**:

```csharp
Dictionary<string, List<int>> mapa = new();
if (edad is >= 18 and < 65) { }
if (cliente is not null) { }
```

### C# 10 (2021, .NET 6)

**`global using` e `ImplicitUsings`:** los `using` que se repetían en todos los archivos se declaran una sola vez para todo el proyecto:

```csharp
// GlobalUsings.cs (o habilitando <ImplicitUsings>enable</ImplicitUsings> en el .csproj)
global using System.Text.Json;
global using MiEmpresa.Dominio;
```

Ojo: `global using` **no** descubre ni instala librerías por ti. Solo evita repetir los `using` de espacios de nombres que ya están disponibles (de .NET o de paquetes NuGet que agregaste). `ImplicitUsings` agrega automáticamente los más comunes (`System`, `System.Linq`, `System.Collections.Generic`, `System.IO`...).

**Espacios de nombres por archivo:**

```csharp
namespace MiApp.Servicios;      // todo el archivo, sin llaves ni un nivel extra de indentación

public class Facturador { }
```

Además: `record struct`, constantes `string` interpoladas, lambdas con tipo natural (`var f = (int x) => x * 2;`) y patrones de propiedades extendidos (`{ Direccion.Ciudad: "Lima" }`).

### C# 11 (2022, .NET 7)

**Raw string literals** ([Texto: char y string](../01-tipos-y-variables/05-Texto%20char%20y%20string.md)):

```csharp
string json = """
    { "nombre": "Ana", "ruta": "C:\datos" }
    """;
```

**`required`** ([Propiedades](../04-poo/03-Propiedades.md)), **patrones de lista** (`numeros is [1, 2, ..]`), **generic math** (`INumber<T>`), **miembros `static abstract` en interfaces** y el modificador **`file`**.

### C# 12 (2023, .NET 8)

**Constructores primarios** ([Constructores y this](../04-poo/04-Constructores%20y%20this.md)):

```csharp
public class Facturador                                       // antes
{
    private readonly IRepositorio _repo;
    public Facturador(IRepositorio repo) => _repo = repo;
}

public class Facturador(IRepositorio repo)                    // después
{
    public void Emitir(Factura f) => repo.Guardar(f);
}
```

**Expresiones de colección** ([Elegir la colección adecuada](../06-colecciones/04-Elegir%20la%20coleccion%20adecuada.md)):

```csharp
List<int> numeros = new List<int> { 1, 2, 3 };    // antes
List<int> numeros2 = [1, 2, 3];                    // después
int[] todos = [.. numeros, .. numeros2, 4];        // propagación
```

Además: parámetros por defecto en lambdas y alias para cualquier tipo (`using Punto = (int X, int Y);`).

### C# 13 (2024, .NET 9)

`params` con cualquier colección (`params ReadOnlySpan<int>`), el nuevo tipo `System.Threading.Lock`, la secuencia de escape `\e`, propiedades `partial` y `ref struct` que implementan interfaces.

### C# 14 (2025, .NET 10)

**La palabra clave `field`** ([Propiedades](../04-poo/03-Propiedades.md)):

```csharp
public string Nombre
{
    get;
    set => field = value?.Trim() ?? throw new ArgumentNullException(nameof(value));
}
```

**Miembros de extensión** (bloques `extension` con propiedades y miembros estáticos de extensión), **asignación condicional de null** (`cliente?.Direccion = nueva;`) y conversiones implícitas a `Span<T>`.

### Qué versión usa tu proyecto

Cada `TargetFramework` trae una versión de C# por defecto: `net8.0` → C# 12, `net9.0` → C# 13, `net10.0` → C# 14. Se puede ver con:

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <!-- <LangVersion>latest</LangVersion>   cambiar la versión de C# (normalmente no hace falta) -->
</PropertyGroup>
```

Si usas una característica más nueva que la de tu proyecto, el compilador da `error CS8652: The feature '...' is currently in Preview` o `error CS9058: Feature '...' is not available in C# 11.0. Please use language version 12.0 or greater.`

-----

## Ejemplo completo

Un mismo programa escrito a la manera de C# 5 y a la manera de C# 14.

**C# 5:**

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace Pedidos
{
    class Program
    {
        static void Main(string[] args)
        {
            var pedidos = new List<Pedido>();
            pedidos.Add(new Pedido("A-1", 120m, "Pagado"));
            pedidos.Add(new Pedido("A-2", 45m, "Pendiente"));
            pedidos.Add(new Pedido("A-3", 300m, "Enviado"));

            foreach (var p in pedidos)
            {
                string accion;
                switch (p.Estado)
                {
                    case "Pagado": accion = "Preparar"; break;
                    case "Enviado": accion = "Seguir"; break;
                    default: accion = "Esperar"; break;
                }
                Console.WriteLine(string.Format("{0}: {1} → {2}", p.Codigo, p.Total, accion));
            }

            var caros = pedidos.Where(p => p.Total > 100).ToList();
            Console.WriteLine(string.Format("Pedidos caros: {0}", caros.Count));
        }
    }

    class Pedido
    {
        private readonly string _codigo;
        private readonly decimal _total;
        private readonly string _estado;

        public Pedido(string codigo, decimal total, string estado)
        {
            _codigo = codigo;
            _total = total;
            _estado = estado;
        }

        public string Codigo { get { return _codigo; } }
        public decimal Total { get { return _total; } }
        public string Estado { get { return _estado; } }
    }
}
```

**C# 14:**

```csharp
List<Pedido> pedidos =
[
    new("A-1", 120m, Estado.Pagado),
    new("A-2", 45m, Estado.Pendiente),
    new("A-3", 300m, Estado.Enviado)
];

foreach (var (codigo, total, estado) in pedidos)
{
    string accion = estado switch
    {
        Estado.Pagado => "Preparar",
        Estado.Enviado => "Seguir",
        _ => "Esperar"
    };
    Console.WriteLine($"{codigo}: {total} → {accion}");
}

Console.WriteLine($"Pedidos caros: {pedidos.Count(p => p is { Total: > 100 })}");

enum Estado { Pendiente, Pagado, Enviado }
record Pedido(string Codigo, decimal Total, Estado Estado);
```

Salida (la misma en las dos versiones):

```text
A-1: 120 → Preparar
A-2: 45 → Esperar
A-3: 300 → Seguir
Pedidos caros: 2
```

La versión moderna usa top-level statements, una expresión de colección, `new()` con tipo de destino, un record posicional con deconstrucción, una expresión `switch`, interpolación, un patrón de propiedades y un enum en lugar de strings mágicos. De 60 líneas a 22, con el mismo comportamiento y más seguridad de tipos.

-----

## Errores comunes

**1. Usar una característica más nueva que la versión del proyecto.**
Qué pasa: `error CS9058: Feature 'collection expressions' is not available in C# 11.0. Please use language version 12.0 or greater.`
Por qué: el `TargetFramework` define la versión de C# por defecto.
Arreglo: actualiza el `TargetFramework` (lo recomendable) o usa la sintaxis anterior.

**2. Modernizar por modernizar.**
Qué pasa: un pull request enorme que cambia la sintaxis de cientos de archivos sin cambiar nada más, difícil de revisar y que genera conflictos con el trabajo de todos.
Por qué: el cambio de estilo no aporta valor por sí solo.
Arreglo: aplica la sintaxis moderna al código que tocas, o hazlo con herramientas automáticas (`dotnet format`) en un cambio aislado y acordado con el equipo.

**3. Abusar de la sintaxis compacta.**
Qué pasa: una expresión `switch` anidada en un ternario dentro de una interpolación, todo en una línea.
Por qué: "más corto" no siempre es "más claro".
Arreglo: usa la sintaxis moderna cuando mejora la legibilidad; si una línea necesita esfuerzo para entenderse, sepárala.

**4. Confundir `global using` con "instalar paquetes".**
Qué pasa: `error CS0246: The type or namespace name 'JsonConvert' could not be found` aunque pusiste el `global using`.
Por qué: el `using` solo importa nombres; el paquete tiene que estar referenciado.
Arreglo: `dotnet add package Newtonsoft.Json` y después el `using`.

-----

## Según la versión de C#

Esta lección es, en sí misma, la tabla de versiones. Resumen:

| C# | .NET | Año | Lo más usado |
| --- | --- | --- | --- |
| 6 | Framework 4.6 | 2015 | Interpolación, `=>` en miembros, `?.`, `nameof` |
| 7.x | Framework 4.7 / Core 2.x | 2017-18 | Tuplas, `out var`, patrones con `is`, funciones locales |
| 8 | Core 3.0 | 2019 | Expresión `switch`, nullable, rangos, `using` sin llaves, `IAsyncEnumerable` |
| 9 | .NET 5 | 2020 | Records, `init`, top-level statements, `new()` |
| 10 | .NET 6 | 2021 | `global using`, namespace por archivo, `record struct` |
| 11 | .NET 7 | 2022 | Raw strings, `required`, patrones de lista, generic math |
| 12 | .NET 8 | 2023 | Constructores primarios, expresiones de colección |
| 13 | .NET 9 | 2024 | `params` con colecciones, `Lock` |
| 14 | .NET 10 | 2025 | `field`, miembros de extensión, `?.` en asignaciones |

-----

## Cuándo sí y cuándo no

**Usa la sintaxis moderna cuando:**

* Reduce ruido sin ocultar la intención: records para datos, expresiones `switch` para mapeos, interpolación, `using` sin llaves.
* El equipo la conoce y el proyecto la soporta.

**Mantén la sintaxis existente cuando:**

* El proyecto tiene una convención establecida y el cambio no aporta claridad real.
* El código debe compilar con una versión de .NET anterior (por ejemplo, librerías que apuntan a .NET Standard 2.0).

-----

## Resumen en 5 líneas

1. Cada versión de C# viene con una versión de .NET; `net10.0` usa C# 14 por defecto.
2. C# 6–8 trajeron interpolación, `=>`, `?.`, tuplas, patrones, expresión `switch` y nullable.
3. C# 9–12 trajeron records, top-level statements, `global using`, namespaces por archivo, constructores primarios y expresiones de colección.
4. C# 13–14 suman `params` con colecciones, la palabra clave `field` y miembros de extensión.
5. Moderniza el código que tocas cuando mejora la claridad; reconoce la sintaxis antigua para trabajar con código heredado.

-----

## Para profundizar

<details>
<summary>Dónde seguir las novedades</summary>

* [Novedades de C#](https://learn.microsoft.com/dotnet/csharp/whats-new/) en la documentación oficial, con una página por versión.
* El repositorio [dotnet/csharplang](https://github.com/dotnet/csharplang) en GitHub, donde se discuten y diseñan las propuestas del lenguaje.
* [sharplab.io](https://sharplab.io): pega código moderno y mira en qué lo convierte el compilador (cómo queda un record, un `foreach` o un `async`). Es la mejor forma de entender el azúcar sintáctico.

</details>

<details>
<summary>Herramientas para modernizar código heredado</summary>

* Los **analizadores de estilo** del IDE sugieren la forma moderna (por ejemplo, "usar expresión switch", "usar coincidencia de patrones") y la aplican con Ctrl + .
* `dotnet format` aplica las preferencias del `.editorconfig` a toda la solución.
* El **.NET Upgrade Assistant** ayuda a migrar proyectos de .NET Framework a .NET moderno.

</details>

-----

## En entrevista

### Respuesta corta (junior)

C# fue agregando características para escribir menos código repetitivo: interpolación de strings, `?.`, tuplas, pattern matching, expresiones `switch`, records, top-level statements, `global using` y constructores primarios, entre otras. Uso la sintaxis moderna cuando hace el código más claro, y reconozco la antigua para trabajar con proyectos heredados.

### Respuesta ampliada (semi-senior)

La evolución de C# va hacia menos ceremonia (top-level statements, namespaces por archivo, `global using`, constructores primarios), más seguridad (tipos de referencia anulables, `required`, `init`), más estilo funcional (records, expresiones `switch`, patrones, inmutabilidad) y más rendimiento (`Span<T>`, `ref struct`, generic math). La versión del lenguaje está ligada al `TargetFramework`. En proyectos existentes, la modernización se hace de forma incremental y automatizada (analizadores, `dotnet format`, Upgrade Assistant), en cambios separados de los funcionales, priorizando la legibilidad sobre la brevedad.

### Preguntas frecuentes de seguimiento

**1. ¿Qué característica moderna de C# te parece más útil?**
Una respuesta sólida: los records y las expresiones `switch` con patrones, porque reducen mucho código y errores; o los tipos de referencia anulables, porque detectan `NullReferenceException` al compilar. Lo importante es justificarlo.

**2. ¿Qué hace `global using`?**
Importa un espacio de nombres para todos los archivos del proyecto, evitando repetir `using` en cada uno. No agrega referencias a paquetes.

**3. ¿De qué depende la versión de C# que puedes usar?**
Del `TargetFramework` del proyecto (o de `LangVersion`, si se configura explícitamente).

-----

## Práctica

**Ejercicio 1.** Moderniza este fragmento usando al menos cuatro características de C# 7 en adelante:

```csharp
public class Resultado
{
    private readonly bool _ok;
    private readonly string _mensaje;
    public Resultado(bool ok, string mensaje) { _ok = ok; _mensaje = mensaje; }
    public bool Ok { get { return _ok; } }
    public string Mensaje { get { return _mensaje; } }
}

public static string Describir(object valor)
{
    if (valor == null) return "nulo";
    if (valor is int)
    {
        int n = (int)valor;
        if (n < 0) return "entero negativo";
        return "entero";
    }
    if (valor is string)
    {
        string s = (string)valor;
        return "texto de " + s.Length + " caracteres";
    }
    return "otro";
}
```

<details>
<summary>Solución</summary>

```csharp
public record Resultado(bool Ok, string Mensaje);

public static string Describir(object? valor) => valor switch
{
    null => "nulo",
    int n when n < 0 => "entero negativo",
    int => "entero",
    string s => $"texto de {s.Length} caracteres",
    _ => "otro"
};
```

Características usadas: record posicional (C# 9), expresión `switch` (C# 8), patrones de tipo con variable y cláusula `when` (C# 7/8), patrón de tipo sin variable (C# 9), interpolación (C# 6), cuerpo de expresión (C# 6/7) y tipo de referencia anulable `object?` (C# 8).

</details>

**Ejercicio 2.** Para cada fragmento, indica desde qué versión de C# es válido:

1. `string s = $"Hola {nombre}";`
2. `var (a, b) = (1, 2);`
3. `int[] x = [1, 2, 3];`
4. `public record Punto(int X, int Y);`
5. `namespace MiApp;`
6. `string json = """{"a":1}""";`

<details>
<summary>Solución</summary>

1. C# 6.
2. C# 7.
3. C# 12.
4. C# 9.
5. C# 10.
6. C# 11.

</details>

-----

## Siguiente lección

Terminaste el último módulo de C# Core & Runtime. Vuelve al [índice general](../README.md) para repasar el recorrido completo.
