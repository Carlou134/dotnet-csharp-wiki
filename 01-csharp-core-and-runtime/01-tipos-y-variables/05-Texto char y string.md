# Texto: char y string

## En una frase

`char` guarda **un** carácter (`'A'`) y `string` guarda una secuencia **inmutable** de caracteres (`"Hola"`); se escriben con literales normales, *verbatim* (`@"..."`) o *raw* (`"""..."""`) y se combinan con concatenación (`+`) o, mejor, con interpolación (`$"..."`).

-----

## Antes de empezar

Conviene que ya sepas:

* Declarar variables y qué es un tipo de referencia, de [Tipos de valor y de referencia](02-Tipos%20de%20valor%20y%20de%20referencia.md).
* Las secuencias de escape básicas, vistas en [Tu primer programa](../00-introduccion/03-Tu%20primer%20programa.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`char`:** tipo de valor que guarda un solo carácter UTF-16. Se escribe entre comillas simples.
* **`string`:** tipo de referencia que guarda una secuencia de `char`. Se escribe entre comillas dobles.
* **Inmutable:** que no puede modificarse después de creado.
* **Secuencia de escape:** `\` seguido de un carácter con significado especial (`\n`, `\"`).
* **Literal verbatim:** string con `@` delante, que no interpreta las secuencias de escape.
* **Raw string literal:** string entre tres o más comillas dobles, que se escribe tal cual.
* **Concatenación:** unir strings con `+`.
* **Interpolación:** insertar valores en un string con `$` y llaves `{}`.

-----

## El problema

Casi todo programa trabaja con texto: mensajes al usuario, nombres, rutas de archivos, JSON, consultas SQL. Y el texto trae complicaciones:

* ¿Cómo escribes comillas **dentro** de un texto entre comillas?
* ¿Cómo escribes una ruta de Windows (`C:\Usuarios\Ana`) si `\` es especial?
* ¿Cómo armas el mensaje `"Hola Ana, tienes 3 mensajes."` con datos que cambian, sin pelearte con espacios y signos `+`?

C# tiene una herramienta concreta para cada caso.

-----

## Cómo funciona

### `char`: un carácter

```csharp
char letra = 'A';
char simbolo = '?';
char enie = 'ñ';
char salto = '\n';         // un char también puede ser una secuencia de escape

Console.WriteLine(char.IsDigit('7'));     // True
Console.WriteLine(char.IsLetter('7'));    // False
Console.WriteLine(char.ToUpper('a'));     // A
Console.WriteLine((int)'A');              // 65 (su código numérico)
```

Comillas **simples** para `char`, **dobles** para `string`. `'Hola'` no compila.

### `string`: una secuencia de caracteres

```csharp
string nombre = "Ana";
string vacio = "";                 // string vacío (longitud 0)
string vacio2 = string.Empty;      // equivalente, más explícito
string? sinValor = null;           // no es lo mismo que vacío: no hay string

char[] letras = { 'H', 'o', 'l', 'a' };
string desdeArray = new string(letras);   // "Hola"
string repetido = new string('-', 10);    // "----------"
```

Puedes leer cada carácter por su posición (empieza en 0):

```csharp
string palabra = "radio";
char primera = palabra[0];     // 'r'
char ultima = palabra[^1];     // 'o' (^1 = el primero desde el final)
int largo = palabra.Length;    // 5
```

### Los strings son inmutables

Un string **no se puede modificar**. Todos los métodos que parecen cambiarlo devuelven uno **nuevo**:

```csharp
string s = "hola";
s[0] = 'H';                  // error CS0200: la propiedad o el indizador es de solo lectura
s.ToUpper();                 // crea "HOLA"... y se pierde, porque no lo guardaste
s = s.ToUpper();             // ahora s apunta al nuevo string "HOLA"
```

¿Por qué inmutable? Porque así un string se puede compartir entre muchas variables o hilos sin que nadie lo cambie "por debajo", y .NET puede reutilizar literales iguales. El costo es que concatenar muchas veces en un bucle crea muchos strings intermedios; la solución es `StringBuilder`, que se ve en [Métodos de string y StringBuilder](06-Metodos%20de%20string%20y%20StringBuilder.md).

### Secuencias de escape

| Secuencia | Significado |
| --- | --- |
| `\'` | Comilla simple |
| `\"` | Comilla doble |
| `\\` | Barra invertida |
| `\n` | Nueva línea |
| `\r` | Retorno de carro (vuelve al inicio de la línea) |
| `\t` | Tabulación horizontal |
| `\0` | Carácter nulo |
| `\e` | Escape (ESC), para colores en terminal (C# 13) |
| `\uNNNN` | Carácter Unicode por su código hexadecimal de 4 dígitos (`\u00F1` = ñ) |
| `\U00NNNNNN` | Carácter Unicode de 8 dígitos (incluye emojis) |

```csharp
string cita = "Ella dijo: \"Hola\"";
string ruta = "C:\\Usuarios\\Ana\\Documentos";
string tabla = "Nombre\tEdad\nAna\t30";
string enie = "Espa\u00F1a";            // España
```

En Windows, un salto de línea "completo" en archivos de texto es `\r\n`. Si necesitas el salto correcto para el sistema operativo actual, usa `Environment.NewLine`.

### Literales verbatim: `@"..."`

Con `@` delante, las barras invertidas son **literales** (no escapan nada) y el texto puede ocupar varias líneas. Para escribir una comilla doble se duplica: `""`.

```csharp
string ruta = @"C:\Usuarios\Ana\Documentos";

string poema = @"Primera línea
Segunda línea";

string cita = @"Ella dijo: ""Hola""";
```

Son ideales para rutas de Windows y expresiones regulares.

### Raw string literals: `"""..."""`

Desde C# 11, el texto entre tres (o más) comillas dobles se escribe **exactamente** como se verá, sin escapar nada:

```csharp
string json = """
    {
      "nombre": "Ana",
      "ruta": "C:\Usuarios\Ana"
    }
    """;
```

Reglas:

* En varias líneas, las comillas de apertura y de cierre van en líneas propias.
* La **indentación de las comillas de cierre** se elimina de todas las líneas. Eso te permite indentar el bloque junto al código sin que los espacios pasen al resultado.
* Si el texto contiene `"""`, abre y cierra con cuatro comillas `""""`.

```csharp
string unaLinea = """Usa "comillas" sin escapar""";
```

Son ideales para JSON, SQL, HTML o cualquier texto con muchas comillas.

### Concatenación con `+`

```csharp
string nombre = "Ana";
int mensajes = 3;

string saludo = "Hola " + nombre + ", tienes " + mensajes + " mensajes.";
```

Si uno de los operandos es `string`, el otro se convierte a texto automáticamente (`mensajes` → `"3"`). El problema es la legibilidad: es fácil olvidar un espacio o un signo.

Cuidado con el orden de evaluación, que es de izquierda a derecha:

```csharp
Console.WriteLine(1 + 2 + "3");    // "33": primero 1 + 2 = 3 (números), luego 3 + "3"
Console.WriteLine("1" + 2 + 3);    // "123": desde el principio es concatenación
```

### Interpolación con `$`

La forma moderna y recomendada:

```csharp
string saludo = $"Hola {nombre}, tienes {mensajes} mensajes.";
```

Entre llaves puede ir **cualquier expresión**:

```csharp
int a = 5, b = 3;
Console.WriteLine($"{a} + {b} = {a + b}");                // 5 + 3 = 8
Console.WriteLine($"Mayúsculas: {nombre.ToUpper()}");     // Mayúsculas: ANA
Console.WriteLine($"¿Mayor de edad? {(edad >= 18 ? "Sí" : "No")}");   // el ternario va entre paréntesis
```

Formato y alineación dentro de las llaves:

```csharp
decimal precio = 1234.5m;
double porcentaje = 0.256;
DateTime fecha = new DateTime(2026, 10, 2);

Console.WriteLine($"{precio:N2}");         // 1,234.50  (número con 2 decimales)
Console.WriteLine($"{precio:C}");          // $1,234.50 (moneda, según la cultura)
Console.WriteLine($"{porcentaje:P1}");     // 25.6%
Console.WriteLine($"{fecha:dd/MM/yyyy}");  // 02/10/2026
Console.WriteLine($"[{nombre,10}]");       // [       Ana] (alineado a la derecha, 10 caracteres)
Console.WriteLine($"[{nombre,-10}]");      // [Ana       ] (alineado a la izquierda)
```

Para escribir una llave literal dentro de un string interpolado, duplícala: `$"{{literal}}"` produce `{literal}`.

Se pueden combinar prefijos: `$@"C:\Usuarios\{nombre}"` (interpolado y verbatim) y `$"""..."""` (interpolado y raw).

### Formato compuesto: `string.Format`

Antes de la interpolación (C# 6), se usaban marcadores numerados. Lo verás en código antiguo y en métodos de logging:

```csharp
string mensaje = string.Format("Hola {0}, tienes {1} mensajes.", nombre, mensajes);
Console.WriteLine("Hola {0}, tienes {1} mensajes.", nombre, mensajes);   // WriteLine también lo acepta
```

`{0}` es el primer argumento después del texto, `{1}` el segundo, y así sucesivamente.

### Comparar strings

```csharp
string a = "hola";
string b = "HOLA";

Console.WriteLine(a == b);                                              // False (distingue mayúsculas)
Console.WriteLine(a.Equals(b, StringComparison.OrdinalIgnoreCase));     // True
Console.WriteLine(string.IsNullOrEmpty(""));                            // True
Console.WriteLine(string.IsNullOrWhiteSpace("   "));                    // True
```

`==` compara el **contenido**, aunque `string` sea un tipo de referencia (ver [Tipos de valor y de referencia](02-Tipos%20de%20valor%20y%20de%20referencia.md)).

-----

## Ejemplo completo

```csharp
string cliente = "Ana Torres";
int cantidad = 3;
decimal precioUnitario = 49.90m;
DateTime fecha = new DateTime(2026, 10, 2);
string carpeta = @"C:\Facturas\2026";

decimal total = cantidad * precioUnitario;

string factura = $"""
    ===== FACTURA =====
    Cliente : {cliente}
    Fecha   : {fecha:dd/MM/yyyy}
    Detalle : {cantidad} x {precioUnitario:N2}
    Total   : {total:N2}
    Archivo : {carpeta}\{cliente.Replace(' ', '_')}.txt
    ===================
    """;

Console.WriteLine(factura);
```

Salida (con cultura en inglés; los separadores varían según la configuración regional):

```text
===== FACTURA =====
Cliente : Ana Torres
Fecha   : 02/10/2026
Detalle : 3 x 49.90
Total   : 149.70
Archivo : C:\Facturas\2026\Ana_Torres.txt
===================
```

Un raw string interpolado (`$"""`) permite escribir la barra invertida sin escaparla e insertar valores con formato.

-----

## Errores comunes

**1. Comillas simples para un texto.**
Qué pasa: `error CS1012: Too many characters in character literal`.
Por qué: `'...'` es un `char`, y solo admite un carácter.
Arreglo: comillas dobles: `"Hola"`.

**2. Barra invertida sin escapar en una ruta.**
Qué pasa: `error CS1009: Unrecognized escape sequence` (por ejemplo `"C:\Users"`, porque `\U` espera un código Unicode).
Por qué: `\` inicia una secuencia de escape.
Arreglo: `"C:\\Users"` o `@"C:\Users"`.

**3. Espacio entre `$` y las comillas.**
Qué pasa: error de compilación.
Por qué: `$` y `"` deben ir juntos: `$"..."`.
Arreglo: quita el espacio.

**4. Olvidar el `$`.**
Qué pasa: se imprime literalmente `Hola {nombre}`.
Por qué: sin `$`, las llaves son texto común.
Arreglo: `$"Hola {nombre}"`.

**5. Creer que un método modifica el string.**
Qué pasa: `texto.Trim();` no cambia nada.
Por qué: los strings son inmutables; el método devuelve uno nuevo.
Arreglo: `texto = texto.Trim();`.

**6. Comparar sin considerar mayúsculas.**
Qué pasa: `"Admin" == "admin"` da `False` y un login o un filtro falla.
Por qué: `==` es sensible a mayúsculas y minúsculas.
Arreglo: `string.Equals(a, b, StringComparison.OrdinalIgnoreCase)`.

-----

## Según la versión de C#

* **C# 6:** interpolación de strings con `$"..."`.
* **C# 8:** `$@"..."` y `@$"..."` en cualquier orden (antes solo `$@`). Índices desde el final (`s[^1]`).
* **C# 10:** strings interpolados constantes (`const string Ruta = $"{Base}/api";`) si todo lo que interpolan son constantes `string`.
* **C# 11:** raw string literals (`"""..."""`) y saltos de línea dentro de las llaves de interpolación.
* **C# 13:** secuencia de escape `\e`.

-----

## Cuándo sí y cuándo no

| Necesitas... | Usa |
| --- | --- |
| Un texto simple | `"..."` |
| Insertar valores | `$"... {valor} ..."` |
| Rutas de Windows o expresiones regulares | `@"..."` |
| JSON, SQL, HTML o texto con muchas comillas | `"""..."""` |
| Unir dos o tres piezas fijas | `+` es aceptable |
| Construir texto dentro de un bucle | `StringBuilder` (siguiente lección) |
| Logging estructurado (`ILogger`) | Plantillas con marcadores, no interpolación: `logger.LogInformation("Usuario {Id}", id)` |

-----

## Resumen en 5 líneas

1. `char` = un carácter entre comillas simples; `string` = texto entre comillas dobles.
2. Los strings son inmutables: los métodos devuelven strings nuevos.
3. `\` escapa caracteres; `@"..."` los toma literalmente; `"""..."""` escribe el texto tal cual.
4. La interpolación `$"Hola {nombre}"` es la forma recomendada de combinar texto y valores.
5. `==` compara el contenido de los strings y distingue mayúsculas de minúsculas.

-----

## Para profundizar

<details>
<summary>Unicode, UTF-16 y por qué Length puede sorprenderte</summary>

Un `char` de .NET es una unidad UTF-16 de 16 bits. La mayoría de los caracteres ocupa un `char`, pero los emojis y algunos caracteres especiales ocupan **dos** (un *par sustituto*):

```csharp
string emoji = "👍";
Console.WriteLine(emoji.Length);   // 2
```

Si necesitas contar lo que una persona percibe como "letras" (por ejemplo, para limitar la longitud de un nombre), usa `StringInfo.LengthInTextElements` o recorre los `Rune`.

</details>

<details>
<summary>Interning: por qué dos literales iguales son el mismo objeto</summary>

El runtime guarda una sola copia de cada literal de string (el *intern pool*). Por eso:

```csharp
string a = "hola";
string b = "hola";
Console.WriteLine(ReferenceEquals(a, b));   // True: misma instancia
```

Los strings construidos en ejecución (concatenando, leyendo de la consola) no se internan automáticamente. Es otra razón para comparar siempre por contenido (`==` o `Equals`) y nunca por referencia.

</details>

-----

## En entrevista

### Respuesta corta (junior)

`char` guarda un solo carácter y `string` una cadena de caracteres. Los strings son inmutables: cada método devuelve uno nuevo. Para combinar texto y variables se usa la interpolación `$"..."`. Con `@` se escriben rutas sin escapar las barras, y con `"""` (raw strings) se escribe texto con comillas sin escaparlas.

### Respuesta ampliada (semi-senior)

`string` es un tipo de referencia inmutable con semántica de valor en `==` y `Equals`. La inmutabilidad permite compartirlo de forma segura entre hilos e internar los literales, pero hace costosa la concatenación repetida, que se resuelve con `StringBuilder`. La interpolación se compila a `DefaultInterpolatedStringHandler` desde C# 10, lo que evita boxing y asignaciones intermedias. Los raw string literals de C# 11 eliminan el escape y gestionan la indentación según las comillas de cierre. Las comparaciones deben indicar `StringComparison` explícitamente: `Ordinal` para identificadores y claves, y `CurrentCulture` para texto que ve el usuario.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué los strings son inmutables?**
Para que sean seguros de compartir entre variables e hilos, para poder internarlos y para que sirvan como claves de diccionario sin cambiar su hash.

**2. ¿Diferencia entre `""`, `string.Empty` y `null`?**
`""` y `string.Empty` son el mismo string vacío (longitud 0). `null` significa que no hay ningún string; acceder a `.Length` lanza `NullReferenceException`.

**3. ¿Interpolación o `string.Format`?**
Interpolación por legibilidad y rendimiento. `string.Format` solo si la plantilla viene de afuera (por ejemplo, de un archivo de recursos para traducción).

-----

## Práctica

**Ejercicio 1.** Escribe estas tres cadenas de tres formas: con escapes, verbatim y raw.

* `C:\Proyectos\Wiki`
* `Él dijo "listo"`

<details>
<summary>Solución</summary>

```csharp
// Con escapes
string r1 = "C:\\Proyectos\\Wiki";
string c1 = "Él dijo \"listo\"";

// Verbatim
string r2 = @"C:\Proyectos\Wiki";
string c2 = @"Él dijo ""listo""";

// Raw
string r3 = """C:\Proyectos\Wiki""";
string c3 = """
    Él dijo "listo"
    """;
```

En `c3`, el texto termina en `"`. En una sola línea, esa comilla quedaría pegada al cierre `"""` y el compilador no sabría dónde termina el literal. Por eso se usa la forma de varias líneas, donde el cierre va en su propia línea.

</details>

**Ejercicio 2.** Con `nombre = "Luis"` y `puntos = 87.456`, imprime usando interpolación: `Jugador: Luis | Puntos: 87.46 |` con el nombre alineado a la izquierda en 8 caracteres.

<details>
<summary>Solución</summary>

```csharp
string nombre = "Luis";
double puntos = 87.456;
Console.WriteLine($"Jugador: {nombre,-8}| Puntos: {puntos:F2} |");
// Jugador: Luis    | Puntos: 87.46 |
```

</details>

-----

## Siguiente lección

[Métodos de string y StringBuilder](06-Metodos%20de%20string%20y%20StringBuilder.md)
