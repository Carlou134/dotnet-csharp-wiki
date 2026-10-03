# Modelos de aplicaciones ASP.NET Core

## En una frase
Minimal APIs, controladores, MVC, Razor Pages y Blazor comparten ASP.NET Core, pero organizan la interfaz y el flujo de trabajo de maneras distintas.

-----

## Antes de empezar
Conviene que ya sepas: [Host, configuración y pipeline HTTP](01-Host%20configuracion%20y%20pipeline%20HTTP.md).

Palabras nuevas: **Razor Pages:** UI centrada en páginas; **Blazor:** UI por componentes; **MVC:** separación entre modelo, vista y controlador.

-----

## El problema
Elegir una plantilla porque «es la más profesional» introduce ceremonias que el problema no necesita. También es incorrecto comparar MAUI con una Web API como si fueran dos alternativas equivalentes: una construye clientes y la otra expone HTTP.

-----

## Cómo funciona
### 1. Comparación
| Modelo | Unidad principal | Uso típico |
| --- | --- | --- |
| Minimal API | Endpoint | APIs pequeñas o vertical slices |
| API con controladores | Controller/action | APIs con convenciones MVC |
| MVC con vistas | Controller + view | Sitios renderizados en servidor |
| Razor Pages | PageModel + página | UI centrada en páginas/formularios |
| Blazor Web App | Componente | UI interactiva con componentes Razor |

### 2. No son exclusivas
Una aplicación puede mapear Minimal APIs junto con controladores o componentes. Combinar modelos tiene costo cognitivo; hazlo por límites claros, no por accidente.

### 3. MVC
El controlador recibe la interacción, coordina un caso de uso y elige una vista. El modelo de dominio no debe depender de la vista ni del controlador.

### 4. Razor Pages
Agrupa handlers y presentación alrededor de una página. Reduce navegación entre carpetas para flujos centrados en formularios.

### 5. Blazor
Usa componentes Razor y distintos modos de renderizado. No significa «cero JavaScript»: el navegador sigue siendo una plataforma web y algunas integraciones requieren interoperabilidad.

### 6. MAUI
.NET MAUI no es un modelo de ASP.NET Core. Es un stack de clientes nativos multiplataforma que puede consumir una API ASP.NET Core.

-----

## Ejemplo completo
Una aplicación puede exponer una API mínima y Razor Pages:
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();

var app = builder.Build();
app.UseHttpsRedirection();
app.UseStaticFiles();

app.MapGet("/api/status", () => TypedResults.Ok(new StatusResponse("ready")));
app.MapRazorPages();
app.Run();

public sealed record StatusResponse(string Status);
```

Respuesta de `GET /api/status`:
```json
{"status":"ready"}
```

-----

## Errores comunes
**1. Decir que MVC es arquitectura completa.** Qué pasa: controladores acumulan negocio y acceso a datos. Por qué: MVC organiza presentación. Arreglo: separa casos de uso y dominio.

**2. Elegir Blazor para evitar aprender web.** Qué pasa: se ignoran HTTP, HTML, CSS y ciclo del navegador. Por qué: Blazor abstrae parte del trabajo, no la plataforma. Arreglo: aprende fundamentos web.

**3. Confundir Razor con Razor Pages.** Qué pasa: se mezclan sintaxis y modelo. Por qué: Razor es sintaxis; Razor Pages es un framework de páginas que la usa. Arreglo: nombra ambos niveles.

-----

## Según la versión de .NET
- **ASP.NET Core 1:** unificó MVC y Web API en un mismo stack.
- **ASP.NET Core 2:** incorporó Razor Pages.
- **.NET 6:** popularizó Minimal APIs y el hosting mínimo.
- **.NET 8-10:** Blazor Web App integra renderizado de servidor e interactividad mediante modos de renderizado.

-----

## Cuándo sí y cuándo no
**Elige por el flujo dominante:** endpoints para servicios, MVC/Razor Pages para HTML de servidor y Blazor para UI por componentes. **No elijas por cantidad de carpetas:** más estructura no garantiza mejor arquitectura.

-----

## Resumen en 5 líneas
1. ASP.NET Core admite varios modelos sobre el mismo host.
2. Minimal APIs y controladores sirven para exponer HTTP.
3. MVC y Razor Pages renderizan HTML en el servidor.
4. Blazor organiza UI interactiva mediante componentes.
5. MAUI crea clientes nativos y puede consumir una API web.

-----

## Para profundizar
<details><summary>¿Se pueden mezclar?</summary>Sí. Puedes combinar endpoints y UI, pero debes documentar límites y políticas compartidas para que la aplicación no tenga dos formas arbitrarias de resolver lo mismo.</details>

-----

## En entrevista
### Respuesta corta (junior)
ASP.NET Core permite APIs, MVC, Razor Pages y Blazor. La elección depende de si entregas datos, HTML de servidor o componentes interactivos.

### Respuesta ampliada (semi-senior)
Los modelos comparten host, DI, configuración y middleware. Cambia la unidad de organización y el pipeline posterior al routing. MVC es presentación, no sustituto de una arquitectura de negocio.

### Preguntas frecuentes de seguimiento
**1. ¿Minimal API significa aplicación pequeña?** No; describe el modelo de endpoints, no el tamaño permitido.

**2. ¿MAUI reemplaza ASP.NET Core?** No; puede ser cliente de un backend ASP.NET Core.

-----

## Práctica
**Ejercicio 1.** Elige un modelo para un panel administrativo con formularios y poco JavaScript.
<details><summary>Solución</summary>Razor Pages o MVC son opciones directas. Razor Pages favorece flujos centrados en páginas; MVC favorece una organización explícita por controladores y vistas.</details>

-----

## Siguiente lección
[MVC, routing y vistas Razor](03-MVC%20routing%20y%20vistas%20Razor.md)
