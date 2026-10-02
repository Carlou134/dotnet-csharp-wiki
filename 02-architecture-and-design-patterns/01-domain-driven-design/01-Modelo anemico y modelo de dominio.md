# Modelo anémico y modelo de dominio

## En una frase

Domain-Driven Design (DDD) propone que el código **modele el negocio**: en lugar de clases que solo guardan datos (modelo **anémico**) y servicios que hacen todo, las clases del dominio **tienen comportamiento**, protegen sus propias reglas y usan el **mismo lenguaje** que la gente del negocio.

-----

## Antes de empezar

Conviene que ya sepas:

* Encapsulación, propiedades con `private set` y constructores que validan: [POO](../../01-csharp-core-and-runtime/04-poo/README.md).
* Excepciones: [Lanzar y crear excepciones](../../01-csharp-core-and-runtime/08-excepciones/02-Lanzar%20y%20crear%20excepciones.md).
* Records e igualdad por valor: [Structs y records](../../01-csharp-core-and-runtime/05-tipos-avanzados/02-Structs%20y%20records.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Dominio:** el área del negocio que resuelve el software (ventas, logística, matrículas).
* **DDD (*Domain-Driven Design*):** enfoque de diseño que pone el modelo del dominio en el centro del software.
* **Modelo anémico:** clases con solo propiedades públicas, sin comportamiento ni reglas.
* **Modelo rico (de dominio):** clases que combinan datos y comportamiento y protegen sus reglas.
* **Invariante:** regla que el modelo debe cumplir siempre ("un pedido confirmado no puede quedar vacío").
* **Lenguaje ubicuo (*ubiquitous language*):** vocabulario común entre expertos del negocio y desarrolladores, usado tal cual en el código.
* **Bloques tácticos:** los patrones de código de DDD: entidades, value objects, agregados, servicios de dominio, repositorios.

-----

## El problema

Un sistema de pedidos típico, después de un par de años:

```csharp
public class Pedido
{
    public int Id { get; set; }
    public string Estado { get; set; } = "";
    public decimal Total { get; set; }
    public List<LineaPedido> Lineas { get; set; } = new();
}
```

Y la lógica, repartida en servicios:

```csharp
// En PedidoService
pedido.Lineas.Add(linea);
pedido.Total += linea.Precio * linea.Cantidad;

// En CarritoController (otro desarrollador, otro día)
pedido.Lineas.Add(otraLinea);                 // ¡se olvidó de actualizar el Total!

// En ReporteService
pedido.Estado = "Enviado";                    // ¿y si el pedido no estaba pagado?

// En cualquier lugar
pedido.Total = -50;                           // nada lo impide
```

Este es el **modelo anémico**: `Pedido` es una bolsa de datos. Las reglas ("el total es la suma de las líneas", "no se envía sin pagar") existen solo en la cabeza de algunos desarrolladores y en `if` dispersos por diez servicios. Cada lugar que toca un pedido puede dejarlo **inconsistente**, y nadie lo detecta hasta que un cliente recibe una factura con un total incorrecto.

-----

## Cómo funciona

### El modelo rico: datos + comportamiento + reglas

```csharp
public class Pedido
{
    private readonly List<LineaPedido> _lineas = new();

    public Guid Id { get; }
    public EstadoPedido Estado { get; private set; } = EstadoPedido.Borrador;
    public IReadOnlyList<LineaPedido> Lineas => _lineas.AsReadOnly();
    public decimal Total => _lineas.Sum(l => l.Subtotal);     // siempre coherente: se calcula

    public Pedido(Guid id) => Id = id;

    public void AgregarLinea(string producto, decimal precio, int cantidad)
    {
        if (Estado != EstadoPedido.Borrador)
            throw new InvalidOperationException("Solo se pueden agregar líneas a un pedido en borrador.");
        _lineas.Add(new LineaPedido(producto, precio, cantidad));
    }

    public void Confirmar()
    {
        if (_lineas.Count == 0)
            throw new InvalidOperationException("No se puede confirmar un pedido vacío.");
        Estado = EstadoPedido.Confirmado;
    }

    public void Enviar()
    {
        if (Estado != EstadoPedido.Pagado)
            throw new InvalidOperationException("Solo se envían pedidos pagados.");
        Estado = EstadoPedido.Enviado;
    }

    // Pagar(), Cancelar()...
}
```

Ahora:

* `pedido.Total = -50` **no compila**: `Total` se calcula, no se asigna.
* `pedido.Lineas.Add(...)` **no compila**: la lista se expone de solo lectura.
* `pedido.Estado = ...` **no compila**: el estado solo cambia con operaciones que validan (`Confirmar`, `Enviar`).
* Las reglas viven en **un solo lugar**: la clase `Pedido`. Ningún servicio puede saltárselas.

Es la encapsulación de [POO](../../01-csharp-core-and-runtime/04-poo/02-Modificadores%20de%20acceso%20y%20encapsulacion.md) llevada al centro del diseño: **el objeto protege sus invariantes**.

### Lenguaje ubicuo: el código habla como el negocio

Si en las reuniones el negocio dice "el cliente **confirma** el pedido" y "el almacén **despacha**", el código debe decir `pedido.Confirmar()` y `pedido.Despachar()`, no `pedido.SetEstado(2)` ni `UpdateOrderStatus(id, "DSP")`.

| El negocio dice | Código anémico | Código con lenguaje ubicuo |
| --- | --- | --- |
| "Agregar un producto al pedido" | `pedido.Lineas.Add(new Linea { ... })` | `pedido.AgregarLinea(producto, precio, cantidad)` |
| "Confirmar el pedido" | `pedido.Estado = 2;` | `pedido.Confirmar();` |
| "Aplicar un cupón" | `pedido.Total -= pedido.Total * 0.1m;` | `pedido.AplicarCupon(cupon);` |

Cuando el código usa las mismas palabras, un experto del negocio puede leer los nombres de los métodos y detectar reglas mal entendidas, y los desarrolladores no tienen que "traducir" en cada conversación.

### Los bloques tácticos de DDD

DDD tiene dos partes:

* **Estratégica:** cómo dividir un sistema grande en partes (*bounded contexts*) y cómo se relacionan. Es tema de arquitectura.
* **Táctica:** cómo escribir el modelo dentro de cada parte. Es lo que ves en este módulo:

| Bloque | Qué es | Lección |
| --- | --- | --- |
| **Entidad** | Objeto con identidad propia que cambia en el tiempo (un pedido, un cliente) | [Entidades](02-Entidades.md) |
| **Value object** | Valor sin identidad, inmutable, comparado por su contenido (dinero, email, dirección) | [Value objects](03-Value%20objects.md) |
| **Agregado** | Grupo de entidades y value objects que cambian juntos, con una raíz que controla el acceso | [Agregados](04-Agregados.md) |
| Servicio de dominio | Lógica del negocio que no pertenece a una sola entidad | Mencionado en [Agregados](04-Agregados.md) |
| Repositorio | Abstracción para guardar y recuperar agregados | Mencionado en [Agregados](04-Agregados.md) |

### El dominio no depende de la tecnología

Una idea clave: las clases del dominio **no saben** de bases de datos, HTTP ni frameworks. `Pedido` no hereda de una clase de Entity Framework, no tiene atributos `[Table]` ni llama a `SaveChanges`. Así:

* Las reglas se prueban con pruebas unitarias simples, sin base de datos.
* Cambiar de base de datos o de framework no toca el corazón del negocio.

Es la base de arquitecturas como **Clean Architecture** y **Hexagonal**, donde el dominio está en el centro y todo lo demás depende de él (y no al revés).

-----

## Ejemplo completo

```csharp
var pedido = new Pedido(Guid.NewGuid());
pedido.AgregarLinea("Teclado", 150m, 2);
pedido.AgregarLinea("Mouse", 60m, 1);
Console.WriteLine($"Total: {pedido.Total:N2} ({pedido.Lineas.Count} líneas, {pedido.Estado})");

Intentar("Enviar sin pagar", () => pedido.Enviar());

pedido.Confirmar();
pedido.Pagar();
pedido.Enviar();
Console.WriteLine($"Estado final: {pedido.Estado}");

Intentar("Agregar línea a un pedido enviado", () => pedido.AgregarLinea("Monitor", 800m, 1));
Intentar("Línea con cantidad 0", () => new Pedido(Guid.NewGuid()).AgregarLinea("Cable", 10m, 0));

static void Intentar(string accion, Action operacion)
{
    try
    {
        operacion();
        Console.WriteLine($"✔ {accion}");
    }
    catch (Exception ex) when (ex is InvalidOperationException or ArgumentException)
    {
        Console.WriteLine($"✘ {accion}: {ex.Message}");
    }
}

enum EstadoPedido { Borrador, Confirmado, Pagado, Enviado, Cancelado }

record LineaPedido
{
    public string Producto { get; }
    public decimal PrecioUnitario { get; }
    public int Cantidad { get; }
    public decimal Subtotal => PrecioUnitario * Cantidad;

    public LineaPedido(string producto, decimal precioUnitario, int cantidad)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(producto);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(precioUnitario);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(cantidad);
        (Producto, PrecioUnitario, Cantidad) = (producto, precioUnitario, cantidad);
    }
}

class Pedido
{
    private readonly List<LineaPedido> _lineas = new();

    public Guid Id { get; }
    public EstadoPedido Estado { get; private set; } = EstadoPedido.Borrador;
    public IReadOnlyList<LineaPedido> Lineas => _lineas.AsReadOnly();
    public decimal Total => _lineas.Sum(l => l.Subtotal);

    public Pedido(Guid id) => Id = id;

    public void AgregarLinea(string producto, decimal precio, int cantidad)
    {
        ExigirEstado(EstadoPedido.Borrador, "Solo se pueden agregar líneas a un pedido en borrador.");
        _lineas.Add(new LineaPedido(producto, precio, cantidad));
    }

    public void Confirmar()
    {
        ExigirEstado(EstadoPedido.Borrador, "Solo se confirma un pedido en borrador.");
        if (_lineas.Count == 0) throw new InvalidOperationException("No se puede confirmar un pedido vacío.");
        Estado = EstadoPedido.Confirmado;
    }

    public void Pagar()
    {
        ExigirEstado(EstadoPedido.Confirmado, "Solo se paga un pedido confirmado.");
        Estado = EstadoPedido.Pagado;
    }

    public void Enviar()
    {
        ExigirEstado(EstadoPedido.Pagado, "Solo se envían pedidos pagados.");
        Estado = EstadoPedido.Enviado;
    }

    private void ExigirEstado(EstadoPedido esperado, string mensaje)
    {
        if (Estado != esperado) throw new InvalidOperationException(mensaje);
    }
}
```

Salida:

```text
Total: 360.00 (2 líneas, Borrador)
✘ Enviar sin pagar: Solo se envían pedidos pagados.
Estado final: Enviado
✘ Agregar línea a un pedido enviado: Solo se pueden agregar líneas a un pedido en borrador.
✘ Línea con cantidad 0: cantidad ('0') must be a non-negative and non-zero value. (Parameter 'cantidad')
Actual value was 0.
```

Todas las reglas del ciclo de vida del pedido están en `Pedido`. No hay forma de "saltarse" un paso desde afuera.

-----

## Errores comunes

**1. Setters públicos "por comodidad".**
Qué pasa: cualquier parte del sistema cambia el estado sin pasar por las reglas.
Por qué: `{ get; set; }` es lo primero que sale al escribir una clase.
Arreglo: `private set` (o solo `get`) y métodos con nombres del negocio que validan.

**2. Exponer la colección interna.**
Qué pasa: `pedido.Lineas.Add(...)` o `Clear()` desde afuera, sin actualizar el total ni validar.
Por qué: devolver `List<T>` entrega el control de la lista.
Arreglo: lista privada y `IReadOnlyList<T>` con `AsReadOnly()`; las modificaciones, con métodos (`AgregarLinea`).

**3. Guardar datos derivados que se desincronizan.**
Qué pasa: un `Total` almacenado que no coincide con la suma de las líneas.
Por qué: alguien modificó las líneas sin recalcular.
Arreglo: calcula los valores derivados (`Total => _lineas.Sum(...)`), o actualízalos dentro de los mismos métodos que cambian las líneas.

**4. Nombres técnicos en lugar del lenguaje del negocio.**
Qué pasa: `UpdateStatus(2)`, `ProcessData()`, `Manager`; el negocio no puede validar el modelo y los desarrolladores se confunden.
Por qué: se piensa en la base de datos o en la tecnología, no en el dominio.
Arreglo: nombres que el negocio usa (`Confirmar`, `Despachar`, `AplicarCupon`).

**5. Aplicar DDD a un CRUD simple.**
Qué pasa: entidades, value objects y agregados para un formulario que solo guarda y lista datos.
Por qué: se usa DDD por moda.
Arreglo: DDD paga cuando hay **reglas de negocio complejas**; para un CRUD, un modelo simple es mejor (KISS).

-----

## Según la versión de C#

DDD es independiente del lenguaje, pero C# moderno lo facilita mucho:

* **C# 9:** `record` (value objects con igualdad por valor en una línea) e `init`.
* **C# 10:** `record struct`.
* **C# 11:** `required`, para garantizar datos obligatorios al crear objetos.
* **.NET 8:** `ArgumentOutOfRangeException.ThrowIfNegativeOrZero` y compañía, para validar con una línea.
* **EF Core 8:** *complex types* para mapear value objects sin trucos.

-----

## Cuándo sí y cuándo no

**Usa un modelo rico (DDD táctico) cuando:**

* El dominio tiene reglas que cambian, estados, validaciones cruzadas y lógica que hoy está dispersa.
* El software es el corazón del negocio (facturación, logística, seguros, matrículas).

**Usa un modelo simple cuando:**

* Es un CRUD sin reglas: formularios de mantenimiento de tablas, catálogos simples.
* Es un prototipo o una herramienta interna pequeña.

Un mismo sistema puede tener las dos cosas: modelo rico en el núcleo del negocio y CRUD simple en los módulos de soporte.

-----

## Resumen en 5 líneas

1. Modelo anémico: solo datos con setters públicos y la lógica dispersa en servicios; las reglas se rompen con facilidad.
2. Modelo rico: las clases del dominio tienen comportamiento y protegen sus invariantes.
3. Setters privados, colecciones de solo lectura y métodos con nombres del negocio que validan.
4. Lenguaje ubicuo: el código usa las mismas palabras que el negocio.
5. DDD táctico = entidades, value objects y agregados; paga en dominios con reglas complejas, no en un CRUD.

-----

## Para profundizar

<details>
<summary>DDD estratégico: bounded contexts</summary>

En un sistema grande, la misma palabra significa cosas distintas: "Producto" en Ventas tiene precio y descuentos; en Almacén tiene peso, ubicación y stock. Intentar un único modelo `Producto` para todo produce una clase gigante y contradictoria. DDD propone dividir el sistema en **bounded contexts** (contextos delimitados), cada uno con su propio modelo y su propio lenguaje, que se comunican por contratos. Esa división suele coincidir con equipos y, a veces, con servicios desplegables (de ahí la relación con los microservicios, aunque DDD no los exige).

</details>

<details>
<summary>Lecturas de referencia</summary>

* *Domain-Driven Design*, de Eric Evans (2003): el libro original, conocido como "el libro azul".
* *Implementing Domain-Driven Design*, de Vaughn Vernon: más práctico, con mucho código.
* *Domain-Driven Design Distilled*, de Vaughn Vernon: una introducción corta.
* La guía de Microsoft [*.NET Microservices: Architecture for Containerized .NET Applications*](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/), con implementaciones en C#.

</details>

-----

## En entrevista

### Respuesta corta (junior)

DDD es un enfoque de diseño que pone el negocio en el centro del código. En lugar de clases que solo tienen propiedades (modelo anémico), las clases del dominio tienen métodos que aplican las reglas del negocio y protegen su estado. Además, el código usa el mismo vocabulario que los expertos del negocio, lo que se llama lenguaje ubicuo.

### Respuesta ampliada (semi-senior)

DDD tiene una parte estratégica (bounded contexts, mapas de contexto, lenguaje ubicuo por contexto) y una táctica (entidades, value objects, agregados, servicios de dominio, eventos de dominio, repositorios). El modelo anémico, señalado por Fowler como antipatrón, separa datos y comportamiento y obliga a replicar las invariantes en los servicios; el modelo rico las encapsula con setters privados, colecciones de solo lectura y operaciones con nombres del negocio. El dominio es independiente de la infraestructura, lo que habilita Clean o Hexagonal Architecture y pruebas unitarias sin base de datos. DDD se justifica en dominios complejos; en módulos CRUD agrega costo sin beneficio.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es un modelo anémico y por qué se considera un antipatrón?**
Clases con solo datos y lógica en servicios externos. Las reglas quedan dispersas y cualquier código puede dejar los objetos en estados inválidos.

**2. ¿Qué es el lenguaje ubicuo?**
El vocabulario compartido entre el negocio y los desarrolladores, usado tal cual en clases, métodos y conversaciones.

**3. ¿Siempre conviene usar DDD?**
No. Conviene en dominios con lógica de negocio compleja; para CRUDs simples agrega complejidad innecesaria.

-----

## Práctica

**Ejercicio 1.** Convierte esta clase anémica en una con comportamiento: el saldo no puede quedar negativo, solo cambia con `Depositar` y `Retirar`, y una cuenta bloqueada no permite retiros.

```csharp
public class CuentaBancaria
{
    public string Numero { get; set; } = "";
    public decimal Saldo { get; set; }
    public bool Bloqueada { get; set; }
}
```

<details>
<summary>Solución</summary>

```csharp
var cuenta = new CuentaBancaria("001-123");
cuenta.Depositar(500);
cuenta.Retirar(120);
Console.WriteLine(cuenta.Saldo);   // 380

cuenta.Bloquear();
try { cuenta.Retirar(10); }
catch (InvalidOperationException ex) { Console.WriteLine(ex.Message); }   // La cuenta está bloqueada.

class CuentaBancaria
{
    public string Numero { get; }
    public decimal Saldo { get; private set; }
    public bool Bloqueada { get; private set; }

    public CuentaBancaria(string numero)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(numero);
        Numero = numero;
    }

    public void Depositar(decimal monto)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(monto);
        Saldo += monto;
    }

    public void Retirar(decimal monto)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(monto);
        if (Bloqueada) throw new InvalidOperationException("La cuenta está bloqueada.");
        if (monto > Saldo) throw new InvalidOperationException("Saldo insuficiente.");
        Saldo -= monto;
    }

    public void Bloquear() => Bloqueada = true;
    public void Desbloquear() => Bloqueada = false;
}
```

</details>

**Ejercicio 2.** Un experto del negocio describe: "Un alumno se **matricula** en un curso si hay **vacantes**; puede **retirarse** antes del inicio de clases; una vez iniciadas, solo puede **abandonar**, y queda registrado." Escribe solo las firmas de los métodos de una clase `Curso` usando el lenguaje ubicuo.

<details>
<summary>Solución</summary>

```csharp
class Curso
{
    public int VacantesDisponibles { get; }
    public bool ClasesIniciadas { get; }

    public void Matricular(Alumno alumno) { /* exige vacantes */ }
    public void Retirar(Alumno alumno) { /* solo antes del inicio */ }
    public void RegistrarAbandono(Alumno alumno) { /* solo después del inicio */ }
    public void IniciarClases() { /* cambia el estado */ }
}
```

Los nombres salen de la conversación con el negocio (`Matricular`, `Retirar`, `RegistrarAbandono`), no de la base de datos (`InsertInscripcion`, `UpdateEstado`).

</details>

-----

## Siguiente lección

[Entidades](02-Entidades.md)
