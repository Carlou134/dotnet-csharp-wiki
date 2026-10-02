# Principios DRY, KISS y refactoring

## En una frase

**DRY** (no te repitas) pide que cada conocimiento del sistema viva en un solo lugar, **KISS** (mantenlo simple) y **YAGNI** (no lo vas a necesitar) piden no complicar el diseño antes de tiempo, el **bajo acoplamiento** permite cambiar una parte sin romper las demás, y el **refactoring** es la práctica continua de mejorar el código sin cambiar lo que hace, para pagar la **deuda técnica**.

-----

## Antes de empezar

Conviene que ya sepas:

* Nombres, comentarios y code smells, de [Nombres, comentarios y code smells](01-Nombres%20comentarios%20y%20code%20smells.md).
* Interfaces e inyección de dependencias, de [Interfaces](../04-poo/08-Interfaces.md).
* Excepciones, de [Excepciones](../08-excepciones/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **DRY (*Don't Repeat Yourself*):** cada pieza de conocimiento debe tener una única representación en el sistema.
* **KISS (*Keep It Simple, Stupid*):** la solución más simple que resuelve el problema suele ser la mejor.
* **YAGNI (*You Aren't Gonna Need It*):** no construyas lo que todavía no necesitas.
* **Sobreingeniería:** diseño más complejo que el problema que resuelve.
* **Acoplamiento:** cuánto depende una parte del código de los detalles de otra.
* **Cohesión:** cuánto se relacionan entre sí las responsabilidades de una misma clase o módulo.
* **Deuda técnica:** el costo futuro de los atajos tomados hoy.
* **Refactoring:** cambiar la estructura interna del código sin cambiar su comportamiento observable.
* **Code review:** revisión del código por otra persona antes de integrarlo.

-----

## El problema

Un sistema de facturación crece durante dos años. Hoy:

* La regla "el IGV es 18 %" aparece en 14 archivos. El gobierno la cambia y alguien actualiza 13. El reporte del archivo 14 sale mal durante tres meses.
* Para guardar un cliente hay una `FabricaAbstractaDeRepositoriosGenericosConfigurable` con cinco capas de herencia "por si algún día cambiamos de base de datos". Nadie se anima a tocarla.
* La clase `ServicioFacturas` crea por dentro su conexión a la base de datos, su cliente de correo y su logger. Para probarla hay que levantar una base de datos real.
* Cada cambio tarda el triple de lo que debería, porque nadie entiende del todo cómo funciona y hay miedo de romper algo.

Nada de esto es un error de sintaxis. Son decisiones de diseño que, acumuladas, hacen que el software sea **caro de cambiar**. Los principios de esta lección son guías para evitarlo.

-----

## Cómo funciona

### DRY: no te repitas

```csharp
// ❌ La regla del impuesto está duplicada
decimal totalFactura = subtotal * 1.18m;
// ... en otro archivo
decimal totalCotizacion = subtotal + subtotal * 0.18m;
// ... en otro
decimal impuestoReporte = ventas * 18 / 100;

// ✅ Una sola fuente de verdad
public static class Impuestos
{
    public const decimal TasaIgv = 0.18m;
    public static decimal AplicarIgv(decimal monto) => monto * (1 + TasaIgv);
}
```

DRY no es solo "no copiar y pegar código": es que **cada conocimiento** (una regla de negocio, un cálculo, una validación, un formato) tenga un único lugar. Cuando la regla cambie, se cambia en un sitio.

Pero cuidado: **no toda similitud es duplicación**. Dos fragmentos que hoy se parecen pero representan conceptos distintos pueden evolucionar por separado:

```csharp
bool EsMayorDeEdad(int edad) => edad >= 18;
bool PuedeAbrirCuenta(int edad) => edad >= 18;
```

Hoy son iguales; mañana el banco puede exigir 21 para abrir una cuenta. Si las unificaste en un solo método, ese cambio obligará a separarlas de nuevo (o, peor, a llenar el método de `if`). Una regla útil es la **regla de tres**: la primera vez lo escribes, la segunda lo duplicas con cuidado, la tercera extraes la abstracción. **Duplicar es más barato que una abstracción equivocada.**

### KISS: mantenlo simple

```csharp
// ❌ Sobreingeniería para un caso simple
public interface IEstrategiaDeSaludo { string Saludar(string n); }
public class EstrategiaSaludoFormal : IEstrategiaDeSaludo { /* ... */ }
public class FabricaDeEstrategiasDeSaludo { /* ... */ }
// ... 4 archivos para imprimir "Hola, Ana"

// ✅
string Saludar(string nombre) => $"Hola, {nombre}";
```

La complejidad tiene un costo: más código que leer, probar y mantener. Usa patrones y capas cuando el problema los justifica (varias implementaciones reales, reglas que cambian seguido), no por si acaso. "Simple" no significa "rápido y sucio": significa **sin partes innecesarias**.

### YAGNI: no lo vas a necesitar

"Agreguemos soporte para varias monedas, por si acaso". "Hagamos el repositorio genérico para cualquier base de datos, por si algún día migramos". La mayoría de esos "por si acaso" nunca llegan, y mientras tanto el código es más complejo y lento de cambiar. Construye lo que el requisito actual pide, **bien diseñado** para que agregar lo próximo sea fácil cuando de verdad haga falta.

### Bajo acoplamiento y alta cohesión

**Acoplamiento:** cuánto sabe una clase de los detalles de otra. **Cohesión:** cuánto se relaciona lo que hace una misma clase.

```csharp
// ❌ Alto acoplamiento: la clase crea y conoce sus dependencias concretas
public class ServicioFacturas
{
    private readonly SqlRepositorioFacturas _repo = new SqlRepositorioFacturas("Server=...");
    private readonly ClienteSmtp _correo = new ClienteSmtp("smtp.empresa.com");

    public void Emitir(Factura f) { _repo.Guardar(f); _correo.Enviar(f.Cliente.Email, "Factura emitida"); }
}

// ✅ Bajo acoplamiento: depende de abstracciones que recibe desde afuera
public class ServicioFacturas
{
    private readonly IRepositorioFacturas _repo;
    private readonly INotificador _notificador;

    public ServicioFacturas(IRepositorioFacturas repo, INotificador notificador)
    {
        _repo = repo;
        _notificador = notificador;
    }

    public void Emitir(Factura f) { _repo.Guardar(f); _notificador.Notificar(f.Cliente, "Factura emitida"); }
}
```

Con la segunda versión puedes cambiar SQL por otra base de datos, o el correo por SMS, sin tocar `ServicioFacturas`, y probarla con implementaciones falsas en memoria. Es el **principio de inversión de dependencias** (la D de SOLID), y la base de la inyección de dependencias de ASP.NET Core.

Alta cohesión significa que una clase tiene **una** razón para cambiar (el **principio de responsabilidad única**, la S de SOLID): `ServicioFacturas` emite facturas; no formatea PDFs ni calcula sueldos.

### Deuda técnica

Cada atajo ("lo arreglo después", "copio esto y listo", "sin pruebas, que no hay tiempo") es como un préstamo: te deja avanzar rápido hoy, pero genera **intereses**: cada cambio futuro cuesta un poco más. La deuda puede estar en el código, en la arquitectura, en la seguridad, en las pruebas o en las dependencias desactualizadas.

Tomar deuda a veces es una decisión válida (llegar a una fecha de lanzamiento), siempre que sea **consciente** y se pague. La deuda que nadie registra ni paga termina haciendo que cada cambio sea lento y riesgoso.

### Refactoring

Refactorizar es **cambiar la estructura interna sin cambiar el comportamiento**. No es reescribir desde cero ni agregar funcionalidades: es renombrar, extraer métodos, eliminar duplicación, simplificar condiciones, mover responsabilidades.

Cómo se hace bien:

1. **Con red de seguridad:** pruebas automatizadas que verifiquen que el comportamiento no cambió.
2. **En pasos pequeños:** un cambio, compilar, ejecutar las pruebas, repetir. Si algo se rompe, sabes exactamente qué fue.
3. **De forma continua:** no es un proyecto que se hace "cada seis meses", sino parte del trabajo diario. La **regla del boy scout**: deja el código un poco mejor de como lo encontraste.
4. **Separado de los cambios de comportamiento:** no mezcles en el mismo commit una refactorización y una funcionalidad nueva.

El IDE automatiza las refactorizaciones más comunes de forma segura: renombrar (F2 / Ctrl + R, R), extraer método o variable, introducir parámetro, mover tipo a su propio archivo (todo con Ctrl + .).

### Code review

Que otra persona revise tu código antes de integrarlo detecta errores, difunde el conocimiento en el equipo y mantiene la consistencia. Para que sea útil:

* Revisa **diseño, claridad y corrección**; el estilo (espacios, llaves) lo debe cubrir una herramienta (`.editorconfig`, `dotnet format`).
* Haz cambios pequeños: un pull request de 200 líneas se revisa bien; uno de 3.000, no.
* Comenta el código, no a la persona.

### Uso correcto de `try`/`catch`

Un caso frecuente de código "sucio":

```csharp
// ❌ captura todo, oculta el error y sigue como si nada
try
{
    GuardarPedido(pedido);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}
```

Las reglas (vistas en [Excepciones](../08-excepciones/README.md)):

* Captura excepciones **específicas**, donde puedas hacer algo útil.
* No las uses para el control de flujo de casos esperados (`TryParse`, `TryGetValue`).
* Registra el error completo (`ex.ToString()`, o mejor, un `ILogger`) y deja que el manejo global responda en las fronteras de la aplicación.
* Si solo quieres registrar y que siga, relanza con `throw;`.

-----

## Ejemplo completo

Refactorizar un cálculo de precios lleno de duplicación y acoplamiento.

**Antes:**

```csharp
class Tienda
{
    public decimal PrecioOnline(decimal precio, string tipoCliente)
    {
        decimal p = precio;
        if (tipoCliente == "vip") p = p - p * 0.15m;
        else if (tipoCliente == "frecuente") p = p - p * 0.05m;
        p = p + p * 0.18m;
        if (p > 500) Console.WriteLine($"[LOG {DateTime.Now}] precio alto: {p}");
        return Math.Round(p, 2);
    }

    public decimal PrecioTienda(decimal precio, string tipoCliente)
    {
        decimal p = precio;
        if (tipoCliente == "vip") p = p - p * 0.15m;
        else if (tipoCliente == "frecuente") p = p - p * 0.05m;
        p = p + 10;                                              // costo de atención en tienda
        p = p + p * 0.18m;
        if (p > 500) Console.WriteLine($"[LOG {DateTime.Now}] precio alto: {p}");
        return Math.Round(p, 2);
    }
}
```

**Después:**

```csharp
var tienda = new Tienda(new LogEnConsola());
Console.WriteLine(tienda.Precio(500m, TipoCliente.Vip, Canal.Online));        // 501.50
Console.WriteLine(tienda.Precio(100m, TipoCliente.Frecuente, Canal.Tienda));   // 123.90

enum TipoCliente { Normal, Frecuente, Vip }
enum Canal { Online, Tienda }

interface ILog { void Advertir(string mensaje); }

class LogEnConsola : ILog
{
    public void Advertir(string mensaje) => Console.WriteLine($"[LOG {DateTime.Now:HH:mm}] {mensaje}");
}

static class ReglasDePrecio
{
    public const decimal TasaIgv = 0.18m;
    public const decimal CostoAtencionEnTienda = 10m;
    public const decimal UmbralPrecioAlto = 500m;

    public static decimal Descuento(TipoCliente tipo) => tipo switch
    {
        TipoCliente.Vip => 0.15m,
        TipoCliente.Frecuente => 0.05m,
        _ => 0m
    };
}

class Tienda
{
    private readonly ILog _log;
    public Tienda(ILog log) => _log = log;

    public decimal Precio(decimal precioBase, TipoCliente tipo, Canal canal)
    {
        decimal precio = precioBase * (1 - ReglasDePrecio.Descuento(tipo));
        if (canal == Canal.Tienda) precio += ReglasDePrecio.CostoAtencionEnTienda;
        precio *= 1 + ReglasDePrecio.TasaIgv;

        if (precio > ReglasDePrecio.UmbralPrecioAlto)
            _log.Advertir($"precio alto: {precio:N2}");

        return Math.Round(precio, 2);
    }
}
```

Salida:

```text
[LOG 10:30] precio alto: 501.50
501.50
123.90
```

* **DRY:** el descuento, el IGV, el redondeo y el log estaban duplicados; ahora existen una vez.
* **Números mágicos:** `0.15`, `0.18`, `10`, `500` tienen nombre.
* **Strings mágicos:** `"vip"` (que admitía `"VIP"` mal escrito sin error) es un enum.
* **Acoplamiento:** el log ya no es `Console.WriteLine` fijo: se inyecta un `ILog`, que en una prueba puede ser un log falso.
* **KISS:** no se agregaron fábricas ni capas: un método, un enum y una clase de reglas.

Antes de esta refactorización deberías tener pruebas que verifiquen los precios de los dos métodos originales, para confirmar que el comportamiento no cambió.

-----

## Errores comunes

**1. Aplicar DRY a la fuerza.**
Qué pasa: un método "genérico" lleno de parámetros y `if` para cubrir casos que solo se parecían.
Por qué: se unificó código que representaba conceptos distintos.
Arreglo: regla de tres; tolera la duplicación hasta entender bien la abstracción.

**2. Sobreingeniería "por si acaso".**
Qué pasa: interfaces con una sola implementación que nunca tendrá otra, capas que solo pasan datos, configuraciones para casos inexistentes.
Por qué: diseñar para un futuro imaginado.
Arreglo: YAGNI; diseña para lo que se necesita hoy, de forma que sea fácil de extender.

**3. Refactorizar sin pruebas.**
Qué pasa: cambios que "solo limpiaban" rompen comportamiento en producción.
Por qué: no había forma de verificar que nada cambió.
Arreglo: escribe pruebas de caracterización antes de refactorizar código heredado.

**4. "La gran reescritura".**
Qué pasa: meses reescribiendo desde cero, con el sistema viejo y el nuevo divergiendo, y nuevos bugs.
Por qué: se subestima el conocimiento acumulado en el código existente.
Arreglo: refactorización incremental (por ejemplo, el patrón *strangler fig*: reemplazar partes de a poco).

**5. Mezclar refactorización y funcionalidades.**
Qué pasa: revisiones imposibles y bugs difíciles de rastrear.
Por qué: no se sabe qué cambio provocó qué.
Arreglo: commits (o pull requests) separados.

**6. Ignorar la deuda técnica.**
Qué pasa: cada sprint se entregan menos funcionalidades y los bugs aumentan.
Por qué: los "intereses" de la deuda crecen.
Arreglo: hazla visible (tickets, notas en el backlog) y reserva tiempo para pagarla de forma regular.

-----

## Según la versión de C#

Los principios no dependen del lenguaje, pero C# moderno facilita aplicarlos:

* **Records (C# 9):** objetos de valor con poco código, lo que reduce la obsesión por primitivos.
* **Expresiones `switch` y patrones (C# 8–9):** reglas como datos, más simples que cadenas de `if`.
* **Constructores primarios (C# 12):** la inyección de dependencias con menos ceremonia: `class Tienda(ILog log)`.
* **.NET:** inyección de dependencias incorporada (`Microsoft.Extensions.DependencyInjection`) y analizadores de código que detectan muchos smells.

-----

## Cuándo sí y cuándo no

**Aplica DRY a:**

* Reglas de negocio, cálculos, validaciones, configuración: cualquier conocimiento que deba cambiar en un solo lugar.

**Acepta algo de duplicación cuando:**

* Los fragmentos representan conceptos distintos o todavía no está clara la abstracción correcta.
* Es código de pruebas: la claridad de cada prueba vale más que no repetirse.

**Refactoriza:**

* Cuando vas a tocar un código para agregar algo (primero lo haces fácil de cambiar, después lo cambias).
* Cuando un code smell te hace perder tiempo cada vez que pasas por ahí.

**No refactorices:**

* Código que funciona y que nadie va a tocar.
* Justo antes de una entrega crítica, sin pruebas.

-----

## Resumen en 5 líneas

1. DRY: cada conocimiento en un solo lugar; pero duplicar es mejor que una abstracción equivocada (regla de tres).
2. KISS y YAGNI: la solución más simple que resuelve el problema actual; nada de "por si acaso".
3. Bajo acoplamiento (depender de interfaces inyectadas) y alta cohesión (una responsabilidad por clase).
4. La deuda técnica son los atajos de hoy que encarecen los cambios de mañana: hazla visible y págala.
5. Refactorizar es mejorar la estructura sin cambiar el comportamiento: con pruebas, en pasos pequeños y de forma continua.

-----

## Para profundizar

<details>
<summary>SOLID en una tabla</summary>

| Principio | Idea | Lo viste en |
| --- | --- | --- |
| **S**ingle Responsibility | Una clase, una razón para cambiar | Esta lección (cohesión) |
| **O**pen/Closed | Abierto a extensión, cerrado a modificación | [Polimorfismo y casting](../04-poo/09-Polimorfismo%20y%20casting.md) |
| **L**iskov Substitution | Las derivadas se pueden usar donde se espera la base | [Herencia](../04-poo/06-Herencia.md) |
| **I**nterface Segregation | Interfaces chicas y específicas | [Interfaces](../04-poo/08-Interfaces.md) |
| **D**ependency Inversion | Depender de abstracciones, no de implementaciones concretas | Esta lección (acoplamiento) |

Son guías para diseñar código fácil de cambiar, no reglas absolutas. Aplicadas sin criterio, producen la sobreingeniería que KISS intenta evitar.

</details>

<details>
<summary>Pruebas de caracterización</summary>

Antes de refactorizar código heredado sin pruebas, se escriben pruebas que **documentan lo que el código hace hoy**, aunque parezca incorrecto. Si después de refactorizar las pruebas siguen pasando, el comportamiento no cambió. Los errores que descubras se corrigen después, en un cambio aparte. El libro de referencia es *Working Effectively with Legacy Code*, de Michael Feathers.

</details>

-----

## En entrevista

### Respuesta corta (junior)

DRY significa no repetir código ni lógica, para que cada cosa se cambie en un solo lugar. KISS significa mantener las soluciones simples. YAGNI, no programar cosas que todavía no se necesitan. Refactorizar es mejorar la estructura del código sin cambiar lo que hace, y ayuda a reducir la deuda técnica, que son los problemas que acumulamos por tomar atajos.

### Respuesta ampliada (semi-senior)

DRY trata del conocimiento, no del texto: dos fragmentos idénticos que representan reglas distintas no deben unificarse, y una abstracción prematura es más costosa que la duplicación. KISS y YAGNI contrapesan la tendencia a la sobreingeniería; el diseño debe ser simple pero extensible, lo que se logra con bajo acoplamiento (dependencias abstractas inyectadas) y alta cohesión (SRP). La deuda técnica es un instrumento que puede ser consciente, pero debe gestionarse. El refactoring es continuo, en pasos pequeños protegidos por pruebas (de caracterización en código heredado), separado de los cambios funcionales y apoyado en las refactorizaciones automáticas del IDE; las grandes reescrituras se reemplazan por migraciones incrementales.

### Preguntas frecuentes de seguimiento

**1. ¿Siempre hay que eliminar la duplicación?**
No. Si el código se parece pero representa conceptos distintos, unificarlo los acopla. Conviene esperar a entender la abstracción (regla de tres).

**2. ¿Qué es la deuda técnica?**
El costo futuro de las decisiones rápidas o subóptimas de hoy: cada cambio posterior cuesta más hasta que se "paga" refactorizando.

**3. ¿Cómo refactorizas código sin pruebas?**
Primero escribo pruebas que capturen el comportamiento actual (de caracterización), y después refactorizo en pasos pequeños, ejecutándolas en cada paso.

-----

## Práctica

**Ejercicio 1.** Este código viola DRY. Refactorízalo para que la validación del email exista una sola vez.

```csharp
void RegistrarCliente(string email)
{
    if (string.IsNullOrWhiteSpace(email) || !email.Contains('@') || email.Length > 100)
        throw new ArgumentException("Email inválido");
    // ...
}

void ActualizarEmail(int id, string email)
{
    if (string.IsNullOrWhiteSpace(email) || !email.Contains('@') || email.Length > 100)
        throw new ArgumentException("Email inválido");
    // ...
}
```

<details>
<summary>Solución</summary>

Una opción es extraer un método de validación. Otra, más robusta, es un tipo propio que solo puede existir si es válido (evita también la obsesión por primitivos):

```csharp
var email = Email.Crear("ana@mail.com");
Console.WriteLine(email);   // ana@mail.com

void RegistrarCliente(Email email) { /* ya es válido por construcción */ }
void ActualizarEmail(int id, Email email) { /* ... */ }

public readonly record struct Email
{
    private const int LargoMaximo = 100;
    public string Valor { get; }

    private Email(string valor) => Valor = valor;

    public static Email Crear(string valor)
    {
        if (string.IsNullOrWhiteSpace(valor) || !valor.Contains('@') || valor.Length > LargoMaximo)
            throw new ArgumentException("Email inválido", nameof(valor));
        return new Email(valor.Trim().ToLowerInvariant());
    }

    public override string ToString() => Valor;
}
```

</details>

**Ejercicio 2.** Un compañero propone, para un sistema que hoy solo exporta a CSV, crear una interfaz `IExportador`, una fábrica de exportadores, un archivo de configuración para elegir el formato y clases vacías para Excel, PDF y XML "por si las piden". ¿Qué principio está en juego y qué le propondrías?

<details>
<summary>Solución</summary>

**YAGNI** y **KISS**: está construyendo para requisitos que no existen. Le propondría implementar solo la exportación a CSV, bien encapsulada en una clase con un nombre claro (`ExportadorCsv`). Si en el futuro se pide un segundo formato, en ese momento se extrae la interfaz `IExportador` (es una refactorización de minutos con el IDE). Las clases vacías y la configuración agregan código que hay que mantener sin aportar valor hoy.

</details>

-----

## Siguiente lección

[Sintaxis moderna de C#](03-Sintaxis%20moderna%20de%20CSharp.md)
