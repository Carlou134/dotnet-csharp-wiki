# Glosario: tecnologías heredadas

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**ABC de WCF:** *Address* (dirección), *Binding* (transporte y codificación) y *Contract* (operaciones) de un servicio WCF. ([WCF](04-WCF%20y%20sus%20reemplazos.md))

**AndAlso / OrElse:** operadores lógicos de VB.NET con cortocircuito; `And` y `Or` evalúan siempre ambos lados. ([VB.NET](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md))

**Ciclo de vida de la página:** secuencia de etapas de Web Forms (`Init`, `Load`, eventos de controles, `PreRender`, `Render`) que se ejecuta en cada petición. ([Web Forms](02-ASP%20NET%20Web%20Forms.md))

**Code-behind:** archivo `.aspx.cs` (o `.aspx.vb`) con la lógica de una página Web Forms. ([Web Forms](02-ASP%20NET%20Web%20Forms.md))

**Control de servidor:** elemento con `runat="server"` que Web Forms convierte en HTML y que dispara eventos en C#. ([Web Forms](02-ASP%20NET%20Web%20Forms.md))

**CoreWCF:** proyecto de la comunidad, con apoyo de Microsoft, que porta el servidor de WCF a .NET moderno. ([WCF](04-WCF%20y%20sus%20reemplazos.md))

**DataContract / DataMember:** atributos de WCF que definen qué tipos y propiedades se serializan en los mensajes. ([WCF](04-WCF%20y%20sus%20reemplazos.md))

**DevExpress:** suite comercial de controles para WinForms, WPF, Web Forms, Blazor y otras plataformas; licencia por desarrollador. ([WinForms](03-Windows%20Forms%20y%20DevExpress.md))

**Diseñador:** editor visual de Visual Studio que genera el código de los controles de un formulario en `Form1.Designer.cs`. ([WinForms](03-Windows%20Forms%20y%20DevExpress.md))

**Enlace de datos (data binding):** conexión entre controles y objetos (`BindingSource`, `BindingList<T>`) para reflejar cambios automáticamente. ([WinForms](03-Windows%20Forms%20y%20DevExpress.md))

**GAC (Global Assembly Cache):** carpeta de Windows donde .NET Framework guarda ensamblados compartidos por todo el sistema. ([.NET Framework](01-NET%20Framework%20frente%20a%20NET%20moderno.md))

**gRPC:** framework RPC con contratos en Protocol Buffers sobre HTTP/2; reemplazo habitual de WCF entre sistemas propios. ([WCF](04-WCF%20y%20sus%20reemplazos.md))

**Handles / WithEvents:** forma declarativa de VB.NET para conectar un método a un evento. ([VB.NET](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md))

**Hilo de interfaz (UI thread):** único hilo que puede tocar los controles de una aplicación de escritorio. ([WinForms](03-Windows%20Forms%20y%20DevExpress.md))

**Invoke / InvokeAsync:** métodos de `Control` para ejecutar código en el hilo de interfaz desde otro hilo. ([WinForms](03-Windows%20Forms%20y%20DevExpress.md))

**Master page:** plantilla común de Web Forms (`.master`) que comparten varias páginas. ([Web Forms](02-ASP%20NET%20Web%20Forms.md))

**Module (VB.NET):** contenedor de miembros compartidos; equivale a una clase `static` de C#. ([VB.NET](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md))

**MVP (Model-View-Presenter):** patrón que saca la lógica de un formulario a un presentador que se puede probar sin interfaz. ([WinForms](03-Windows%20Forms%20y%20DevExpress.md))

**.NET Framework:** implementación original de .NET, solo para Windows, en mantenimiento desde la versión 4.8.1. ([.NET Framework](01-NET%20Framework%20frente%20a%20NET%20moderno.md))

**.NET moderno:** plataforma multiplataforma que empezó como .NET Core y desde .NET 5 se llama solo ".NET". ([.NET Framework](01-NET%20Framework%20frente%20a%20NET%20moderno.md))

**.NET Standard:** especificación de APIs comunes para compartir bibliotecas entre .NET Framework y .NET moderno. ([.NET Framework](01-NET%20Framework%20frente%20a%20NET%20moderno.md))

**Option Strict / Option Explicit:** opciones de VB.NET que prohíben conversiones implícitas peligrosas y el uso de variables sin declarar. ([VB.NET](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md))

**packages.config:** archivo de los proyectos antiguos con la lista de paquetes NuGet, reemplazado por `<PackageReference>`. ([.NET Framework](01-NET%20Framework%20frente%20a%20NET%20moderno.md))

**Postback:** envío del formulario a la misma página Web Forms para procesar el evento de un control. ([Web Forms](02-ASP%20NET%20Web%20Forms.md))

**Proyecto estilo SDK:** formato moderno del `.csproj` (`<Project Sdk="...">`); el antiguo lista cada archivo y usa `ToolsVersion`. ([.NET Framework](01-NET%20Framework%20frente%20a%20NET%20moderno.md))

**ServiceContract / OperationContract:** atributos que convierten una interfaz y sus métodos en un contrato de servicio WCF. ([WCF](04-WCF%20y%20sus%20reemplazos.md))

**SOAP:** protocolo de mensajes en XML con sobre, cabeceras y cuerpo. ([WCF](04-WCF%20y%20sus%20reemplazos.md))

**TFM (Target Framework Moniker):** identificador del framework de destino: `net48`, `netstandard2.0`, `net10.0`. ([.NET Framework](01-NET%20Framework%20frente%20a%20NET%20moderno.md))

**VB6:** Visual Basic 6, lenguaje anterior a .NET basado en COM; no es VB.NET. ([VB.NET](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md))

**VB.NET:** Visual Basic sobre .NET; compila a IL como C#. ([VB.NET](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md))

**ViewState:** estado de los controles de Web Forms serializado en el campo oculto `__VIEWSTATE`, que viaja en cada postback. ([Web Forms](02-ASP%20NET%20Web%20Forms.md))

**WCF (Windows Communication Foundation):** framework de servicios de .NET Framework para SOAP, TCP y colas. ([WCF](04-WCF%20y%20sus%20reemplazos.md))

**Web Form:** página `.aspx` con marcado y controles de servidor, compilada como una clase derivada de `System.Web.UI.Page`. ([Web Forms](02-ASP%20NET%20Web%20Forms.md))

**Windows Forms (WinForms):** framework de interfaces de escritorio para Windows basado en formularios y controles. ([WinForms](03-Windows%20Forms%20y%20DevExpress.md))

**WSDL:** documento XML que describe un servicio SOAP: operaciones, mensajes y tipos. ([WCF](04-WCF%20y%20sus%20reemplazos.md))
