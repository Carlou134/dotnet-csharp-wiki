# ASP.NET Web Forms

## En una frase

Web Forms (2002) simulaba programar una aplicación de escritorio en la web: controles de servidor con eventos (`Button_Click`), un ciclo de vida de página y el estado guardado en un campo oculto (**ViewState**) que viaja en cada *postback*; para mantenerlo necesitas entender ese ciclo, y para migrarlo, saber que no tiene equivalente en .NET moderno.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo reconocer un proyecto de .NET Framework: [.NET Framework frente a .NET moderno](01-NET%20Framework%20frente%20a%20NET%20moderno.md).
* HTTP sin estado, formularios y verbos: [MVC, routing y vistas Razor](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/03-MVC%20routing%20y%20vistas%20Razor.md).
* Eventos y delegados en C#: [Delegados y eventos](../../01-csharp-core-and-runtime/09-delegados-y-eventos/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Web Form (`.aspx`):** página con marcado HTML y controles de servidor, compilada como una clase que hereda de `System.Web.UI.Page`.
* **Code-behind (`.aspx.cs`):** archivo de C# (o VB) con la lógica de la página: manejadores de eventos y `Page_Load`.
* **Control de servidor:** elemento con `runat="server"` (`<asp:Button>`, `<asp:GridView>`) que el servidor convierte en HTML y que dispara eventos en C#.
* **Postback:** el formulario se envía (POST) a la misma página para procesar un evento de un control.
* **ViewState:** estado de los controles serializado en el campo oculto `__VIEWSTATE`, que viaja al navegador y vuelve en cada postback.
* **Ciclo de vida de la página:** secuencia de etapas (`Init`, `Load`, eventos de controles, `PreRender`, `Render`) que se ejecuta en cada petición.
* **Master page (`.master`):** plantilla común (cabecera, menú) que comparten varias páginas.

-----

## El problema

Te toca corregir un error en una pantalla de pedidos hecha en Web Forms. El usuario elige un cliente en una lista desplegable, toca "Buscar" y la lista vuelve **siempre** al primer cliente: la búsqueda nunca usa el elegido. El código:

```csharp
protected void Page_Load(object sender, EventArgs e)
{
    ddlClientes.DataSource = _servicio.ListarClientes();   // se ejecuta en CADA petición, también en el postback
    ddlClientes.DataBind();
}

protected void btnBuscar_Click(object sender, EventArgs e)
{
    gvPedidos.DataSource = _servicio.PedidosDe(ddlClientes.SelectedValue);   // siempre el primero
    gvPedidos.DataBind();
}
```

Lo que pasa, y por qué cuesta verlo si vienes de ASP.NET Core:

1. **`Page_Load` corre antes del evento del botón**, en cada petición. Al volver a enlazar la lista, se pierde la selección que el usuario envió.
2. La página pesa 900 KB: la grilla de 2000 filas está guardada **completa** en `__VIEWSTATE`, que sube y baja en cada clic.
3. La lógica de negocio (descuentos, validaciones) está en los manejadores de eventos, mezclada con la manipulación de controles. No se puede probar sin levantar la página.
4. En otra pantalla, un `SqlDataSource` del marcado arma el SQL concatenando un `TextBox`: inyección SQL desde el formulario.

Las consecuencias:

* errores que dependen del orden del ciclo de vida, invisibles para quien no lo conoce;
* páginas pesadas por un estado que nadie dimensionó;
* lógica atrapada en la capa de presentación, lo que encarece cualquier migración.

-----

## Cómo funciona

### 1. El modelo: eventos de escritorio sobre HTTP

```text
GET /Pedidos.aspx
  servidor: crea la página → Init → Load → Render (HTML + __VIEWSTATE oculto)
                                                     │
navegador: el usuario elige "Pendientes" en la lista │
                                                     ▼
POST /Pedidos.aspx  (formulario completo + __VIEWSTATE + __EVENTTARGET=ddlEstado)
  servidor: crea la página DE NUEVO → Init → carga ViewState → Load (IsPostBack = true)
            → evento ddlEstado_SelectedIndexChanged → PreRender → Render (HTML nuevo + ViewState nuevo)
```

HTTP no tiene estado; Web Forms lo **simula** reconstruyendo la página en cada postback a partir del ViewState y de los valores del formulario. Cada clic que dispara un evento de servidor es un POST completo de la página.

