# Despliegue en IIS

## En una frase

IIS es el servidor web de Windows; hospeda aplicaciones de .NET Framework **dentro** de su proceso (`w3wp.exe`) y aplicaciones ASP.NET Core a través del **ASP.NET Core Module**, dentro del proceso (*in-process*) o como proxy hacia Kestrel (*out-of-process*); desplegar en IIS no es legacy, pero sí exige entender *application pools*, el `web.config` y cómo diagnosticar los errores 500.3x.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo arranca ASP.NET Core y qué es Kestrel: [Host, configuración y pipeline HTTP](../../03-aspnet-core-apis/00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).
* Diferencias entre .NET Framework y .NET moderno: [.NET Framework frente a .NET moderno](../01-tecnologias-heredadas/01-NET%20Framework%20frente%20a%20NET%20moderno.md).
* Logging: [Logging con ILogger](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/04-Logging%20con%20ILogger.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **IIS (*Internet Information Services*):** servidor web de Windows.
* **Sitio (*site*):** aplicación web publicada en IIS con uno o más *bindings* (protocolo, IP, puerto, nombre de host).
* **Application pool:** grupo de aplicaciones que comparten un proceso de trabajo (`w3wp.exe`), su identidad y su configuración de reciclado.
* **Reciclado:** reinicio periódico o por condiciones del proceso del *application pool*.
* **ASP.NET Core Module (ANCM):** módulo nativo de IIS que hospeda o reenvía las peticiones a una aplicación ASP.NET Core.
* **Hosting Bundle:** instalador que agrega el runtime de .NET, el runtime de ASP.NET Core y el ANCM al servidor.
* **In-process / out-of-process:** la aplicación corre dentro de `w3wp.exe`, o en un proceso aparte con Kestrel y el ANCM como proxy.
* **`app_offline.htm`:** archivo que, colocado en la carpeta de la aplicación, la detiene y muestra su contenido mientras se despliega.

-----

## El problema

El equipo publica la nueva API .NET 10 en el mismo servidor IIS donde corren desde hace años las aplicaciones Web Forms. Copian la carpeta de `bin\Release` al servidor, crean el sitio y:

1. **HTTP 500.19 — Internal Server Error.** El `web.config` menciona `AspNetCoreModuleV2`, que no existe en el servidor: nadie instaló el *Hosting Bundle*.
2. Lo instalan. Ahora: **HTTP 500.30 — ASP.NET Core app failed to start.** Sin más detalle en el navegador.
3. Alguien pone la API en el mismo *application pool* que la aplicación Web Forms "para no crear otro". La aplicación vieja empieza a fallar al reciclarse.
4. Para desplegar una versión nueva, copian los archivos con la aplicación corriendo: `The process cannot access the file 'Tienda.Api.dll' because it is being used by another process`.
5. Mientras tanto, los usuarios de la aplicación Web Forms se quejan de que "cada tanto los saca del sistema": el *application pool* se recicla cada 29 horas por defecto y la sesión `InProc` se pierde.

Las consecuencias:

* despliegues que fallan con códigos de error que nadie sabe leer;
* aplicaciones que se afectan entre sí por compartir un proceso;
* pérdida de sesiones y cortes que parecen aleatorios.

-----

## Cómo funciona

### 1. Las piezas de IIS

```text
IIS
├── Sitio "Tienda"            bindings: https://tienda.empresa.com:443 (certificado)
│   ├── /                     aplicación → carpeta C:\inetpub\tienda-api    → pool "TiendaApi"
│   └── /facturacion          aplicación → carpeta C:\inetpub\facturacion   → pool "Facturacion"
└── Application pools
    ├── "TiendaApi"     .NET CLR: Sin código administrado · identidad: IIS AppPool\TiendaApi
    └── "Facturacion"   .NET CLR: v4.0 · pipeline integrado · identidad: IIS AppPool\Facturacion
```

* Cada *application pool* es uno o más procesos `w3wp.exe`. Lo que pasa en un pool (caída, reciclado, consumo de memoria) no afecta a los otros.
* La **identidad** del pool (por defecto `IIS AppPool\NombreDelPool`) es el usuario de Windows con el que corre la aplicación: necesita permisos de lectura sobre su carpeta y de escritura donde guarde logs o archivos.

### 2. .NET Framework en IIS

* El CLR de .NET Framework se carga **dentro** de `w3wp.exe` (pool con ".NET CLR v4.0").
* La configuración está en el `Web.config`: `<system.web>` (ASP.NET) y `<system.webServer>` (IIS).
* El **reciclado** reinicia el proceso: todo lo que esté en memoria (sesión `InProc`, cachés, variables `static`) se pierde. Por defecto, cada 1740 minutos (29 horas) y tras 20 minutos sin peticiones (*idle timeout*).

### 3. ASP.NET Core en IIS

```text
In-process (por defecto)                         Out-of-process
navegador → IIS (w3wp.exe)                       navegador → IIS (w3wp.exe)
              └─ ANCM                                          └─ ANCM ──proxy HTTP──► dotnet Tienda.Api.dll
                  └─ tu app (IISHttpServer)                                              (Kestrel)
              un solo proceso: más rápido                      dos procesos: aislamiento, una app por proceso
```

* El pool debe tener **".NET CLR: Sin código administrado"**: ASP.NET Core no usa el CLR de .NET Framework.
* `dotnet publish` genera un `web.config` mínimo para el ANCM:

```xml
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet" arguments=".\Tienda.Api.dll"
                  stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="inprocess">
        <environmentVariables>
          <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
        </environmentVariables>
      </aspNetCore>
    </system.webServer>
  </location>
</configuration>
```

* La configuración de la **aplicación** sigue en `appsettings.json` y variables de entorno; el `web.config` solo le dice a IIS cómo arrancarla.
* Con *in-process*, un *application pool* puede hospedar **una sola** aplicación ASP.NET Core.

### 4. Los errores que vas a ver

| Código | Significa | Causa típica | Dónde mirar |
| --- | --- | --- | --- |
| **500.19** | Configuración inválida | Falta el *Hosting Bundle* (módulo desconocido) o el `web.config` está mal formado | Instalar el *Hosting Bundle*; validar el XML |
| **500.30** | La app falló al arrancar (in-process) | Excepción en `Program.cs`: configuración faltante, cadena de conexión, permisos | Visor de eventos (Aplicación) y log de *stdout* |
| **500.31** | No encontró el runtime | Falta el runtime de la versión que pide la app | Instalar el *Hosting Bundle* de esa versión |
| **500.32** | No pudo cargar la DLL | Arquitectura incorrecta (32/64 bits) | "Habilitar aplicaciones de 32 bits" del pool frente a la publicación |
| **502.5** | Falló el proceso (out-of-process) | El proceso de Kestrel no arrancó | Visor de eventos y *stdout* |
| **503** | Servicio no disponible | El *application pool* está detenido (a veces por fallas repetidas: *rapid-fail protection*) | Administrador de IIS, estado del pool |

### 5. Diagnosticar un 500.30

1. **Visor de eventos** → Registros de Windows → Aplicación: el ANCM registra la excepción de arranque.
2. **Log de *stdout*:** en el `web.config`, `stdoutLogEnabled="true"`, crear la carpeta `logs` y darle permisos de escritura a la identidad del pool. **Apagarlo después**: no rota y llena el disco.
3. **Arrancar a mano** en el servidor, desde la carpeta publicada: `dotnet .\Tienda.Api.dll`. La excepción aparece en la consola.
4. Revisar diferencias de entorno: `ASPNETCORE_ENVIRONMENT` (sin definir = `Production`, que puede no tener la configuración que sí tenías en `Development`).

### 6. Desplegar sin bloquear archivos

```text
1. Copiar app_offline.htm a la carpeta de la aplicación   → el ANCM detiene la app y libera las DLL
2. Copiar los archivos nuevos (dotnet publish -c Release -o publicado)
3. Borrar app_offline.htm                                 → la próxima petición arranca la versión nueva
```

Web Deploy (`msdeploy`) y las tareas de despliegue de Azure DevOps o GitHub Actions automatizan estos pasos. Para no cortar el servicio, se usan dos instancias detrás de un balanceador o despliegues *blue/green*.

### 7. Configuración del pool que conviene revisar

| Opción | Por defecto | Recomendación |
| --- | --- | --- |
| .NET CLR | v4.0 | "Sin código administrado" para ASP.NET Core |
| Modo de inicio | OnDemand | `AlwaysRunning` (+ precarga del sitio) si el primer request no puede ser lento |
| Idle timeout | 20 minutos | 0 o más alto si la app debe estar siempre caliente |
| Reciclado periódico | 1740 minutos | Un horario fijo de baja demanda, o desactivarlo si la app no tiene fugas |
| Identidad | ApplicationPoolIdentity | Mantenerla; dar permisos mínimos sobre carpetas y recursos |

-----

## Ejemplo completo

API mínima .NET 10 con un endpoint de diagnóstico que dice cómo está hospedada. Proyecto `dotnet new web -o Tienda.Api`. `Program.cs`:

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Hosting.Server;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/diagnostico", (IServer servidor, IWebHostEnvironment entorno) => TypedResults.Ok(new
{
    Proceso = Process.GetCurrentProcess().ProcessName,
    Servidor = servidor.GetType().Name,
    Entorno = entorno.EnvironmentName,
    Runtime = System.Runtime.InteropServices.RuntimeInformation.FrameworkDescription
}));

