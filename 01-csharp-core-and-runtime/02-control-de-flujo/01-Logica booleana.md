# Lógica booleana

## En una frase

El tipo `bool` solo vale `true` o `false`; los **operadores de comparación** (`==`, `<`, `>=`...) producen un `bool` y los **operadores lógicos** (`&&`, `||`, `!`, `^`) combinan varios `bool` en uno, que después decide qué camino sigue el programa.

-----

## Antes de empezar

Conviene que ya sepas:

* Declarar variables y operar con números, de [Números y operadores](../01-tipos-y-variables/04-Numeros%20y%20operadores.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Booleano (`bool`):** tipo con solo dos valores: `true` y `false`.
* **Expresión booleana:** cualquier expresión que produce un `bool`, como `edad >= 18`.
* **Operador de comparación:** compara dos valores y devuelve `bool`.
* **Operador lógico:** combina valores booleanos (Y, O, NO, O exclusivo).
* **Tabla de verdad:** tabla que muestra el resultado de un operador lógico para cada combinación de entradas.
* **Cortocircuito:** cuando el resultado ya se conoce con el primer operando, el segundo no se evalúa.

-----

## El problema

Un programa útil toma decisiones: "si el usuario es mayor de edad **y** aceptó los términos, crea la cuenta"; "si la contraseña está vacía **o** tiene menos de 8 caracteres, muestra un error". Antes de poder decidir, necesitas una forma de **expresar preguntas de sí o no** y combinarlas.

Para eso existe la lógica booleana, la base de todas las decisiones del código.

-----

## Cómo funciona

### El tipo `bool`

```csharp
bool estaActivo = true;
bool tienePermiso = false;

Console.WriteLine(estaActivo);   // True
```

`true` y `false` son palabras clave, **no** strings: `"true"` es un texto de 4 letras. Tampoco son números: a diferencia de C o JavaScript, en C# `if (1)` no compila.

### Operadores de comparación

| Operador | Significa | Ejemplo | Resultado |
| --- | --- | --- | --- |
| `==` | Igual a | `5 == 5` | `true` |
| `!=` | Distinto de | `5 != 3` | `true` |
| `<` | Menor que | `3 < 75` | `true` |
| `>` | Mayor que | `3 > 75` | `false` |
| `<=` | Menor o igual | `5 <= 5` | `true` |
| `>=` | Mayor o igual | `4 >= 5` | `false` |

```csharp
int edad = 20;
bool esMayor = edad >= 18;               // true
bool esAna = "Ana" == "Ana";             // true (los strings se comparan por contenido)
bool iguales = true == false;            // false
```

`=` **asigna**; `==` **compara**. Confundirlos es uno de los errores más comunes al empezar.

### Operadores lógicos

| Operador | Nombre | Devuelve `true` si... |
| --- | --- | --- |
| `&&` | Y (AND) | **ambos** son `true` |
| `\|\|` | O (OR) | **al menos uno** es `true` |
| `!` | NO (NOT) | el operando es `false` (invierte el valor) |
| `^` | O exclusivo (XOR) | son **distintos** entre sí |

### Tabla de verdad

| A | B | `A && B` | `A \|\| B` | `A ^ B` | `!A` |
| --- | --- | --- | --- | --- | --- |
| `true` | `true` | `true` | `true` | `false` | `false` |
| `true` | `false` | `false` | `true` | `true` | `false` |
| `false` | `true` | `false` | `true` | `true` | `true` |
| `false` | `false` | `false` | `false` | `false` | `true` |

```csharp
bool y = (4 > 1) && (2 < 7);     // true && true   → true
bool o = (8 > 6) || (3 > 6);     // true || false  → true
bool no = !(1 < 3);              // !true          → false
bool xor = (5 > 1) ^ (2 > 1);    // true ^ true    → false
```

### Evaluar expresiones combinadas paso a paso

```csharp
bool respuesta = (9 < 3) || (100 < 45);
bool otra = ((3439 > 40) && (1 < 3)) || respuesta;
```

1. `(9 < 3) || (100 < 45)` → `false || false` → `false`. `respuesta` es `false`.
2. `((3439 > 40) && (1 < 3))` → `(true && true)` → `true`.
3. `true || respuesta` → `true || false` → `true`. `otra` es `true`.

Precedencia: `!` primero, luego las comparaciones, luego `&&` y al final `||`. Así, `a || b && c` se evalúa como `a || (b && c)`. Usa paréntesis para que la intención sea evidente.

### Cortocircuito: `&&` y `||` no siempre evalúan los dos lados

* En `A && B`, si `A` es `false`, el resultado ya es `false`: **`B` no se evalúa**.
* En `A || B`, si `A` es `true`, el resultado ya es `true`: **`B` no se evalúa**.

Esto es muy útil para protegerse de errores:

```csharp
string? nombre = null;

if (nombre != null && nombre.Length > 3)   // si nombre es null, nunca se llama a .Length
{
    Console.WriteLine("Nombre largo");
}
```

Si invirtieras el orden (`nombre.Length > 3 && nombre != null`), obtendrías un `NullReferenceException`. **El orden importa.**

Existen `&` y `|` sin duplicar, que con `bool` evalúan **siempre** los dos lados. Casi nunca son lo que quieres en una condición.

### Leyes de De Morgan: negar una condición compuesta

Para negar una condición con `&&` u `||`, se niega cada parte y se cambia el operador:

```text
!(A && B)  ==  !A || !B
!(A || B)  ==  !A && !B
```

```csharp
// "No es (mayor de edad y con permiso)" equivale a "es menor o no tiene permiso"
bool bloqueado = !(edad >= 18 && tienePermiso);
bool bloqueado2 = edad < 18 || !tienePermiso;   // misma lógica, más fácil de leer
```

### Patrones relacionales (C# 9+)

Para comparar una variable con varios valores, existen patrones más legibles:

```csharp
int temperatura = 22;

bool agradable = temperatura is >= 18 and <= 26;     // en lugar de temperatura >= 18 && temperatura <= 26
bool extremo = temperatura is < 0 or > 40;
bool noCero = temperatura is not 0;
```

`and`, `or` y `not` solo funcionan dentro de un patrón (después de `is`); fuera de él se siguen usando `&&`, `||` y `!`.

-----

## Ejemplo completo

```csharp
Console.Write("Edad: ");
int.TryParse(Console.ReadLine(), out int edad);

Console.Write("¿Aceptas los términos? (s/n): ");
bool aceptaTerminos = Console.ReadLine()?.Trim().ToLower() == "s";

Console.Write("Contraseña: ");
string contrasena = Console.ReadLine() ?? "";

bool esMayor = edad >= 18;
bool contrasenaValida = contrasena.Length >= 8 && contrasena.Any(char.IsDigit);
bool puedeRegistrarse = esMayor && aceptaTerminos && contrasenaValida;

Console.WriteLine($"Mayor de edad: {esMayor}");
Console.WriteLine($"Aceptó términos: {aceptaTerminos}");
Console.WriteLine($"Contraseña válida: {contrasenaValida}");
Console.WriteLine($"Registro permitido: {puedeRegistrarse}");
Console.WriteLine($"Rango de edad típico de estudiante: {edad is >= 17 and <= 25}");
```

Ejecución de ejemplo:

```text
Edad: 20
¿Aceptas los términos? (s/n): s
Contraseña: secreto123
Mayor de edad: True
Aceptó términos: True
Contraseña válida: True
Registro permitido: True
Rango de edad típico de estudiante: True
```

Guardar cada condición en una variable con nombre (`esMayor`, `contrasenaValida`) hace que la condición final se lea casi como una frase. `Any(char.IsDigit)` es de LINQ: comprueba si algún carácter es un dígito.

-----

## Errores comunes

**1. Usar `=` en lugar de `==`.**
Qué pasa: `if (edad = 18)` da `error CS0029: Cannot implicitly convert type 'int' to 'bool'`.
Por qué: `=` asigna; la condición de un `if` debe ser `bool`.
Arreglo: `if (edad == 18)`.

**2. Comparar con el string `"true"`.**
Qué pasa: `error CS0019: Operator '==' cannot be applied to operands of type 'bool' and 'string'`.
Por qué: `true` (bool) y `"true"` (string) son tipos distintos.
Arreglo: `if (activo)` o `if (activo == true)`.

**3. Usar un número como condición.**
Qué pasa: `if (cantidad)` da `error CS0029: Cannot implicitly convert type 'int' to 'bool'`.
Por qué: en C#, los números no se convierten a `bool` (en C y JavaScript sí).
Arreglo: `if (cantidad != 0)`.

**4. Ordenar mal las condiciones con cortocircuito.**
Qué pasa: `NullReferenceException` en `texto.Length > 0 && texto != null`.
Por qué: se evalúa de izquierda a derecha; `.Length` se ejecuta antes de comprobar `null`.
Arreglo: pon primero la comprobación que protege: `texto != null && texto.Length > 0`, o usa `!string.IsNullOrEmpty(texto)`.

**5. Escribir rangos como en matemáticas.**
Qué pasa: `18 <= edad <= 65` da `error CS0019: Operator '<=' cannot be applied to operands of type 'bool' and 'int'`.
Por qué: primero se evalúa `18 <= edad` (un `bool`) y luego se intenta comparar ese `bool` con 65.
Arreglo: `edad >= 18 && edad <= 65` o `edad is >= 18 and <= 65`.

-----

## Según la versión de C#

* **C# 7:** `is` con patrones de tipo y constantes (`x is null`).
* **C# 9:** patrones relacionales (`is >= 18`) y lógicos (`and`, `or`, `not`): `x is not null`, `n is > 0 and < 10`.

-----

## Cuándo sí y cuándo no

**Extrae condiciones a variables con nombre cuando:**

* La condición tiene más de dos partes. `if (puedeRegistrarse)` se entiende mejor que una línea con cinco operadores.

**Usa patrones (`is >= ... and <= ...`) cuando:**

* Comparas **una misma variable** contra un rango o varios valores.

**No uses `&` / `|` con `bool` cuando:**

* Quieres una condición normal: pierdes el cortocircuito. Solo tienen sentido si el lado derecho **debe** ejecutarse siempre (y eso suele indicar un diseño confuso).

-----

## Resumen en 5 líneas

1. `bool` solo vale `true` o `false`; no es un número ni un string.
2. Comparación: `==`, `!=`, `<`, `>`, `<=`, `>=`. `=` asigna, `==` compara.
3. Lógicos: `&&` (ambos), `||` (alguno), `!` (invierte), `^` (distintos).
4. `&&` y `||` hacen cortocircuito: ordena las condiciones para que la primera proteja a la segunda.
5. Desde C# 9: `x is >= 1 and <= 10`, `x is not null`.

-----

## Para profundizar

<details>
<summary>bool? y la lógica de tres valores</summary>

`bool?` puede valer `true`, `false` o `null` (desconocido). Con `&` y `|`, C# aplica lógica de tres valores: `null & false` es `false` (ya se sabe que es falso) y `null | true` es `true`. Con `&&` y `||`, en cambio, no compila con `bool?`. Para una condición, convierte primero: `if (acepta == true)` o `if (acepta ?? false)`.

</details>

<details>
<summary>Operadores de bits</summary>

Con enteros, `&`, `|`, `^` y `~` operan **bit a bit**, y `<<` y `>>` desplazan bits. Se usan con banderas (`[Flags] enum`), máscaras y protocolos binarios:

```csharp
int permisos = 0b0101;           // leer (1) + ejecutar (4)
bool puedeLeer = (permisos & 0b0001) != 0;   // true
```

No los necesitas para la lógica cotidiana.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`bool` representa verdadero o falso. Los operadores de comparación (`==`, `<`, `>=`...) devuelven un `bool`, y los lógicos (`&&`, `||`, `!`) los combinan. `&&` y `||` hacen cortocircuito: si con el primer operando ya se conoce el resultado, el segundo no se evalúa.

### Respuesta ampliada (semi-senior)

C# no convierte implícitamente enteros ni referencias a `bool`, lo que elimina errores típicos de C como `if (x = 0)`. `&&` y `||` hacen cortocircuito y se aprovechan como guardas (`x != null && x.Valido`); `&` y `|` con `bool` evalúan ambos lados y con `bool?` implementan lógica de tres valores. Desde C# 9, los patrones relacionales y lógicos (`is >= 0 and < 10`, `is not null`) expresan rangos y negaciones de forma más legible y, además, sirven en expresiones `switch`. Para condiciones complejas conviene extraer variables o métodos con nombre y aplicar De Morgan para simplificar las negaciones.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `&&` y `&`?**
`&&` hace cortocircuito; `&` evalúa siempre ambos lados. Con enteros, `&` es el AND bit a bit.

**2. ¿Qué es el cortocircuito y para qué sirve?**
No evaluar el segundo operando si el primero ya determina el resultado. Sirve para evitar errores (`obj != null && obj.Prop`) y trabajo innecesario.

**3. ¿Por qué `x is not null` y no `x != null`?**
`is not null` no puede ser alterado por una sobrecarga del operador `!=` y expresa la intención con claridad.

-----

## Práctica

**Ejercicio 1.** Sin ejecutar, ¿qué valor tiene cada variable?

```csharp
int a = 5, b = 10;
bool r1 = a > 3 && b < 5;
bool r2 = a > 3 || b < 5;
bool r3 = !(a == 5) ^ (b == 10);
bool r4 = a is > 0 and < 10;
```

<details>
<summary>Solución</summary>

* `r1` = `true && false` = `false`.
* `r2` = `true || false` = `true`.
* `r3` = `!true ^ true` = `false ^ true` = `true`.
* `r4` = `true` (5 está entre 0 y 10).

</details>

**Ejercicio 2.** Un año es bisiesto si es divisible por 4, **excepto** los divisibles por 100, **salvo** que también sean divisibles por 400. Escribe la expresión booleana `esBisiesto` para una variable `int anio`.

<details>
<summary>Solución</summary>

```csharp
bool esBisiesto = (anio % 4 == 0 && anio % 100 != 0) || anio % 400 == 0;
```

Pruebas: 2024 → `true`, 1900 → `false`, 2000 → `true`, 2026 → `false`.

</details>

-----

## Siguiente lección

[Condicionales](02-Condicionales.md)
