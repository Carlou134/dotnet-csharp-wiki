# Enumeraciones

## En una frase

Un `enum` es un tipo de valor que define un **conjunto fijo de constantes con nombre** (`Pequeno`, `Mediano`, `Grande`) respaldadas por un entero, para reemplazar números o strings "mágicos" por nombres que el compilador verifica.

-----

## Antes de empezar

Conviene que ya sepas:

* Condicionales y la expresión `switch`, de [Condicionales](../02-control-de-flujo/02-Condicionales.md).
* Conversiones explícitas (cast) y `TryParse`, de [Conversiones de tipos](../01-tipos-y-variables/03-Conversiones%20de%20tipos.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Enumeración (`enum`):** tipo que define un conjunto de constantes con nombre.
* **Miembro de un enum:** cada una de esas constantes (`DrinkSize.Medium`).
* **Tipo subyacente:** el tipo entero que guarda el valor (`int` por defecto).
* **`[Flags]`:** atributo que indica que los miembros se pueden combinar como bits.
* **Operación bit a bit:** operar sobre los bits de un número con `|`, `&`, `~`.
* **Número mágico:** un literal sin nombre cuyo significado no es evidente (`if (estado == 3)`).

-----

## El problema

Quieres representar el estado de un pedido. Sin enums, tienes dos opciones malas:

```csharp
int estado = 2;               // ¿2 es "enviado" o "entregado"? Hay que buscarlo en algún comentario
if (estado == 3) { /* ... */ }

string estado2 = "Enviado";   // ¿"enviado", "Enviado" o "ENVIADO"? Un error de tipeo compila sin problemas
if (estado2 == "Envaido") { /* nunca entra, y nadie se entera */ }
```

Con números, el código no se entiende. Con strings, el compilador no te protege de errores de tipeo y cualquier texto es "válido". Necesitas un tipo con un **conjunto cerrado** de valores, con nombres legibles y verificados al compilar.

-----

## Cómo funciona

### Declarar y usar un enum

```csharp
enum EstadoPedido
{
    Pendiente,
    Pagado,
    Enviado,
    Entregado,
    Cancelado
}
```

```csharp
EstadoPedido estado = EstadoPedido.Pagado;

if (estado == EstadoPedido.Pagado)
{
    Console.WriteLine("Preparar el envío");
}

estado = EstadoPedido.Enviado;
Console.WriteLine(estado);          // Enviado (ToString devuelve el nombre)
```

* Los miembros se separan con **comas** y no llevan `;`.
* Se accede con `NombreDelEnum.Miembro`.
* `EstadoPedido.Envaido` no compila: el error aparece en el editor, no en producción.
* Convención: nombre del enum en **singular** y PascalCase; miembros en PascalCase.

### Valores numéricos implícitos y explícitos

Por defecto, el primer miembro vale `0` y cada uno suma 1:

```csharp
enum Talla
{
    Pequena,   // 0
    Mediana,   // 1
    Grande     // 2
}

Console.WriteLine((int)Talla.Grande);   // 2
Talla t = (Talla)1;                     // Mediana
```

Puedes asignar valores propios cuando tienen significado:

```csharp
enum TallaOnzas
{
    Pequena = 12,
    Mediana = 16,
    Grande = 20
}

enum CodigoHttp
{
    Ok = 200,
    NoEncontrado = 404,
    ErrorServidor = 500
}
```

Y cambiar el tipo subyacente (cualquier entero: `byte`, `short`, `long`...):

```csharp
enum Prioridad : byte { Baja, Media, Alta }
```

### Enums y `switch`

Los enums brillan con `switch`, porque el conjunto de casos está cerrado:

```csharp
decimal Precio(Talla talla) => talla switch
{
    Talla.Pequena => 3m,
    Talla.Mediana => 4m,
    Talla.Grande => 5m,
    _ => throw new ArgumentOutOfRangeException(nameof(talla))
};
```

El brazo `_` es necesario: como verás más abajo, una variable de tipo enum **puede** contener valores que no están en la lista.

### Métodos de la clase `Enum`

```csharp
string[] nombres = Enum.GetNames<Talla>();        // ["Pequena", "Mediana", "Grande"]
Talla[] valores = Enum.GetValues<Talla>();        // [Pequena, Mediana, Grande]
bool existe = Enum.IsDefined(Talla.Grande);       // true
bool existe2 = Enum.IsDefined((Talla)7);          // false

foreach (Talla t in Enum.GetValues<Talla>())
{
    Console.WriteLine($"{t} = {(int)t}");
}
```

Las versiones genéricas (`GetNames<T>()`, `GetValues<T>()`, `IsDefined<T>()`) existen desde **.NET 5**. En código antiguo verás la forma con `typeof`:

```csharp
string[] nombres = Enum.GetNames(typeof(Talla));
Array valores = Enum.GetValues(typeof(Talla));
```

### De texto a enum: `Parse` y `TryParse`

```csharp
Talla a = Enum.Parse<Talla>("Mediana");                       // Mediana
Talla b = Enum.Parse<Talla>("mediana", ignoreCase: true);     // Mediana

if (Enum.TryParse("Grande", out Talla c))
{
    Console.WriteLine(c);                                     // Grande
}

Enum.Parse<Talla>("ExtraGrande");   // ArgumentException: el valor no está definido
```

Igual que con los números (ver [Conversiones de tipos](../01-tipos-y-variables/03-Conversiones%20de%20tipos.md)), usa `TryParse` para datos que vienen de afuera.

### La trampa: un enum puede valer cualquier número

Un enum es, en el fondo, un entero con nombres. El compilador **no** impide valores fuera de la lista:

```csharp
Talla rara = (Talla)99;                          // compila y se ejecuta
Console.WriteLine(rara);                         // 99

Enum.TryParse("99", out Talla otra);             // devuelve TRUE: "99" es un número válido
Console.WriteLine(Enum.IsDefined(otra));         // False
```

Por eso, cuando un enum llega desde afuera (una API, una base de datos, el usuario), valídalo con `Enum.IsDefined`. Y además:

```csharp
Talla sinAsignar = default;   // 0 → Pequena, ¡aunque nadie lo eligió!
```

El valor por defecto de un enum es siempre `0`. Si `0` no es un valor válido, define un miembro explícito para representarlo:

```csharp
enum EstadoPedido
{
    Desconocido = 0,    // el valor por defecto tiene un significado claro
    Pendiente = 1,
    Pagado = 2
}
```

### Banderas: `[Flags]`

Algunos valores no son excluyentes: un archivo puede tener permisos de lectura **y** de escritura a la vez. Con `[Flags]`, cada miembro es una **potencia de 2** (un bit distinto) y se combinan con operaciones bit a bit:

```csharp
[Flags]
enum Permisos
{
    Ninguno = 0,
    Leer = 1,          // 0001
    Escribir = 2,      // 0010
    Ejecutar = 4,      // 0100
    Borrar = 8,        // 1000
    LeerEscribir = Leer | Escribir    // combinación con nombre
}
```

```csharp
Permisos p = Permisos.Leer | Permisos.Escribir;      // combinar con OR
Console.WriteLine(p);                                // LeerEscribir

p |= Permisos.Ejecutar;                              // agregar
Console.WriteLine(p);                                // LeerEscribir, Ejecutar (usa la combinación con nombre)

bool puedeEscribir = p.HasFlag(Permisos.Escribir);   // comprobar → true
bool puedeBorrar = (p & Permisos.Borrar) != 0;       // comprobar con AND → false

p &= ~Permisos.Escribir;                             // quitar con AND + NOT
Console.WriteLine(p);                                // Leer, Ejecutar
```

| Operación | Sintaxis |
| --- | --- |
| Combinar | `a \| b` |
| Agregar | `p \|= Permisos.X` |
| Comprobar | `p.HasFlag(Permisos.X)` o `(p & Permisos.X) != 0` |
| Quitar | `p &= ~Permisos.X` |

Reglas de diseño: incluye `Ninguno = 0`, usa potencias de 2 y nombra el enum en **plural** (`Permisos`). Usa `[Flags]` solo para opciones combinables: mezclar en un mismo enum de flags valores excluyentes (como `Pequeno`, `Mediano`, `Grande`) permite combinaciones absurdas como "pequeño y grande a la vez".

-----

## Ejemplo completo

```csharp
var pedidos = new (int Id, EstadoPedido Estado, Extras Extras)[]
{
    (1, EstadoPedido.Pagado, Extras.Envoltorio | Extras.Tarjeta),
    (2, EstadoPedido.Enviado, Extras.Ninguno),
    (3, EstadoPedido.Cancelado, Extras.EnvioExpress),
    (4, (EstadoPedido)42, Extras.Ninguno)            // simula un dato corrupto de la base de datos
};

foreach (var (id, estado, extras) in pedidos)
{
    if (!Enum.IsDefined(estado))
    {
        Console.WriteLine($"Pedido {id}: estado inválido ({(int)estado})");
        continue;
    }

    string accion = estado switch
    {
        EstadoPedido.Pendiente => "Esperar pago",
        EstadoPedido.Pagado => "Preparar",
        EstadoPedido.Enviado => "Seguir envío",
        EstadoPedido.Entregado => "Pedir reseña",
        EstadoPedido.Cancelado => "Reembolsar",
        _ => "Revisar"
    };

    decimal costoExtras = 0;
    if (extras.HasFlag(Extras.Envoltorio)) costoExtras += 5m;
    if (extras.HasFlag(Extras.Tarjeta)) costoExtras += 2m;
    if (extras.HasFlag(Extras.EnvioExpress)) costoExtras += 15m;

    Console.WriteLine($"Pedido {id}: {estado,-10} → {accion,-14} extras: {extras} ({costoExtras:N2})");
}

Console.Write("Escribe un estado para filtrar: ");
if (Enum.TryParse(Console.ReadLine(), ignoreCase: true, out EstadoPedido filtro) && Enum.IsDefined(filtro))
{
    Console.WriteLine($"Filtrando por {filtro} (código {(int)filtro})");
}
else
{
    Console.WriteLine($"Estados válidos: {string.Join(", ", Enum.GetNames<EstadoPedido>())}");
}

enum EstadoPedido
{
    Desconocido = 0,
    Pendiente,
    Pagado,
    Enviado,
    Entregado,
    Cancelado
}

[Flags]
enum Extras
{
    Ninguno = 0,
    Envoltorio = 1,
    Tarjeta = 2,
    EnvioExpress = 4
}
```

Salida (si escribes "enviado"):

```text
Pedido 1: Pagado     → Preparar       extras: Envoltorio, Tarjeta (7.00)
Pedido 2: Enviado    → Seguir envío   extras: Ninguno (0.00)
Pedido 3: Cancelado  → Reembolsar     extras: EnvioExpress (15.00)
Pedido 4: estado inválido (42)
Escribe un estado para filtrar: enviado
Filtrando por Enviado (código 3)
```

-----

## Errores comunes

**1. Terminar los miembros con `;`.**
Qué pasa: `error CS1003: Syntax error, ',' expected`.
Por qué: los miembros de un enum se separan con comas.
Arreglo: `enum Nivel { Uno, Dos, Tres }`.

**2. Asignar un entero sin cast.**
Qué pasa: `Talla t = 1;` da `error CS0266: Cannot implicitly convert type 'int' to 'Talla'`.
Por qué: un enum es un tipo propio, aunque guarde un entero. (Solo el literal `0` se convierte implícitamente).
Arreglo: `Talla t = (Talla)1;`, o mejor, `Talla t = Talla.Mediana;`.

**3. Intentar valores de tipo `string`.**
Qué pasa: `enum Color { Rojo = "rojo" }` da `error CS0029: Cannot implicitly convert type 'string' to 'int'`.
Por qué: el tipo subyacente de un enum siempre es entero.
Arreglo: usa el nombre del miembro (`Color.Rojo.ToString()`) o un diccionario `Color → string`.

**4. Confiar en que un enum siempre tiene un valor definido.**
Qué pasa: un `switch` sin `_` lanza `SwitchExpressionException` con un valor como `(Talla)99`.
Por qué: el compilador permite cualquier entero.
Arreglo: valida con `Enum.IsDefined` lo que viene de afuera e incluye siempre un brazo por defecto.

**5. `[Flags]` sin potencias de 2.**
Qué pasa: `Leer = 1, Escribir = 2, Ejecutar = 3` → `Leer | Escribir` es igual a `Ejecutar`.
Por qué: 3 es `0011`, la combinación de los bits 1 y 2.
Arreglo: 1, 2, 4, 8, 16... o con desplazamientos: `1 << 0`, `1 << 1`, `1 << 2`.

-----

## Según la versión de C#

* **C# 7.3:** restricción genérica `where T : Enum` para escribir métodos genéricos sobre enums.
* **.NET Core 2.0 / .NET 5:** `Enum.Parse<T>` y, desde .NET 5, `GetNames<T>`, `GetValues<T>` e `IsDefined<T>`, que evitan `typeof(...)` y casts.
* **C# 8 / 9:** expresiones `switch` y patrones (`estado is EstadoPedido.Pagado or EstadoPedido.Enviado`), que combinan muy bien con enums.

-----

## Cuándo sí y cuándo no

**Usa un enum cuando:**

* Hay un conjunto **pequeño, fijo y conocido al compilar** de opciones: estados, tipos, niveles, días.

**Usa `[Flags]` cuando:**

* Las opciones son **combinables** (permisos, opciones de configuración).

**Usa otra cosa cuando:**

* Los valores cambian sin recompilar (categorías de productos que define el usuario): una tabla en la base de datos.
* Cada opción tiene mucho comportamiento propio: clases con polimorfismo, en lugar de un `switch` repetido en diez lugares.

-----

## Resumen en 5 líneas

1. `enum Nombre { A, B, C }` define constantes con nombre; por defecto valen 0, 1, 2...
2. Se puede asignar valores explícitos y cambiar el tipo subyacente (`enum X : byte`).
3. `Enum.GetValues<T>()`, `GetNames<T>()`, `IsDefined()`, `Parse<T>()` y `TryParse()` permiten recorrer, validar y convertir.
4. Un enum puede contener cualquier entero: valida lo que viene de afuera y define qué significa el `0`.
5. `[Flags]` + potencias de 2 permite combinar opciones con `|`, comprobar con `HasFlag` y quitar con `& ~`.

-----

## Para profundizar

<details>
<summary>Métodos genéricos sobre enums</summary>

Con la restricción `where T : struct, Enum` se pueden escribir utilidades que sirven para cualquier enum:

```csharp
static T ParseOrDefault<T>(string texto, T porDefecto) where T : struct, Enum =>
    Enum.TryParse(texto, ignoreCase: true, out T valor) && Enum.IsDefined(valor) ? valor : porDefecto;

Talla t = ParseOrDefault("enorme", Talla.Mediana);   // Mediana
```

Las restricciones genéricas se estudian en [Restricciones y varianza](05-Restricciones%20y%20varianza.md).

</details>

<details>
<summary>Enums en APIs y bases de datos</summary>

* En JSON, por defecto `System.Text.Json` serializa los enums como **números**. Para usar los nombres, se agrega `JsonStringEnumConverter`. Los nombres son más legibles y más robustos ante cambios de orden.
* En Entity Framework se guardan como enteros por defecto; se pueden guardar como texto con `.HasConversion<string>()`.
* **Nunca reordenes ni insertes miembros** en medio de un enum que ya se guardó como número: los datos existentes cambiarían de significado. Asigna valores explícitos a los enums que se persisten.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un enum es un tipo de valor que define un conjunto de constantes con nombre, respaldadas por un entero. Mejora la legibilidad y evita errores frente a usar números o strings. Por defecto, los valores empiezan en 0, pero se pueden asignar a mano. Con el atributo `[Flags]` y potencias de 2 se pueden combinar varios valores.

### Respuesta ampliada (semi-senior)

Un enum es un tipo de valor con un tipo subyacente entero (`int` por defecto) que no restringe los valores en ejecución: `(T)99` es válido, por lo que los datos externos se validan con `Enum.IsDefined`, teniendo en cuenta que `TryParse` acepta strings numéricos. El `default` es 0, así que conviene que el 0 tenga significado (`None`/`Unknown`). Los enums `[Flags]` modelan conjuntos de bits combinables con `|`, `&` y `~`; `HasFlag` está optimizado por el JIT desde .NET Core 2.1. Al persistirlos, conviene usar valores explícitos o serializarlos por nombre. Cuando un `switch` sobre un enum se repite en muchos lugares, suele ser señal de reemplazarlo por polimorfismo.

### Preguntas frecuentes de seguimiento

**1. ¿Un enum puede tener valores de texto?**
No. Su tipo subyacente siempre es un entero. Para asociar textos se usan atributos (`[Description]`), diccionarios o el propio nombre del miembro.

**2. ¿Qué pasa si casteo un número que no existe en el enum?**
Compila y funciona: la variable guarda ese número. Hay que validarlo con `Enum.IsDefined`.

**3. ¿Para qué sirve `[Flags]`?**
Para representar combinaciones de opciones como bits. Además, hace que `ToString()` muestre los nombres combinados (`"Leer, Escribir"`).

-----

## Práctica

**Ejercicio 1.** Crea un enum `DiaSemana` (Lunes = 1 ... Domingo = 7) y un método `EsLaborable(DiaSemana dia)` con una expresión `switch` y patrones. Luego pide un número al usuario y muestra el día y si es laborable, validando la entrada.

<details>
<summary>Solución</summary>

```csharp
Console.Write("Número de día (1-7): ");
if (int.TryParse(Console.ReadLine(), out int n) && Enum.IsDefined((DiaSemana)n))
{
    var dia = (DiaSemana)n;
    Console.WriteLine($"{dia}: {(EsLaborable(dia) ? "laborable" : "fin de semana")}");
}
else
{
    Console.WriteLine("Día inválido");
}

static bool EsLaborable(DiaSemana dia) => dia switch
{
    DiaSemana.Sabado or DiaSemana.Domingo => false,
    _ => true
};

enum DiaSemana
{
    Lunes = 1, Martes, Miercoles, Jueves, Viernes, Sabado, Domingo
}
```

</details>

**Ejercicio 2.** Modela las opciones de una pizza (`Queso`, `Jamon`, `Champinones`, `Aceitunas`) como un enum `[Flags]`. Crea un pedido con queso y jamón, agrega aceitunas, quita el jamón y muestra el resultado y si tiene champiñones.

<details>
<summary>Solución</summary>

```csharp
var pedido = Ingredientes.Queso | Ingredientes.Jamon;
pedido |= Ingredientes.Aceitunas;
pedido &= ~Ingredientes.Jamon;

Console.WriteLine(pedido);                                     // Queso, Aceitunas
Console.WriteLine(pedido.HasFlag(Ingredientes.Champinones));   // False

[Flags]
enum Ingredientes
{
    Ninguno = 0,
    Queso = 1,
    Jamon = 2,
    Champinones = 4,
    Aceitunas = 8
}
```

</details>

-----

## Siguiente lección

[Structs y records](02-Structs%20y%20records.md)