app.MapGet("/", () => "Tienda.Api funcionando");

app.Run();
```

Publicación para IIS:

```text
dotnet publish -c Release -o C:\inetpub\tienda-api
```

En el servidor: instalar el *Hosting Bundle* de .NET 10, crear el *application pool* `TiendaApi` con ".NET CLR: Sin código administrado", crear el sitio apuntando a `C:\inetpub\tienda-api` con ese pool y darle a `IIS AppPool\TiendaApi` permiso de lectura sobre la carpeta.

Respuesta de `GET /diagnostico` según dónde corre (la versión exacta del runtime depende del parche instalado):

```text
dotnet run (Kestrel, Development)
  {"proceso":"Tienda.Api","servidor":"KestrelServerImpl","entorno":"Development","runtime":".NET 10.0.x"}

IIS in-process (Production)
  {"proceso":"w3wp","servidor":"IISHttpServer","entorno":"Production","runtime":".NET 10.0.x"}

IIS out-of-process (hostingModel="outofprocess")
  {"proceso":"dotnet","servidor":"KestrelServerImpl","entorno":"Production","runtime":".NET 10.0.x"}
```

Qué observar:

* En *in-process*, la aplicación vive dentro de `w3wp.exe` y el servidor es `IISHttpServer`: no hay Kestrel. En *out-of-process*, hay un proceso `dotnet` con Kestrel detrás del ANCM.
* Con `dotnet run`, el proceso se llama como el ejecutable de la app (`Tienda.Api`); en *out-of-process* el `web.config` lanza `dotnet Tienda.Api.dll`, por eso el proceso es `dotnet`.
* El entorno en IIS es `Production` salvo que el `web.config` (o el sistema) defina `ASPNETCORE_ENVIRONMENT`. Si tu app solo tiene configuración en `appsettings.Development.json`, en IIS va a fallar con un 500.30.
* Este endpoint es una herramienta de diagnóstico: en producción, protégelo o quítalo, porque expone detalles del servidor.

-----

## Errores comunes

**1. No instalar el *Hosting Bundle*.**
Qué pasa: 500.19 (módulo desconocido) o 500.31 (runtime no encontrado).
Por qué: IIS no tiene el ANCM ni el runtime de .NET moderno.
Arreglo: instalar el *Hosting Bundle* de la versión correcta y reiniciar IIS (`iisreset`) o el servidor.

**2. Pool con ".NET CLR v4.0" para ASP.NET Core.**
Qué pasa: funciona, pero carga el CLR de .NET Framework sin necesidad, y puede confundir el diagnóstico.
Por qué: el valor por defecto del pool es para aplicaciones de Framework.
Arreglo: "Sin código administrado" en los pools de ASP.NET Core.

**3. Compartir un *application pool* entre aplicaciones.**
Qué pasa: un reciclado, una fuga de memoria o una caída de una aplicación afecta a las demás; con ASP.NET Core *in-process*, la segunda aplicación ni siquiera arranca.
Por qué: un pool es un proceso compartido.
Arreglo: un pool por aplicación.

**4. Copiar archivos con la aplicación en ejecución.**
Qué pasa: archivos bloqueados y despliegues a medias.
Por qué: las DLL están cargadas en el proceso.
Arreglo: `app_offline.htm` antes de copiar, o Web Deploy, que lo hace automáticamente.

**5. Dejar el log de *stdout* activado.**
Qué pasa: archivos que crecen sin rotar hasta llenar el disco.
Por qué: el log de *stdout* es para diagnosticar el arranque, no para producción.
Arreglo: activarlo, diagnosticar y desactivarlo; para logs permanentes, un proveedor de logging real.

**6. Guardar estado en memoria con reciclado activo.**
Qué pasa: usuarios que pierden la sesión "cada tanto", cachés que se vacían.
Por qué: el reciclado reinicia el proceso cada 29 horas por defecto, y también tras inactividad.
Arreglo: sesión fuera del proceso (SQL Server, Redis) y reciclado en horarios controlados.

-----

## Según la versión de .NET

* **IIS 7 (Windows Server 2008):** pipeline integrado y *application pools* con identidades virtuales.
* **ASP.NET Core 1.x–2.1:** solo *out-of-process* (Kestrel detrás de IIS).
* **ASP.NET Core 2.2:** *hosting in-process* con el ANCM V2.
* **ASP.NET Core 3.0:** *in-process* pasa a ser el modelo por defecto.
* **.NET 6+:** el *Hosting Bundle* instala runtimes de varias versiones en paralelo; cada app usa la que pide.
* **.NET 8+:** mejoras en el ANCM (por ejemplo, mensajes de error más claros y cierre ordenado).

-----

## Cuándo sí y cuándo no

**Usa IIS cuando:** la organización tiene servidores Windows administrados, necesitas autenticación de Windows integrada, convives con aplicaciones de .NET Framework o el equipo de infraestructura ya opera IIS.

**Usa contenedores o Linux cuando:** buscas despliegues reproducibles, escalado horizontal, orquestación o reducir costos de licencias ([Contenedores Docker para APIs .NET](../../02-architecture-and-design-patterns/05-cloud-native-y-contenedores/01-Contenedores%20Docker%20para%20APIs%20NET.md)).

**No uses *out-of-process* cuando:** no tienes un motivo concreto (aislamiento, varias apps en un pool por requisitos especiales): *in-process* es más rápido y es el valor por defecto.

-----

## Resumen en 5 líneas

1. IIS hospeda .NET Framework dentro de `w3wp.exe` y ASP.NET Core a través del ASP.NET Core Module.
2. ASP.NET Core necesita el *Hosting Bundle* y un pool "Sin código administrado"; un pool por aplicación.
3. *In-process* (por defecto) corre dentro de `w3wp.exe`; *out-of-process* usa Kestrel detrás de IIS.
4. 500.19, 500.30 y 500.31 se diagnostican con el visor de eventos, el log de *stdout* y arrancando la app a mano.
5. `app_offline.htm` para desplegar sin bloqueos; el reciclado borra todo lo que esté en memoria.

-----

## Para profundizar

<details>
<summary>Autenticación de Windows en IIS</summary>

En intranets, IIS puede autenticar a los usuarios con sus cuentas de Windows (Kerberos/NTLM) sin pantalla de login. En ASP.NET Core *in-process* se activa en IIS (autenticación de Windows habilitada, anónima deshabilitada) y en la app con `builder.Services.AddAuthentication(IISDefaults.AuthenticationScheme)`. Es uno de los motivos más comunes para seguir usando IIS en aplicaciones internas.

</details>

<details>
<summary>Rapid-fail protection y el 503</summary>

Si el proceso de un pool falla varias veces seguidas en poco tiempo (por defecto, 5 fallas en 5 minutos), IIS **detiene el pool** para no entrar en un ciclo de reinicios, y el sitio responde 503 hasta que alguien lo vuelva a iniciar. Si ves un 503 después de un despliegue, revisa el estado del pool y el visor de eventos: probablemente hubo varios 500.30 antes.

</details>

-----

## En entrevista

### Respuesta corta (junior)

IIS es el servidor web de Windows. Para publicar una app ASP.NET Core hay que instalar el Hosting Bundle, crear un application pool sin código administrado y un sitio que apunte a la carpeta publicada. Por defecto la app corre in-process, dentro de `w3wp.exe`.

### Respuesta ampliada (semi-senior)

Distingo el hosting de .NET Framework (CLR dentro de `w3wp.exe`, configuración en `Web.config`) del de ASP.NET Core con el ANCM, *in-process* por defecto u *out-of-process* con Kestrel. Uso un pool por aplicación con "Sin código administrado" y permisos mínimos para su identidad. Diagnostico 500.19 (falta el Hosting Bundle o `web.config` inválido), 500.30 (excepción de arranque: visor de eventos, *stdout* temporal, arrancar a mano y revisar `ASPNETCORE_ENVIRONMENT`) y 500.31 (runtime faltante). Despliego con `app_offline.htm` o Web Deploy, y tengo en cuenta el reciclado: nada de estado importante en memoria.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es un application pool?**
Un grupo de aplicaciones que comparten un proceso `w3wp.exe`, su identidad y su configuración de reciclado.

**2. ¿In-process o out-of-process?**
In-process por defecto (más rápido); out-of-process solo si hace falta aislar en otro proceso.

**3. ¿Cómo diagnosticas un 500.30?**
Visor de eventos, log de stdout temporal y ejecutando `dotnet app.dll` en el servidor.

**4. ¿Por qué los usuarios pierden la sesión cada tanto?**
Por el reciclado del pool con sesión `InProc`: el proceso se reinicia y la memoria se pierde.

-----

## Práctica

**Ejercicio 1.** Después de publicar, la API responde 500.30. Al ejecutar `dotnet Tienda.Api.dll` en el servidor aparece `InvalidOperationException: Falta ConnectionStrings:Tienda.` En tu máquina funciona. ¿Qué pasó y cómo lo arreglas sin poner la cadena en el repositorio?

<details>
<summary>Solución</summary>

En tu máquina la cadena está en `appsettings.Development.json` o en *user secrets*, que solo se cargan en `Development`. En IIS el entorno es `Production` y no hay ninguna fuente con ese valor.

Opciones, de mejor a peor:

1. Variable de entorno en el servidor o en el `web.config` publicado:

```xml
<environmentVariables>
  <environmentVariable name="ConnectionStrings__Tienda" value="Server=...;Database=Tienda;Integrated Security=true" />