### 2. El ciclo de vida (lo mínimo que hay que saber)

| Etapa | Qué pasa | Qué haces ahí |
| --- | --- | --- |
| `PreInit` | Se eligen master page y tema | Asignar master page dinámica |
| `Init` | Se crean los controles | **Crear controles dinámicos** (deben recrearse en cada petición) |
| Carga de ViewState y datos enviados | Los controles recuperan su estado y los valores del formulario | — |
| `Load` | `Page_Load` | Cargar datos **solo si `!IsPostBack`** |
| Eventos de controles | `Button_Click`, `SelectedIndexChanged`... | Reaccionar a la acción del usuario |
| `PreRender` | Últimos cambios antes de generar HTML | Ajustes finales de la interfaz |
| `Render` | Se genera el HTML y se guarda el ViewState | — |

La regla que resuelve el error del problema:

```csharp
protected void Page_Load(object sender, EventArgs e)
{
    if (!IsPostBack)          // solo en la primera carga (GET)
    {
        ddlClientes.DataSource = _servicio.ListarClientes();
        ddlClientes.DataBind();
    }
    // en los postbacks, la lista se reconstruye desde el ViewState y conserva la selección
}
```

### 3. ViewState: cómodo y caro

```html
<input type="hidden" name="__VIEWSTATE" id="__VIEWSTATE" value="/wEPDwUKLTE2..." />
```

* Guarda el estado de los controles (items de una lista, filas de una grilla, propiedades cambiadas) serializado en Base64.
* Viaja **en cada respuesta y en cada postback**: una grilla grande puede generar cientos de KB.
* Está firmado (MAC) para que el cliente no lo modifique; **nunca** desactives `EnableViewStateMac`. Con la clave de firma (`machineKey`) filtrada, un atacante puede fabricar un ViewState malicioso y ejecutar código en el servidor.
* Se puede desactivar por control (`EnableViewState="false"`) cuando el dato se vuelve a cargar en cada petición.

### 4. Controles de servidor y code-behind

```aspx
<asp:Button ID="btnConfirmar" runat="server" Text="Confirmar" OnClick="btnConfirmar_Click" />
```

* `runat="server"` hace que exista un campo `btnConfirmar` en la clase de la página (declarado en el archivo `.aspx.designer.cs`, que genera Visual Studio).
* El servidor genera el HTML; el `id` en el navegador puede ser distinto (`ctl00_MainContent_btnConfirmar`) por los contenedores. Para JavaScript, usa `ClientID` o `ClientIDMode="Static"`.
* El atributo `OnClick` conecta el evento con el método del code-behind.

### 5. Otras piezas que vas a encontrar

| Pieza | Para qué | Cuidado |
| --- | --- | --- |
| Master pages (`.master`) | Diseño común | El `id` de los controles cambia dentro del contenido |
| User controls (`.ascx`) | Componentes reutilizables | Mismo ciclo de vida, anidado |
| `UpdatePanel` (AJAX) | Postback parcial sin recargar la página | Sigue enviando el formulario y el ViewState completos |
| `Session` | Estado por usuario en el servidor | En modo `InProc` se pierde cuando IIS recicla el *application pool* |
| `Global.asax` | Eventos de aplicación (`Application_Start`, `Application_Error`) | Equivalente aproximado al `Program.cs` |
| `SqlDataSource` / `ObjectDataSource` | Enlace de datos declarativo en el marcado | SQL en la vista; revisar parámetros |
| `Response.Redirect(url)` | Redirigir | Lanza `ThreadAbortException` para cortar la página; usa `Response.Redirect(url, false)` + `Context.ApplicationInstance.CompleteRequest()` |

### 6. Seguridad que viene de fábrica (y que no hay que apagar)

* **Validación de peticiones:** rechaza entradas con `<script>` por defecto (`HttpRequestValidationException`). Desactivarla (`ValidateRequest="false"`) obliga a codificar toda la salida.
* **Event validation:** verifica que los valores enviados (por ejemplo, el item elegido de una lista) sean de los que el servidor generó.
* **Codificación:** `<%: valor %>` codifica HTML; `<%= valor %>` **no**. En controles, `Label.Text` no codifica: usa `Server.HtmlEncode`.

### 7. Migrar: no hay traducción directa

