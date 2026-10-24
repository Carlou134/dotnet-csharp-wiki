# Windows Forms y DevExpress

## En una frase

Windows Forms (WinForms) es el framework de escritorio para Windows basado en formularios, controles y eventos; a diferencia de Web Forms, **sigue vivo en .NET moderno**, y sus problemas típicos no son de la tecnología sino del código: lógica en los manejadores de eventos, hilos que tocan controles y suites comerciales como DevExpress de las que depende media aplicación.

-----

## Antes de empezar

Conviene que ya sepas:

* Eventos y delegados: [Delegados y eventos](../../01-csharp-core-and-runtime/09-delegados-y-eventos/README.md).
* `async`/`await` y por qué `async void` solo se admite en eventos de interfaz: [Asincronía y archivos](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).
* Plataformas y TFM: [.NET Framework frente a .NET moderno](01-NET%20Framework%20frente%20a%20NET%20moderno.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Windows Forms (WinForms):** framework de interfaces de escritorio para Windows basado en formularios (`Form`) y controles.
* **Diseñador:** editor visual de Visual Studio que genera el código de los controles en `Form1.Designer.cs` (`InitializeComponent`).
* **Hilo de interfaz (*UI thread*):** el único hilo que puede tocar los controles; procesa los mensajes de la ventana.
* **`Invoke` / `InvokeAsync`:** métodos de `Control` para ejecutar código en el hilo de interfaz desde otro hilo.
* **Enlace de datos (*data binding*):** conexión entre controles y objetos (`BindingSource`, `BindingList<T>`) para que los cambios se reflejen solos.
* **MVP (*Model-View-Presenter*):** patrón que saca la lógica del formulario a un presentador que se puede probar sin interfaz.
* **DevExpress:** suite comercial de controles (grillas, reportes, editores) para WinForms, WPF, Web Forms, Blazor y más.

-----

## El problema

Una aplicación de escritorio de facturación, en WinForms con DevExpress, lleva 12 años en producción. El formulario principal:

```csharp
private void btnCargar_Click(object sender, EventArgs e)
{
    var conexion = new SqlConnection("Server=...;Password=...");    // cadena en el código
    var tabla = new DataTable();
    new SqlDataAdapter("SELECT * FROM Facturas WHERE Cliente = '" + txtCliente.Text + "'", conexion).Fill(tabla);
    gridControl1.DataSource = tabla;

    foreach (DataRow fila in tabla.Rows)
    {
        if ((decimal)fila["Total"] > 10000) { /* 80 líneas de reglas de descuento */ }
    }
}
```

Lo que pasa:

1. **La ventana se congela** ("No responde") los 20 segundos que tarda la consulta: todo corre en el hilo de interfaz.
2. Alguien lo "arregla" con `Task.Run(...)` y ahora la aplicación se cae con `InvalidOperationException: Cross-thread operation not valid: Control 'gridControl1' accessed from a thread other than the thread it was created on.`
3. La regla de descuentos vive dentro de un `Click`: no se puede probar ni reutilizar en la API nueva que pide el negocio.
4. Inyección SQL desde un `TextBox` y la contraseña de la base dentro del ejecutable que se instala en 300 PCs.
5. Actualizar DevExpress de la versión 19.2 a la actual requiere renovar licencias, actualizar todas sus DLL a la vez y revisar cambios de API en decenas de formularios.

Las consecuencias:

* una interfaz que se congela o se cae según quién "optimizó";
* reglas de negocio prisioneras de los formularios;
* secretos y SQL inseguro distribuidos en cada instalación;
* una dependencia comercial que condiciona cualquier actualización.

-----

## Cómo funciona

### 1. El modelo: formularios, controles y eventos

```csharp
var boton = new Button { Text = "Cargar", Dock = DockStyle.Top };
boton.Click += async (s, e) => await CargarAsync();
formulario.Controls.Add(boton);
```

* Un `Form` contiene controles; cada control expone eventos (`Click`, `TextChanged`, `SelectionChanged`).
* Con el diseñador de Visual Studio, el código de los controles se genera en `Form1.Designer.cs` dentro de `InitializeComponent()`. **No se edita a mano**: el diseñador lo reescribe.
* La lógica va en `Form1.cs` (los manejadores de eventos)... y ahí es donde suele acumularse todo.

### 2. Un solo hilo toca la interfaz

```text
Hilo de interfaz: [mensaje: clic] → btnCargar_Click → [mensaje: repintar] → [mensaje: tecla] → ...
                         │
                         └─ si aquí se hace una consulta de 20 s, ningún otro mensaje se procesa:
                            la ventana "No responde"
```

| Forma | ¿Congela? | ¿Seguro con los controles? |
| --- | --- | --- |
| Consulta síncrona en el `Click` | Sí | Sí |
| `Task.Run` que toca controles | No | **No**: `InvalidOperationException` (cross-thread) |
| `async` + `await` de E/S en el `Click` | No | Sí: después del `await` se vuelve al hilo de interfaz |
| Trabajo en otro hilo + `Invoke`/`InvokeAsync` para tocar controles | No | Sí |

El patrón moderno es el tercero:

```csharp
private async void btnCargar_Click(object sender, EventArgs e)   // async void: se admite en eventos de UI
{
    btnCargar.Enabled = false;
    try
    {
        var facturas = await _servicio.ListarAsync(txtCliente.Text);   // no bloquea el hilo de interfaz
        bindingSource.DataSource = facturas;                            // de vuelta en el hilo de interfaz
    }
    catch (Exception ex)
    {
        MessageBox.Show(this, ex.Message, "Error");   // una excepción en async void sin capturar cierra la app
    }
    finally
    {
        btnCargar.Enabled = true;
    }
}
```

`await` captura el contexto de sincronización de WinForms y retoma la ejecución en el hilo de interfaz, por eso se pueden tocar controles después. Y como es `async void`, **todo** debe ir en `try`/`catch`: una excepción que escape termina la aplicación.

### 3. Enlace de datos

```csharp
var pedidos = new BindingList<PedidoFila>(lista);
var origen = new BindingSource { DataSource = pedidos };
grilla.DataSource = origen;
```

* `BindingList<T>` avisa a la grilla cuando se agregan o quitan elementos.
* Para que la grilla refleje el cambio de **una propiedad** de un elemento, la clase debe implementar `INotifyPropertyChanged`, o hay que pedir el repintado (`origen.ResetBindings(false)`).
* `DataTable` con `SqlDataAdapter` es el estilo antiguo: funciona, pero sin tipos ni separación entre datos y lógica.

### 4. Sacar la lógica del formulario: MVP

```text
Vista (Form)                 Presentador                         Modelo / servicios
 - controles                  - recibe eventos de la vista        - reglas de negocio
 - expone IVistaPedidos  ◄──► - llama a servicios                 - acceso a datos
 - sin lógica                 - le dice a la vista qué mostrar
```

El formulario implementa una interfaz (`IVistaPedidos { void MostrarPedidos(...); event EventHandler CargarSolicitado; }`) y el presentador trabaja contra esa interfaz. Así, la lógica se prueba con una vista falsa, sin abrir ventanas. Aun sin MVP completo, el mínimo es: **el formulario llama a un servicio; el servicio no conoce controles**.

### 5. WinForms en .NET moderno

| | .NET Framework 4.8 | .NET 10 |
| --- | --- | --- |
| Proyecto | Clásico o estilo SDK con `net48` | `<TargetFramework>net10.0-windows</TargetFramework>` + `<UseWindowsForms>true</UseWindowsForms>` |
| Alta resolución (DPI) | Limitada y propensa a problemas | `PerMonitorV2` y mejoras continuas |
| Novedades | Ninguna | Async (`Control.InvokeAsync`, .NET 9), modo oscuro (experimental desde .NET 9) |
| Plataforma | Windows | Windows (WinForms no es multiplataforma, ni en .NET moderno) |

Migrar una aplicación WinForms de Framework a .NET moderno suele ser **mucho más simple** que migrar Web Forms: el modelo es el mismo. Los bloqueos habituales son controles de terceros sin versión para .NET moderno, `System.Web` usado indebidamente o APIs eliminadas (Remoting, `AppDomain`).

### 6. DevExpress

```csharp
// WinForms: GridControl con su vista (requiere licencia y paquetes de DevExpress)
gridControl1.DataSource = facturas;
gridView1.Columns["Total"].DisplayFormat.FormatType = DevExpress.Utils.FormatType.Numeric;
gridView1.Columns["Total"].DisplayFormat.FormatString = "c2";
gridView1.FocusedRowChanged += (s, e) => MostrarDetalle(gridView1.GetFocusedRow() as FacturaFila);
```

| Familia | Plataforma | Prefijos típicos |
| --- | --- | --- |
| WinForms | Escritorio | `GridControl`/`GridView`, `XtraEditors`, `XtraReports`, `TextEdit`, `LookUpEdit` |
| ASP.NET Web Forms | Web clásica | `ASPxGridView`, `ASPxComboBox`, `ASPxCallbackPanel` |
| Blazor / ASP.NET Core | Web moderna | `DxGrid`, `DxComboBox` |

Lo que tienes que saber al mantener o migrar un sistema con DevExpress:

* **Licencia por desarrollador** (suscripción anual). Las versiones instaladas en producción no requieren licencia en cada PC, pero compilar y actualizar sí.
* **Todas las DLL de DevExpress deben tener la misma versión** (por ejemplo, 23.2.x). Mezclar versiones produce errores de carga en tiempo de ejecución.
* Los paquetes vienen de un **feed NuGet privado** con una clave personal, o del instalador. Sin acceso, el proyecto no compila en otra máquina.
* El código de negocio suele quedar **atado a los eventos de la grilla** (`CustomRowCellEdit`, `CustomColumnDisplayText`). Al migrar, esa lógica hay que extraerla antes de cambiar de plataforma.
* DevExpress tiene versiones para .NET moderno y para Blazor, con APIs **parecidas pero no iguales**: pasar de `ASPxGridView` (Web Forms) a `DxGrid` (Blazor) es una reescritura de la pantalla, no un cambio de nombre.

-----

## Ejemplo completo

Aplicación WinForms en .NET 10 (solo Windows), con el formulario armado en código para que se pueda copiar entero. Crea el proyecto con `dotnet new winforms -o Pedidos.Escritorio` y reemplaza `Program.cs` (y borra `Form1.cs` y `Form1.Designer.cs`).

`Program.cs`:

```csharp
using System.ComponentModel;

namespace Pedidos.Escritorio;

internal static class Program
{
    [STAThread]
    private static void Main()
    {
        ApplicationConfiguration.Initialize();
        Application.Run(new FormularioPedidos(new ServicioPedidos()));
    }
}

public sealed class FormularioPedidos : Form
{
    private readonly ServicioPedidos _servicio;
    private readonly BindingSource _origen = new();
    private readonly DataGridView _grilla = new() { Dock = DockStyle.Fill, ReadOnly = true, AutoGenerateColumns = true };
    private readonly Button _botonCargar = new() { Text = "Cargar pedidos", Dock = DockStyle.Top, Height = 36 };
    private readonly Button _botonConfirmar = new() { Text = "Confirmar seleccionado", Dock = DockStyle.Top, Height = 36 };
    private readonly Label _estado = new() { Dock = DockStyle.Bottom, Height = 24, Text = "Listo." };

    public FormularioPedidos(ServicioPedidos servicio)
    {
        _servicio = servicio;
        Text = "Pedidos";
        Width = 520;
        Height = 360;

        _grilla.DataSource = _origen;
        Controls.Add(_grilla);            // Fill primero; los Dock Top/Bottom se acomodan alrededor
        Controls.Add(_botonConfirmar);
        Controls.Add(_botonCargar);
        Controls.Add(_estado);

        _botonCargar.Click += BotonCargar_Click;
        _botonConfirmar.Click += BotonConfirmar_Click;
    }

    // async void: la única excepción admitida, porque es un manejador de evento de interfaz
    private async void BotonCargar_Click(object? sender, EventArgs e)
    {
        _botonCargar.Enabled = false;
        _estado.Text = "Cargando...";
        try
        {
            List<PedidoFila> pedidos = await _servicio.ListarAsync();   // la ventana sigue respondiendo
            _origen.DataSource = new BindingList<PedidoFila>(pedidos);  // de vuelta en el hilo de interfaz
            _estado.Text = $"{pedidos.Count} pedidos cargados.";
        }
        catch (Exception ex)
        {
            _estado.Text = "Error al cargar.";
            MessageBox.Show(this, ex.Message, "Error", MessageBoxButtons.OK, MessageBoxIcon.Error);
        }
        finally
        {
            _botonCargar.Enabled = true;
        }
    }

    private async void BotonConfirmar_Click(object? sender, EventArgs e)
    {
        if (_origen.Current is not PedidoFila seleccionado)
        {
            _estado.Text = "Selecciona un pedido.";
            return;
        }

        try
        {
            await _servicio.ConfirmarAsync(seleccionado.Codigo);
            seleccionado.Estado = "Confirmado";
            _origen.ResetCurrentItem();   // la fila no notifica cambios: se pide el repintado
            _estado.Text = $"{seleccionado.Codigo} confirmado.";
        }
        catch (InvalidOperationException ex)
        {
            _estado.Text = ex.Message;
        }
    }
}

public sealed class PedidoFila
{
    public required string Codigo { get; init; }
    public required string Cliente { get; init; }
    public string Estado { get; set; } = "Pendiente";
}

// La lógica no conoce controles: se puede probar y reutilizar en otra interfaz
public sealed class ServicioPedidos
{
    private readonly List<PedidoFila> _pedidos =
    [
        new() { Codigo = "P-001", Cliente = "Ana" },
        new() { Codigo = "P-002", Cliente = "Luis", Estado = "Confirmado" },
        new() { Codigo = "P-003", Cliente = "Marta" }
    ];

    public async Task<List<PedidoFila>> ListarAsync()
    {
        await Task.Delay(1500);   // simula una consulta lenta a la base o a una API
        return [.. _pedidos];
    }

    public async Task ConfirmarAsync(string codigo)
    {
        await Task.Delay(300);
        PedidoFila pedido = _pedidos.Single(p => p.Codigo == codigo);
        if (pedido.Estado == "Confirmado")
        {
            throw new InvalidOperationException($"{codigo} ya estaba confirmado.");
        }
        pedido.Estado = "Confirmado";
    }
}
```

Comportamiento al ejecutarlo con `dotnet run`:

```text
Inicio                        → ventana "Pedidos", grilla vacía, estado "Listo."
Clic en "Cargar pedidos"      → el botón se deshabilita, estado "Cargando..."; la ventana se puede mover y redimensionar
Después de 1,5 s              → grilla con P-001/Ana/Pendiente, P-002/Luis/Confirmado, P-003/Marta/Pendiente
                                estado "3 pedidos cargados."
Seleccionar P-001, "Confirmar"→ la fila pasa a Confirmado; estado "P-001 confirmado."
Seleccionar P-002, "Confirmar"→ estado "P-002 ya estaba confirmado."
```

Qué observar:

* Durante los 1,5 s de carga la ventana **responde**: `await` libera el hilo de interfaz. Con una llamada síncrona, aparecería "No responde".
* Después del `await` se modifican `_origen` y `_estado` sin `Invoke`: el contexto de sincronización devolvió la ejecución al hilo de interfaz.
* Los manejadores son `async void` y capturan todas las excepciones esperables: una excepción sin capturar en un `async void` cierra la aplicación.
* `PedidoFila` no implementa `INotifyPropertyChanged`, así que después de cambiar `Estado` se llama a `ResetCurrentItem()` para que la grilla repinte la fila.
* `ServicioPedidos` no sabe que existe una ventana. Con DevExpress, cambiaría el `DataGridView` por un `GridControl`, pero el servicio sería el mismo.

-----

## Errores comunes

**1. Trabajo largo en el hilo de interfaz.**
Qué pasa: la ventana se congela y Windows la marca como "No responde".
Por qué: el hilo que pinta la ventana está ocupado esperando la consulta.
Arreglo: `async`/`await` con APIs asíncronas de E/S; `Task.Run` solo para trabajo de CPU.

**2. Tocar controles desde otro hilo.**
Qué pasa: `InvalidOperationException: Cross-thread operation not valid: Control '...' accessed from a thread other than the thread it was created on.`
Por qué: los controles solo pueden usarse desde el hilo que los creó.
Arreglo: volver al hilo de interfaz con `await` (sin `ConfigureAwait(false)` en el código del formulario) o con `Invoke`/`InvokeAsync`.

**3. `async void` sin `try`/`catch`.**
Qué pasa: una excepción en un manejador cierra la aplicación entera.
Por qué: no hay un `Task` que la capture; se relanza en el contexto de sincronización.
Arreglo: `try`/`catch` en cada manejador `async void`, y un manejador global (`Application.ThreadException`) como red de seguridad.

**4. Editar `Form1.Designer.cs` a mano.**
Qué pasa: los cambios desaparecen la próxima vez que alguien abre el diseñador, o el diseñador deja de abrir el formulario.
Por qué: ese archivo lo regenera Visual Studio.
Arreglo: cambios visuales desde el diseñador; lógica en `Form1.cs`.

**5. Lógica de negocio en los eventos (y en los eventos de la grilla de DevExpress).**
Qué pasa: reglas imposibles de probar y que hay que reescribir al cambiar de interfaz.
Por qué: el modelo de eventos invita a escribir todo en `Click` o en `CustomRowCellEdit`.
Arreglo: servicios sin dependencias de la interfaz; el formulario solo traduce eventos en llamadas.

**6. Mezclar versiones de DevExpress.**
Qué pasa: `FileLoadException` o `TypeLoadException` al abrir formularios.
Por qué: los ensamblados de DevExpress dependen de su misma versión exacta.
Arreglo: actualizar todos los paquetes de DevExpress juntos, con su herramienta de migración de proyectos.

-----

## Según la versión de .NET

* **.NET Framework 1.0 (2002):** Windows Forms.
* **.NET Framework 2.0 (2005):** `DataGridView`, `BindingSource`, `ToolStrip`.
* **.NET Framework 4.5 (2012):** `async`/`await` en manejadores de eventos.
* **.NET Core 3.0 (2019):** WinForms llega a .NET Core, solo en Windows y open source.
* **.NET 5–8:** diseñador para proyectos .NET moderno en Visual Studio, mejoras de DPI y accesibilidad.
* **.NET 9:** `Control.InvokeAsync` y modo oscuro experimental.
* **.NET 10:** continúa la modernización (mejoras del diseñador, de DPI y de las APIs asíncronas).

-----

## Cuándo sí y cuándo no

**Usa o mantén WinForms cuando:** la aplicación es de escritorio para Windows, interna, con formularios de datos (ABM, reportes), y el equipo la conoce. Para eso sigue siendo productivo, y en .NET 10 tiene soporte activo.

**Migra de .NET Framework a .NET 10 cuando:** quieres soporte de largo plazo, mejor rendimiento y DPI, y los controles de terceros tienen versión para .NET moderno. Suele ser una migración razonable.

**No elijas WinForms cuando:** la aplicación debe correr en macOS, Linux, web o móviles (considera Blazor, MAUI o Avalonia), o la interfaz necesita un diseño visual muy moderno (WPF o WinUI ofrecen más).

-----

## Resumen en 5 líneas

1. WinForms es escritorio para Windows con formularios, controles y eventos, y sigue soportado en .NET 10.
2. Solo el hilo de interfaz toca los controles: `async`/`await` para no congelar la ventana, `Invoke` si vienes de otro hilo.
3. `async void` solo en manejadores de eventos, siempre con `try`/`catch`.
4. La lógica va en servicios sin controles (idealmente con MVP); el formulario solo traduce eventos.
5. DevExpress: licencia por desarrollador, todas las DLL en la misma versión y lógica atrapada en eventos de la grilla que hay que extraer antes de migrar.

-----

## Para profundizar

<details>
<summary>Migrar WinForms de .NET Framework a .NET 10</summary>

Los pasos habituales: convertir el `.csproj` a estilo SDK con `net10.0-windows` y `UseWindowsForms`, migrar `packages.config` a `PackageReference`, reemplazar dependencias sin versión para .NET moderno, revisar `App.config` (la configuración de `System.Configuration` sigue disponible con el paquete `System.Configuration.ConfigurationManager`) y probar en pantallas con distinta escala de DPI. Las herramientas de modernización de Microsoft (Upgrade Assistant, en proceso de ser reemplazado por la modernización asistida con GitHub Copilot) automatizan parte del trabajo.

</details>

<details>
<summary>WPF: el otro escritorio de .NET</summary>

WPF (2006) usa XAML, gráficos vectoriales y un enlace de datos mucho más potente, y suele combinarse con el patrón MVVM. También existe en .NET moderno, solo para Windows. Si heredas una aplicación WPF, los conceptos de esta lección (hilo de interfaz, `async void` en eventos, lógica fuera de la vista) aplican igual; cambian el marcado (XAML) y el mecanismo de enlace (`INotifyPropertyChanged`, comandos).

</details>

-----

## En entrevista

### Respuesta corta (junior)

WinForms es el framework de escritorio para Windows con formularios y controles que manejan eventos. Sigue existiendo en .NET moderno. Para no congelar la ventana uso `async`/`await` en los manejadores de eventos, y no toco controles desde otros hilos.

### Respuesta ampliada (semi-senior)

En WinForms un solo hilo procesa los mensajes de la ventana: uso `async`/`await` en los manejadores (`async void` con `try`/`catch`, la única excepción a esa regla), `Task.Run` solo para CPU y `Invoke`/`InvokeAsync` si vengo de otro hilo. Saco la lógica de los formularios a servicios o a presentadores (MVP) para poder probarla y reutilizarla. Sé que migrar WinForms de Framework a .NET 10 suele ser viable porque el modelo es el mismo, y que los bloqueos son los controles de terceros. Con DevExpress, tengo en cuenta la licencia por desarrollador, que todas las DLL vayan en la misma versión y que la lógica metida en eventos de la grilla hay que extraerla antes de cambiar de plataforma.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué se congela una ventana de WinForms?**
Porque una operación larga ocupa el hilo de interfaz y no se procesan los mensajes de repintado ni de entrada.

**2. ¿Cuándo está bien `async void`?**
En manejadores de eventos de interfaz, siempre con manejo de excepciones.

**3. ¿WinForms funciona en .NET 10?**
Sí, solo en Windows.

**4. ¿Qué cuidarías al actualizar DevExpress?**
Actualizar todas sus DLL a la misma versión, tener licencia vigente y revisar los cambios de API en los formularios.

-----

## Práctica

**Ejercicio 1.** Este código se cae con la excepción de *cross-thread*. Corrígelo sin cambiar `CalcularTotales`, que es CPU intensivo y síncrono.

```csharp
private void btnCalcular_Click(object sender, EventArgs e)
{
    Task.Run(() =>
    {
        decimal total = CalcularTotales();
        lblTotal.Text = total.ToString("C");
    });
}
```

<details>
<summary>Solución</summary>

```csharp
private async void btnCalcular_Click(object sender, EventArgs e)
{
    btnCalcular.Enabled = false;
    try
    {
        decimal total = await Task.Run(CalcularTotales);   // el cálculo, en otro hilo
        lblTotal.Text = total.ToString("C");                // el control, en el hilo de interfaz
    }
    catch (Exception ex)
    {
        MessageBox.Show(this, ex.Message, "Error");
    }
    finally
    {
        btnCalcular.Enabled = true;
    }
}
```

`Task.Run` sigue siendo correcto porque el trabajo es de CPU; lo que cambia es que el resultado se usa **después del `await`**, cuando ya se volvió al hilo de interfaz. Además, deshabilitar el botón evita lanzar el cálculo dos veces.

</details>

**Ejercicio 2.** Implementa `INotifyPropertyChanged` en `PedidoFila` para que la grilla se actualice sola al cambiar `Estado`, y quita el `ResetCurrentItem()` del ejemplo.

<details>
<summary>Solución</summary>

```csharp
public sealed class PedidoFila : INotifyPropertyChanged
{
    private string _estado = "Pendiente";

    public required string Codigo { get; init; }
    public required string Cliente { get; init; }

    public string Estado
    {
        get => _estado;
        set
        {
            if (_estado == value) return;
            _estado = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Estado)));
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}
```

Con `BindingList<PedidoFila>`, la lista escucha `PropertyChanged` de cada elemento y avisa a la grilla, que repinta la fila sin `ResetCurrentItem()`. Ojo: el evento debe dispararse en el hilo de interfaz; si `Estado` cambia desde otro hilo, vuelve primero con `await` o `InvokeAsync`.

</details>

-----

## Siguiente lección

[WCF y sus reemplazos](04-WCF%20y%20sus%20reemplazos.md)
