# Condicionales

## En una frase

Los condicionales eligen qué código se ejecuta según una condición: `if` / `else if` / `else` para decisiones generales, `switch` para comparar un valor contra varios casos y el operador ternario `? :` (o la expresión `switch`) cuando la decisión produce un **valor**.

-----

## Antes de empezar

Conviene que ya sepas:

* Escribir expresiones booleanas con comparaciones y `&&`, `||`, `!`, de [Lógica booleana](01-Logica%20booleana.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Flujo de control:** el orden en que se ejecutan las instrucciones de un programa.
* **Bloque:** un grupo de sentencias entre llaves `{ }`.
* **Sentencia condicional:** estructura que ejecuta un bloque solo si se cumple una condición.
* **`switch`:** estructura que compara un valor contra varios casos (`case`).
* **Operador ternario:** `condición ? valorSiTrue : valorSiFalse`, la versión de `if/else` que produce un valor.
* **Expresión `switch`:** versión de `switch` que devuelve un valor (C# 8).
* **Patrón:** una forma de describir qué valores coinciden: una constante, un rango, un tipo.

-----

## El problema

Sin condicionales, un programa ejecuta siempre las mismas instrucciones en el mismo orden. No podría:

* Mostrar "Aprobado" o "Desaprobado" según la nota.
* Aplicar un descuento solo a clientes VIP.
* Responder distinto según la opción que elige el usuario en un menú.

Las condicionales permiten que el programa **tome caminos distintos** según los datos.

-----

## Cómo funciona

### `if`

Ejecuta el bloque solo si la condición es `true`:

```csharp
string color = "azul";

if (color == "azul")
{
    Console.WriteLine("El color es azul");
}

Console.WriteLine("Esto se ejecuta siempre");
```

* La condición va **entre paréntesis** y debe ser un `bool`.
* El código condicional va **entre llaves**.
* La convención de .NET es poner la llave de apertura en su propia línea e indentar con **4 espacios**.

### `if ... else`

`else` define qué hacer cuando la condición es `false`:

```csharp
int nota = 11;

if (nota >= 13)
{
    Console.WriteLine("Aprobado");
}
else
{
    Console.WriteLine("Desaprobado");
}
```

Se ejecuta **exactamente uno** de los dos bloques.

### `else if`: varias condiciones en cadena

```csharp
int nota = 17;

if (nota >= 18)
{
    Console.WriteLine("Excelente");
}
else if (nota >= 14)
{
    Console.WriteLine("Bueno");
}
else if (nota >= 11)
{
    Console.WriteLine("Suficiente");
}
else
{
    Console.WriteLine("Insuficiente");
}
// Imprime "Bueno"
```

* Las condiciones se evalúan **de arriba hacia abajo** y se ejecuta **solo la primera** que sea `true`. Por eso el orden importa: si `nota >= 11` estuviera primero, una nota de 17 imprimiría "Suficiente".
* El `else` final es opcional y atrapa todo lo que no cumplió ninguna condición.

### Llaves opcionales (y por qué ponerlas igual)

Si el bloque tiene una sola sentencia, las llaves son opcionales:

```csharp
if (nota >= 13)
    Console.WriteLine("Aprobado");
```

Es legal, pero arriesgado: si después agregas una segunda línea creyendo que está dentro del `if`, quedará **fuera**. Muchos equipos exigen llaves siempre.

### Retorno temprano (*guard clauses*)

En lugar de anidar `if` dentro de `if`, sal antes cuando algo no es válido:

```csharp
// Anidado: difícil de seguir
if (usuario != null)
{
    if (usuario.Activo)
    {
        if (usuario.Saldo > 0)
        {
            Procesar(usuario);
        }
    }
}

// Con guardas: lineal
if (usuario is null) return;
if (!usuario.Activo) return;
if (usuario.Saldo <= 0) return;

Procesar(usuario);
```

### `switch`: un valor contra varios casos

Cuando comparas **una misma variable** contra muchos valores, `switch` es más claro que una cadena de `else if`:

```csharp
string color = "rojo";

switch (color)
{
    case "azul":
        Console.WriteLine("Color frío");
        break;
    case "rojo":
    case "naranja":                       // varios casos comparten el mismo código
        Console.WriteLine("Color cálido");
        break;
    default:
        Console.WriteLine("Color desconocido");
        break;
}
```

Reglas:

* Cada sección termina con `break` (o `return`, `throw`, `continue` dentro de un bucle). C# **no permite** "caer" de un caso al siguiente si el caso tiene código.
* Varias etiquetas `case` seguidas, sin código entre ellas, comparten el mismo bloque.
* `default` es **opcional**: se ejecuta si ningún caso coincide.
* Funciona con enteros, `char`, `string`, `enum`, `bool` y, desde C# 7, con patrones de cualquier tipo.

`switch` con patrones y condiciones (`when`):

```csharp
int temperatura = 35;

switch (temperatura)
{
    case < 0:
        Console.WriteLine("Helada");
        break;
    case >= 0 and < 25:
        Console.WriteLine("Templado");
        break;
    case var t when t > 40:
        Console.WriteLine("Peligro");
        break;
    default:
        Console.WriteLine("Calor");
        break;
}
// Imprime "Calor"
```

### El operador ternario `? :`

Cuando la decisión sirve para elegir **un valor**, el ternario es más compacto que un `if/else`:

```csharp
int nota = 15;
string estado = nota >= 13 ? "Aprobado" : "Desaprobado";
```

Se lee: "¿`nota >= 13`? Si es así, `"Aprobado"`; si no, `"Desaprobado"`".

* Las dos ramas deben ser del mismo tipo (o convertibles entre sí).
* Es una **expresión**: produce un valor, así que se puede usar dentro de otras expresiones, por ejemplo en una interpolación: `$"Estado: {(nota >= 13 ? "OK" : "NO")}"`.

Se pueden encadenar, pero se vuelven ilegibles rápido:

```csharp
string nivel = nota >= 18 ? "Excelente" : nota >= 14 ? "Bueno" : "Regular";   // límite razonable
```

Si necesitas más de dos niveles, usa una expresión `switch`.

### La expresión `switch` (C# 8+)

Es la versión de `switch` que **devuelve un valor**. Más corta y sin `break`:

```csharp
int nota = 17;

string nivel = nota switch
{
    >= 18 => "Excelente",
    >= 14 => "Bueno",
    >= 11 => "Suficiente",
    _ => "Insuficiente"          // _ es el "default"
};
```

```csharp
string dia = "sábado";

bool esFinDeSemana = dia switch
{
    "sábado" or "domingo" => true,
    _ => false
};
```

* Cada brazo tiene la forma `patrón => valor`, separados por comas.
* Se evalúan de arriba hacia abajo y gana el primero que coincide.
* `_` (descarte) coincide con cualquier cosa. Si no lo pones y ningún brazo coincide, se lanza una excepción en ejecución, y el compilador te avisa con una advertencia.

### Decisiones rápidas con `null`: `??` y `?.`

Dos operadores condicionales muy usados con valores que pueden ser `null`:

```csharp
string? nombre = null;

string mostrar = nombre ?? "Anónimo";       // si nombre es null, usa "Anónimo"
int? largo = nombre?.Length;                // si nombre es null, no llama a Length y da null
nombre ??= "Invitado";                      // asigna solo si es null
```

-----

## Ejemplo completo

Calculadora de envío según el destino y el peso:

```csharp
Console.Write("Destino (local/nacional/internacional): ");
string destino = (Console.ReadLine() ?? "").Trim().ToLower();

Console.Write("Peso en kg: ");
if (!double.TryParse(Console.ReadLine(), out double peso) || peso <= 0)
{
    Console.WriteLine("Peso inválido.");
    return;                                      // guarda: salir si el dato no sirve
}

decimal tarifaBase = destino switch
{
    "local" => 5m,
    "nacional" => 15m,
    "internacional" => 50m,
    _ => -1m
};

if (tarifaBase < 0)
{
    Console.WriteLine("Destino desconocido.");
    return;
}

decimal recargo;
if (peso > 20)
{
    recargo = 30m;
}
else if (peso > 5)
{
    recargo = 10m;
}
else
{
    recargo = 0m;
}

decimal total = tarifaBase + recargo;
string mensaje = total > 40 ? "Envío premium" : "Envío estándar";

Console.WriteLine($"Tarifa: {tarifaBase} + recargo: {recargo} = {total} ({mensaje})");
```

Ejecución de ejemplo:

```text
Destino (local/nacional/internacional): nacional
Peso en kg: 8
Tarifa: 15 + recargo: 10 = 25 (Envío estándar)
```

Se usan los tres estilos donde cada uno encaja mejor: una expresión `switch` para mapear valores, `if/else if` para rangos con lógica y un ternario para elegir un texto.

-----

## Errores comunes

**1. Olvidar `break` en un `switch`.**
Qué pasa: `error CS0163: Control cannot fall through from one case label ('case "azul":') to another`.
Por qué: C# prohíbe que la ejecución pase de un caso con código al siguiente.
Arreglo: termina cada sección con `break`, `return` o `throw`.

**2. Usar en el `switch` una variable sin asignar.**
Qué pasa: `error CS0165: Use of unassigned local variable 'color'`.
Por qué: `string color; switch (color)` lee una variable sin valor.
Arreglo: asigna un valor antes del `switch`.

**3. Ordenar mal las condiciones de un `else if`.**
Qué pasa: una nota de 19 imprime "Suficiente".
Por qué: la primera condición verdadera gana (`nota >= 11` ya es verdadera).
Arreglo: de la condición más específica a la más general.

**4. Punto y coma después del `if`.**
Qué pasa: `if (x > 5);` compila con `warning CS0642: Possible mistaken empty statement` y el bloque siguiente se ejecuta **siempre**.
Por qué: el `;` es una sentencia vacía: el `if` controla "nada".
Arreglo: quita el `;`.

**5. Tipos distintos en las ramas del ternario.**
Qué pasa: `var x = condicion ? 1 : "uno";` da `error CS0173: Type of conditional expression cannot be determined because there is no implicit conversion between 'int' and 'string'`.
Por qué: el ternario produce un único tipo.
Arreglo: haz que las dos ramas sean del mismo tipo.

**6. Expresión `switch` sin caso por defecto.**
Qué pasa: `warning CS8509: The switch expression does not handle all possible values` y, si llega un valor no previsto, `SwitchExpressionException` en ejecución.
Por qué: la expresión debe producir un valor para cualquier entrada.
Arreglo: agrega un brazo `_ => ...`.

-----

## Según la versión de C#

* **C# 7:** patrones en `switch` (`case int n:`) y cláusulas `when`.
* **C# 8:** expresiones `switch`, patrones de propiedades (`{ Activo: true }`) y `??=`.
* **C# 9:** patrones relacionales (`case < 0:`) y lógicos (`and`, `or`, `not`) dentro de `switch`.
* **C# 11:** patrones de lista (`[1, 2, ..]`).

Si lees código anterior a 2019, verás largas cadenas de `if/else if` o `switch` con `break` donde hoy se usaría una expresión `switch`.

-----

## Cuándo sí y cuándo no

| Situación | Herramienta |
| --- | --- |
| Una o dos condiciones con lógica variada | `if` / `else` |
| Rangos o condiciones distintas en cadena | `else if` |
| Comparar un valor con muchos casos y **ejecutar acciones** | `switch` (sentencia) |
| Comparar un valor con muchos casos y **obtener un valor** | Expresión `switch` |
| Elegir entre dos valores | Ternario `? :` |
| Valor por defecto si algo es `null` | `??` |

**Evita:**

* Ternarios anidados de más de dos niveles.
* `if` anidados de más de dos o tres niveles: usa guardas o extrae métodos.

-----

## Resumen en 5 líneas

1. `if (condición) { }` ejecuta un bloque si la condición es `true`; `else` cubre el caso contrario.
2. En `else if`, gana la primera condición verdadera: ordena de lo más específico a lo más general.
3. `switch` compara un valor contra casos; cada sección termina en `break` y `default` es opcional.
4. El ternario `c ? a : b` y la expresión `switch` producen valores.
5. `??` da un valor por defecto si algo es `null`; las guardas (`if (...) return;`) evitan anidar.

-----

## Para profundizar

<details>
<summary>Patrones de propiedades y de tipo</summary>

Las expresiones `switch` pueden mirar dentro de los objetos:

```csharp
decimal Descuento(Cliente c) => c switch
{
    { EsVip: true, Antiguedad: > 5 } => 0.20m,
    { EsVip: true } => 0.10m,
    { Antiguedad: > 10 } => 0.05m,
    _ => 0m
};
```

Y comprobar el tipo:

```csharp
string Describir(object o) => o switch
{
    int n when n < 0 => "entero negativo",
    int => "entero",
    string s => $"texto de {s.Length} caracteres",
    null => "nulo",
    _ => "otra cosa"
};
```

Se retoma en [Polimorfismo y casting](../04-poo/09-Polimorfismo%20y%20casting.md).

</details>

<details>
<summary>goto case: el único "fall-through" de C#</summary>

Si de verdad necesitas que un caso continúe en otro, debes decirlo explícitamente:

```csharp
switch (nivel)
{
    case 2:
        Console.WriteLine("Acceso a reportes");
        goto case 1;
    case 1:
        Console.WriteLine("Acceso básico");
        break;
}
```

Es raro y suele indicar que el diseño se puede simplificar.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`if/else` ejecuta un bloque u otro según una condición booleana; `else if` encadena varias condiciones y se ejecuta la primera verdadera. `switch` compara un valor contra varios casos, y cada caso termina en `break`. El operador ternario `? :` elige entre dos valores en una sola línea.

### Respuesta ampliada (semi-senior)

C# no permite *fall-through* implícito en `switch` (CS0163), lo que evita una clase entera de bugs de C. Desde C# 7 el `switch` admite patrones de tipo y `when`; desde C# 8, las expresiones `switch` permiten mapear entradas a valores de forma declarativa, con patrones de propiedades, relacionales y lógicos, y el compilador avisa si la expresión no es exhaustiva. Para la legibilidad se prefieren las guardas sobre el anidamiento. Cuando un `switch` sobre el tipo de un objeto crece y se repite en muchos lugares, suele ser señal de reemplazarlo por polimorfismo.

### Preguntas frecuentes de seguimiento

**1. ¿Es obligatorio `default` en un `switch`?**
No en la sentencia `switch`. En la expresión `switch` conviene un brazo `_`, porque si ningún brazo coincide se lanza una excepción.

**2. ¿Qué diferencia hay entre la sentencia y la expresión `switch`?**
La sentencia ejecuta acciones y usa `case`/`break`. La expresión devuelve un valor, usa `=>` y no necesita `break`.

**3. ¿Cuándo reemplazarías un `switch` por polimorfismo?**
Cuando el `switch` decide comportamiento según el tipo de objeto y se repite en varios lugares: agregar un tipo nuevo obligaría a modificar todos esos `switch` (viola el principio abierto/cerrado).

-----

## Práctica

**Ejercicio 1.** Reescribe esta cadena de `if` como una expresión `switch`:

```csharp
string mensaje;
if (codigo == 200) mensaje = "OK";
else if (codigo == 404) mensaje = "No encontrado";
else if (codigo == 500) mensaje = "Error del servidor";
else mensaje = "Desconocido";
```

<details>
<summary>Solución</summary>

```csharp
string mensaje = codigo switch
{
    200 => "OK",
    404 => "No encontrado",
    500 => "Error del servidor",
    _ => "Desconocido"
};
```

</details>

**Ejercicio 2.** Escribe un programa que pida un número del 1 al 7 y muestre el día de la semana. Si el número está fuera de rango o no es un número, debe mostrar un mensaje de error. Usa una guarda y un `switch`.

<details>
<summary>Solución</summary>

```csharp
Console.Write("Número de día (1-7): ");
if (!int.TryParse(Console.ReadLine(), out int n) || n is < 1 or > 7)
{
    Console.WriteLine("Debe ser un número entre 1 y 7.");
    return;
}

string dia = n switch
{
    1 => "Lunes",
    2 => "Martes",
    3 => "Miércoles",
    4 => "Jueves",
    5 => "Viernes",
    6 => "Sábado",
    _ => "Domingo"
};

Console.WriteLine(dia);
```

</details>

-----

## Siguiente lección

[Arrays](03-Arrays.md)