| Destino | Por qué encaja | Costo |
| --- | --- | --- |
| **Blazor Server** | Modelo mental parecido: componentes, eventos en C#, estado en el servidor | Reescribir las páginas; los controles de terceros (DevExpress, Telerik) tienen versiones Blazor |
| **Razor Pages** | Formularios simples, una página = un archivo + su modelo | Pasar a un modelo sin eventos de control: GET/POST explícitos |
| **API + frontend SPA** | Interfaces muy interactivas | Dos aplicaciones en lugar de una |

En todos los casos, el primer paso es el mismo: **sacar la lógica de negocio del code-behind** a clases normales (`netstandard2.0`), probarla y dejar las páginas como capa fina. Lo que quede en el code-behind se reescribe; lo que salió, se reutiliza. La migración por partes, con YARP y los *System.Web adapters*, se ve en [Estrategias de migración](../02-iis-y-migracion/02-Estrategias%20de%20migracion.md).

-----

## Ejemplo completo

Proyecto **ASP.NET Web Application (.NET Framework)**, plantilla *Web Forms*, con .NET Framework 4.8 en Visual Studio (Windows). C# 7.3, que es la versión por defecto para `net48`.

`Pedidos.aspx`:

```aspx
<%@ Page Language="C#" AutoEventWireup="true" CodeBehind="Pedidos.aspx.cs" Inherits="Tienda.Web.Pedidos" %>
<!DOCTYPE html>
<html>
<head runat="server">
    <title>Pedidos</title>
</head>
<body>
    <form id="form1" runat="server">
        Estado:
        <asp:DropDownList ID="ddlEstado" runat="server" AutoPostBack="true"
                          OnSelectedIndexChanged="ddlEstado_SelectedIndexChanged">
            <asp:ListItem Text="Todos" Value="" />
            <asp:ListItem Text="Pendientes" Value="Pendiente" />
            <asp:ListItem Text="Confirmados" Value="Confirmado" />
        </asp:DropDownList>

        <asp:GridView ID="gvPedidos" runat="server" AutoGenerateColumns="false"
                      DataKeyNames="Id" OnRowCommand="gvPedidos_RowCommand">
            <Columns>
                <asp:BoundField DataField="Codigo" HeaderText="Código" />
                <asp:BoundField DataField="Estado" HeaderText="Estado" />
                <asp:ButtonField CommandName="Confirmar" Text="Confirmar" />
            </Columns>
        </asp:GridView>

        <asp:Label ID="lblMensaje" runat="server" />
    </form>
</body>
</html>
```

`Pedidos.aspx.cs`:

```csharp
using System;
using System.Web.UI;
using System.Web.UI.WebControls;

namespace Tienda.Web
{
    public partial class Pedidos : Page
    {
        // Web Forms clásico no tiene DI integrada (4.7.2+ permite configurarla con WebObjectActivator)
        private readonly ServicioPedidos _servicio = new ServicioPedidos();

        protected void Page_Load(object sender, EventArgs e)
        {
            if (!IsPostBack)
            {
                CargarGrilla();
            }
        }

        protected void ddlEstado_SelectedIndexChanged(object sender, EventArgs e)
        {
            CargarGrilla();
        }

        protected void gvPedidos_RowCommand(object sender, GridViewCommandEventArgs e)
        {
            if (e.CommandName != "Confirmar") return;

            int fila = Convert.ToInt32(e.CommandArgument);
            int id = (int)gvPedidos.DataKeys[fila].Value;
            try
            {
                _servicio.Confirmar(id);
                lblMensaje.Text = "Pedido confirmado.";
            }
            catch (InvalidOperationException ex)
            {
                lblMensaje.Text = Server.HtmlEncode(ex.Message);   // Label.Text no codifica solo
            }
            CargarGrilla();
        }

        private void CargarGrilla()
        {
            gvPedidos.DataSource = _servicio.Listar(ddlEstado.SelectedValue);
            gvPedidos.DataBind();
        }
    }
}
```

