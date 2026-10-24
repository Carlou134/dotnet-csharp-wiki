# VB.NET para desarrolladores de C#

## En una frase

VB.NET compila al mismo IL y usa la misma biblioteca de .NET que C#, así que si sabes C# ya sabes el 90 % de VB.NET; lo que te falta es la sintaxis y un puñado de diferencias que causan errores reales (`And` frente a `AndAlso`, `Option Strict`, sensibilidad a mayúsculas), porque vas a **leer y mantener** código VB, no a empezar proyectos nuevos en él.

-----

## Antes de empezar

Conviene que ya sepas:

* C# con clases, propiedades, eventos, LINQ y `async`/`await`: [C# Core & Runtime](../../README.md).
* Qué es el IL y por qué distintos lenguajes comparten la plataforma: [Introducción](../../01-csharp-core-and-runtime/00-introduccion/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **VB.NET:** Visual Basic sobre .NET (2002 en adelante); compila a IL como C#.
* **VB6:** Visual Basic 6 (1998), el lenguaje anterior a .NET, basado en COM; no es VB.NET.
* **`Option Strict`:** opción que prohíbe conversiones implícitas con pérdida de datos y el enlace tardío; con `Off`, el compilador deja pasar errores que aparecen en ejecución.
* **`Option Explicit`:** obliga a declarar las variables con `Dim`.
* **`Module`:** contenedor de miembros compartidos; equivale a una clase `static` de C#.
* **`AndAlso` / `OrElse`:** operadores lógicos con cortocircuito; `And` / `Or` evalúan siempre ambos lados.
* **`Handles` / `WithEvents`:** forma declarativa de VB para conectar un método a un evento.

-----

## El problema

Te asignan un error en un módulo de facturación en VB.NET. El reporte: "a veces se cae al facturar a clientes sin dirección".

```vb
Public Function CalcularEnvio(cliente As Cliente) As Decimal
    If cliente.Direccion IsNot Nothing And cliente.Direccion.Provincia = "Lima" Then
        Return 0
    End If
    Dim recargo = cliente.Puntos / 10
    Return 15 + recargo
End Function
```

Lo que pasa, y por qué no lo ves si lees VB "como si fuera C#":

1. **`And` no corta.** Con `Direccion` en `Nothing`, `And` evalúa igual el lado derecho y lanza `NullReferenceException`. En C#, `&&` habría cortado. El operador con cortocircuito en VB es `AndAlso`.
2. **El archivo tiene `Option Strict Off`** (el valor por defecto en muchos proyectos viejos). `cliente.Puntos / 10` con `Puntos` entero da un `Double` (en VB, `/` siempre es división real), que se convierte en silencio a `Decimal` al sumarse: errores de redondeo en la facturación, sin un solo aviso del compilador.
3. Buscas el método `ObtenerCliente` con la búsqueda sensible a mayúsculas del editor y no aparece: está escrito `obtenerCliente` en un lado y `ObtenerCliente` en otro. **VB no distingue mayúsculas**: para el compilador es el mismo identificador.
4. En otro archivo, un evento se conecta con `Handles btnFacturar.Click` y no encuentras ningún `+=`: no sabes de dónde sale la llamada.

Las consecuencias:

* errores de lógica que un desarrollador de C# no reconoce porque los operadores "se parecen";
* conversiones silenciosas que solo fallan en ejecución;
* tiempo perdido buscando código que sí existe, escrito de otra manera.

-----

## Cómo funciona

### 1. Lo esencial: es la misma plataforma

```text
Pedido.vb  ──vbc──┐
                  ├──► IL ──► mismo CLR, misma BCL, mismos paquetes NuGet
Pedido.cs  ──csc──┘
```

Una biblioteca en VB se referencia desde C# (y al revés) sin ningún problema. Una solución puede tener proyectos en los dos lenguajes. Por eso migrar de VB a C# puede hacerse **proyecto por proyecto**.

### 2. Equivalencias de sintaxis

| C# | VB.NET |
| --- | --- |
| `int total = 0;` | `Dim total As Integer = 0` |
| `var total = 0;` | `Dim total = 0` (con `Option Infer On`) |
| `string? nombre = null;` | `Dim nombre As String = Nothing` |
| `if (a && b) { } else if (c) { } else { }` | `If a AndAlso b Then ... ElseIf c Then ... Else ... End If` |
| `a \|\| b`, `!a` | `a OrElse b`, `Not a` |
| `a == b`, `a != b` | `a = b`, `a <> b` |
| `obj == null`, `obj != null` | `obj Is Nothing`, `obj IsNot Nothing` |
| `"Hola " + nombre` | `"Hola " & nombre` |
| `$"Total: {total}"` | `$"Total: {total}"` |
| `switch` | `Select Case ... Case ... Case Else ... End Select` |
| `foreach (var x in lista) { }` | `For Each x In lista ... Next` |
| `for (int i = 0; i < 10; i++)` | `For i = 0 To 9 ... Next` |
| `void Metodo() { }` | `Sub Metodo() ... End Sub` |
| `int Metodo() { return 1; }` | `Function Metodo() As Integer ... Return 1 ... End Function` |
| `ref int x` | `ByRef x As Integer` (`ByVal` es el valor por defecto) |
| `static class Utilidades` | `Module Utilidades` |
| `static` (miembro) | `Shared` |
| `this`, `base` | `Me`, `MyBase` |
| `List<string>` | `List(Of String)` |
| `x => x * 2` | `Function(x) x * 2` (`Sub(x) ...` si no devuelve valor) |
| `throw new InvalidOperationException()` | `Throw New InvalidOperationException()` |
| `catch (Exception ex)` | `Catch ex As Exception` |
| `using (var c = ...) { }` | `Using c = ... ... End Using` |
| `typeof(Pedido)`, `(Pedido)obj`, `obj as Pedido` | `GetType(Pedido)`, `CType(obj, Pedido)` / `DirectCast(obj, Pedido)`, `TryCast(obj, Pedido)` |
| `int / int` (división entera) | `\` para división entera; `/` es siempre división real |
| `// comentario` | `' comentario` |

### 3. Las diferencias que causan errores

| Diferencia | Riesgo | Qué hacer |
| --- | --- | --- |
| `And` / `Or` no cortocircuitan | `NullReferenceException`, efectos secundarios dobles | Usar `AndAlso` / `OrElse` en condiciones |
| `Option Strict Off` | Conversiones con pérdida y enlace tardío sin aviso | `Option Strict On` en cada archivo o en el proyecto (`<OptionStrict>On</OptionStrict>`) |
| No distingue mayúsculas | Búsquedas que no encuentran nada; parámetros que "pisan" propiedades con el mismo nombre | Búsquedas sin distinguir mayúsculas; `Me.Propiedad` para desambiguar |
| `/` siempre es división real | `7 / 2` da `3.5`, no `3` | `\` para división entera |
| Arreglos: `Dim a(5) As Integer` | Tiene **6** elementos (el número es el índice superior) | Recordarlo al portar bucles |
| `Nothing` en tipos de valor | `Dim n As Integer = Nothing` da `0`, no "sin valor" | `Integer?` para un valor nulo |
| `=` compara y asigna según el contexto | Confusión al leer condiciones | Leer con atención; no existe `==` |

### 4. Eventos: `Handles` y `WithEvents`

```vb
' Forma declarativa (típica en WinForms con VB)
Private WithEvents btnFacturar As Button

Private Sub btnFacturar_Click(sender As Object, e As EventArgs) Handles btnFacturar.Click
    Facturar()
End Sub

' Forma explícita, equivalente al += de C#
AddHandler pedido.Confirmado, AddressOf AlConfirmar
RemoveHandler pedido.Confirmado, AddressOf AlConfirmar

' Disparar un evento (equivale a Confirmado?.Invoke(this, EventArgs.Empty))
RaiseEvent Confirmado(Me, EventArgs.Empty)
```

Si no encuentras dónde se conecta un evento, busca `Handles` al final de la firma del método.

### 5. El espacio de nombres `Microsoft.VisualBasic`

El código VB antiguo usa funciones heredadas de VB6: `Len(s)`, `Mid(s, 2, 3)`, `CInt(x)`, `IsNothing(x)`, `MsgBox(...)`, `Format(...)`. Funcionan (están en `Microsoft.VisualBasic`, disponible también desde C#), pero conviene reemplazarlas por las equivalentes de .NET (`s.Length`, `s.Substring(1, 3)`, `Integer.Parse`, `Is Nothing`) al tocar el código: son más claras para cualquier desarrollador .NET y facilitan una conversión futura a C#. Ojo: `Mid` empieza a contar en **1**; `Substring`, en **0**.

### 6. El estado de VB.NET

* Desde 2020, Microsoft mantiene VB.NET con soporte en .NET moderno, pero **sin funciones nuevas del lenguaje**: no sigue la evolución de C# (no hay *records*, *pattern matching* moderno ni constructores primarios).
* En .NET moderno funciona para consola, bibliotecas, WinForms, WPF y *workers*. **ASP.NET Core con Razor no tiene soporte para VB**: las vistas y componentes van en C#.
* No admite *top-level statements*: siempre hay un `Module` con `Sub Main`.
* Para convertir código a C#, la extensión **Code Converter** (ICSharpCode) de Visual Studio traduce archivos o proyectos enteros; el resultado compila en la mayoría de los casos, pero conviene revisarlo (sobre todo `And`/`Or`, divisiones y funciones de `Microsoft.VisualBasic`).

### 7. ¿Y VB6?

VB6 (1998) no es .NET: genera código nativo y componentes COM, corre sobre su propio runtime y se programa con el IDE de Visual Studio 6. Si encuentras archivos `.frm`, `.bas`, `.cls` o `.vbp`, es VB6. Microsoft mantiene el runtime en Windows, pero el IDE no tiene soporte. Su migración a .NET es una reescritura (con herramientas comerciales de ayuda); no tiene relación directa con esta lección.

-----

## Ejemplo completo

Consola en VB.NET con .NET 10: `dotnet new console -lang VB -o FacturacionVb`. Reemplaza `Program.vb`:

```vb
Option Strict On
Option Explicit On
Option Infer On

Imports System.Globalization

Module Program
    ' VB no admite Async Sub Main: GetAwaiter().GetResult() en Main es la excepción aceptada
    Sub Main(args As String())
        MainAsync().GetAwaiter().GetResult()
    End Sub

    Private Async Function MainAsync() As Task
        Dim pedido As New Pedido("P-001", "Ana")
        AddHandler pedido.Confirmado, AddressOf AlConfirmar

        pedido.AgregarLinea("Teclado", 150D, 1)
        pedido.AgregarLinea("Mouse", 60D, 2)
        Console.WriteLine($"Total: {pedido.Total.ToString("F2", CultureInfo.InvariantCulture)}")

        ' AndAlso corta: con And, cliente.Length lanzaría NullReferenceException
        Dim cliente As String = Nothing
        If cliente IsNot Nothing AndAlso cliente.Length > 0 Then
            Console.WriteLine($"Cliente: {cliente}")
        Else
            Console.WriteLine("Sin cliente")
        End If

        Select Case pedido.Total
            Case Is >= 1000D
                Console.WriteLine("Envío prioritario")
            Case 100D To 999.99D
                Console.WriteLine("Envío estándar gratis")
            Case Else
                Console.WriteLine("Envío con costo")
        End Select

        ' LINQ con sintaxis de consulta y lambdas con Function(...)
        Dim caros = From l In pedido.Lineas
                    Where l.Subtotal > 100D
                    Select l.Producto
        Console.WriteLine("Líneas de más de 100: " & String.Join(", ", caros))

        Await pedido.ConfirmarAsync()

        Try
            pedido.AgregarLinea("Cable", 10D, 1)
        Catch ex As InvalidOperationException
            Console.WriteLine("Error: " & ex.Message)
        End Try

        Dim valor As Integer = 5
        Duplicar(valor)
        Console.WriteLine($"ByRef: {valor}")

        Console.WriteLine($"7 / 2 = {7 / 2} | 7 \ 2 = {7 \ 2}")
    End Function

    Private Sub AlConfirmar(sender As Object, e As EventArgs)
        Console.WriteLine("Evento: pedido confirmado")
    End Sub

    Private Sub Duplicar(ByRef numero As Integer)
        numero *= 2
    End Sub
End Module

Public Class LineaPedido
    Public Sub New(producto As String, precio As Decimal, cantidad As Integer)
        ' VB no distingue mayúsculas: "producto" y "Producto" son el mismo nombre; Me. desambigua
        Me.Producto = producto
        Me.Precio = precio
        Me.Cantidad = cantidad
    End Sub

    Public ReadOnly Property Producto As String
    Public ReadOnly Property Precio As Decimal
    Public ReadOnly Property Cantidad As Integer

    Public ReadOnly Property Subtotal As Decimal
        Get
            Return Precio * Cantidad
        End Get
    End Property
End Class

Public Class Pedido
    Private ReadOnly _lineas As New List(Of LineaPedido)
    Private _confirmado As Boolean

    Public Event Confirmado As EventHandler

    Public Sub New(codigo As String, cliente As String)
        Me.Codigo = codigo
        Me.Cliente = cliente
    End Sub

    Public ReadOnly Property Codigo As String
    Public ReadOnly Property Cliente As String

    Public ReadOnly Property Lineas As IReadOnlyList(Of LineaPedido)
        Get
            Return _lineas.AsReadOnly()
        End Get
    End Property

    Public ReadOnly Property Total As Decimal
        Get
            Return _lineas.Sum(Function(l) l.Subtotal)
        End Get
    End Property

    Public Sub AgregarLinea(producto As String, precio As Decimal, cantidad As Integer)
        If _confirmado Then
            Throw New InvalidOperationException("El pedido ya está confirmado.")
        End If
        _lineas.Add(New LineaPedido(producto, precio, cantidad))
    End Sub

    Public Async Function ConfirmarAsync() As Task
        Await Task.Delay(100)
        _confirmado = True
        RaiseEvent Confirmado(Me, EventArgs.Empty)
    End Function
End Class
```

Salida (con cultura `en-US`; con `es-AR`, por ejemplo, `3.5` se mostraría como `3,5`):

```text
Total: 270.00
Sin cliente
Envío estándar gratis
Líneas de más de 100: Teclado, Mouse
Evento: pedido confirmado
Error: El pedido ya está confirmado.
ByRef: 10
7 / 2 = 3.5 | 7 \ 2 = 3
```

Qué observar:

* Todo el programa usa tipos de .NET que ya conoces (`List(Of T)`, `Task`, `InvalidOperationException`, LINQ): cambia la sintaxis, no la plataforma.
* `AndAlso` evitó el `NullReferenceException` con `cliente` en `Nothing`; con `And`, el programa se habría caído en esa línea.
* `Select Case` admite rangos (`100D To 999.99D`) y comparaciones (`Is >= 1000D`), algo que en C# requiere patrones relacionales.
* `7 / 2` da `3.5` y `7 \ 2` da `3`: la división entera tiene su propio operador.
* Los constructores usan `Me.` porque el parámetro `producto` y la propiedad `Producto` son, para VB, el mismo nombre.
* `Option Strict On` arriba del archivo: sin él, el compilador aceptaría conversiones implícitas que fallan o pierden datos en ejecución.

-----

## Errores comunes

**1. Usar `And` / `Or` en condiciones.**
Qué pasa: `NullReferenceException` o métodos con efectos secundarios que se ejecutan aunque el primer lado ya decidió el resultado.
Por qué: `And` y `Or` evalúan siempre los dos operandos.
Arreglo: `AndAlso` y `OrElse` en condiciones; `And`/`Or` solo para operaciones de bits.

**2. Mantener `Option Strict Off`.**
Qué pasa: conversiones con pérdida (`Double` a `Integer`), enlace tardío sobre `Object` y errores que el compilador de C# habría detectado.
Por qué: era el valor por defecto en muchos proyectos y facilita escribir rápido.
Arreglo: activar `Option Strict On` en el proyecto y corregir los errores que aparecen (cada uno es un error real potencial).

**3. Olvidar que VB no distingue mayúsculas.**
Qué pasa: búsquedas que no encuentran usos; un parámetro con el mismo nombre que una propiedad la oculta.
Por qué: `Total`, `total` y `TOTAL` son el mismo identificador.
Arreglo: búsquedas sin distinguir mayúsculas y `Me.` para referirse a miembros de la clase.

**4. Portar divisiones y arreglos sin revisar.**
Qué pasa: al convertir a C#, `a / b` con enteros pasa de división real a entera, y `Dim a(5)` (6 elementos) se convierte en `new int[5]` (5 elementos).
Por qué: las reglas de ambos lenguajes difieren.
Arreglo: revisar cada `/` y cada declaración de arreglo en la conversión; los convertidores lo hacen bien casi siempre, pero no siempre.

**5. Funciones de VB6 sin revisar al convertir.**
Qué pasa: índices desplazados en uno (`Mid` cuenta desde 1) o comportamientos distintos ante `Nothing` (`Len(Nothing)` devuelve 0; `s.Length` lanza excepción).
Por qué: las funciones de `Microsoft.VisualBasic` replican reglas de VB6.
Arreglo: reemplazarlas por APIs de .NET con atención a esos detalles.

-----

## Según la versión de .NET

* **VB.NET 7.0 (2002):** primera versión sobre .NET Framework 1.0; rompe compatibilidad con VB6.
* **VB 9 (2008):** LINQ con sintaxis de consulta, inferencia de tipos (`Option Infer`), literales XML.
* **VB 11 (2012):** `Async` / `Await`.
* **VB 14 (2015):** interpolación de cadenas, `?.`, `NameOf`, propiedades autoimplementadas de solo lectura.
* **VB 16.9 (2020–2021):** última actualización relevante; consume `init` de C# pero no los declara.
* **2020:** Microsoft anuncia que VB.NET se mantiene estable en .NET moderno, sin nuevas funciones del lenguaje.

-----

## Cuándo sí y cuándo no

**Lee y mantén VB.NET cuando:** heredas un sistema escrito en VB. Con esta lección y tu C#, puedes corregir y extender código VB con seguridad.

**Convierte a C# cuando:** el sistema va a seguir creciendo, necesitas funciones modernas del lenguaje (C# 12+), ASP.NET Core con Razor o el equipo ya no tiene gente que domine VB. Hazlo proyecto por proyecto, aprovechando que los dos lenguajes conviven en una solución.

**No empieces proyectos nuevos en VB.NET cuando:** puedes elegir: el lenguaje no evoluciona y el ecosistema (ejemplos, plantillas, bibliotecas, ASP.NET Core) está en C#.

-----

## Resumen en 5 líneas

1. VB.NET compila al mismo IL y usa la misma BCL que C#: cambia la sintaxis, no la plataforma.
2. `AndAlso`/`OrElse` cortocircuitan; `And`/`Or` no.
3. `Option Strict On` siempre: sin él, el compilador deja pasar conversiones y enlace tardío peligrosos.
4. VB no distingue mayúsculas, `/` es división real y `Dim a(5)` tiene 6 elementos.
5. VB.NET está soportado pero no evoluciona: se mantiene y, si el sistema crece, se convierte a C# proyecto por proyecto.

-----

## Para profundizar

<details>
<summary>Convivencia de VB y C# en una misma solución</summary>

Una solución puede tener `Facturacion.Dominio.vbproj` y `Facturacion.Api.csproj`, y el proyecto C# referencia al de VB como a cualquier otro. Lo que no se puede es mezclar lenguajes **dentro** de un mismo proyecto. Una estrategia de migración habitual: los proyectos nuevos se crean en C#, los existentes en VB se convierten cuando hay que hacer cambios grandes en ellos, y la solución funciona durante todo el proceso.

</details>

<details>
<summary>Literales XML en VB</summary>

VB permite escribir XML directamente en el código y consultarlo con una sintaxis propia:

```vb
Dim pedido = <pedido codigo="P-001"><cliente>Ana</cliente></pedido>
Dim nombre = pedido.<cliente>.Value      ' "Ana"
Dim codigo = pedido.@codigo              ' "P-001"
```

C# no tiene equivalente directo (se usa `XElement` de `System.Xml.Linq`). Si lo encuentras al convertir código, es de las partes que los convertidores traducen a llamadas de LINQ to XML y conviene revisar a mano.

</details>

-----

## En entrevista

### Respuesta corta (junior)

VB.NET es otro lenguaje de .NET que compila al mismo IL que C#, así que usa las mismas bibliotecas. Las diferencias principales son de sintaxis, y hay que tener cuidado con `And` frente a `AndAlso` (el segundo cortocircuita) y con activar `Option Strict On`.

### Respuesta ampliada (semi-senior)

Leo VB.NET sin problema porque comparte plataforma con C#: traduzco `Dim`, `Sub`/`Function`, `Handles`, `Is Nothing` y lambdas con `Function(x)`. Presto atención a las diferencias que causan errores: `And`/`Or` sin cortocircuito, `Option Strict Off` con conversiones silenciosas, identificadores sin distinción de mayúsculas, `/` como división real y los límites de arreglos. Sé que VB está soportado en .NET moderno pero sin funciones nuevas y sin Razor en ASP.NET Core, así que para sistemas que crecen propongo convertir a C# proyecto por proyecto, con herramientas como Code Converter y revisión de los puntos delicados.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `And` y `AndAlso`?**
`AndAlso` cortocircuita como `&&`; `And` evalúa siempre ambos lados.

**2. ¿Una biblioteca VB se puede usar desde C#?**
Sí: compila a IL y se referencia como cualquier otro ensamblado.

**3. ¿VB.NET tiene futuro?**
Está soportado en .NET moderno, pero Microsoft no agrega funciones nuevas al lenguaje.

**4. ¿VB6 y VB.NET son lo mismo?**
No: VB6 es anterior a .NET, basado en COM; su migración es una reescritura.

-----

## Práctica

**Ejercicio 1.** Encuentra los tres problemas y corrige este método.

```vb
Public Function Promedio(notas As List(Of Integer)) As Integer
    If notas IsNot Nothing And notas.Count > 0 Then
        Return notas.Sum() / notas.Count
    End If
    Return 0
End Function
```

<details>
<summary>Solución</summary>

1. `And` no corta: con `notas` en `Nothing`, `notas.Count` lanza `NullReferenceException`.
2. `/` devuelve `Double`; con `Option Strict Off` se convierte en silencio a `Integer` **redondeando** (VB usa redondeo bancario), y con `Option Strict On` no compila.
3. El tipo de retorno `Integer` pierde los decimales del promedio.

```vb
Public Function Promedio(notas As List(Of Integer)) As Double
    If notas IsNot Nothing AndAlso notas.Count > 0 Then
        Return notas.Sum() / notas.Count
    End If
    Return 0
End Function
```

Si el negocio quisiera la división entera, se usaría `\`; lo importante es que sea una decisión explícita.

</details>

**Ejercicio 2.** Traduce a C# la clase `Pedido` del ejemplo completo.

<details>
<summary>Solución</summary>

```csharp
public class Pedido(string codigo, string cliente)
{
    private readonly List<LineaPedido> _lineas = [];
    private bool _confirmado;

    public event EventHandler? Confirmado;

    public string Codigo { get; } = codigo;
    public string Cliente { get; } = cliente;
    public IReadOnlyList<LineaPedido> Lineas => _lineas.AsReadOnly();
    public decimal Total => _lineas.Sum(l => l.Subtotal);

    public void AgregarLinea(string producto, decimal precio, int cantidad)
    {
        if (_confirmado)
        {
            throw new InvalidOperationException("El pedido ya está confirmado.");
        }
        _lineas.Add(new LineaPedido(producto, precio, cantidad));
    }

    public async Task ConfirmarAsync()
    {
        await Task.Delay(100);
        _confirmado = true;
        Confirmado?.Invoke(this, EventArgs.Empty);
    }
}
```

`RaiseEvent` se convierte en `Confirmado?.Invoke(...)` (en VB, `RaiseEvent` ya contempla que no haya suscriptores). Las propiedades `ReadOnly` asignadas en el constructor pasan a propiedades `{ get; }` inicializadas desde el constructor primario, algo que VB no tiene.

</details>

-----

## Siguiente lección

Terminaste el módulo. Continúa con [Despliegue en IIS](../02-iis-y-migracion/01-Despliegue%20en%20IIS.md) o vuelve al [índice](README.md).
