# Programación orientada a objetos

En esta carpeta aprendes a **crear tus propios tipos**: a definir clases con datos y comportamiento, a **proteger su estado** con modificadores de acceso y propiedades, a **construir objetos válidos** con constructores, a distinguir lo que pertenece a la clase (**estático**) de lo que pertenece a cada objeto, y a aplicar los pilares de la POO: **herencia**, **abstracción**, **interfaces** y **polimorfismo**. Cierra con la clase `object`, raíz de todos los tipos, y con el **encadenamiento de métodos**.

-----

## Antes de empezar

Conviene que ya tengas:

* Métodos, parámetros, valores de retorno y lambdas: [Métodos](../03-metodos/README.md).
* La diferencia entre tipos de valor y de referencia: [Tipos de valor y de referencia](../01-tipos-y-variables/02-Tipos%20de%20valor%20y%20de%20referencia.md). Es clave: todas las clases son tipos de referencia.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Clases y objetos](01-Clases%20y%20objetos.md) | Definir clases, instanciar con `new`, campos, métodos de instancia e inicializadores | Métodos |
| [2. Modificadores de acceso y encapsulación](02-Modificadores%20de%20acceso%20y%20encapsulacion.md) | `public`, `private`, `protected`, `internal`, accesos por defecto, invariantes y `readonly` | Clases y objetos |
| [3. Propiedades](03-Propiedades.md) | `get`/`set`, validación, autoimplementadas, `init`, `required`, calculadas y `field` | Modificadores de acceso |
| [4. Constructores y this](04-Constructores%20y%20this.md) | Constructores, validación, `this`, sobrecarga, `: this(...)`, orden de inicialización y constructores primarios | Propiedades |
| [5. Miembros estáticos](05-Miembros%20estaticos.md) | Campos, métodos, constructores y clases `static`; `const` frente a `static readonly`; `Main` explicado | Constructores |
| [6. Herencia](06-Herencia.md) | Clase base y derivada, `protected`, `base(...)`, `sealed` y composición frente a herencia | Miembros estáticos |
| [7. Virtual, override y clases abstractas](07-Virtual%20override%20y%20clases%20abstractas.md) | `virtual`/`override`, `new`, miembros y clases `abstract`, Template Method | Herencia |
| [8. Interfaces](08-Interfaces.md) | Contratos, implementación múltiple, implementación explícita, métodos por defecto y comparación con clases abstractas | Virtual y abstract |
| [9. Polimorfismo y casting](09-Polimorfismo%20y%20casting.md) | Tipo estático frente a dinámico, upcasting, downcasting, `is`, `as` y pattern matching | Interfaces |
| [10. La clase Object](10-La%20clase%20Object.md) | `ToString`, `Equals`, `GetHashCode`, `GetType`, `==` frente a `Equals` y boxing | Polimorfismo |
| [11. Encadenamiento de métodos](11-Encadenamiento%20de%20metodos.md) | `return this`, APIs fluidas, encadenamiento inmutable y patrón Builder | Constructores y this |

-----

## El mapa completo en una mirada

```
Clase           ->  class Bosque { campos, propiedades, constructores, métodos }
Objeto          ->  var b = new Bosque("Amazonas");      (tipo de referencia)

Encapsular      ->  campos private + operaciones public que protegen las reglas
Propiedades     ->  { get; set; }  { get; private set; }  { get; init; }  => calculada
Constructor     ->  Bosque(string nombre) { ... }   : this(...)   : base(...)
static          ->  pertenece a la clase (una sola copia): Bosque.Cantidad, Math.Max

Los 4 pilares:
Abstracción     ->  abstract class Figura { abstract double Area(); }
Encapsulación   ->  private + propiedades + validación
Herencia        ->  class Circulo : Figura          (una sola clase base)
Polimorfismo    ->  Figura f = new Circulo(); f.Area();   → ejecuta la versión de Circulo

Contratos       ->  interface IFigura { double Area(); }   (muchas por clase)
Casting         ->  upcast implícito · downcast (T)x · x is T t · x as T
object          ->  ToString() · Equals() + GetHashCode() · GetType()
Fluido          ->  new Builder().A().B().Construir()      (return this)
```

-----

## Cómo leer los fragmentos de código

Los bloques de las secciones "Cómo funciona" son **fragmentos**: muestran una idea y a veces omiten partes. Los de "Ejemplo completo" y "Práctica" son **programas completos** que compilan en un `Program.cs`.

Si pegas un fragmento en `Program.cs` y obtienes `error CS8803: Top-level statements must precede namespace and type declarations`, mueve las declaraciones de `class`/`interface` **debajo** de las sentencias sueltas (o a su propio archivo `.cs`). En un `Program.cs` con top-level statements, primero va el código que se ejecuta y después los tipos.

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con código chico y explicado.
5. **Ejemplo completo:** todo junto en un programa que compila.
6. **Errores comunes:** lo que suele salir mal, por qué pasa, cómo se arregla y el código de error del compilador.
7. **Según la versión de C#:** qué cambió entre versiones, para reconocer código antiguo y moderno.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento frecuentes.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya sabes modelar un dominio con clases, interfaces y polimorfismo. El siguiente paso es elegir el tipo correcto para cada dato (enums, structs, records, nulos y genéricos): [Tipos avanzados](../05-tipos-avanzados/README.md).
