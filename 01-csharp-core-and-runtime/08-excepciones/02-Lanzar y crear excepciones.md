# Lanzar y crear excepciones

## En una frase

Con `throw` tu código **señala** que algo no puede continuar (un argumento inválido, un estado incorrecto), usando las excepciones de .NET o **excepciones propias** que heredan de `Exception`; y cuando relanzas una excepción capturada, `throw;` conserva la traza original mientras que `throw ex;` la pierde.

-----

## Antes de empezar

Conviene que ya sepas:

* `try`, `catch`, `finally`, propagación y filtros, de [Manejo de excepciones](01-Manejo%20de%20excepciones.md).
* Herencia y constructores con `: base(...)`, de [Herencia](../04-poo/06-Herencia.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`throw`:** sentencia (o expresión) que lanza una excepción.
* **Cláusula de guarda:** validación al inicio de un método que lanza si los datos no sirven.
* **Relanzar:** volver a lanzar la excepción capturada con `throw;`.
* **Excepción interna (*inner exception*):** la excepción original que otra "envuelve" para agregar contexto.
* **Excepción personalizada:** clase propia que hereda de `Exception` para representar un error del dominio.
* **Patrón Result:** devolver un objeto que indica éxito o error en lugar de lanzar una excepción.

-----

## El problema

Escribes un método que transfiere dinero:

```csharp
void Transferir(Cuenta origen, Cuenta destino, decimal monto)
{
    origen.Saldo -= monto;
    destino.Saldo += monto;
}
```

¿Qué pasa si `monto` es negativo? ¿Si `origen` es `null`? ¿Si no hay saldo suficiente? El método hace cosas absurdas en silencio (una transferencia negativa "roba" dinero al destino) o falla más adelante con un `NullReferenceException` que no explica nada.

Necesitas que el método **rechace** los datos inválidos en el momento, con un mensaje claro, y que quien lo llama pueda distinguir "argumento inválido" de "saldo insuficiente", porque se manejan distinto.

-----

## Cómo funciona

### `throw`: lanzar una excepción

```csharp
public void Transferir(Cuenta origen, Cuenta destino, decimal monto)
{
    if (origen is null) throw new ArgumentNullException(nameof(origen));
    if (destino is null) throw new ArgumentNullException(nameof(destino));
    if (monto <= 0) throw new ArgumentOutOfRangeException(nameof(monto), monto, "El monto debe ser positivo.");
    if (origen.Saldo < monto) throw new InvalidOperationException("Saldo insuficiente.");

    origen.Saldo -= monto;
    destino.Saldo += monto;
}
```

* `throw` termina el método de inmediato: lo que sigue no se ejecuta.
* Validar al principio (**cláusulas de guarda**) garantiza que el resto del método trabaje con datos válidos: si algo falla, falla **pronto** y **cerca** del origen (*fail fast*).
* `nameof(monto)` produce `"monto"` y se actualiza solo si renombras el parámetro.

### Elegir la excepción adecuada

| Situación | Excepción |
| --- | --- |
| Un argumento es `null` y no debería | `ArgumentNullException` |
| Un argumento está fuera del rango válido | `ArgumentOutOfRangeException` |
| Un argumento es inválido por otro motivo | `ArgumentException` |
| El objeto no está en un estado que permita la operación | `InvalidOperationException` |
| La operación no está soportada (por diseño) | `NotSupportedException` |
| Todavía no implementado (temporal, durante el desarrollo) | `NotImplementedException` |
| Un formato de texto es incorrecto | `FormatException` |
| Se canceló una operación | `OperationCanceledException` |

Nunca lances `Exception` a secas, ni `NullReferenceException` o `IndexOutOfRangeException` a mano: son para el runtime.

**Atención con los constructores.** Estos dos no hacen lo mismo:

```csharp
throw new ArgumentOutOfRangeException("La cantidad no puede ser negativa.");     // ❌ ese texto es el NOMBRE DEL PARÁMETRO
throw new ArgumentOutOfRangeException(nameof(cantidad), "La cantidad no puede ser negativa.");   // ✅
```

En `ArgumentException` es al revés: `new ArgumentException(mensaje, nombreParametro)`. Revisa la firma.

### Atajos para las guardas (`ThrowIf...`)

.NET moderno trae métodos estáticos que validan y lanzan en una línea:

```csharp
ArgumentNullException.ThrowIfNull(origen);
ArgumentException.ThrowIfNullOrWhiteSpace(titular);
ArgumentOutOfRangeException.ThrowIfNegativeOrZero(monto);
ArgumentOutOfRangeException.ThrowIfGreaterThan(porcentaje, 100);
ObjectDisposedException.ThrowIf(_cerrado, this);
```

El nombre del parámetro se captura automáticamente (con `CallerArgumentExpression`), así que no hace falta `nameof`.

### `throw` como expresión

Desde C# 7, `throw` puede ir donde se espera un valor:

```csharp
_nombre = nombre ?? throw new ArgumentNullException(nameof(nombre));

string Categoria(int edad) => edad >= 0
    ? (edad < 18 ? "Menor" : "Adulto")
    : throw new ArgumentOutOfRangeException(nameof(edad));

decimal Tarifa(string zona) => zona switch
{
    "norte" => 10m,
    "sur" => 12m,
    _ => throw new ArgumentException($"Zona desconocida: {zona}", nameof(zona))
};
```

### Relanzar: `throw;` frente a `throw ex;`

A veces capturas una excepción para registrarla y la dejas seguir:

```csharp
try
{
    ProcesarPedido(pedido);
}
catch (Exception ex)
{
    Log($"Falló el pedido {pedido.Id}: {ex.Message}");
    throw;          // ✅ relanza la MISMA excepción con su traza original
}
```

```csharp
catch (Exception ex)
{
    Log(ex.Message);
    throw ex;       // ❌ reinicia la traza: parece que el error ocurrió AQUÍ
}
```

Con `throw ex;`, la traza de pila empieza en este `catch` y pierdes la información de dónde ocurrió realmente el error. Es un error clásico de entrevista y de revisión de código. (Si solo quieres registrar, un filtro `when (Registrar(ex))` que devuelva `false` lo hace sin siquiera capturar).

### Envolver con contexto: `InnerException`

Cuando traduces un error técnico a uno de más alto nivel, **envuelve** la excepción original:

```csharp
try
{
    string json = File.ReadAllText(ruta);
    return JsonSerializer.Deserialize<Config>(json)!;
}
catch (IOException ex)
{
    throw new ConfiguracionInvalidaException($"No se pudo leer la configuración desde '{ruta}'.", ex);
}
```

El segundo argumento queda en `InnerException`. Quien la reciba ve un mensaje con contexto de negocio, y en el log (`ex.ToString()`) aparecen las dos excepciones con sus trazas.

### Excepciones personalizadas

Las excepciones de .NET describen errores técnicos. Para errores de **tu dominio**, crea las tuyas:

```csharp
public class SaldoInsuficienteException : Exception
{
    public decimal SaldoDisponible { get; }
    public decimal MontoSolicitado { get; }

    public SaldoInsuficienteException(decimal saldoDisponible, decimal montoSolicitado)
        : base($"Saldo insuficiente: disponible {saldoDisponible:N2}, solicitado {montoSolicitado:N2}.")
    {
        SaldoDisponible = saldoDisponible;
        MontoSolicitado = montoSolicitado;
    }

    public SaldoInsuficienteException(string mensaje, Exception? interna = null) : base(mensaje, interna) { }
}
```

Reglas y convenciones:

* Hereda de `Exception` (o de una más específica, como `InvalidOperationException`, si encaja).
* El nombre termina en **`Exception`**.
* Pasa el mensaje a `base(...)` y ofrece un constructor que acepte la excepción interna.
* Agrega **propiedades** con los datos útiles para manejar el error (no obligues a parsear el mensaje).

Quien llama puede capturarla de forma específica y reaccionar con esos datos:

```csharp
try
{
    banco.Transferir(cuentaA, cuentaB, 500m);
}
catch (SaldoInsuficienteException ex)
{
    Console.WriteLine($"Te faltan {ex.MontoSolicitado - ex.SaldoDisponible:N2}.");
}
```

### Excepciones o resultados

No todo "fallo" es excepcional. Una regla útil:

* **Excepción:** cuando el método **no puede cumplir lo que promete** por un error de programación o una situación anormal (argumento inválido, archivo de configuración corrupto, base de datos caída).
* **Resultado (`bool TryX(out ...)`, `null`, un tipo `Result<T>`):** cuando el "fallo" es un caso **esperado y frecuente** (el usuario escribió mal, la búsqueda no encontró nada, la validación de un formulario).

```csharp
// Esperado: el código de descuento puede no existir
if (descuentos.TryGetValue(codigo, out var d)) { /* ... */ }

// No esperado: el sistema no puede funcionar sin esto
var cadena = config["ConnectionString"] ?? throw new InvalidOperationException("Falta la cadena de conexión.");
```

-----

## Ejemplo completo

```csharp
var banco = new Banco();
var ana = banco.AbrirCuenta("Ana", 1000m);
var luis = banco.AbrirCuenta("Luis", 50m);

IntentarTransferir(banco, ana, luis, 300m);
IntentarTransferir(banco, luis, ana, 1000m);
IntentarTransferir(banco, ana, luis, -20m);

try
{
    banco.AbrirCuenta("   ", 10m);
}
catch (ArgumentException ex)
{
    Console.WriteLine($"No se abrió la cuenta: {ex.Message}");
}

Console.WriteLine($"Saldos finales → Ana: {ana.Saldo:N2}, Luis: {luis.Saldo:N2}");

static void IntentarTransferir(Banco banco, Cuenta origen, Cuenta destino, decimal monto)
{
    try
    {
        banco.Transferir(origen, destino, monto);
        Console.WriteLine($"OK: {origen.Titular} → {destino.Titular} {monto:N2}");
    }
    catch (SaldoInsuficienteException ex)
    {
        Console.WriteLine($"Rechazada: a {origen.Titular} le faltan {ex.MontoSolicitado - ex.SaldoDisponible:N2}");
    }
    catch (ArgumentOutOfRangeException ex) when (ex.ParamName == "monto")
    {
        Console.WriteLine($"Monto inválido ({ex.ActualValue}): debe ser positivo");
    }
}

class Cuenta
{
    public string Titular { get; }
    public decimal Saldo { get; internal set; }
    public Cuenta(string titular, decimal saldo) => (Titular, Saldo) = (titular, saldo);
}

class Banco
{
    public Cuenta AbrirCuenta(string titular, decimal saldoInicial)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(titular);
        ArgumentOutOfRangeException.ThrowIfNegative(saldoInicial);
        return new Cuenta(titular, saldoInicial);
    }

    public void Transferir(Cuenta origen, Cuenta destino, decimal monto)
    {
        ArgumentNullException.ThrowIfNull(origen);
        ArgumentNullException.ThrowIfNull(destino);
        if (monto <= 0)
            throw new ArgumentOutOfRangeException(nameof(monto), monto, "El monto debe ser positivo.");
        if (origen.Saldo < monto)
            throw new SaldoInsuficienteException(origen.Saldo, monto);

        origen.Saldo -= monto;
        destino.Saldo += monto;
    }
}

class SaldoInsuficienteException : Exception
{
    public decimal SaldoDisponible { get; }
    public decimal MontoSolicitado { get; }

    public SaldoInsuficienteException(decimal disponible, decimal solicitado)
        : base($"Saldo insuficiente: disponible {disponible:N2}, solicitado {solicitado:N2}.")
    {
        SaldoDisponible = disponible;
        MontoSolicitado = solicitado;
    }
}
```

Salida:

```text
OK: Ana → Luis 300.00
Rechazada: a Luis le faltan 650.00
Monto inválido (-20): debe ser positivo
No se abrió la cuenta: The value cannot be an empty string or composed entirely of whitespace. (Parameter 'titular')
Saldos finales → Ana: 700.00, Luis: 350.00
```

El `Banco` solo valida y lanza; `IntentarTransferir`, que está en la frontera con el usuario, decide qué mensaje mostrar para cada tipo de error.

-----

## Errores comunes

**1. Relanzar con `throw ex;`.**
Qué pasa: la traza de pila apunta al `catch`, no al lugar real del error.
Por qué: `throw ex` lanza la excepción como si fuera nueva.
Arreglo: `throw;`. Si necesitas agregar contexto, envuelve: `throw new MiException("...", ex);`.

**2. Confundir el orden de los argumentos del constructor.**
Qué pasa: el mensaje aparece como nombre del parámetro (`Parameter 'La cantidad no puede ser negativa.'`).
Por qué: `ArgumentOutOfRangeException(string)` recibe el **nombre del parámetro**, no el mensaje.
Arreglo: `new ArgumentOutOfRangeException(nameof(param), "mensaje")`.

**3. Lanzar `Exception` genérica.**
Qué pasa: quien llama no puede distinguir este error de cualquier otro sin leer el texto del mensaje.
Por qué: `catch (Exception)` captura todo.
Arreglo: usa una excepción específica de .NET o una propia.

**4. Perder la excepción original al traducirla.**
Qué pasa: el log muestra "No se pudo leer la configuración" sin la causa.
Por qué: no pasaste la excepción interna.
Arreglo: `throw new MiException("mensaje", ex);`.

**5. Lanzar dentro de un `finally`.**
Qué pasa: la excepción original se pierde y la reemplaza la del `finally`.
Por qué: solo puede propagarse una.
Arreglo: el `finally` debe ser simple y no fallar.

**6. Excepciones para la validación de formularios.**
Qué pasa: un `try/catch` por cada campo, y mensajes de error que se descubren de a uno.
Por qué: los errores de validación del usuario son esperados.
Arreglo: valida todos los campos y devuelve una lista de errores (patrón Result o librerías como FluentValidation).

-----

## Según la versión de C#

* **C# 6:** `nameof` y filtros `when`.
* **C# 7:** expresiones `throw` (`??`, ternario, `switch`, cuerpos de expresión).
* **.NET 6:** `ArgumentNullException.ThrowIfNull`.
* **.NET 7 / 8:** `ArgumentException.ThrowIfNullOrEmpty`/`ThrowIfNullOrWhiteSpace` y la familia `ArgumentOutOfRangeException.ThrowIfNegative`, `ThrowIfZero`, `ThrowIfGreaterThan`...
* **C# 10:** `CallerArgumentExpression`, que permite a esos métodos conocer el nombre del argumento.

-----

## Cuándo sí y cuándo no

**Lanza una excepción cuando:**

* Un argumento viola el contrato del método (guardas al inicio de los métodos públicos).
* El objeto está en un estado que no permite la operación.
* Ocurre algo anormal que el método no puede resolver.

**Crea una excepción propia cuando:**

* Representa un error del dominio que alguien va a querer capturar específicamente (y quizás con datos extra).

**No lances (devuelve un resultado) cuando:**

* El "fallo" es parte del flujo normal: búsquedas sin resultados, entradas inválidas del usuario, validaciones.

-----

## Resumen en 5 líneas

1. `throw new TipoDeExcepcion(...)` termina el método y señala el error; valida al inicio con guardas (*fail fast*).
2. Usa la excepción más específica: `ArgumentNullException`, `ArgumentOutOfRangeException`, `InvalidOperationException`...
3. Relanza con `throw;` (conserva la traza), nunca con `throw ex;`; para agregar contexto, envuelve con `InnerException`.
4. Las excepciones propias heredan de `Exception`, terminan en `Exception` y pueden llevar propiedades con datos.
5. Excepciones para lo anormal; `TryX`/resultados para lo esperado.

-----

## Para profundizar

<details>
<summary>El patrón Result</summary>

En lugar de lanzar, un método puede devolver un objeto que indique éxito o error:

```csharp
public record Resultado<T>(bool Exito, T? Valor, string? Error)
{
    public static Resultado<T> Ok(T valor) => new(true, valor, null);
    public static Resultado<T> Falla(string error) => new(false, default, error);
}

Resultado<decimal> CalcularDescuento(string codigo) =>
    codigo == "VERANO" ? Resultado<decimal>.Ok(0.1m) : Resultado<decimal>.Falla("Código inválido");
```

Hace explícito en la firma que la operación puede fallar y obliga a quien llama a considerarlo. Es habitual en arquitecturas limpias y en la capa de aplicación; las excepciones quedan para los errores realmente inesperados.

</details>

<details>
<summary>ExceptionDispatchInfo: relanzar desde otro lugar</summary>

Si necesitas guardar una excepción y relanzarla más tarde (por ejemplo, en otro hilo) sin perder la traza original:

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo? capturada = null;
try { /* ... */ }
catch (Exception ex) { capturada = ExceptionDispatchInfo.Capture(ex); }

capturada?.Throw();   // relanza conservando la traza original
```

Es lo que usa `await` por dentro para que las excepciones de un `Task` conserven su traza.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Se lanza una excepción con `throw new ...`, por ejemplo `ArgumentNullException` si un parámetro es `null`. Para relanzar una excepción capturada se usa `throw;`, porque `throw ex;` pierde la traza de pila original. Las excepciones personalizadas son clases que heredan de `Exception`, cuyo nombre termina en `Exception`.

### Respuesta ampliada (semi-senior)

Las guardas al inicio de los métodos públicos implementan *fail fast* y se escriben hoy con los helpers `ThrowIfNull`/`ThrowIfNegative`, que usan `CallerArgumentExpression`. `throw;` preserva la traza mientras `throw ex;` la reinicia; para traducir errores técnicos a errores de dominio se envuelve la causa en `InnerException`. Las excepciones propias deben aportar datos tipados (propiedades), no solo texto, y heredar de la base más adecuada. Las excepciones se reservan para violaciones de contrato y situaciones anormales; los resultados esperados se modelan con `TryX` o un tipo `Result`, por claridad y por el costo de lanzar. En las fronteras (middleware, manejadores globales) se registran con `ex.ToString()` y se traducen a respuestas (por ejemplo, `ProblemDetails`).

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `throw` y `throw ex`?**
`throw;` relanza la misma excepción conservando su traza; `throw ex;` reinicia la traza en el punto del `catch`.

**2. ¿Cuándo crearías una excepción personalizada?**
Cuando representa un error del dominio que alguien necesita capturar de forma específica, idealmente con datos adicionales.

**3. ¿Lanzar excepciones o devolver un resultado?**
Excepciones para lo inesperado o las violaciones de contrato; resultados (`TryX`, `Result<T>`) para fallos esperados y frecuentes.

-----

## Práctica

**Ejercicio 1.** Agrega guardas a este constructor para que `nombre` no sea nulo ni vacío, `edad` esté entre 0 y 130 y `email` contenga una `@`. Usa las excepciones adecuadas.

```csharp
class Usuario
{
    public Usuario(string nombre, int edad, string email) { /* ... */ }
}
```

<details>
<summary>Solución</summary>

```csharp
class Usuario
{
    public string Nombre { get; }
    public int Edad { get; }
    public string Email { get; }

    public Usuario(string nombre, int edad, string email)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(nombre);
        ArgumentOutOfRangeException.ThrowIfNegative(edad);
        ArgumentOutOfRangeException.ThrowIfGreaterThan(edad, 130);
        ArgumentNullException.ThrowIfNull(email);
        if (!email.Contains('@'))
            throw new ArgumentException("El email debe contener '@'.", nameof(email));

        Nombre = nombre;
        Edad = edad;
        Email = email;
    }
}
```

</details>

**Ejercicio 2.** Crea una excepción `ProductoNoEncontradoException` con una propiedad `Codigo`, lánzala desde un método `Buscar(string codigo)` cuando el código no exista en un diccionario, y captúrala mostrando el código. Luego reflexiona: ¿sería mejor un `TryBuscar`?

<details>
<summary>Solución</summary>

```csharp
var catalogo = new Dictionary<string, string> { ["A1"] = "Teclado" };

try
{
    Console.WriteLine(Buscar(catalogo, "Z9"));
}
catch (ProductoNoEncontradoException ex)
{
    Console.WriteLine($"No existe el producto {ex.Codigo}");
}

static string Buscar(Dictionary<string, string> catalogo, string codigo) =>
    catalogo.TryGetValue(codigo, out var nombre) ? nombre : throw new ProductoNoEncontradoException(codigo);

class ProductoNoEncontradoException : Exception
{
    public string Codigo { get; }
    public ProductoNoEncontradoException(string codigo) : base($"No se encontró el producto '{codigo}'.")
        => Codigo = codigo;
}
```

Depende del contexto: si buscar un código inexistente es normal (el usuario lo tipeó), un `TryBuscar` o devolver `null` es mejor. Si el código viene de datos que **deberían** ser consistentes (un pedido que referencia un producto), que no exista es anormal y la excepción está justificada.

</details>

-----

## Siguiente lección

Terminaste el módulo de excepciones. Continúa con [Delegados y eventos](../09-delegados-y-eventos/README.md).
