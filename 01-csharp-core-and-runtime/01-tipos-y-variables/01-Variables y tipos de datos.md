# Variables y tipos de datos

## En una frase

Una variable es un nombre que guarda un valor en memoria, y en C# cada variable tiene un **tipo fijo** (`int`, `double`, `string`, `bool`...) que el compilador conoce y verifica antes de ejecutar.

-----

## Antes de empezar

Conviene que ya sepas:

* Escribir y ejecutar un programa con `Console.WriteLine`, como en [Tu primer programa](../00-introduccion/03-Tu%20primer%20programa.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Variable:** un nombre asociado a un espacio de memoria donde se guarda un valor que puede cambiar.
* **Tipo de dato:** define qué valores puede guardar una variable, cuánta memoria ocupa y qué operaciones admite.
* **Declarar:** anunciar una variable con su tipo y su nombre.
* **Inicializar:** asignarle su primer valor.
* **Literal:** un valor escrito directamente en el código: `42`, `3.14`, `"hola"`, `true`.
* **Tipado estático:** los tipos se conocen y verifican al compilar, no al ejecutar.
* **Tipado fuerte:** el lenguaje no mezcla tipos incompatibles sin una conversión explícita.
* **Constante:** un valor con nombre que no puede cambiar.
* **Palabra clave (*keyword*):** palabra reservada del lenguaje, como `int`, `if` o `class`.

-----

## El problema

Para la computadora todo son bits. La secuencia `01000001` puede ser el número 65 o la letra `A`. Sin un tipo, no hay forma de saber qué operaciones tienen sentido: ¿se puede elevar al cuadrado un texto? ¿Pasar a mayúsculas un número?

En lenguajes con tipado dinámico, como JavaScript, ese error aparece **al ejecutar**, a veces en producción:

```js
let precio = "100";
let total = precio * 2 + 1;   // 201... ¿o "1001"? Depende del operador.
```

C# te obliga a decir de qué tipo es cada dato y verifica **al compilar** que lo uses bien. El error aparece en tu editor, no en la pantalla de tu cliente.

-----

## Cómo funciona

### Declarar e inicializar

La forma general es `tipo nombre = valor;`:

```csharp
int edad = 32;                 // declarar e inicializar en una línea

string pais;                   // declarar...
pais = "Perú";                 // ...e inicializar después

edad = 33;                     // cambiar el valor (mismo tipo)
Console.WriteLine(edad);       // 33
```

Una vez declarada, la variable **conserva su tipo para siempre**. `edad = "treinta";` no compila.

### Los tipos que más vas a usar

| Tipo | Guarda | Literal de ejemplo |
| --- | --- | --- |
| `int` | Enteros | `42`, `-7`, `1_000_000` |
| `double` | Decimales (uso general, ciencia) | `3.14`, `-0.5` |
| `decimal` | Decimales exactos (dinero) | `19.99m` |
| `bool` | Verdadero o falso | `true`, `false` |
| `char` | Un solo carácter | `'A'`, `'ñ'` |
| `string` | Texto | `"Hola"` |

### Todos los tipos numéricos integrados

| Tipo | Tamaño | Rango | Sufijo del literal |
| --- | --- | --- | --- |
| `sbyte` | 1 byte | -128 a 127 | — |
| `byte` | 1 byte | 0 a 255 | — |
| `short` | 2 bytes | -32.768 a 32.767 | — |
| `ushort` | 2 bytes | 0 a 65.535 | — |
| `int` | 4 bytes | ±2.147 millones (aprox.) | — |
| `uint` | 4 bytes | 0 a 4.294 millones (aprox.) | `u` |
| `long` | 8 bytes | ±9,2 × 10¹⁸ | `L` |
| `ulong` | 8 bytes | 0 a 1,8 × 10¹⁹ | `ul` |
| `nint` / `nuint` | 4 u 8 bytes | Depende del procesador (32 o 64 bits) | — |
| `float` | 4 bytes | ~6-9 dígitos de precisión | `f` |
| `double` | 8 bytes | ~15-17 dígitos de precisión | `d` (opcional) |
| `decimal` | 16 bytes | 28-29 dígitos exactos | `m` |

Además: `bool` (1 byte en la práctica), `char` (2 bytes, un carácter UTF-16) y `string` (tamaño variable).

```csharp
bool estaAbierto = true;
byte edadMascota = 12;
sbyte temperatura = -5;
char nota = 'A';
short altura = 180;
ushort ramas = 33;
int nivelDelMar = -24;
uint anio = 2026u;
long poblacionMundial = 8_100_000_000L;
ulong estrellas = 100_000_000_000ul;
nint paginas = 412;
nuint kilometros = 2597;
float alturaJirafa = 5.5f;
double pesoHipopotamo = 1_500.75;
decimal saldo = 1_493_867.23m;
string mensaje = "Hola";
```

No hace falta memorizar los rangos. En la práctica: **`int`** para enteros, **`long`** si `int` no alcanza, **`double`** para decimales en general y **`decimal`** para dinero. Cada número se ve a fondo en [Números y operadores](04-Numeros%20y%20operadores.md).

### Alias de C# y tipos de .NET

Cada palabra clave de tipo es un **alias** de un tipo de .NET. Son exactamente lo mismo:

| Alias C# | Tipo .NET |
| --- | --- |
| `int` | `System.Int32` |
| `long` | `System.Int64` |
| `double` | `System.Double` |
| `decimal` | `System.Decimal` |
| `bool` | `System.Boolean` |
| `char` | `System.Char` |
| `string` | `System.String` |
| `object` | `System.Object` |

Por eso `int.Parse(...)` e `Int32.Parse(...)` son la misma llamada. La convención es usar el alias (`int`, `string`).

### Tipado implícito con `var`

Si el tipo es obvio por el valor, puedes dejar que el compilador lo **infiera**:

```csharp
var ciudad = "Lima";     // el compilador decide: string
var total = 10;          // int
var precio = 9.99;       // double (¡no decimal!)
var monto = 9.99m;       // decimal
```

`var` **no** es tipado dinámico. La variable sigue siendo `string`, `int`, etc. para siempre:

```csharp
var ciudad = "Lima";
ciudad = 25;             // error CS0029: no se puede convertir int a string
```

Reglas de `var`:

* Solo para variables **locales** (dentro de un método).
* Debe inicializarse en la misma línea: `var x;` no compila (CS0818).
* No se puede inicializar con `null` solo: `var x = null;` no compila (CS0815).

### Constantes

Cuando un valor **nunca** cambia, decláralo con `const`:

```csharp
const double Pi = 3.14159;
const int DiasPorSemana = 7;
const string Saludo = "Hola";

Pi = 3;   // error CS0131: el lado izquierdo de una asignación debe ser una variable...
```

* El valor debe conocerse **al compilar** (literales u otras constantes). `const DateTime Hoy = DateTime.Now;` no compila.
* Por convención se nombran en **PascalCase** (`DiasPorSemana`). No es una regla del compilador, es una convención de .NET.

### Valores por defecto

Cada tipo tiene un valor por defecto, que obtienes con `default`:

```csharp
int a = default;       // 0
bool b = default;      // false
double c = default;    // 0
char d = default;      // '\0'
string? e = default;   // null
```

Las **variables locales** no reciben ese valor automáticamente: el compilador exige asignarlas antes de leerlas.

```csharp
int contador;
Console.WriteLine(contador);   // error CS0165: uso de la variable local no asignada 'contador'
```

Los **campos** de una clase y los elementos de un array, en cambio, sí empiezan con su valor por defecto. Se ve en [Arrays](../02-control-de-flujo/03-Arrays.md) y en [Clases y objetos](../04-poo/01-Clases%20y%20objetos.md).

### Nombres válidos y convenciones

Un nombre debe empezar con letra o `_` y puede contener letras, dígitos y `_`. No puede ser una palabra clave.

```csharp
int edadUsuario;     // ✅ camelCase: convención para variables locales
int _contador;       // ✅ válido (convención para campos privados)
int edad2;           // ✅ válido
int 2edad;           // ❌ CS1001: no puede empezar con dígito
int edad-usuario;    // ❌ el guion se interpreta como resta
int class;           // ❌ CS1041: 'class' es palabra clave
int @class;          // ✅ válido con @ delante (evítalo salvo que no haya alternativa)
```

| Elemento | Convención | Ejemplo |
| --- | --- | --- |
| Variable local, parámetro | camelCase | `nombreCliente` |
| Campo privado | _camelCase | `_saldo` |
| Constante, método, propiedad, clase | PascalCase | `MaxIntentos`, `CalcularTotal` |

Usa nombres que expliquen el contenido: `diasDesdeUltimoPago` es mejor que `d` o `x1`.

-----

## Ejemplo completo

```csharp
const decimal TasaImpuesto = 0.18m;

string producto = "Teclado mecánico";
int cantidad = 3;
decimal precioUnitario = 149.90m;
bool envioGratis = cantidad >= 3;

decimal subtotal = precioUnitario * cantidad;
decimal impuesto = subtotal * TasaImpuesto;
decimal total = subtotal + impuesto;

Console.WriteLine($"Producto: {producto}");
Console.WriteLine($"Cantidad: {cantidad}");
Console.WriteLine($"Subtotal: {subtotal}");
Console.WriteLine($"Impuesto: {impuesto}");
Console.WriteLine($"Total: {total}");
Console.WriteLine($"Envío gratis: {envioGratis}");
```

Salida:

```text
Producto: Teclado mecánico
Cantidad: 3
Subtotal: 449.70
Impuesto: 80.9460
Total: 530.6460
Envío gratis: True
```

Se usó `decimal` porque es dinero, `const` para un valor fijo y `bool` para una condición. (El separador decimal que se muestra depende de la configuración regional del sistema: puede salir `449,70`).

-----

## Errores comunes

**1. Usar una variable sin declararla.**
Qué pasa: `error CS0103: The name 'datos' does not exist in the current context`.
Por qué: en C#, toda variable se declara con su tipo (o con `var`) antes de usarse.
Arreglo: `string datos = "...";`.

**2. Asignar un valor de otro tipo.**
Qué pasa: `error CS0266: Cannot implicitly convert type 'double' to 'int'. An explicit conversion exists (are you missing a cast?)`.
Por qué: `int puntaje = 45.39;` perdería la parte decimal, y C# no pierde datos en silencio.
Arreglo: usa el tipo correcto (`double puntaje = 45.39;`) o convierte explícitamente. Se ve en [Conversiones de tipos](03-Conversiones%20de%20tipos.md).

**3. Leer una variable local sin asignarla.**
Qué pasa: `error CS0165: Use of unassigned local variable`.
Por qué: el compilador garantiza que nunca leas basura de la memoria.
Arreglo: inicializa la variable al declararla.

**4. Olvidar el sufijo `m` o `f`.**
Qué pasa: `error CS0664: Literal of type double cannot be implicitly converted to type 'decimal'; use an 'M' suffix`.
Por qué: un literal con punto decimal es `double` por defecto.
Arreglo: `decimal precio = 9.99m;` y `float x = 1.5f;`.

**5. Usar `l` minúscula como sufijo de `long`.**
Qué pasa: `warning CS0078: The 'l' suffix is easily confused with the digit '1'`.
Por qué: `25000l` se confunde visualmente con `250001`.
Arreglo: usa `L` mayúscula: `25000L`.

**6. Declarar dos veces el mismo nombre.**
Qué pasa: `error CS0128: A local variable named 'x' is already defined in this scope`.
Por qué: un nombre identifica una sola variable dentro de su ámbito.
Arreglo: cambia el nombre o reutiliza la variable sin volver a declararla.

-----

## Según la versión de C#

* **C# 3:** `var` (tipado implícito para variables locales).
* **C# 7:** separador de dígitos `_` (`1_000_000`) y literales binarios (`0b1010`).
* **C# 7.1:** el literal `default` sin tipo (`int x = default;`). Antes se escribía `default(int)`.
* **C# 9:** `nint` y `nuint` (enteros del tamaño nativo del procesador).
* **C# 10:** constantes `string` construidas con interpolación de otras constantes.

-----

## Cuándo sí y cuándo no

**Usa `var` cuando:**

* El tipo es evidente en la misma línea: `var cliente = new Cliente();`, `var nombres = new List<string>();`.

**Escribe el tipo explícito cuando:**

* No es obvio: `var resultado = Calcular();` obliga a quien lee a ir a ver qué devuelve `Calcular`.
* El tipo importa para la corrección: `decimal total = 0;` frente a `var total = 0;`, que es `int`.

**Usa `const` cuando:**

* El valor es fijo y conocido al compilar (días de la semana, límites de reglas de negocio que no cambian).
* Si el valor se calcula al ejecutar o puede cambiar entre versiones de una librería, usa `static readonly` (se ve en [Miembros estáticos](../04-poo/05-Miembros%20estaticos.md)).

-----

## Resumen en 5 líneas

1. Una variable se declara con `tipo nombre = valor;` y conserva ese tipo para siempre.
2. C# es de tipado estático y fuerte: los errores de tipo aparecen al compilar.
3. Tipos básicos: `int`, `double`, `decimal` (dinero), `bool`, `char`, `string`.
4. `var` infiere el tipo, pero no lo vuelve dinámico.
5. `const` define valores fijos conocidos al compilar; las locales deben asignarse antes de leerse.

-----

## Para profundizar

<details>
<summary>Tipos predefinidos y tipos definidos por el usuario</summary>

Los tipos se agrupan de dos maneras distintas, que conviene no mezclar:

* **Según quién los define:** *predefinidos* (los que trae el lenguaje: `int`, `bool`, `string`, `object`) y *definidos por el usuario* (los que creas tú con `class`, `struct`, `enum`, `interface`, `record`, `delegate`).
* **Según cómo se guardan y copian:** *tipos de valor* (`int`, `double`, `bool`, `char`, `struct`, `enum`) y *tipos de referencia* (`string`, `object`, `class`, `interface`, arrays, `delegate`).

Las dos clasificaciones se cruzan: `int` es predefinido y de valor; `string` es predefinido y de referencia; una `class` tuya es definida por el usuario y de referencia; un `struct` tuyo es definido por el usuario y de valor. La segunda clasificación se estudia en [Tipos de valor y de referencia](02-Tipos%20de%20valor%20y%20de%20referencia.md).

</details>

<details>
<summary>dynamic: el escape del tipado estático</summary>

`dynamic` desactiva la verificación de tipos al compilar: las operaciones se resuelven al ejecutar.

```csharp
dynamic x = "hola";
x = 10;              // compila
x.MetodoQueNoExiste(); // compila, pero falla al ejecutar (RuntimeBinderException)
```

Existe para interoperar con COM, lenguajes dinámicos o JSON sin esquema. En código normal, evítalo: pierdes justamente lo que hace seguro a C#.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una variable guarda un valor y en C# siempre tiene un tipo. C# es de tipado estático, así que el compilador verifica los tipos antes de ejecutar. Los tipos más usados son `int`, `double`, `decimal`, `bool`, `char` y `string`. `var` deja que el compilador infiera el tipo, pero la variable sigue siendo de tipo fijo.

### Respuesta ampliada (semi-senior)

C# es de tipado estático y fuerte: el tipo se fija al compilar y las conversiones con posible pérdida de datos son explícitas. Las palabras clave (`int`, `string`) son alias de tipos del CTS (`System.Int32`, `System.String`). `var` es inferencia en tiempo de compilación, no tipado dinámico; para eso existe `dynamic`, que se resuelve en ejecución y se evita salvo en interop. Para dinero se usa `decimal` por su representación en base 10. El compilador exige la asignación definitiva de las variables locales, mientras que los campos y los elementos de array se inicializan con `default`. Para valores fijos se usa `const` (valor incrustado en el código que la usa) o `static readonly` (valor resuelto en ejecución).

### Preguntas frecuentes de seguimiento

**1. ¿`var` hace que C# sea de tipado dinámico?**
No. El tipo se infiere al compilar y no cambia. El tipado dinámico es `dynamic`.

**2. ¿Qué diferencia hay entre `string` y `String`?**
Ninguna: `string` es un alias de `System.String`. Por convención se usa `string`.

**3. ¿Por qué `decimal` para dinero y no `double`?**
`double` usa base 2 y no puede representar exactamente valores como 0.1; `decimal` usa base 10 y sí. Se ve en [Números y operadores](04-Numeros%20y%20operadores.md).

**4. ¿Qué diferencia hay entre `const` y `readonly`?**
`const` se resuelve al compilar y solo admite tipos primitivos y `string`. `readonly` se asigna en ejecución (en la declaración o en el constructor) y admite cualquier tipo.

-----

## Práctica

**Ejercicio 1.** Elige el tipo más adecuado para cada dato y declara la variable:

1. La cantidad de alumnos de un curso.
2. El saldo de una cuenta bancaria.
3. Si un usuario está activo.
4. La inicial de un nombre.
5. La distancia de la Tierra a la Luna en metros (384.400.000).

<details>
<summary>Solución</summary>

```csharp
int cantidadAlumnos = 35;
decimal saldo = 1_250.75m;
bool estaActivo = true;
char inicial = 'C';
long distanciaTierraLuna = 384_400_000;   // cabe en int, pero long da margen si la cifra crece
```

La distancia cabe en `int` (máximo ~2.147 millones), así que `int` también es correcto. Lo importante es saber justificar la elección.

</details>

**Ejercicio 2.** Encuentra los errores de este código sin ejecutarlo:

```csharp
var nombre;
nombre = "Ana";
int edad = "30";
decimal precio = 19.99;
const int Max;
```

<details>
<summary>Solución</summary>

* `var nombre;` → CS0818: `var` debe inicializarse en la misma línea.
* `int edad = "30";` → CS0029: un `string` no se convierte implícitamente a `int`.
* `decimal precio = 19.99;` → CS0664: falta el sufijo `m`.
* `const int Max;` → CS0145: una constante debe tener un valor.

</details>

-----

## Siguiente lección

[Tipos de valor y de referencia](02-Tipos%20de%20valor%20y%20de%20referencia.md)
