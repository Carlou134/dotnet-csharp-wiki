# Nombres, comentarios y code smells

## En una frase

Código limpio es código **fácil de entender y de cambiar**: empieza por **nombres** que expliquen la intención, **comentarios** que digan el porqué (no el qué), y por aprender a reconocer los **code smells**, señales de que algo en el diseño merece una mejora.

-----

## Antes de empezar

Conviene que ya sepas:

* Las convenciones de nombres de .NET (camelCase, PascalCase), de [Variables y tipos de datos](../01-tipos-y-variables/01-Variables%20y%20tipos%20de%20datos.md).
* Métodos, clases, enums y constantes: los módulos de [Métodos](../03-metodos/README.md), [POO](../04-poo/README.md) y [Enumeraciones](../05-tipos-avanzados/01-Enumeraciones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Código limpio:** código que se entiende rápido, se cambia con seguridad y no sorprende.
* **Buena práctica:** forma probada de resolver un problema común.
* **Code smell ("olor"):** señal superficial que suele indicar un problema más profundo en el diseño.
* **Número mágico:** literal sin nombre cuyo significado no es evidente.
* **Método largo / clase grande ("God class"):** que hace demasiadas cosas.
* **Legibilidad:** facilidad con la que una persona entiende el código.

-----

## El problema

Mira este método. Funciona. ¿Qué hace?

```csharp
public double Calc(List<int> l, int t)
{
    double r = 0;
    foreach (var x in l)
    {
        if (t == 1) r += x * 0.18;
        else if (t == 2) r += x * 0.08;
        else r += x;
    }
    // suma
    return r;
}
```

Para entenderlo hay que adivinar qué es `l`, qué significa `t == 1`, por qué `0.18`. El comentario "suma" no ayuda. Ahora compáralo:

```csharp
public decimal CalcularImpuestoTotal(IEnumerable<decimal> precios, TipoImpuesto tipo)
{
    decimal tasa = tipo switch
    {
        TipoImpuesto.Igv => 0.18m,
        TipoImpuesto.Reducido => 0.08m,
        _ => 0m
    };
    return precios.Sum(precio => precio * tasa);
}
```

Se lee como una frase. Y al reescribirlo aparece una duda que el original escondía: con un tipo desconocido, la versión vieja sumaba el **precio completo** como si fuera impuesto. ¿Era intencional o un bug? Con nombres claros, ese tipo de errores salta a la vista.

El código se escribe una vez y se **lee** decenas de veces: al revisarlo, depurarlo o cambiarlo. Cada minuto que alguien pierde descifrándolo es un costo, y cada malentendido, un bug.

-----

## Cómo funciona

### Nombres que revelan la intención

El nombre debe responder **qué es** o **qué hace**, sin necesidad de un comentario:

```csharp
// ❌
int d;
List<Usuario> lst;
bool flag;
void Proc();

// ✅
int diasDesdeUltimaModificacion;
List<Usuario> usuariosActivos;
bool tienePermisoDeEscritura;
void EnviarFacturaPorCorreo();
```

Pautas:

| Elemento | Convención | Pauta | Ejemplo |
| --- | --- | --- | --- |
| Variable local, parámetro | camelCase | Sustantivo descriptivo | `precioUnitario` |
| Campo privado | _camelCase | Igual que una variable | `_repositorio` |
| Método | PascalCase | **Verbo** + complemento | `CalcularTotal`, `ObtenerUsuariosActivos` |
| Clase, record, struct | PascalCase | **Sustantivo singular** | `Factura`, `Usuario` |
| Interfaz | `I` + PascalCase | Capacidad o rol | `IRepositorio`, `INotificador` |
| Booleano | camelCase / PascalCase | Pregunta de sí o no | `estaActivo`, `EsValido`, `TieneStock` |
| Constante | PascalCase | Describe el valor | `MaxIntentosDeLogin` |
| Método asíncrono | PascalCase + `Async` | — | `GuardarAsync` |

Más pautas:

* **Pronunciables y buscables:** `fechaGeneracion` en lugar de `fGenYmd`. Buscar `MaxIntentos` en el código es fácil; buscar `3`, imposible.
* **Sin codificaciones ni prefijos de tipo:** `nombre`, no `strNombre` ni `sNombre`.
* **Sin números para distinguir:** `cliente1` y `cliente2` no dicen nada; `remitente` y `destinatario`, sí. (Los números son válidos cuando forman parte del dominio: `Direccion2`, `Http2`).
* **Una palabra por concepto:** no mezcles `Obtener`, `Traer` y `Buscar` para lo mismo en distintas clases.
* **Longitud proporcional al alcance:** `i` está bien en un `for` de tres líneas; un campo de clase necesita un nombre completo.
* **Un idioma por proyecto:** todo en español o todo en inglés (en muchas empresas, todo en inglés). Mezclar (`GetUsuariosActivos`) confunde.

### Comentarios: el porqué, no el qué

El mejor comentario es el que no hace falta porque el código se explica solo. Antes de comentar, intenta **mejorar el nombre** o **extraer un método**:

```csharp
// ❌ comentario que repite el código
// incrementa el contador
contador++;

// ❌ comentario que compensa un mal nombre
// verifica si el usuario puede comprar
if (u.E && u.S > 0 && !u.B) { }

// ✅ el código se explica solo
if (usuario.PuedeComprar()) { }
```

Comentarios que **sí** aportan:

```csharp
// El SAT exige redondear cada línea antes de sumar, no el total (norma 2025-17).
decimal total = lineas.Sum(l => Math.Round(l.Importe, 2));

// TODO: reemplazar por la API v2 cuando el proveedor la habilite (ticket PAG-142).
var respuesta = await _clienteLegado.ConsultarAsync(id);

/// <summary>Calcula el precio final aplicando descuentos y el impuesto vigente.</summary>
/// <param name="precioBase">Precio sin impuestos. Debe ser positivo.</param>
public decimal CalcularPrecioFinal(decimal precioBase) { /* ... */ }
```

* Explican **decisiones** o **restricciones externas** que el código no puede expresar.
* Advierten consecuencias no obvias ("este método no es seguro para hilos").
* Documentan la API pública (`///`).

Lo que **no** se comenta:

* Lo obvio.
* El historial de cambios ("modificado por Juan el 3/5"): para eso está git.
* **Código comentado** "por si acaso": se borra; si hace falta, está en el historial.

Un comentario desactualizado es peor que ninguno: miente.

### Números mágicos

```csharp
// ❌ ¿qué es 3? ¿y 86400? ¿y 0.18?
if (intentos > 3) Bloquear();
var expira = DateTime.Now.AddSeconds(86400);
decimal impuesto = subtotal * 0.18m;

// ✅
const int MaxIntentosDeLogin = 3;
const decimal TasaIgv = 0.18m;

if (intentos > MaxIntentosDeLogin) Bloquear();
var expira = DateTime.Now.AddDays(1);          // a veces basta una API más expresiva
decimal impuesto = subtotal * TasaIgv;
```

Si el valor puede cambiar sin recompilar (una tasa, un límite), sácalo a la **configuración** (`appsettings.json`).

Para conjuntos de valores relacionados, usa un **enum** en lugar de números sueltos:

```csharp
// ❌
if (opcion == 1) Agregar();
else if (opcion == 2) Quitar();
else if (opcion == 3) Listar();

// ✅
enum OpcionMenu { Agregar = 1, Quitar = 2, Listar = 3, Salir = 4 }

switch ((OpcionMenu)opcion)
{
    case OpcionMenu.Agregar: Agregar(); break;
    case OpcionMenu.Quitar: Quitar(); break;
    case OpcionMenu.Listar: Listar(); break;
}
```

### Code smells: señales de alerta

Un *code smell* no es un error: el código funciona. Es un **indicio** de que el diseño va a costar caro cuando haya que cambiarlo. Los más frecuentes:

| Smell | Señal | Remedio habitual |
| --- | --- | --- |
| **Nombres pobres** | `x`, `data`, `Manager`, `Helper`, `Procesar2` | Renombrar (F2 en el IDE) |
| **Método largo** | Más de ~20-30 líneas, varios niveles de anidamiento | Extraer métodos con nombre |
| **Lista larga de parámetros** | Más de 3-4 parámetros | Agrupar en un objeto (record) |
| **Clase grande ("God class")** | Cientos de líneas, muchas responsabilidades | Dividir por responsabilidad |
| **Código duplicado** | El mismo bloque en varios lugares | Extraer y reutilizar (ver DRY en la próxima lección) |
| **Números mágicos** | Literales sin nombre | Constantes, enums, configuración |
| **Anidamiento profundo** | `if` dentro de `if` dentro de `for`... | Cláusulas de guarda, extraer métodos |
| **Banderas booleanas como parámetro** | `Exportar(datos, true, false)` | Dos métodos con nombre, o un enum |
| **Switch repetido sobre un tipo** | El mismo `switch (tipo)` en varios lugares | Polimorfismo |
| **Obsesión por primitivos** | `string email`, `decimal monto` sin validar en todas partes | Tipos propios (`Email`, `Dinero`) |
| **Comentarios que explican código confuso** | Un párrafo antes de 5 líneas crípticas | Reescribir el código para que no haga falta |
| **Código muerto** | Métodos o variables que nadie usa | Borrarlo |

Los smells existen a varios niveles: dentro de un método (anidamiento, variables mal nombradas), en una clase (demasiadas responsabilidades), entre clases (acoplamiento excesivo) y en la aplicación (capas mezcladas).

### Funciones pequeñas que hacen una cosa

```csharp
// ❌ un método que valida, calcula, guarda y notifica
public void ProcesarPedido(Pedido p)
{
    if (p.Items.Count == 0) throw new InvalidOperationException("Pedido vacío");
    if (p.Cliente is null) throw new InvalidOperationException("Sin cliente");
    decimal total = 0;
    foreach (var i in p.Items) total += i.Precio * i.Cantidad;
    if (p.Cliente.EsVip) total *= 0.9m;
    p.Total = total;
    _db.Pedidos.Add(p);
    _db.SaveChanges();
    _correo.Enviar(p.Cliente.Email, $"Tu pedido por {total:N2} fue registrado");
}

// ✅ cada paso tiene nombre; el método principal se lee como un índice
public void ProcesarPedido(Pedido pedido)
{
    Validar(pedido);
    pedido.Total = CalcularTotal(pedido);
    Guardar(pedido);
    NotificarAlCliente(pedido);
}
```

El método principal cuenta **qué** pasa; los detalles de **cómo** están en métodos chicos, cada uno fácil de leer, probar y cambiar.

-----

## Ejemplo completo

Refactorizar paso a paso un programa de menú lleno de smells:

**Antes:**

```csharp
var t = new List<string>();
int o = 0;
do
{
    Console.WriteLine("1-Agregar 2-Quitar 3-Listar 4-Salir");
    o = int.Parse(Console.ReadLine()!);
    if (o == 1)
    {
        Console.Write("Tarea: ");
        var x = Console.ReadLine();
        if (x != null && x != "" && x.Length < 50) t.Add(x);
    }
    else if (o == 2)
    {
        for (int i = 0; i < t.Count; i++) Console.WriteLine((i + 1) + ". " + t[i]);
        Console.Write("Número: ");
        var n = int.Parse(Console.ReadLine()!);
        if (n > 0 && n <= t.Count) t.RemoveAt(n - 1);
    }
    else if (o == 3)
    {
        for (int i = 0; i < t.Count; i++) Console.WriteLine((i + 1) + ". " + t[i]);
    }
} while (o != 4);
```

**Después:**

```csharp
const int LargoMaximoTarea = 50;
var tareas = new List<string>();
OpcionMenu opcion;

do
{
    opcion = PedirOpcion();
    switch (opcion)
    {
        case OpcionMenu.Agregar: AgregarTarea(tareas); break;
        case OpcionMenu.Quitar: QuitarTarea(tareas); break;
        case OpcionMenu.Listar: MostrarTareas(tareas); break;
    }
} while (opcion != OpcionMenu.Salir);

static OpcionMenu PedirOpcion()
{
    Console.WriteLine("1-Agregar 2-Quitar 3-Listar 4-Salir");
    return int.TryParse(Console.ReadLine(), out int valor) && Enum.IsDefined((OpcionMenu)valor)
        ? (OpcionMenu)valor
        : OpcionMenu.Invalida;
}

static void AgregarTarea(List<string> tareas)
{
    Console.Write("Tarea: ");
    string? descripcion = Console.ReadLine()?.Trim();
    if (EsDescripcionValida(descripcion))
        tareas.Add(descripcion!);
    else
        Console.WriteLine($"La tarea no puede estar vacía ni superar {LargoMaximoTarea} caracteres.");
}

static void QuitarTarea(List<string> tareas)
{
    MostrarTareas(tareas);
    Console.Write("Número: ");
    if (int.TryParse(Console.ReadLine(), out int numero) && numero >= 1 && numero <= tareas.Count)
        tareas.RemoveAt(numero - 1);
    else
        Console.WriteLine("Número inválido.");
}

static void MostrarTareas(List<string> tareas)
{
    for (int i = 0; i < tareas.Count; i++)
        Console.WriteLine($"{i + 1}. {tareas[i]}");
}

static bool EsDescripcionValida(string? descripcion) =>
    !string.IsNullOrWhiteSpace(descripcion) && descripcion.Length <= LargoMaximoTarea;

enum OpcionMenu { Invalida = 0, Agregar = 1, Quitar = 2, Listar = 3, Salir = 4 }
```

Qué cambió y por qué:

* **Nombres:** `t`, `o`, `x`, `n` → `tareas`, `opcion`, `descripcion`, `numero`.
* **Números mágicos:** `1`, `2`, `3`, `4` y `50` → `OpcionMenu` y `LargoMaximoTarea`.
* **Duplicación:** el bucle de listado estaba dos veces → `MostrarTareas`.
* **Robustez:** `int.Parse` (que hace caer el programa con texto) → `TryParse`.
* **Responsabilidades:** cada opción es un método con nombre; el bucle principal se lee de un vistazo.

El comportamiento es el mismo; el código es más largo en líneas, pero **mucho más corto en tiempo de comprensión**.

-----

## Errores comunes

**1. Nombres genéricos.**
Qué pasa: clases como `Manager`, `Helper`, `Utils`, `Processor` que terminan acumulando de todo.
Por qué: el nombre no limita la responsabilidad.
Arreglo: nombres que digan qué hacen (`CalculadoraDeImpuestos`, `ExportadorCsv`); si no encuentras un buen nombre, probablemente la clase hace demasiado.

**2. Abreviar para ahorrar teclas.**
Qué pasa: `calcTotPedCli` se lee una vez y se descifra cien.
Por qué: el autocompletado del IDE escribe por ti; el lector no tiene autocompletado para entender.
Arreglo: nombres completos.

**3. Comentar en lugar de mejorar.**
Qué pasa: comentarios largos que explican código confuso y que nadie actualiza cuando el código cambia.
Por qué: es más fácil escribir un comentario que refactorizar.
Arreglo: extrae un método con un nombre que diga lo que decía el comentario.

**4. Dejar código comentado.**
Qué pasa: bloques muertos que confunden ("¿esto se usa? ¿lo vuelvo a activar?").
Por qué: miedo a perderlo.
Arreglo: bórralo; git lo recuerda.

**5. Refactorizar sin red de seguridad.**
Qué pasa: al "limpiar", se rompe un comportamiento que nadie notó hasta producción.
Por qué: no había pruebas que verificaran que el comportamiento siguió igual.
Arreglo: pruebas automatizadas antes de refactorizar, y cambios pequeños (próxima lección).

-----

## Según la versión de C#

El lenguaje fue sumando herramientas que hacen más fácil escribir código claro:

* **C# 6:** `nameof`, interpolación de strings, miembros con cuerpo de expresión.
* **C# 7–9:** pattern matching, funciones locales, records y expresiones `switch`, que reemplazan cadenas largas de `if`.
* **C# 10–12:** espacios de nombres por archivo, `global using`, constructores primarios: menos ruido en cada archivo.

La sintaxis moderna se ve en detalle en [Sintaxis moderna de C#](03-Sintaxis%20moderna%20de%20CSharp.md).

-----

## Cuándo sí y cuándo no

**Aplica estas prácticas siempre que:**

* El código vaya a durar más que una tarde: casi siempre.

**Equilibra cuando:**

* Un script descartable o un prototipo de prueba no necesita la misma pulcritud que el núcleo del negocio. Pero cuidado: muchos prototipos terminan en producción.
* La regla choca con la claridad: "métodos de menos de 20 líneas" es una guía, no una ley. Un `switch` de 30 líneas que mapea valores puede ser perfectamente claro.

**Herramientas que ayudan:**

* Los analizadores de .NET y un archivo `.editorconfig` hacen cumplir convenciones automáticamente en todo el equipo.
* El IDE renombra (F2, Ctrl + R, R) y extrae métodos (Ctrl + .) de forma segura.

-----

## Resumen en 5 líneas

1. El código se lee mucho más de lo que se escribe: optimiza para quien lo lee.
2. Nombres que revelan la intención: verbos para métodos, sustantivos para clases, preguntas para booleanos, sin abreviaturas.
3. Comentarios para el porqué (decisiones, restricciones externas), no para el qué; nunca código comentado.
4. Reemplaza los números mágicos por constantes, enums o configuración.
5. Los code smells (métodos largos, muchos parámetros, duplicación, anidamiento) son señales para refactorizar.

-----

## Para profundizar

<details>
<summary>.editorconfig: reglas de estilo para todo el equipo</summary>

Un archivo `.editorconfig` en la raíz del repositorio define convenciones que el IDE y el compilador aplican:

```ini
[*.cs]
indent_size = 4
dotnet_naming_rule.campos_privados.symbols = campos_privados
dotnet_naming_rule.campos_privados.style = guion_bajo
dotnet_naming_rule.campos_privados.severity = warning
dotnet_naming_symbols.campos_privados.applicable_kinds = field
dotnet_naming_symbols.campos_privados.applicable_accessibilities = private
dotnet_naming_style.guion_bajo.required_prefix = _
dotnet_naming_style.guion_bajo.capitalization = camel_case
csharp_style_var_when_type_is_apparent = true:suggestion
```

Con `dotnet format` se puede aplicar el estilo a toda la solución, y en CI se puede hacer fallar el build si no se cumple.

</details>

<details>
<summary>Lecturas recomendadas</summary>

* *Clean Code*, de Robert C. Martin: el libro que popularizó el término (con ejemplos en Java, pero las ideas aplican igual).
* *Refactoring*, de Martin Fowler: el catálogo de code smells y de refactorizaciones.
* Las [convenciones de código de C#](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions) de Microsoft.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Código limpio es código fácil de leer y mantener. Se logra con nombres descriptivos (PascalCase para métodos y clases, camelCase para variables), métodos cortos que hacen una sola cosa, evitando números mágicos y usando comentarios solo para explicar el porqué. Un code smell es una señal de que algo en el código podría estar mal diseñado, como un método muy largo o código duplicado.

### Respuesta ampliada (semi-senior)

La legibilidad se optimiza para el lector: nombres que revelan la intención y que pertenecen al lenguaje del dominio, funciones con un solo nivel de abstracción, parámetros reducidos (agrupados en objetos), y tipos propios en lugar de primitivos para conceptos del dominio. Los comentarios documentan decisiones y la API pública; el resto debe expresarse en el código. Los code smells (método largo, clase dios, obsesión por primitivos, `switch` repetido sobre tipos, cirugía de escopeta) orientan refactorizaciones concretas del catálogo de Fowler. El equipo automatiza convenciones con `.editorconfig`, analizadores y `dotnet format`, y deja las discusiones de estilo fuera de las revisiones de código.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es un code smell?**
Un indicio en el código (método largo, duplicación, demasiados parámetros) de un posible problema de diseño. No es un error, pero suele hacer el código más difícil de cambiar.

**2. ¿Cuándo escribirías un comentario?**
Para explicar por qué se tomó una decisión, una restricción externa o un efecto no evidente, y para documentar la API pública. No para repetir lo que el código ya dice.

**3. ¿Qué es un número mágico y cómo lo evitas?**
Un literal sin nombre cuyo significado no es evidente. Se reemplaza por una constante con nombre, un enum o un valor de configuración.

-----

## Práctica

**Ejercicio 1.** Mejora los nombres y elimina los números mágicos de este método:

```csharp
public bool Chk(string s, int a)
{
    if (s.Length < 8) return false;
    if (a > 5) return false;
    return s.Any(char.IsDigit);
}
```

<details>
<summary>Solución</summary>

```csharp
private const int LargoMinimoContrasena = 8;
private const int MaxIntentosFallidos = 5;

public bool PuedeIniciarSesion(string contrasena, int intentosFallidos)
{
    bool contrasenaValida = contrasena.Length >= LargoMinimoContrasena && contrasena.Any(char.IsDigit);
    bool cuentaBloqueada = intentosFallidos > MaxIntentosFallidos;
    return contrasenaValida && !cuentaBloqueada;
}
```

Las variables intermedias con nombre (`contrasenaValida`, `cuentaBloqueada`) hacen que la última línea se lea como la regla de negocio.

</details>

**Ejercicio 2.** Identifica al menos cuatro code smells en este fragmento y propón un remedio para cada uno:

```csharp
public void Exportar(List<object[]> d, bool b1, bool b2, string r, int f)
{
    // exportar
    if (f == 1) { /* 40 líneas para CSV */ }
    else if (f == 2) { /* 40 líneas para Excel */ }
    // if (f == 3) { /* PDF, no funciona */ }
}
```

<details>
<summary>Solución</summary>

1. **Nombres pobres** (`d`, `b1`, `b2`, `r`, `f`): renombrar según su significado.
2. **Banderas booleanas como parámetros** (`b1`, `b2`): reemplazar por un objeto de opciones (`OpcionesExportacion`) o por métodos distintos.
3. **Número mágico** (`f == 1`, `f == 2`): un enum `FormatoExportacion`.
4. **Método largo / switch sobre un tipo**: una interfaz `IExportador` con implementaciones `ExportadorCsv` y `ExportadorExcel` (polimorfismo).
5. **Comentario inútil** ("exportar") y **código comentado** (el bloque de PDF): borrarlos.
6. **Obsesión por primitivos** (`List<object[]>`): un tipo con nombre para cada fila.

</details>

-----

## Siguiente lección

[Principios DRY, KISS y refactoring](02-Principios%20DRY%20KISS%20y%20refactoring.md)
