# Estrategias de migración

## En una frase

Migrar un sistema heredado no es "reescribirlo en .NET 10": es elegir, **parte por parte**, entre actualizar en el lugar (bibliotecas y WinForms), reemplazar de forma incremental detrás de un proxy (*strangler fig* con YARP, para aplicaciones web) o reescribir, siempre con pruebas que protejan el comportamiento y sin dejar de entregar valor durante meses.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo reconocer cada tecnología: [.NET Framework frente a .NET moderno](../01-tecnologias-heredadas/01-NET%20Framework%20frente%20a%20NET%20moderno.md).
* Qué tiene equivalente y qué no: [ASP.NET Web Forms](../01-tecnologias-heredadas/02-ASP%20NET%20Web%20Forms.md) y [WCF y sus reemplazos](../01-tecnologias-heredadas/04-WCF%20y%20sus%20reemplazos.md).
* Pruebas de integración: [WebApplicationFactory y pruebas de API en memoria](../../05-testing-y-calidad/02-pruebas-de-integracion/01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Big bang:** reescribir el sistema completo y reemplazarlo de una sola vez.
* **Actualización en el lugar (*in-place*):** cambiar el framework de destino de un proyecto y corregir lo que se rompe, sin reescribirlo.
* **Strangler fig (higuera estranguladora):** patrón en el que un sistema nuevo crece alrededor del viejo y lo va reemplazando ruta por ruta hasta que el viejo puede apagarse.
* **YARP (*Yet Another Reverse Proxy*):** biblioteca de Microsoft para construir proxies inversos en ASP.NET Core.
* **System.Web adapters:** paquetes de Microsoft que permiten compartir sesión y autenticación, y usar APIs con forma de `System.Web`, entre una app ASP.NET clásica y una ASP.NET Core durante la migración.
* **Pruebas de caracterización:** pruebas que capturan el comportamiento actual del sistema (aunque sea raro) para detectar cambios no intencionales durante la migración.

-----

## El problema

Una empresa decide "modernizar" su sistema de gestión: Web Forms + WCF + VB.NET, 400 pantallas, 12 años de reglas de negocio. El plan aprobado:

> "Congelamos el sistema actual. En 8 meses tenemos la versión nueva en .NET 10 con Angular. Después apagamos el viejo."

Lo que pasa:

1. **El mes 8 llega con el 40 % hecho.** Nadie había medido cuántas reglas había escondidas en el code-behind y en procedimientos almacenados.
2. **El negocio no se congela.** Durante esos meses llegan cambios legales que hay que hacer en el sistema viejo... y también en el nuevo. Dos sistemas, doble trabajo.
3. **El día del cambio**, el sistema nuevo calcula distinto los descuentos en tres casos que nadie documentó ("siempre fue así"). Hay que volver al viejo con datos ya migrados.
4. El equipo descubre que el servicio WCF lo consumen dos sistemas externos que no estaban en el inventario.
5. Después de 14 meses, el proyecto se cancela. El sistema viejo sigue ahí, ahora con menos gente que lo entienda.

Las consecuencias:

* reescrituras que nunca alcanzan al sistema que reemplazan;
* comportamientos implícitos que se pierden porque nadie los escribió en pruebas;
* meses sin entregar valor al negocio.

-----

## Cómo funciona

### 1. Las tres estrategias

| Estrategia | Cómo | Funciona bien para | Riesgo |
| --- | --- | --- | --- |
| **Actualización en el lugar** | Cambiar el TFM a `net10.0`, convertir el `.csproj` a estilo SDK, reemplazar APIs que no existen | Bibliotecas de clases, consolas, servicios de Windows, WinForms, WPF | Bajo, si hay pruebas; las dependencias de terceros pueden bloquear |
| **Incremental (*strangler fig*)** | Una app nueva delante de la vieja; las rutas se migran de a una | Aplicaciones web (Web Forms, MVC 5, Web API 2), servicios grandes | Medio: convivencia de sesión y autenticación, dos despliegues por un tiempo |
| **Reescritura (*big bang*)** | Sistema nuevo desde cero y cambio de una vez | Sistemas chicos, o cuando el dominio cambió tanto que lo viejo ya no sirve | Alto: plazos, reglas perdidas, negocio congelado |

Regla práctica: **actualización en el lugar** donde el modelo de programación no cambia; **incremental** donde sí cambia (web); **reescritura** solo si es chico o si no queda otra.

### 2. Antes de tocar nada: inventario

```text
Por cada proyecto de la solución:
  · TFM y estilo de proyecto              (el detector de la lección 1 del módulo anterior)
  · tecnologías: System.Web, WCF, Remoting, WF, COM
  · dependencias NuGet y su soporte para .NET moderno (o netstandard2.0)
  · controles de terceros (DevExpress, Telerik) y su versión para .NET moderno
  · consumidores: ¿quién llama a esto? (servicios WCF, APIs, bases compartidas)
  · cobertura de pruebas
```

El inventario convierte "migrar el sistema" en una lista de tareas estimables. Lo que no aparece en el inventario (como los dos consumidores externos del problema) es lo que destruye el plan.

### 3. Pruebas de caracterización: proteger lo que hay

Antes de mover una regla, se escribe una prueba que capture lo que el sistema **hace hoy**, incluso si parece raro:

```csharp
[Theory]
[InlineData(999.99, "Mayorista", 0.05)]   // así se comporta hoy, aunque nadie sabe por qué no es 0.10
[InlineData(1000.00, "Mayorista", 0.10)]
[InlineData(1000.00, "Minorista", 0.00)]
public void Descuento_ComportamientoActual(decimal total, string tipoCliente, decimal esperado) =>
    Assert.Equal(esperado, CalculadoraDescuentos.Calcular(total, tipoCliente));
```

Si la versión nueva da otro resultado, la prueba falla y alguien **decide** si es un error del viejo o del nuevo, en lugar de descubrirlo en producción.

### 4. Strangler fig con YARP

```text
Hoy:               usuarios ──► Web Forms (IIS)

Durante:           usuarios ──► App ASP.NET Core nueva (YARP)
                                 ├── /api/pedidos/*   → implementado en la app nueva
                                 ├── /clientes/*      → implementado en la app nueva
                                 └── todo lo demás    → proxy al Web Forms viejo

Al final:          usuarios ──► App ASP.NET Core nueva   (el Web Forms se apaga)
```

* La app nueva recibe **todo** el tráfico. Lo que ya migró lo atiende ella; el resto lo reenvía al sistema viejo con YARP.
* Cada ruta migrada es un despliegue chico y reversible: si falla, se vuelve a enviar esa ruta al sistema viejo.
* El negocio no se congela: los cambios urgentes se hacen donde vive esa parte hoy.

### 5. Lo difícil de la convivencia: sesión y autenticación

Si un usuario inicia sesión en el Web Forms viejo, la app nueva tiene que reconocerlo, y viceversa. Los **System.Web adapters** (`Microsoft.AspNetCore.SystemWebAdapters`) resuelven esto:

* **Sesión remota:** la app nueva lee y escribe la sesión que guarda la app vieja.
* **Autenticación remota:** la app nueva le pregunta a la vieja quién es el usuario (con su cookie de Forms Authentication).
* **APIs con forma de `System.Web`:** código que usa `HttpContext.Current` o `Request.QueryString` puede moverse a la app nueva con menos cambios.

### 6. Orden recomendado para una aplicación web

1. Inventario y pruebas de caracterización de las reglas críticas.
2. Sacar la lógica de negocio del code-behind a bibliotecas `netstandard2.0` (sirven a la app vieja y a la nueva).
3. Actualizar en el lugar las bibliotecas que se pueda (a `netstandard2.0` o *multi-targeting*).
4. Crear la app ASP.NET Core con YARP delante de la vieja, con sesión y autenticación compartidas.
5. Migrar ruta por ruta, empezando por lo de menor riesgo y mayor valor; medir y comparar.
6. Migrar los datos y el acceso a datos (EF6 → EF Core) como un paso separado.
7. Cuando no queda tráfico al sistema viejo, apagarlo.

### 7. Herramientas

| Herramienta | Para | Nota |
| --- | --- | --- |
| .NET Upgrade Assistant | Analizar dependencias y convertir proyectos (`.csproj`, TFM, paquetes) | Microsoft lo está reemplazando por la modernización asistida con GitHub Copilot; verifica la herramienta vigente |
| Code Converter (ICSharpCode) | VB.NET → C# | Revisar `And`/`Or`, divisiones y funciones de VB6 |
| YARP | Proxy de la app nueva hacia la vieja | `Yarp.ReverseProxy` |
| System.Web adapters | Sesión, autenticación y APIs compartidas | `Microsoft.AspNetCore.SystemWebAdapters` |
| CoreWCF | Mantener contratos SOAP en .NET moderno | Ver la lección de WCF |
| `dotnet ef dbcontext scaffold` | Generar un modelo EF Core desde la base existente | Reemplaza al `.edmx` |

-----

## Ejemplo completo

Dos aplicaciones .NET 10: un "sistema heredado" simulado y la app nueva con YARP, que atiende `/api/pedidos` y reenvía todo lo demás. En la realidad, el heredado sería el Web Forms en IIS.

**Heredado simulado** (`dotnet new web -o Legado`). `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.UseUrls("http://localhost:5100");
var app = builder.Build();

// Responde cualquier ruta, como haría el Web Forms con sus .aspx
app.MapFallback((HttpContext context) => $"Sistema heredado: {context.Request.Path}");

app.Run();
```

**App nueva** (`dotnet new web -o Tienda.Nueva`, y `dotnet add package Yarp.ReverseProxy`). `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.UseUrls("http://localhost:5000");
builder.Services.AddHttpForwarder();

var app = builder.Build();

// Rutas ya migradas: las atiende la app nueva
app.MapGet("/api/pedidos", () => TypedResults.Ok(new[]
{
    new { Codigo = "P-001", Origen = "nuevo" },
    new { Codigo = "P-002", Origen = "nuevo" }
}));

// Todo lo demás: al sistema heredado, con la menor prioridad para que gane cualquier ruta migrada
app.MapForwarder("/{**catch-all}", "http://localhost:5100")
   .Add(endpoint => ((RouteEndpointBuilder)endpoint).Order = int.MaxValue);

app.Run();
```

Arranca primero `Legado` y después `Tienda.Nueva`. Peticiones siempre a la app nueva (puerto 5000):

```text
GET http://localhost:5000/api/pedidos
→ 200 [{"codigo":"P-001","origen":"nuevo"},{"codigo":"P-002","origen":"nuevo"}]

GET http://localhost:5000/Pedidos.aspx
→ 200 Sistema heredado: /Pedidos.aspx

GET http://localhost:5000/Clientes/Editar.aspx?id=7
→ 200 Sistema heredado: /Clientes/Editar.aspx
```

Qué observar:

* El usuario solo conoce una dirección (la app nueva). No sabe qué parte responde el sistema viejo y cuál el nuevo.
* `/api/pedidos` gana porque es una ruta concreta; el *catch-all* tiene `Order = int.MaxValue` para ser siempre la última opción.
* Migrar otra pantalla es agregar su endpoint en la app nueva: desde el siguiente despliegue, esa ruta deja de llegar al heredado. Si algo sale mal, se quita el endpoint y la ruta vuelve al heredado.
* El heredado sigue funcionando sin cambios: puede seguir recibiendo correcciones mientras dura la migración.
* En un sistema real faltarían la sesión y la autenticación compartidas (System.Web adapters), los encabezados reenviados (`X-Forwarded-*`) y el monitoreo de qué rutas siguen yendo al heredado.

-----

## Errores comunes

**1. Reescribir todo de una vez (big bang) en un sistema grande.**
Qué pasa: plazos que se duplican, negocio congelado, doble mantenimiento y un cambio de alto riesgo.
Por qué: se subestiman las reglas implícitas y el sistema viejo sigue cambiando.
Arreglo: incremental con *strangler fig*; reescritura solo para piezas chicas.

**2. Migrar sin pruebas de caracterización.**
Qué pasa: el sistema nuevo "funciona" pero calcula distinto en casos que nadie documentó.
Por qué: el comportamiento real no está escrito en ningún lado.
Arreglo: capturar el comportamiento actual en pruebas antes de mover cada regla.

**3. Inventario incompleto.**
Qué pasa: consumidores externos, tareas programadas o procedimientos almacenados que aparecen a mitad de la migración.
Por qué: se miró el código, no las integraciones ni la base.
Arreglo: inventariar también consumidores (logs de IIS, tráfico de red), jobs y objetos de la base.

**4. Migrar la plataforma y el diseño al mismo tiempo.**
Qué pasa: cambios enormes difíciles de probar ("ya que estamos, pasemos a microservicios y a otra base").
Por qué: se mezclan objetivos.
Arreglo: primero mover a .NET moderno con el mismo comportamiento; después rediseñar, con pruebas en verde.

**5. Olvidar la sesión y la autenticación compartidas.**
Qué pasa: los usuarios tienen que volver a iniciar sesión al pasar de una pantalla vieja a una nueva, o pierden datos de sesión.
Por qué: cada app tiene su propia cookie y su propia sesión.
Arreglo: System.Web adapters (sesión y autenticación remotas) o un proveedor de identidad común (OIDC) para ambas.

-----

## Según la versión de .NET

* **.NET Core 3.1 / .NET 5:** se popularizan las migraciones en el lugar de bibliotecas y WinForms/WPF.
* **2021:** YARP 1.0, el proxy inverso de Microsoft para ASP.NET Core.
* **2022–2023:** System.Web adapters y la guía oficial de migración incremental de ASP.NET a ASP.NET Core.
* **2023:** .NET Upgrade Assistant como extensión de Visual Studio, con soporte para migraciones incrementales.
* **YARP 2.x:** `AddHttpForwarder` y `MapForwarder` para reenvío directo sin configuración de rutas y clusters.
* **2025:** Microsoft empieza a reemplazar Upgrade Assistant por la modernización asistida con GitHub Copilot.

-----

## Cuándo sí y cuándo no

**Migra cuando:** necesitas soporte de largo plazo, otra plataforma (Linux, contenedores, nube), bibliotecas que ya no soportan .NET Framework, o el sistema va a seguir creciendo y cuesta encontrar quien lo mantenga.

**Usa *strangler fig* cuando:** es una aplicación web grande que debe seguir funcionando y recibiendo cambios durante la migración.

**No migres cuando:** el sistema es estable, cambia poco, tiene soporte (4.8.1 sobre un Windows soportado) y el costo de migrar supera el beneficio. "Es viejo" no es, por sí solo, una razón.

-----

## Resumen en 5 líneas

1. Tres estrategias: en el lugar (bibliotecas, WinForms), incremental con *strangler fig* (web) y reescritura (solo si es chico).
2. Primero inventario completo y pruebas de caracterización; sin eso, la migración es a ciegas.
3. La lógica de negocio sale del code-behind a bibliotecas `netstandard2.0` que usan el sistema viejo y el nuevo.
4. YARP pone la app nueva delante de la vieja y permite migrar ruta por ruta, de forma reversible.
5. Sesión y autenticación compartidas con System.Web adapters; datos y diseño, en pasos separados.

-----

## Para profundizar

<details>
<summary>Comparar respuestas del sistema viejo y del nuevo</summary>

Para rutas críticas, una técnica es el *shadowing*: la app nueva envía la petición a **ambas** implementaciones, responde con la vieja y registra las diferencias entre las dos respuestas. Así descubres casos donde el nuevo calcula distinto, con tráfico real y sin afectar a los usuarios. Requiere cuidado con operaciones que modifican datos (solo se aplica a lecturas o a entornos de prueba).

</details>

<details>
<summary>Migrar la base de datos sin cortar el servicio</summary>

Si la migración incluye cambiar el esquema, se aplica el patrón *expand/contract* visto en [Migraciones y consultas](../../04-data-access-and-transactions/01-entity-framework-core/04-Migraciones%20y%20consultas.md): primero se agregan columnas o tablas nuevas compatibles con ambos sistemas, después se migran los datos y el código, y recién al final se borra lo viejo. Mientras convivan el sistema heredado y el nuevo, los dos deben poder trabajar con la misma base.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para migrar un sistema de .NET Framework, las bibliotecas y las apps de escritorio se pueden actualizar en el lugar cambiando el target framework. Las aplicaciones web, como Web Forms, se migran de a poco: una app nueva en ASP.NET Core recibe el tráfico y le pasa al sistema viejo lo que todavía no se migró, con YARP.

### Respuesta ampliada (semi-senior)

Empiezo por un inventario completo (TFM, tecnologías, dependencias, controles de terceros, consumidores, jobs y base) y pruebas de caracterización de las reglas críticas. Actualizo en el lugar lo que mantiene el modelo de programación (bibliotecas a `netstandard2.0`, WinForms, consolas) y para la web aplico *strangler fig*: una app ASP.NET Core con YARP delante, sesión y autenticación compartidas con System.Web adapters, y migración ruta por ruta, reversible. Mantengo los contratos externos (por ejemplo, SOAP con CoreWCF), separo la migración de plataforma del rediseño y de la migración de datos, y evito el *big bang* salvo en piezas chicas. Y no migro sin un motivo concreto.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es el patrón strangler fig?**
Un sistema nuevo que crece alrededor del viejo y lo reemplaza de a partes, hasta que el viejo puede apagarse.

**2. ¿Por qué no reescribir todo?**
Por el riesgo: reglas implícitas que se pierden, plazos que se extienden y negocio congelado.

**3. ¿Qué son las pruebas de caracterización?**
Pruebas que fijan el comportamiento actual para detectar cambios no intencionales al migrar.

**4. ¿Cómo comparten sesión la app vieja y la nueva?**
Con los System.Web adapters (sesión y autenticación remotas) o un proveedor de identidad común.

-----

## Práctica

**Ejercicio 1.** Clasifica cada proyecto en una estrategia: (a) `Facturacion.Comun` (`net472`, solo lógica y LINQ); (b) `Facturacion.Web` (Web Forms, 200 páginas); (c) `Facturacion.Escritorio` (WinForms `net48` con DevExpress 22.2); (d) `Facturacion.Servicios` (WCF `basicHttpBinding`, consumido por un proveedor externo).

<details>
<summary>Solución</summary>

* (a) **En el lugar:** pasar a `netstandard2.0` (o *multi-targeting* `net472;net10.0`). Es la base que comparten los demás durante la migración.
* (b) **Incremental:** app ASP.NET Core con YARP delante, System.Web adapters, migración de páginas a Blazor o Razor Pages de a una.
* (c) **En el lugar:** `net10.0-windows`, verificando que la versión de DevExpress tenga soporte para .NET moderno (puede requerir actualizar DevExpress y su licencia).
* (d) **CoreWCF** manteniendo el contrato SOAP para el proveedor, en .NET 10; endpoints gRPC o REST adicionales para consumidores nuevos.

El orden sugerido: (a) primero, porque destraba a los demás; después (c) y (d), que son más directos; (b) es el más largo y avanza en paralelo.

</details>

**Ejercicio 2.** Agrega al ejemplo la ruta migrada `GET /clientes/{id:int}`, y explica cómo volverías atrás si en producción aparece un error en esa ruta.

<details>
<summary>Solución</summary>

```csharp
app.MapGet("/clientes/{id:int}", (int id) => TypedResults.Ok(new { Id = id, Nombre = "Ana", Origen = "nuevo" }));
```

Desde el despliegue, `GET /clientes/7` lo responde la app nueva; `/clientes/Editar.aspx?id=7` (que no coincide con la plantilla `{id:int}`) sigue yendo al heredado.

Para volver atrás: quitar el endpoint y desplegar. La ruta vuelve a caer en el *catch-all* y llega al sistema heredado, que nunca dejó de tener esa funcionalidad. Para revertir sin desplegar, el endpoint puede consultar un *feature flag* (por ejemplo, con `Microsoft.FeatureManagement`) y, si está apagado, reenviar la petición al heredado con `IHttpForwarder` en lugar de responderla: apagar la bandera devuelve esa ruta al sistema viejo al instante.

</details>

-----

## Siguiente lección

Terminaste el bloque de legacy y migración. Vuelve al [índice del módulo](README.md) o al [índice del bloque](../README.md).
