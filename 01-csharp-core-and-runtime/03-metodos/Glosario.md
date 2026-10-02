# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**`Action<...>`.** Delegado de .NET para métodos que no devuelven nada. `Action<string>` recibe un `string`.

**Ámbito (*scope*).** Zona del código donde un nombre existe y se puede usar. Los parámetros y las variables locales tienen como ámbito su método.

**Argumento.** Valor concreto que se pasa al llamar a un método. En `Saludar("Ana")`, el argumento es `"Ana"`.

**Argumento con nombre.** Argumento que indica a qué parámetro va: `Configurar(d: 4)`. Permite cambiar el orden y saltar opcionales.

**Argumento posicional.** Argumento que se asigna por su posición en la llamada.

**Captura.** Cuando una lambda usa variables del método donde fue creada. Se captura la variable, no una copia de su valor.

**Clausura (*closure*).** Una función junto con las variables que captura. El compilador la implementa con una clase oculta.

**Cuerpo.** Código entre llaves que se ejecuta al llamar a un método.

**Cuerpo de expresión (*expression-bodied*).** Forma corta de un método de una sola expresión: `static int Doble(int x) => x * 2;`.

**Delegado.** Tipo que representa un método con cierta firma. Permite guardar métodos en variables y pasarlos como argumentos.

**Deconstrucción.** Separar una tupla en variables individuales: `var (min, max) = ObtenerExtremos(...)`.

**Descarte (`_`).** Variable de usar y tirar para un valor que no interesa: `int.TryParse(s, out _)`.

**Firma.** El nombre de un método más los tipos (y modificadores) de sus parámetros. No incluye el tipo de retorno.

**Función de orden superior.** Método que recibe o devuelve otro método.

**Función local.** Método declarado dentro de otro método. Con top-level statements, los métodos de `Program.cs` son funciones locales y no admiten sobrecarga.

**`Func<...>`.** Delegado de .NET para métodos que devuelven un valor. El último tipo es el de retorno: `Func<int, string>` recibe un `int` y devuelve un `string`.

**`in`.** Modificador de parámetro: se pasa por referencia, pero el método no puede modificarlo.

**Lambda (expresión lambda).** Método anónimo escrito donde se usa: `x => x * 2`.

**Llamar (invocar).** Ejecutar un método escribiendo su nombre seguido de paréntesis.

**Método.** Bloque de código con nombre que realiza una tarea. En C#, toda función pertenece a un tipo.

**Método anónimo.** Método sin nombre. Las lambdas son la forma moderna de escribirlos.

**`out`.** Modificador de parámetro: el método debe asignarlo y el llamador recibe el valor. Es la base del patrón `TryParse`.

**`params`.** Modificador del último parámetro que permite pasar una cantidad variable de argumentos.

**Parámetro.** Variable declarada en la definición de un método que recibe el valor de un argumento.

**Parámetro opcional.** Parámetro con un valor por defecto constante; el argumento puede omitirse. Va al final de la lista.

**`Predicate<T>`.** Delegado de .NET para métodos que reciben un `T` y devuelven `bool`.

**`ref`.** Modificador de parámetro: el método recibe la variable del llamador y puede modificarla.

**Resolución de sobrecarga.** Proceso por el que el compilador elige qué sobrecarga llamar según los argumentos.

**`return`.** Sentencia que termina un método y, si corresponde, devuelve un valor.

**Sobrecarga (*overload*).** Cada una de las versiones de un método con el mismo nombre y distintos parámetros.

**Tipo de retorno.** El tipo del valor que devuelve un método, escrito antes de su nombre. `void` si no devuelve nada.

**Tupla.** Grupo de valores, posiblemente de tipos distintos, tratado como una unidad: `(int Min, int Max)`. Es un tipo de valor mutable.

**Valor por defecto.** Valor que toma un parámetro opcional cuando no se pasa el argumento.

**`void`.** Tipo de retorno que indica que el método no devuelve nada.
