# WCF y sus reemplazos

## En una frase

WCF (*Windows Communication Foundation*) es el framework de servicios de .NET Framework, casi siempre usado con SOAP sobre HTTP: un **contrato** (interfaz), un **binding** (cómo viaja el mensaje) y una **dirección**; .NET moderno solo incluye el cliente, el servidor existe gracias a **CoreWCF**, y para servicios nuevos el reemplazo es gRPC o REST.

-----

## Antes de empezar

Conviene que ya sepas:

* Plataformas y qué no existe en .NET moderno: [.NET Framework frente a .NET moderno](01-NET%20Framework%20frente%20a%20NET%20moderno.md).
* Diseño de APIs HTTP: [Principios REST y códigos de estado](../../03-aspnet-core-apis/01-diseno-de-apis-rest/01-Principios%20REST%20y%20codigos%20de%20estado.md).
* Interfaces y `async`/`await`: [POO](../../01-csharp-core-and-runtime/04-poo/README.md) y [Asincronía](../../01-csharp-core-and-runtime/10-asincronia-y-archivos/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **WCF:** framework de .NET Framework para construir y consumir servicios con distintos protocolos (SOAP/HTTP, TCP, colas).
* **SOAP:** protocolo de mensajes en XML con un sobre (`Envelope`), cabeceras y cuerpo.
* **WSDL:** documento XML que describe un servicio SOAP (operaciones, mensajes, tipos); es lo que lee "Agregar referencia de servicio".
* **ABC de WCF:** *Address* (dirección), *Binding* (transporte y codificación) y *Contract* (operaciones).
* **`[ServiceContract]` / `[OperationContract]`:** atributos que convierten una interfaz y sus métodos en un contrato de servicio.
* **`[DataContract]` / `[DataMember]`:** atributos que definen qué tipos y propiedades se serializan en los mensajes.
* **CoreWCF:** proyecto de la comunidad (con apoyo de Microsoft) que porta el servidor de WCF a .NET moderno.
* **gRPC:** framework RPC con contratos en Protocol Buffers sobre HTTP/2.

-----

## El problema

El sistema de facturación expone un servicio WCF desde 2012. Lo consumen el ERP de la empresa, dos aplicaciones de escritorio y un proveedor externo. El equipo quiere mover todo a .NET 10 y contenedores Linux.

Primer intento:

1. Crean un proyecto .NET 10 y copian `Service1.svc.cs`. No compila: `System.ServiceModel.ServiceHost` y la configuración `<system.serviceModel>` del `Web.config` no existen en .NET moderno.
2. Alguien propone "reescribirlo como API REST, que es lo moderno". El ERP y el proveedor externo **solo** hablan SOAP con ese WSDL: no van a cambiar su integración por nuestra migración.
3. Mientras discuten, otro equipo agrega un cliente nuevo en .NET 10 que llama al servicio... copiando el `Reference.cs` generado en 2012, con `System.ServiceModel` de .NET Framework. No compila.
4. Al revisar la configuración aparece `netTcpBinding` con seguridad de Windows para las aplicaciones de escritorio, y `wsHttpBinding` con sesiones fiables para el proveedor: no todo se porta igual.

Las consecuencias:

* migraciones bloqueadas por un servicio del que dependen sistemas que no controlas;
* propuestas de reescritura que rompen integraciones externas;
* clientes nuevos que no saben cómo llamar a un servicio SOAP desde .NET moderno.

-----

## Cómo funciona

### 1. El ABC de WCF

```text
Address   http://servidor/Facturacion/ServicioPedidos.svc      dónde
Binding   basicHttpBinding (SOAP 1.1 sobre HTTP, texto XML)     cómo viaja
Contract  IServicioPedidos { ObtenerPedido(codigo) }            qué operaciones
```

```csharp
[ServiceContract]
public interface IServicioPedidos
{
    [OperationContract]
    PedidoDto ObtenerPedido(string codigo);
}

[DataContract(Namespace = "http://tienda.example/pedidos")]
public class PedidoDto
{
    [DataMember] public string Codigo { get; set; }
    [DataMember] public decimal Total { get; set; }
}
```

El mismo contrato puede exponerse con varios *bindings* a la vez (HTTP para el proveedor, TCP para la red interna), cada uno en su dirección.

### 2. Los bindings que vas a encontrar

| Binding | Transporte y formato | Uso típico | En .NET moderno |
| --- | --- | --- | --- |
| `basicHttpBinding` | SOAP 1.1, XML, HTTP | Interoperabilidad (Java, ERP, terceros) | Cliente ✔ · CoreWCF ✔ |
| `wsHttpBinding` | SOAP 1.2 + WS-* (seguridad, sesiones fiables, transacciones) | Integraciones empresariales | Cliente parcial · CoreWCF parcial |
| `netTcpBinding` | Binario sobre TCP | Red interna entre aplicaciones .NET | Cliente ✔ · CoreWCF ✔ |
| `netNamedPipeBinding` | Binario por *pipes* en la misma máquina | Comunicación entre procesos locales | Cliente ✔ (versiones recientes) · CoreWCF ✔ |
| `webHttpBinding` | REST/JSON "a la WCF" | APIs HTTP antes de Web API | No: usa ASP.NET Core |
| `netMsmqBinding` | Colas MSMQ | Mensajería asíncrona | No: usa un broker (RabbitMQ, Service Bus) |

### 3. Un mensaje SOAP

```xml
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <ObtenerPedido xmlns="http://tempuri.org/">
      <codigo>P-001</codigo>
    </ObtenerPedido>
  </s:Body>
</s:Envelope>
```

(Forma del mensaje con `basicHttpBinding`; la cabecera HTTP `SOAPAction` indica la operación: `http://tempuri.org/IServicioPedidos/ObtenerPedido`.)

* `http://tempuri.org/` es el espacio de nombres **por defecto** de los contratos. Si el servidor y el cliente no coinciden en espacios de nombres (del contrato o de los `DataContract`), los mensajes llegan con campos vacíos o fallan.
* Los errores viajan como **SOAP Faults** y en C# se manejan como `FaultException`.

### 4. Configuración en .NET Framework

```xml
<system.serviceModel>
  <services>
    <service name="Facturacion.ServicioPedidos">
      <endpoint address="" binding="basicHttpBinding" contract="Facturacion.IServicioPedidos" />
      <endpoint address="mex" binding="mexHttpBinding" contract="IMetadataExchange" />
    </service>
  </services>
  <behaviors>
    <serviceBehaviors>
      <behavior>
        <serviceMetadata httpGetEnabled="true" />
        <serviceDebug includeExceptionDetailInFaults="false" />
      </behavior>
    </serviceBehaviors>
  </behaviors>
</system.serviceModel>
```

Casi todo WCF se configura en XML. `includeExceptionDetailInFaults="true"` devuelve el *stack trace* al cliente: útil en desarrollo, **fuga de información** en producción.

### 5. WCF en .NET moderno

| Necesitas | Solución |
| --- | --- |
| **Consumir** un servicio WCF desde .NET 10 | Paquetes `System.ServiceModel.Http` (y `.NetTcp`, `.Primitives`...); generar el cliente con `dotnet-svcutil` o "Agregar referencia de servicio WCF" en Visual Studio |
| **Exponer** el mismo contrato SOAP desde .NET 10 | CoreWCF (`CoreWCF.Http`, `CoreWCF.NetTcp`, ...), hospedado en ASP.NET Core |
| Un servicio **nuevo** entre sistemas propios | gRPC (contratos `.proto`, HTTP/2, binario, *streaming*) |
| Un servicio **nuevo** para clientes web o terceros | REST/JSON con ASP.NET Core |

**CoreWCF no es 100 % de WCF:** soporta los *bindings* y comportamientos más usados, pero no todo WS-* (por ejemplo, transacciones distribuidas WS-AtomicTransaction). Antes de comprometerte, inventaría los *bindings* y comportamientos que usa el servicio y verifícalos contra la lista de compatibilidad de CoreWCF.

### 6. Estrategia de migración habitual

```text
Hoy:    clientes SOAP ──► WCF en .NET Framework / IIS
Paso 1: clientes SOAP ──► CoreWCF en .NET 10 (mismo contrato, mismo WSDL) ── Linux/contenedores
Paso 2: clientes nuevos ──► gRPC o REST en el mismo servicio .NET 10
Paso 3: cuando los clientes viejos migran, se retira el endpoint SOAP (o se mantiene si hay terceros)
```

La clave: **no obligar a los consumidores a migrar al mismo tiempo que tú**. El contrato SOAP es una promesa con terceros.

-----

## Ejemplo completo

Dos aplicaciones .NET 10: un servidor SOAP con CoreWCF y un cliente con `ChannelFactory`. Funciona en Windows, Linux o macOS.

**Servidor** (`dotnet new web -o Pedidos.Soap`, y en esa carpeta `dotnet add package CoreWCF.Http` y `dotnet add package CoreWCF.Primitives`). `Program.cs`:

```csharp
using System.Runtime.Serialization;
using CoreWCF;
using CoreWCF.Configuration;
using CoreWCF.Description;

var builder = WebApplication.CreateBuilder(args);
builder.WebHost.UseUrls("http://localhost:5050");

builder.Services.AddServiceModelServices();
builder.Services.AddServiceModelMetadata();
builder.Services.AddSingleton<IServiceBehavior, UseRequestHeadersForMetadataAddressBehavior>();

var app = builder.Build();

app.UseServiceModel(servicios =>
{
    servicios.AddService<ServicioPedidos>();
    servicios.AddServiceEndpoint<ServicioPedidos, IServicioPedidos>(new BasicHttpBinding(), "/ServicioPedidos.svc");

    var metadatos = app.Services.GetRequiredService<ServiceMetadataBehavior>();
    metadatos.HttpGetEnabled = true;   // publica el WSDL en ?wsdl
});

app.Run();

[ServiceContract]
public interface IServicioPedidos
{
    [OperationContract]
    PedidoDto ObtenerPedido(string codigo);
}

[DataContract(Namespace = "http://tienda.example/pedidos")]
public class PedidoDto
{
    [DataMember] public string Codigo { get; set; } = "";
    [DataMember] public string Cliente { get; set; } = "";
    [DataMember] public decimal Total { get; set; }
}

public class ServicioPedidos : IServicioPedidos
{
    private static readonly Dictionary<string, PedidoDto> Pedidos = new()
    {
        ["P-001"] = new PedidoDto { Codigo = "P-001", Cliente = "Ana", Total = 270m },
        ["P-002"] = new PedidoDto { Codigo = "P-002", Cliente = "Luis", Total = 85.5m }
    };

    public PedidoDto ObtenerPedido(string codigo) =>
        Pedidos.TryGetValue(codigo, out PedidoDto? pedido)
            ? pedido
            : throw new FaultException($"No existe el pedido {codigo}.");
}
```

**Cliente** (`dotnet new console -o Pedidos.Cliente`, y `dotnet add package System.ServiceModel.Http`). `Program.cs`:

```csharp
using System.Globalization;
using System.Runtime.Serialization;
using System.ServiceModel;

const string Direccion = "http://localhost:5050/ServicioPedidos.svc";

var fabrica = new ChannelFactory<IServicioPedidos>(new BasicHttpBinding(), new EndpointAddress(Direccion));
IServicioPedidos cliente = fabrica.CreateChannel();

try
{
    PedidoDto pedido = await cliente.ObtenerPedidoAsync("P-001");
    Console.WriteLine($"{pedido.Codigo} | {pedido.Cliente} | {pedido.Total.ToString("F2", CultureInfo.InvariantCulture)}");

    await cliente.ObtenerPedidoAsync("P-999");
}
catch (FaultException ex)
{
    Console.WriteLine($"Error SOAP: {ex.Message}");
}
finally
{
    ((IClientChannel)cliente).Close();
    fabrica.Close();
}

using var http = new HttpClient();
HttpResponseMessage wsdl = await http.GetAsync($"{Direccion}?wsdl");
Console.WriteLine($"WSDL: {(int)wsdl.StatusCode} {wsdl.Content.Headers.ContentType?.MediaType}");

// El contrato del lado cliente: mismo nombre de interfaz, operación y espacios de nombres que el servidor.
// El método termina en Async y devuelve Task: WCF lo asocia a la operación "ObtenerPedido".
[ServiceContract]
public interface IServicioPedidos
{
    [OperationContract]
    Task<PedidoDto> ObtenerPedidoAsync(string codigo);
}

[DataContract(Namespace = "http://tienda.example/pedidos")]
public class PedidoDto
{
    [DataMember] public string Codigo { get; set; } = "";
    [DataMember] public string Cliente { get; set; } = "";
    [DataMember] public decimal Total { get; set; }
}
```

Ejecuta primero el servidor (`dotnet run` en `Pedidos.Soap`) y después el cliente. Salida del cliente:

```text
P-001 | Ana | 270.00
Error SOAP: No existe el pedido P-999.
WSDL: 200 text/xml
```

Qué observar:

* El servidor no tiene `ServiceHost` ni `Web.config`: CoreWCF se hospeda dentro de ASP.NET Core y se configura en código, pero el contrato (`[ServiceContract]`, `[DataContract]`) es el mismo modelo de WCF.
* El cliente redefine el contrato con los atributos de `System.ServiceModel`. Funciona porque coinciden el nombre de la interfaz, el de la operación y el espacio de nombres del `DataContract`. En la práctica, este código lo genera `dotnet-svcutil` a partir del WSDL.
* La operación es síncrona en el servidor y asíncrona en el cliente: WCF asocia `ObtenerPedidoAsync` a la operación `ObtenerPedido`. El mensaje SOAP es el mismo.
* La `FaultException` del servidor viaja como SOAP Fault y llega al cliente como `FaultException`, con el mensaje y sin el *stack trace*.
* El WSDL publicado es lo que necesitan el ERP o un cliente Java para seguir integrándose sin cambios.

-----

## Errores comunes

**1. Reescribir un servicio SOAP como REST sin acordarlo con los consumidores.**
Qué pasa: integraciones externas que dejan de funcionar el día de la migración.
Por qué: el contrato SOAP (WSDL, espacios de nombres, `SOAPAction`) es una interfaz pública.
Arreglo: migrar el servidor con CoreWCF manteniendo el contrato, y ofrecer REST o gRPC como endpoints adicionales.

**2. Espacios de nombres distintos entre servidor y cliente.**
Qué pasa: los objetos llegan con propiedades en `null` o `0`, sin ninguna excepción.
Por qué: el serializador ignora los elementos XML cuyo espacio de nombres no coincide.
Arreglo: generar el cliente desde el WSDL (`dotnet-svcutil`) o fijar el mismo `Namespace` en contratos y `DataContract` de ambos lados.

**3. `includeExceptionDetailInFaults="true"` en producción.**
Qué pasa: el cliente recibe *stack traces*, nombres de servidores o consultas SQL.
Por qué: la opción de depuración quedó activa.
Arreglo: desactivarla y lanzar `FaultException` (o `FaultException<TDetalle>`) con mensajes pensados para el cliente.

**4. No cerrar ni abortar los canales del cliente.**
Qué pasa: conexiones agotadas y llamadas que quedan esperando hasta el *timeout*.
Por qué: cada canal mantiene recursos de red; tras una excepción de comunicación queda en estado `Faulted`.
Arreglo: `Close()` en el camino normal y `Abort()` si el canal falló; no uses `using` sobre un canal WCF, porque `Dispose` llama a `Close` y puede lanzar otra excepción que oculta la original.

**5. Asumir que CoreWCF soporta todo.**
Qué pasa: la migración se traba a mitad de camino por un comportamiento WS-* no soportado.
Por qué: CoreWCF porta lo más usado, no todo WCF.
Arreglo: inventariar *bindings*, seguridad y comportamientos antes de empezar y verificarlos contra la documentación de CoreWCF.

-----

## Según la versión de .NET

* **.NET Framework 3.0 (2006):** WCF, junto con WPF y WF.
* **.NET Framework 3.5 / 4.0:** `webHttpBinding` (REST), configuración simplificada y activación sin archivo `.svc`.
* **.NET Framework 4.5 (2012):** operaciones asíncronas basadas en `Task` en contratos.
* **.NET Core 1.0+:** solo el cliente de WCF (paquetes `System.ServiceModel.*`), para HTTP y TCP.
* **CoreWCF 1.0 (2022):** servidor WCF para .NET moderno, con apoyo de Microsoft.
* **ASP.NET Core 3.0 (2019):** gRPC integrado como alternativa recomendada para servicios nuevos entre sistemas propios.

-----

## Cuándo sí y cuándo no

**Mantén SOAP/WCF (o CoreWCF) cuando:** hay consumidores que dependen del contrato SOAP y no puedes coordinar su migración, o el estándar de la industria con la que te integras es SOAP.

**Usa gRPC cuando:** creas servicios nuevos entre sistemas propios, necesitas rendimiento, contratos estrictos o *streaming*.

**Usa REST/JSON cuando:** los consumidores son navegadores, aplicaciones móviles o terceros que esperan una API HTTP convencional.

**No uses WCF en .NET Framework para algo nuevo cuando:** puedes elegir; ni siquiera para SOAP, porque CoreWCF en .NET moderno cubre ese caso.

-----

## Resumen en 5 líneas

1. WCF define servicios con contrato, binding y dirección; casi siempre SOAP sobre HTTP con `basicHttpBinding`.
2. .NET moderno incluye el cliente WCF; el servidor se porta con CoreWCF, hospedado en ASP.NET Core.
3. El contrato SOAP (WSDL, espacios de nombres) es una promesa con los consumidores: no se rompe en la migración.
4. Servicios nuevos: gRPC entre sistemas propios, REST para web y terceros.
5. Antes de migrar, inventaría bindings y comportamientos WS-*: no todo está en CoreWCF.

-----

## Para profundizar

<details>
<summary>Generar el cliente con <code>dotnet-svcutil</code></summary>

```text
dotnet tool install --global dotnet-svcutil
dotnet-svcutil http://localhost:5050/ServicioPedidos.svc?wsdl --namespace "*,Pedidos.Cliente.Soap"
```

Genera un archivo `Reference.cs` con la interfaz del contrato, los `DataContract` y una clase cliente (`ServicioPedidosClient`) lista para usar, con métodos asíncronos. Es el equivalente moderno de "Agregar referencia de servicio" de Visual Studio para .NET Framework. No edites el archivo generado: si el servicio cambia, se vuelve a generar.

</details>

<details>
<summary>De contrato WCF a contrato gRPC</summary>

Un contrato WCF se traduce casi uno a uno a un archivo `.proto`:

```protobuf
service ServicioPedidos {
  rpc ObtenerPedido (ObtenerPedidoRequest) returns (PedidoDto);
}
message ObtenerPedidoRequest { string codigo = 1; }
message PedidoDto { string codigo = 1; string cliente = 2; string total = 3; }
```

Las diferencias: cada operación recibe **un** mensaje (no varios parámetros), los campos se numeran (la numeración es el contrato, no el nombre) y no hay `decimal` nativo (se usa `string` o un mensaje propio con unidades y nanos). Para equipos con muchos contratos WCF, la biblioteca protobuf-net.Grpc permite definir servicios gRPC con interfaces de C# al estilo WCF, sin escribir `.proto` a mano.

</details>

-----

## En entrevista

### Respuesta corta (junior)

WCF es el framework de servicios de .NET Framework; se usa sobre todo con SOAP. Se define con un contrato (interfaz con `[ServiceContract]`), un binding (cómo viaja) y una dirección. En .NET moderno solo existe el cliente; para el servidor está CoreWCF, y para servicios nuevos se usa gRPC o REST.

### Respuesta ampliada (semi-senior)

Al migrar un servicio WCF, primero inventarío *bindings* y comportamientos: `basicHttpBinding` y `netTcpBinding` los cubre CoreWCF, pero parte de WS-* no. Si hay consumidores externos, mantengo el contrato SOAP con CoreWCF en .NET 10 (mismo WSDL y espacios de nombres) y agrego endpoints gRPC o REST para clientes nuevos, en lugar de forzar una migración coordinada. Del lado cliente uso los paquetes `System.ServiceModel.*` y `dotnet-svcutil`, cierro o aborto los canales correctamente y no expongo detalles de excepciones en los faults. Para servicios nuevos entre sistemas propios elijo gRPC; para web y terceros, REST.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es el ABC de WCF?**
*Address*, *Binding* y *Contract*: dónde, cómo y qué.

**2. ¿Se puede consumir WCF desde .NET 10?**
Sí, con los paquetes cliente `System.ServiceModel.*`.

**3. ¿Y exponer un servicio WCF desde .NET 10?**
Con CoreWCF, para los *bindings* y comportamientos que soporta.

**4. ¿Por qué no reescribir todo como REST?**
Porque los consumidores SOAP existentes dependen del contrato; cambiarlo los rompe.

-----

## Práctica

**Ejercicio 1.** Agrega al servicio la operación `ListarPorCliente(string cliente)` que devuelva `PedidoDto[]`, y llámala desde el cliente. ¿Qué tienes que cambiar en cada lado?

<details>
<summary>Solución</summary>

**Servidor**, en la interfaz y en la implementación:

```csharp
[OperationContract]
PedidoDto[] ListarPorCliente(string cliente);

public PedidoDto[] ListarPorCliente(string cliente) =>
    Pedidos.Values.Where(p => p.Cliente == cliente).ToArray();
```

**Cliente**, en su copia del contrato:

```csharp
[OperationContract]
Task<PedidoDto[]> ListarPorClienteAsync(string cliente);

PedidoDto[] deAna = await cliente.ListarPorClienteAsync("Ana");
```

Los dos contratos deben cambiar en sincronía; por eso, en proyectos reales, el cliente se regenera desde el WSDL con `dotnet-svcutil` después de cada cambio del servidor. Agregar una operación es compatible con los clientes existentes (no la llaman); renombrar o quitar una, no.

</details>

**Ejercicio 2.** Un servicio WCF usa `wsHttpBinding` con `reliableSession` y transacciones WS-AtomicTransaction, consumido solo por otra aplicación interna en .NET. ¿Qué estrategia de migración propones?

<details>
<summary>Solución</summary>

Como el único consumidor es interno y propio, no hace falta preservar el contrato SOAP: conviene migrar ambos lados juntos a **gRPC** (o REST) en .NET 10. Las transacciones distribuidas WS-AT no tienen equivalente directo en CoreWCF ni en gRPC, así que hay que rediseñar esa parte: operaciones idempotentes, patrón outbox o una saga con compensaciones ([Entrega confiable e idempotencia](../../02-architecture-and-design-patterns/02-event-driven-architecture/03-Entrega%20confiable%20e%20idempotencia.md)). Las sesiones fiables se reemplazan por reintentos con idempotencia ([Resiliencia de servicios](../../02-architecture-and-design-patterns/03-resiliencia-de-servicios/README.md)).

CoreWCF solo tendría sentido si hubiera consumidores externos que no pudieran cambiar, y aun así habría que verificar qué parte de esas funciones WS-* soporta.

</details>

-----

## Siguiente lección

[VB.NET para desarrolladores de C#](05-VB%20NET%20para%20desarrolladores%20de%20CSharp.md)