`ServicioPedidos.cs` (la lógica fuera del code-behind, lista para moverse a una biblioteca compartida):

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace Tienda.Web
{
    public class PedidoFila
    {
        public int Id { get; set; }
        public string Codigo { get; set; }
        public string Estado { get; set; }
    }

    public class ServicioPedidos
    {
        // static: compartido por TODOS los usuarios del sitio (solo para el ejemplo; el lock lo protege)
        private static readonly object Bloqueo = new object();
        private static readonly List<PedidoFila> Datos = new List<PedidoFila>
        {
            new PedidoFila { Id = 1, Codigo = "P-001", Estado = "Pendiente" },
            new PedidoFila { Id = 2, Codigo = "P-002", Estado = "Confirmado" },
            new PedidoFila { Id = 3, Codigo = "P-003", Estado = "Pendiente" }
        };

        public List<PedidoFila> Listar(string estado)
        {
            lock (Bloqueo)
            {
                return Datos.Where(p => string.IsNullOrEmpty(estado) || p.Estado == estado)
                            .Select(p => new PedidoFila { Id = p.Id, Codigo = p.Codigo, Estado = p.Estado })
                            .ToList();
            }
        }

        public void Confirmar(int id)
        {
            lock (Bloqueo)
            {
                PedidoFila pedido = Datos.Single(p => p.Id == id);
                if (pedido.Estado == "Confirmado")
                {
                    throw new InvalidOperationException("El pedido " + pedido.Codigo + " ya estaba confirmado.");
                }
                pedido.Estado = "Confirmado";
            }
        }
    }
}
```

Comportamiento al ejecutarlo con IIS Express desde Visual Studio:

```text
GET  Pedidos.aspx                        → tabla con P-001 (Pendiente), P-002 (Confirmado), P-003 (Pendiente)
Elegir "Pendientes" en la lista          → postback automático; tabla con P-001 y P-003; la lista queda en "Pendientes"
"Confirmar" en P-001                     → postback; mensaje "Pedido confirmado."; con el filtro, la tabla muestra solo P-003
Elegir "Todos" y "Confirmar" en P-002    → mensaje "El pedido P-002 ya estaba confirmado."
```

El HTML generado incluye los campos ocultos `__VIEWSTATE`, `__EVENTVALIDATION` y, en los postbacks de la lista, `__EVENTTARGET` con el nombre del control que disparó el evento (su tamaño varía según el contenido de la grilla).

Qué observar:

* `Page_Load` carga la grilla solo con `!IsPostBack`; en los postbacks, la recarga ocurre **en el evento** que corresponde, después de que los controles recuperaron su estado.
* La lista desplegable conserva "Pendientes" después de cada postback gracias al ViewState: no hubo que guardar nada a mano.
* `DataKeyNames="Id"` guarda los ids de las filas en el ViewState; `DataKeys[fila]` los recupera sin exponer el id en el HTML visible.
* El mensaje de error se codifica con `Server.HtmlEncode` porque `Label.Text` escribe HTML crudo.
* `ServicioPedidos` no conoce `Page` ni controles: es el tipo de clase que se lleva tal cual a una biblioteca `netstandard2.0` cuando se migra a Blazor o Razor Pages.

-----

## Errores comunes

**1. Cargar datos en `Page_Load` sin `!IsPostBack`.**
Qué pasa: selecciones que vuelven al primer elemento, eventos de controles que no se disparan o que ven datos viejos.
Por qué: `Page_Load` corre en cada petición, antes de los eventos, y el enlace de datos pisa el estado enviado.
Arreglo: `if (!IsPostBack)` para la carga inicial; recargar en el manejador del evento cuando corresponda.

**2. ViewState gigante.**
Qué pasa: páginas de cientos de KB, lentas en redes móviles y en cada postback.
Por qué: grillas y listas grandes guardan todo su contenido en el ViewState.
Arreglo: `EnableViewState="false"` en controles que se recargan siempre, paginación en las grillas y no guardar objetos grandes en `ViewState[...]`.

**3. Lógica de negocio en el code-behind.**
Qué pasa: reglas duplicadas entre páginas, imposibles de probar y que hay que reescribir enteras al migrar.
Por qué: el modelo de eventos invita a escribir todo en `btnGuardar_Click`.
Arreglo: el code-behind traduce entre controles y servicios; las reglas viven en clases propias.

**4. Controles dinámicos creados en `Page_Load` o en un evento.**
Qué pasa: los controles desaparecen o pierden sus valores en el siguiente postback.
Por qué: el ViewState solo se aplica a controles que ya existen en esa etapa; los dinámicos hay que recrearlos en cada petición, en `Init`.
Arreglo: crear los controles dinámicos en `Page_Init` (o `OnInit`) en todas las peticiones.

**5. `Response.Redirect(url)` dentro de un `try`/`catch` general.**
Qué pasa: se registra una `ThreadAbortException` como error en cada redirección, o el código posterior se ejecuta de forma inesperada.
Por qué: `Redirect(url)` aborta el hilo para cortar la página.
Arreglo: `Response.Redirect(url, false)` seguido de `Context.ApplicationInstance.CompleteRequest()`.

**6. Desactivar las protecciones para "que funcione".**
Qué pasa: XSS (`ValidateRequest="false"` sin codificar la salida) o ejecución remota de código (`EnableViewStateMac="false"`, `machineKey` filtrada).
Por qué: las protecciones molestaban con un caso puntual.
Arreglo: resolver el caso puntual (codificar, validar) y mantener las protecciones activas.

-----

## Según la versión de .NET

* **ASP.NET 1.0 (2002):** Web Forms con .NET Framework 1.0.
* **ASP.NET 2.0 (2005):** master pages, temas, controles de datos (`GridView`, `SqlDataSource`), membresía.
* **ASP.NET AJAX (2007, integrado en 3.5):** `UpdatePanel` y `ScriptManager`.
* **ASP.NET 4.0 (2010):** `ClientIDMode` para controlar los `id` del HTML y ruteo de URLs para Web Forms.
* **ASP.NET 4.5 (2012):** enlace de datos con modelos tipados (`ItemType`, `SelectMethod`) y validación no intrusiva.
* **.NET Framework 4.7.2 (2018):** inyección de dependencias en páginas y controles mediante `WebObjectActivator`.
* **.NET Core / .NET moderno:** Web Forms no fue portado; Microsoft recomienda Blazor como destino natural.

-----

## Cuándo sí y cuándo no

**Mantén Web Forms cuando:** la aplicación es estable, interna, funciona en un Windows soportado y los cambios son pocos. Mejora lo que tocas (sacar lógica del code-behind, reducir ViewState) sin reescribir todo.

**Planifica la migración cuando:** la aplicación sigue creciendo, el equipo no encuentra gente que la mantenga, necesitas otra plataforma (Linux, contenedores) o el ViewState y los postbacks limitan la experiencia de usuario.

**No escribas Web Forms nuevo cuando:** puedes elegir. Para una pantalla nueva en un sistema Web Forms, evalúa hacerla como Blazor o Razor Pages detrás de YARP, en lugar de agrandar lo heredado.

-----

## Resumen en 5 líneas

1. Web Forms simula eventos de escritorio sobre HTTP: cada evento de servidor es un postback que reconstruye la página.
2. El ciclo de vida importa: `Init` (controles dinámicos), `Load` (carga inicial con `!IsPostBack`), eventos, `Render`.
3. ViewState guarda el estado de los controles en un campo oculto firmado: cómodo, pero puede pesar mucho.
4. La lógica de negocio debe salir del code-behind; es lo único que se reutiliza al migrar.
5. No hay migración automática: el destino natural es Blazor, o Razor Pages para formularios simples.

-----

## Para profundizar

<details>
<summary>Web Forms y Blazor: parecidos y diferencias</summary>

Los dos tienen componentes con estado y eventos en C# (`OnClick`), lo que hace a Blazor el destino más natural para un equipo de Web Forms. Las diferencias son de fondo: Blazor no usa ViewState ni postbacks completos (en Blazor Server, el estado vive en el servidor y los cambios viajan por SignalR como diferencias del DOM; en WebAssembly, corre en el navegador), no tiene el ciclo de vida de página de Web Forms y sus componentes se componen de otra manera. Microsoft publicó la guía *Blazor for ASP.NET Web Forms developers* para ese salto.

</details>

<details>
<summary>Diagnosticar qué hay en el ViewState</summary>

El ViewState es Base64 de un formato binario de .NET (*LosFormatter*/*ObjectStateFormatter*). Con herramientas de decodificación de ViewState (extensiones del navegador o utilidades de línea de comandos) puedes ver qué controles aportan más tamaño. Activar el *trace* de la página (`Trace="true"` en la directiva `@ Page`, solo en desarrollo) muestra el tamaño del ViewState por control y los tiempos de cada etapa del ciclo de vida.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Web Forms es el framework web original de .NET Framework. Usa páginas `.aspx` con controles de servidor y eventos como `Button_Click`, y guarda el estado en el ViewState, un campo oculto que viaja en cada postback. Es importante usar `if (!IsPostBack)` en `Page_Load` para no recargar datos en cada evento.

### Respuesta ampliada (semi-senior)

Conozco el ciclo de vida (controles dinámicos en `Init`, carga inicial con `!IsPostBack` en `Load`, eventos, `PreRender`, `Render`) y por qué el ViewState hace pesadas las páginas con grillas grandes; lo desactivo por control cuando el dato se recarga siempre. Mantengo las protecciones (MAC del ViewState, validación de peticiones, event validation) y codifico la salida de los `Label`. Al mantener un sistema Web Forms, saco la lógica del code-behind a servicios que se pueden probar, en `netstandard2.0`, que es lo que después se reutiliza al migrar a Blazor (el destino más natural por el modelo de componentes y eventos) o a Razor Pages, idealmente por partes detrás de YARP.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es un postback?**
El envío del formulario a la misma página para procesar el evento de un control de servidor.

**2. ¿Para qué sirve `IsPostBack`?**
Para distinguir la primera carga (GET) de los envíos de eventos y no recargar datos que pisen el estado enviado.

**3. ¿Por qué el ViewState es un riesgo de seguridad?**
Si se desactiva su firma o se filtra la `machineKey`, un atacante puede enviar un ViewState malicioso que el servidor deserializa.

**4. ¿A qué migrarías una aplicación Web Forms?**
A Blazor por el modelo parecido de componentes y eventos, o a Razor Pages si son formularios simples; primero, sacando la lógica del code-behind.

-----

## Práctica

**Ejercicio 1.** Esta página crea un `TextBox` dinámico en el clic de un botón, y en el siguiente postback el valor escrito desaparece. Explica por qué y corrígelo.

```csharp
protected void btnAgregar_Click(object sender, EventArgs e)
{
    var txt = new TextBox { ID = "txtNota" };
    phNotas.Controls.Add(txt);
}
```

<details>
<summary>Solución</summary>

El control solo existe en la petición donde se hizo clic. En el postback siguiente, la página se reconstruye **sin** ese `TextBox`, así que no hay control que reciba el valor enviado ni el ViewState. Hay que recordar que se agregó (por ejemplo, en el ViewState de la página) y recrearlo en **cada** petición, temprano:

```csharp
private bool MostrarNota
{
    get { return ViewState["MostrarNota"] as bool? ?? false; }
    set { ViewState["MostrarNota"] = value; }
}

