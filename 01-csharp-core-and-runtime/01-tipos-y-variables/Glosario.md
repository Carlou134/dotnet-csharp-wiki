# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Alias de tipo.** Palabra clave de C# que representa un tipo de .NET: `int` es `System.Int32` y `string` es `System.String`.

**Asignación compuesta.** Abreviatura que opera y asigna a la vez: `x += 3` equivale a `x = x + 3`.

**Boxing.** Convertir un tipo de valor en un objeto del heap, por ejemplo al asignar un `int` a una variable `object`. Lo inverso es *unboxing*.

**Cast.** Sintaxis `(tipo)valor` que pide una conversión explícita: `(int)3.9`.

**`char`.** Tipo de valor que guarda un solo carácter UTF-16. Se escribe entre comillas simples: `'A'`.

**Clausura (*closure*).** Objeto oculto que crea el compilador para que una lambda pueda seguir usando las variables locales que captura.

**Comparación ordinal.** Comparar strings carácter por carácter según su código numérico, sin reglas de idioma. Es exacta y rápida.

**Concatenación.** Unir strings con el operador `+`.

**Constante (`const`).** Valor con nombre que no puede cambiar y que debe conocerse al compilar.

**Conversión explícita (restricción, *narrowing*).** Conversión que puede perder información y requiere un cast: `double` → `int`.

**Conversión implícita (ampliación, *widening*).** Conversión automática que no pierde información: `int` → `long`.

**Cultura.** Configuración regional que define cómo se escriben números, monedas y fechas (`3.5` frente a `3,5`). Se representa con `CultureInfo`.

**`decimal`.** Tipo numérico de 16 bytes y base 10, exacto para fracciones decimales. Es el indicado para dinero. Sus literales llevan el sufijo `m`.

**Declarar.** Anunciar una variable con su tipo y su nombre: `int edad;`.

**`default`.** Palabra clave que obtiene el valor por defecto de un tipo: `0`, `false`, `null`...

**Desbordamiento (*overflow*).** Situación en la que un valor no cabe en su tipo. Por defecto "da la vuelta" en silencio; con `checked`, lanza `OverflowException`.

**División entera.** División entre dos enteros, que descarta la parte decimal: `7 / 2` es `3`.

**`double`.** Tipo de punto flotante de 8 bytes, base 2. Rápido, pero no representa exactamente valores como `0.1`.

**Excepción.** Error que ocurre en tiempo de ejecución y, si nadie lo maneja, detiene el programa. Por ejemplo, `FormatException`.

**GC (Garbage Collector).** El recolector de basura: libera automáticamente los objetos del heap que ya no son alcanzables.

**Heap (montón).** Zona de memoria donde viven los objetos creados con `new` y los arrays.

**Índice.** Posición de un elemento dentro de un string o un array, empezando en 0. `^1` indica el último.

**Inicializar.** Asignar el primer valor a una variable.

**Inmutable.** Que no se puede modificar una vez creado. Los strings son inmutables.

**Interpolación.** Insertar valores dentro de un string con `$` y llaves: `$"Hola {nombre}"`.

**LIFO (*Last In, First Out*).** "Último en entrar, primero en salir". Así funciona el stack.

**Literal.** Valor escrito directamente en el código: `42`, `3.14`, `"hola"`, `true`.

**Literal verbatim.** String con `@` delante que no interpreta las secuencias de escape: `@"C:\ruta"`.

**Módulo (`%`).** Operador que devuelve el resto de una división entera: `7 % 3` es `1`.

**`NaN` (*Not a Number*).** Valor especial de `double` que representa un resultado indefinido, como `0.0 / 0.0` o `Math.Sqrt(-1)`.

**`null`.** Valor de una referencia que no apunta a ningún objeto.

**Operador.** Símbolo que realiza una operación: `+`, `==`, `&&`, `++`.

**`Parse`.** Método que convierte texto a otro tipo y lanza una excepción si el texto no es válido.

**Precedencia.** Orden en que se evalúan los operadores: `*` antes que `+`.

**Propiedad.** Dato que expone un objeto y se usa sin paréntesis, como `texto.Length`.

**Raw string literal.** String entre tres o más comillas dobles que se escribe exactamente como se verá, sin escapes. Desde C# 11.

**Redondeo bancario.** Redondear el punto medio al número par más cercano: 2.5 → 2, 3.5 → 4. Es el modo por defecto de `Math.Round`.

**Referencia.** Valor que identifica dónde está un objeto en memoria. Las variables de tipos de referencia guardan una.

**Secuencia de escape.** `\` seguido de un carácter con significado especial dentro de un string: `\n`, `\t`, `\"`.

**Stack (pila).** Zona de memoria donde se guardan las variables locales y los datos de cada llamada a método. Se libera sola al terminar el método.

**`string`.** Tipo de referencia inmutable que representa texto.

**`StringBuilder`.** Clase con un búfer modificable para construir texto de forma eficiente en muchos pasos.

**Subcadena (*substring*).** Una parte de un string.

**Tipado estático.** Los tipos se conocen y verifican al compilar.

**Tipado fuerte.** El lenguaje no mezcla tipos incompatibles sin una conversión explícita.

**Tipo de dato.** Define qué valores puede guardar una variable, cuánta memoria ocupa y qué operaciones admite.

**Tipo de referencia.** Tipo cuya variable guarda una referencia a un objeto: clases, `string`, arrays, interfaces. Al asignarlo se copia la referencia.

**Tipo de valor.** Tipo cuya variable contiene el dato: numéricos, `bool`, `char`, `struct`, `enum`. Al asignarlo se copia el dato.

**Truncar.** Cortar la parte decimal sin redondear: `(int)9.99` es `9`.

**`TryParse`.** Método que intenta convertir texto a otro tipo; devuelve `true` o `false` sin lanzar excepciones.

**Variable.** Nombre asociado a un espacio de memoria donde se guarda un valor.

**`var`.** Palabra clave que deja que el compilador infiera el tipo de una variable local. El tipo sigue siendo fijo.