</environmentVariables>
```

2. Un gestor de secretos (Azure Key Vault) leído al arrancar.
3. Un `appsettings.Production.json` **solo en el servidor**, fuera del control de versiones.

`__` (doble guion bajo) reemplaza a `:` en los nombres de variables de entorno. Con `Integrated Security`, la identidad del pool (`IIS AppPool\TiendaApi`) debe tener un login en SQL Server.

</details>

**Ejercicio 2.** Escribe el script de PowerShell que despliega una nueva versión usando `app_offline.htm`.

<details>
<summary>Solución</summary>

```powershell
$destino = "C:\inetpub\tienda-api"
$origen  = "C:\builds\tienda-api\publicado"

Set-Content -Path "$destino\app_offline.htm" -Value "<h1>Actualizando, volvemos en un minuto.</h1>"
Start-Sleep -Seconds 2   # da tiempo al ANCM a detener la app y liberar las DLL

Copy-Item -Path "$origen\*" -Destination $destino -Recurse -Force -Exclude "app_offline.htm"

Remove-Item -Path "$destino\app_offline.htm"
Invoke-WebRequest -Uri "https://tienda.empresa.com/" -UseBasicParsing | Out-Null   # calienta la app
```

Mientras existe `app_offline.htm`, IIS responde su contenido con un 503 y la aplicación queda detenida. La última línea hace una primera petición para que la app arranque antes de que llegue un usuario.

</details>

-----

## Siguiente lección

[Estrategias de migración](02-Estrategias%20de%20migracion.md)