protected override void LoadViewState(object savedState)
{
    base.LoadViewState(savedState);
    if (MostrarNota) CrearTextBox();   // el ViewState de la página ya está disponible aquí
}

protected void btnAgregar_Click(object sender, EventArgs e)
{
    if (!MostrarNota)
    {
        MostrarNota = true;
        CrearTextBox();
    }
}

private void CrearTextBox()
{
    phNotas.Controls.Add(new TextBox { ID = "txtNota" });
}
```

Se recrea en `LoadViewState` (después de recuperar el ViewState de la página y antes de que los controles reciban sus valores) para que el `TextBox` exista con el mismo `ID` y recupere lo escrito. Esta complejidad es una de las razones por las que los controles dinámicos en Web Forms son fuente de errores.

</details>

**Ejercicio 2.** Indica qué partes del ejemplo completo reutilizarías tal cual al migrar a Blazor y cuáles reescribirías.

<details>
<summary>Solución</summary>

* **Reutilizar:** `ServicioPedidos` y `PedidoFila` (no dependen de `System.Web`); se mueven a una biblioteca `netstandard2.0` o `net10.0` y se registran en DI. Conviene reemplazar la lista `static` por un repositorio real.
* **Reescribir:** `Pedidos.aspx` y `Pedidos.aspx.cs`. En Blazor serían un componente `Pedidos.razor` con un `<select @bind="estado" @bind:after="Cargar">`, una tabla con `@foreach` y un botón `@onclick="() => Confirmar(p.Id)"`. Desaparecen `IsPostBack`, el ViewState y `DataKeys`: el estado vive en campos del componente.

Cuanta más lógica haya en clases como `ServicioPedidos` y menos en el code-behind, más barata es la migración.

</details>

-----

## Siguiente lección

[Windows Forms y DevExpress](03-Windows%20Forms%20y%20DevExpress.md)
