# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Abstracción.** Pilar de la POO: modelar solo lo esencial de un concepto y ocultar los detalles. Las clases abstractas y las interfaces son sus herramientas.

**`abstract` (miembro).** Miembro sin implementación que toda clase derivada concreta debe implementar con `override`.

**Accesor (`get` / `set` / `init`).** Bloques de una propiedad que se ejecutan al leerla (`get`), al asignarla (`set`) o al asignarla solo durante la creación (`init`).

**API fluida (*fluent API*).** API diseñada para encadenar llamadas y leerse como una frase: `builder.A().B().C()`.

**`as`.** Operador que intenta un cast y devuelve `null` si el objeto no es de ese tipo.

**`base`.** Palabra clave para llamar al constructor (`: base(...)`) o a los miembros (`base.Metodo()`) de la clase base.

**Boxing.** Envolver un tipo de valor en un objeto del heap para tratarlo como `object`.

**Builder (patrón).** Objeto auxiliar que construye otro paso a paso y lo entrega con un método final, como `Construir()`.

**Campo.** Variable que pertenece a una clase. Cada objeto tiene su propia copia (salvo que sea `static`).

**Campo de respaldo (*backing field*).** Campo privado donde una propiedad guarda realmente su dato.

**Casting.** Tratar una referencia como otro tipo de su jerarquía: hacia arriba (upcasting) o hacia abajo (downcasting).

**Clase.** Definición de un tipo propio con datos (campos, propiedades) y comportamiento (métodos).

**Clase abstracta.** Clase marcada `abstract`: no se puede instanciar y sirve como base. Puede tener estado, constructores y métodos concretos.

**Clase base (superclase, padre).** Clase de la que hereda otra.

**Clase derivada (subclase, hija).** Clase que hereda de otra.

**Clase estática.** Clase que solo tiene miembros estáticos y no se puede instanciar, como `Math` o `Console`.

**Composición.** Construir una clase usando objetos de otras como campos ("tiene un"), en lugar de heredar.

**Constructor.** Método especial con el nombre de la clase y sin tipo de retorno que inicializa el objeto al hacer `new`.

**Constructor estático.** Constructor marcado `static` que se ejecuta una sola vez, antes del primer uso de la clase.

**Constructor primario.** Parámetros declarados junto al nombre de la clase: `class Bosque(string nombre)`. Desde C# 12.

**Despacho dinámico.** Decidir al ejecutar qué implementación de un método virtual se llama, según el tipo real del objeto.

**Downcasting.** Tratar una referencia de tipo base como un tipo derivado. Es explícito y puede fallar (`InvalidCastException`).

**Encadenamiento de métodos.** Llamar a un método sobre el resultado del anterior en una sola expresión.

**Encapsulación.** Pilar de la POO: ocultar el estado interno y exponer operaciones que mantienen al objeto siempre válido.

**Ensamblado (*assembly*).** El `.dll` o `.exe` de un proyecto. Es la unidad de visibilidad de `internal`.

**`Equals`.** Método de `object` que indica si dos objetos son iguales. Por defecto, en clases, compara referencias.

**Estado.** Los valores de los campos de un objeto en un momento dado.

**`GetHashCode`.** Método de `object` que devuelve un número usado por diccionarios y conjuntos. Objetos iguales deben tener el mismo hash.

**Herencia.** Pilar de la POO: una clase obtiene los miembros de otra con `class Derivada : Base`.

**Implementación explícita.** Implementar un miembro de interfaz de forma que solo sea visible a través de esa interfaz: `void IInterfaz.Metodo()`.

**Inicializador de objeto.** Sintaxis para asignar propiedades al crear un objeto: `new Bosque { Nombre = "x" }`.

**Instancia (objeto).** Ejemplar concreto creado a partir de una clase.

**Instanciar.** Crear un objeto con `new`.

**Interfaz.** Contrato que declara miembros que una clase se compromete a implementar. Una clase puede implementar muchas.

**`internal`.** Accesible desde cualquier código del mismo ensamblado (proyecto).

**Invariante.** Regla que el estado de un objeto debe cumplir siempre, como "el saldo nunca es negativo".

**`is`.** Operador que comprueba si un objeto es compatible con un tipo; con patrón (`is Perro p`) también lo convierte.

**Método de instancia.** Método que trabaja con los datos de un objeto concreto y se llama sobre él.

**Miembro.** Cualquier elemento declarado dentro de una clase: campos, propiedades, métodos, constructores, eventos.

**Miembro estático.** Miembro que pertenece a la clase y no a los objetos; hay una sola copia.

**Modificador de acceso.** Palabra clave que define desde dónde se puede usar un tipo o miembro: `public`, `private`, `protected`, `internal`...

**`new` (ocultación).** En una derivada, declara un miembro que oculta al de la base sin sobrescribirlo. Se decide por el tipo de la variable.

**`object` (`System.Object`).** La clase base de todos los tipos de .NET.

**`override`.** Marca un miembro que reemplaza a uno `virtual` o `abstract` de la clase base.

**Polimorfismo.** Pilar de la POO: la misma llamada se comporta distinto según el tipo real del objeto.

**`private`.** Accesible solo dentro de la clase que lo declara.

**Propiedad.** Miembro que se usa como un campo, pero ejecuta código al leerse o asignarse.

**Propiedad autoimplementada.** Propiedad cuyo campo de respaldo genera el compilador: `{ get; set; }`.

**Propiedad calculada.** Propiedad que calcula su valor a partir de otros datos cada vez que se lee: `public double Area => Ancho * Alto;`.

**`protected`.** Accesible en la clase y en sus clases derivadas.

**`public`.** Accesible desde cualquier lugar.

**`readonly`.** Campo que solo se puede asignar en su declaración o en el constructor.

**Record.** Tipo pensado para datos, con igualdad por valor, `ToString` y `with` generados automáticamente.

**`required`.** Modificador que obliga a asignar una propiedad al crear el objeto. Desde C# 11.

**`sealed`.** Impide heredar de una clase o seguir sobrescribiendo un miembro.

**`static`.** Indica que un miembro pertenece a la clase y no a cada objeto.

**`this`.** Referencia al objeto sobre el que se está ejecutando el código. `: this(...)` encadena constructores.

**Tipo dinámico (real).** El tipo del objeto al que apunta una variable, conocido al ejecutar.

**Tipo estático (declarado).** El tipo de la variable, conocido al compilar. Decide qué miembros se pueden usar.

**`ToString`.** Método de `object` que devuelve una representación en texto. Lo usan `Console.WriteLine` y la interpolación.

**Upcasting.** Tratar un objeto derivado como su tipo base. Es implícito y siempre seguro.

**`value`.** Palabra clave que, dentro de un `set`, representa el valor que se está asignando.

**`virtual`.** Marca un miembro de la base que las derivadas pueden sobrescribir con `override`.
