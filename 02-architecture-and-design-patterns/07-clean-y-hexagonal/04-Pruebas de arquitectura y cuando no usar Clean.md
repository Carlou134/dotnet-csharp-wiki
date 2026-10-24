# Pruebas de arquitectura y cuándo no usar Clean

## En una frase

Una arquitectura que solo vive en un diagrama se degrada con cada *pull request*; las **pruebas de arquitectura** convierten sus reglas en pruebas que fallan cuando alguien las rompe, y la otra mitad del criterio es saber cuándo Clean es la herramienta equivocada: en sistemas simples, sus capas solo agregan costo.

-----

## Antes de empezar

Conviene que ya sepas:

* La estructura de cuatro proyectos: [Estructura de una solución Clean en .NET](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md).
* xUnit y `Theory`: [xUnit: Fact, Theory y aserciones](../../05-testing-y-calidad/01-pruebas-unitarias/02-xUnit%20Fact%20Theory%20y%20aserciones.md).
* Organización por caso de uso: [CQRS y mediator con MediatR](../06-cqrs-y-mediator/02-CQRS%20y%20mediator%20con%20MediatR.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Prueba de arquitectura:** prueba automática que verifica reglas estructurales del código (dependencias, visibilidad, nombres).
* **NetArchTest:** biblioteca para escribir pruebas de arquitectura en .NET con una API fluida.
* **Erosión de la arquitectura:** degradación gradual de las reglas de diseño por cambios pequeños que nadie revisa.
* **Vertical Slice Architecture:** organización por funcionalidad completa (endpoint, lógica y datos juntos) en lugar de por capas técnicas.
* **Monolito modular:** una sola aplicación desplegable dividida en módulos con fronteras claras y dependencias controladas.
* **Capa pasamanos (*pass-through*):** capa que solo reenvía llamadas a la siguiente sin agregar nada.

-----

## El problema

Dos equipos, dos problemas opuestos.

**Equipo A** armó la solución de la lección 3 con cuidado. Un año después, revisando una alerta de seguridad, alguien encuentra:

```csharp
// Tienda.Dominio/Pedido.cs
using Microsoft.AspNetCore.Http;           // para "loguear el usuario actual"

// Tienda.Aplicacion/ConfirmarPedido.cs
using Tienda.Infraestructura.Persistencia; // se hizo public "temporalmente"
```

Nadie lo decidió. Fueron dos cambios chicos, cada uno aprobado en un *code review* con prisa. La arquitectura se **erosionó** sin que ninguna prueba fallara.

**Equipo B** aplicó Clean "completo" a un panel interno de administración con doce tablas y ninguna regla de negocio:

```text
Endpoint → Command → Validator → Handler → IRepository → Repository → DbContext
   + PedidoDto, PedidoResponse, PedidoEntity, PedidoMapper, IPedidoService, PedidoService...
```

Agregar un campo a "Proveedor" toca once archivos en cinco proyectos. `ProveedorService.Obtener(id)` solo llama a `repositorio.Obtener(id)`, que solo llama a `db.Proveedores.Find(id)`.

Las consecuencias:

* reglas de arquitectura que se rompen en silencio;
* arquitecturas sobredimensionadas que multiplican el costo de cada cambio sin proteger ninguna regla.

-----

## Cómo funciona

### 1. Reglas como pruebas

```csharp
[Fact]
public void Dominio_NoDependeDeOtrasCapasNiDeFrameworks()
{
    TestResult resultado = Types.InAssembly(typeof(Pedido).Assembly)
        .ShouldNot()
        .HaveDependencyOnAny("Tienda.Aplicacion", "Tienda.Infraestructura", "Tienda.Api",
                             "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore")
        .GetResult();

    Assert.True(resultado.IsSuccessful, Fallas(resultado));
}
```

NetArchTest inspecciona los ensamblados compilados (con Mono.Cecil) y evalúa la regla sobre cada tipo. Si falla, `FailingTypeNames` dice **qué tipos** la rompen: el mensaje de la prueba apunta al archivo culpable.

### 2. Qué reglas vale la pena verificar

| Regla | Por qué |
| --- | --- |
| Dominio no depende de Aplicación, Infraestructura, Api, EF Core ni ASP.NET Core | Es la regla de dependencias |
| Aplicación no depende de Infraestructura, Api, EF Core ni ASP.NET Core | Los casos de uso se prueban sin infraestructura |
| Los tipos de Infraestructura (salvo el registro) no son públicos | Los endpoints no se saltean la aplicación |
| Las entidades del dominio no tienen *setters* públicos | El modelo rico no se degrada a anémico |
| Los handlers o casos de uso son `sealed` / siguen una convención de nombres | Consistencia (opcional, más discutible) |

Las dos primeras son imprescindibles **si** todo vive en un solo proyecto (las referencias no te protegen). Con proyectos separados, la referencia equivocada no compila, pero un paquete NuGet agregado al dominio sí: las pruebas cubren ese hueco.

### 3. Dónde viven y cuándo corren

Un proyecto `tests/Tienda.ArquitecturaTests` que referencia a todos los proyectos de `src`. Corre con `dotnet test` como cualquier otra prueba: en local y en CI, en cada *pull request*. Una regla rota bloquea el *merge* igual que un error de compilación.

### 4. Clean no es la única opción

| Estilo | Organiza por | Brilla cuando | Cuesta |
| --- | --- | --- | --- |
| Capas simples (API + servicios + EF Core) | Tipo técnico | CRUD, poca lógica, equipo chico | Las reglas se dispersan si el dominio crece |
| **Clean / Hexagonal** | Dirección de dependencias | Dominio con reglas, larga vida, infraestructura cambiante | Más proyectos, mapeos, indirección |
| Vertical Slice | Funcionalidad (cada caso de uso es una "rebanada" completa) | Muchos casos de uso independientes, poco compartido | Duplicación si hay mucho dominio común |
| Monolito modular | Módulos del negocio con fronteras (Pedidos, Pagos, Inventario) | Sistemas grandes con áreas separables | Disciplina para no acoplar módulos |

No son excluyentes: un monolito modular puede tener Clean dentro de cada módulo, y una rebanada vertical puede respetar la regla de dependencias para su dominio.

### 5. Señales de sobrediseño

* **Capas pasamanos:** `Service.Obtener` llama a `Repository.Obtener`, que llama a `db.Find`. Nadie agrega nada.
* **Una interfaz por clase**, todas con una sola implementación y ninguna prueba que use otra.
* **Mapeos en cadena:** `Entity → Model → Dto → Response`, con las mismas propiedades.
* **Agregar un campo toca más de cuatro o cinco archivos** en un CRUD sin reglas.
* **El dominio es anémico:** entidades con `{ get; set; }` y toda la "lógica" en servicios que asignan propiedades. Tienes el costo de Clean sin el dominio que la justifica.

### 6. Cómo decidir

```text
¿Hay reglas de negocio reales que proteger (estados, invariantes, cálculos)?
 ├─ No ──► capas simples o vertical slices; EF Core directo en los endpoints/handlers está bien
 └─ Sí ──► ¿el sistema vivirá años y cambiará de infraestructura o de mecanismos de entrada?
            ├─ No ──► dominio separado (sin EF Core) + handlers simples; sin todo el andamiaje
            └─ Sí ──► Clean / Hexagonal con proyectos y pruebas de arquitectura
¿Es grande, con áreas del negocio separables? ──► monolito modular (con Clean adentro si aplica)
```

La regla de dependencias (el dominio no conoce la infraestructura) casi siempre paga lo que cuesta. Lo que se ajusta al tamaño del sistema es **cuánto andamiaje** pones alrededor.

-----

## Ejemplo completo

Proyecto de pruebas de arquitectura para la solución de la [lección 3](03-Estructura%20de%20una%20solucion%20Clean%20en%20NET.md):

```text
dotnet new xunit -o tests/Tienda.ArquitecturaTests
dotnet add tests/Tienda.ArquitecturaTests reference src/Tienda.Dominio src/Tienda.Aplicacion src/Tienda.Infraestructura src/Tienda.Api
dotnet add tests/Tienda.ArquitecturaTests package NetArchTest.Rules --version 1.3.2
```

`tests/Tienda.ArquitecturaTests/ReglasDeArquitecturaTests.cs`:

```csharp
using System.Reflection;
using NetArchTest.Rules;
using Tienda.Aplicacion;
using Tienda.Dominio;

namespace Tienda.ArquitecturaTests;

public class ReglasDeArquitecturaTests
{
    private static readonly Assembly Dominio = typeof(Pedido).Assembly;
    private static readonly Assembly Aplicacion = typeof(CrearPedido).Assembly;
    private static readonly Assembly Infraestructura = typeof(Tienda.Infraestructura.DependencyInjection).Assembly;

    [Fact]
    public void Dominio_NoDependeDeOtrasCapasNiDeFrameworks()
    {
        TestResult resultado = Types.InAssembly(Dominio)
            .ShouldNot()
            .HaveDependencyOnAny("Tienda.Aplicacion", "Tienda.Infraestructura", "Tienda.Api",
                                 "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore")
            .GetResult();

        Assert.True(resultado.IsSuccessful, Fallas(resultado));
    }

    [Fact]
    public void Aplicacion_NoDependeDeInfraestructuraNiDeFrameworksWeb()
    {
        TestResult resultado = Types.InAssembly(Aplicacion)
            .ShouldNot()
            .HaveDependencyOnAny("Tienda.Infraestructura", "Tienda.Api",
                                 "Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore")
            .GetResult();

        Assert.True(resultado.IsSuccessful, Fallas(resultado));
    }

    [Fact]
    public void Infraestructura_SoloExponeElRegistro()
    {
        TestResult resultado = Types.InAssembly(Infraestructura)
            .That().AreClasses()
            .And().DoNotHaveName("DependencyInjection")
            .Should().NotBePublic()
            .GetResult();

        Assert.True(resultado.IsSuccessful, Fallas(resultado));
    }

    [Fact]
    public void Dominio_EntidadesSinSettersPublicos()
    {
        var conSetterPublico = Dominio.GetTypes()
            .Where(t => t.IsClass && t.Namespace == "Tienda.Dominio"
                        && !typeof(Exception).IsAssignableFrom(t))   // Exception trae HelpLink y Source con set público
            .SelectMany(t => t.GetProperties(BindingFlags.Public | BindingFlags.Instance)
                .Where(p => p.SetMethod is { IsPublic: true })
                .Select(p => $"{t.Name}.{p.Name}"))
            .ToList();

        Assert.True(conSetterPublico.Count == 0, $"Setters públicos: {string.Join(", ", conSetterPublico)}");
    }

    private static string Fallas(TestResult resultado) =>
        $"Tipos que rompen la regla: {string.Join(", ", resultado.FailingTypeNames ?? [])}";
}
```

Ejecución con `dotnet test`. Resultado sobre la solución de la lección 3: **4 pruebas superadas, 0 con error**.

Ahora rompe una regla a propósito: en `Pedido.cs` agrega `using Microsoft.AspNetCore.Http;` y un campo `private readonly IHttpContextAccessor? _accesor;` (para eso el dominio necesitaría `<FrameworkReference Include="Microsoft.AspNetCore.App" />` en su `.csproj`, que es justamente el cambio "inocente" del problema). La prueba `Dominio_NoDependeDeOtrasCapasNiDeFrameworks` falla con el mensaje:

```text
Tipos que rompen la regla: Tienda.Dominio.Pedido
```

Qué observar:

* Las tres primeras reglas usan NetArchTest; la cuarta, reflexión simple. No hace falta una biblioteca para todo: una regla es una prueba como cualquier otra.
* El mensaje de fallo nombra el tipo culpable: el revisor del *pull request* sabe exactamente qué mirar.
* `Infraestructura_SoloExponeElRegistro` protege lo que en la lección 3 resolvía `internal`: si alguien vuelve público el `DbContext`, falla.
* `Dominio_EntidadesSinSettersPublicos` mantiene el modelo rico: un `{ get; set; }` agregado "para que EF Core funcione" rompe la prueba.
* Estas pruebas tardan milisegundos: no hay excusa para no correrlas en cada *build*.

-----

## Errores comunes

**1. Confiar en el diagrama y en el *code review*.**
Qué pasa: erosión gradual; un año después las capas existen solo de nombre.
Por qué: cada cambio chico parece aceptable y nadie ve el conjunto.
Arreglo: pruebas de arquitectura en CI para las reglas que importan.

**2. Aplicar Clean completo a un CRUD.**
Qué pasa: once archivos por campo nuevo, capas pasamanos y un equipo que odia la arquitectura.
Por qué: se elige la estructura antes de mirar el problema.
Arreglo: capas simples o vertical slices cuando no hay reglas que proteger; Clean cuando el dominio lo justifica.

**3. Clean con dominio anémico.**
Qué pasa: todo el costo (proyectos, puertos, mapeos) y entidades que son bolsas de propiedades.
Por qué: se copió la estructura sin el modelado de dominio que le da sentido.
Arreglo: mueve las reglas a las entidades ([Modelo anémico y modelo de dominio](../01-domain-driven-design/01-Modelo%20anemico%20y%20modelo%20de%20dominio.md)), o reconoce que no hace falta Clean.

**4. Pruebas de arquitectura sobre detalles cosméticos.**
Qué pasa: reglas sobre nombres o sufijos que fallan por motivos que a nadie le importan, y el equipo empieza a ignorarlas.
Por qué: se verifican convenciones de estilo en lugar de reglas de diseño.
Arreglo: pocas reglas, las que protegen dependencias y encapsulamiento.

**5. Mezclar estilos sin decidir.**
Qué pasa: la mitad del sistema en capas, un cuarto en vertical slices y el resto "como salió".
Por qué: cada desarrollador eligió su estilo.
Arreglo: documenta la decisión (un ADR corto) y, si conviven estilos, que sea por módulo y a propósito.

-----

## Según la versión de .NET

* **NetArchTest (2019):** inspirado en ArchUnit (Java); `NetArchTest.Rules` 1.3.x es la versión estable.
* **ArchUnitNET:** alternativa más expresiva (reglas sobre capas definidas una vez y reutilizadas).
* **2018–2020:** se popularizan *Vertical Slice Architecture* (Jimmy Bogard) y los monolitos modulares como alternativas a las capas tradicionales.
* **.NET 8 / 9:** plantillas de la comunidad para Clean Architecture y monolitos modulares; .NET Aspire facilita orquestar módulos y servicios en desarrollo.

-----

## Cuándo sí y cuándo no

**Escribe pruebas de arquitectura cuando:** la arquitectura tiene reglas que importan (dependencias, visibilidad) y el código lo tocan varias personas o durante años.

**Usa Clean / Hexagonal cuando:** hay reglas de negocio reales, larga vida e infraestructura o mecanismos de entrada que pueden cambiar.

**No uses Clean completo cuando:** es un CRUD, un prototipo, una herramienta interna sin reglas o un servicio chico de transformación de datos. Ahí, capas simples o vertical slices dan más con menos.

-----

## Resumen en 5 líneas

1. Las arquitecturas se erosionan con cambios chicos; las pruebas de arquitectura lo detectan en cada *build*.
2. Verifica pocas reglas que importen: dependencias del dominio y la aplicación, visibilidad de infraestructura, modelo rico.
3. Clean no es la única opción: capas simples, vertical slices y monolitos modulares tienen su lugar.
4. Capas pasamanos, una interfaz por clase y mapeos en cadena son señales de sobrediseño.
5. La regla de dependencias casi siempre conviene; el andamiaje alrededor se ajusta al problema.

-----

## Para profundizar

<details>
<summary>Vertical Slice con la regla de dependencias</summary>

En Vertical Slice, cada funcionalidad es una carpeta con su endpoint, su handler y su acceso a datos (`Features/Pedidos/Confirmar/`). Se puede usar EF Core directamente en el handler, sin repositorios. Aun así, puedes mantener un proyecto `Dominio` sin dependencias con las entidades ricas: las rebanadas usan el dominio para las reglas y EF Core para persistir. Es una combinación muy usada: menos capas que Clean, pero con el núcleo protegido.

</details>

<details>
<summary>ADR: dejar escrita la decisión</summary>

Un *Architecture Decision Record* es un archivo Markdown corto en el repositorio (`docs/adr/0003-clean-architecture.md`) con contexto, decisión, alternativas consideradas y consecuencias. Cuando alguien pregunta en dos años "¿por qué hay cuatro proyectos para esto?", la respuesta está ahí, con la fecha y las razones. Las pruebas de arquitectura verifican la decisión; el ADR la explica.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las pruebas de arquitectura verifican automáticamente reglas como "el dominio no depende de EF Core". En .NET se escriben con NetArchTest dentro de un proyecto de pruebas. Clean Architecture no siempre conviene: en un CRUD simple agrega capas sin beneficio.

### Respuesta ampliada (semi-senior)

Protejo la arquitectura con pocas pruebas que importan: dependencias del dominio y la aplicación, tipos de infraestructura no públicos y entidades sin setters públicos, en un proyecto que corre en CI. Elijo el estilo según el problema: Clean o Hexagonal cuando hay reglas de negocio reales y larga vida; capas simples o Vertical Slice para CRUD y herramientas; monolito modular para sistemas grandes con áreas separables, a veces con Clean adentro. Reconozco el sobrediseño (capas pasamanos, interfaces 1:1, mapeos en cadena, dominio anémico) y documento la decisión en un ADR.

### Preguntas frecuentes de seguimiento

**1. Si las capas son proyectos separados, ¿para qué pruebas de arquitectura?**
Porque nada impide agregar un paquete de infraestructura al dominio ni volver público un tipo interno.

**2. ¿Qué es Vertical Slice?**
Organizar por funcionalidad completa (endpoint, lógica y datos juntos) en lugar de por capas técnicas.

**3. ¿Cuándo no usarías Clean?**
En CRUD sin reglas, prototipos o servicios chicos de transformación de datos.

**4. ¿Qué es una capa pasamanos?**
Una que solo reenvía llamadas sin agregar comportamiento: señal de que sobra.

-----

## Práctica

**Ejercicio 1.** Escribe una prueba que verifique que ningún tipo del dominio tiene un constructor público sin parámetros (los agregados se crean con métodos de fábrica).

<details>
<summary>Solución</summary>

```csharp
[Fact]
public void Dominio_SinConstructoresPublicosSinParametros()
{
    var infractores = Dominio.GetTypes()
        .Where(t => t.IsClass && !t.IsAbstract && t.Namespace == "Tienda.Dominio"
                    && !typeof(Exception).IsAssignableFrom(t))
        .Where(t => t.GetConstructor(Type.EmptyTypes) is { IsPublic: true })
        .Select(t => t.Name)
        .ToList();

    Assert.True(infractores.Count == 0, $"Constructores públicos sin parámetros: {string.Join(", ", infractores)}");
}
```

Las excepciones se excluyen porque no son entidades. `GetConstructor(Type.EmptyTypes)` sin `BindingFlags` solo encuentra constructores públicos, así que los privados que usa EF Core no cuentan como infracción.

</details>

**Ejercicio 2.** Te piden un sistema para registrar asistencia a capacitaciones: alta de cursos, inscripción de empleados y un reporte de asistencia. No hay reglas especiales. ¿Qué arquitectura propones y por qué?

<details>
<summary>Solución</summary>

Capas simples o Vertical Slice: una API (Minimal APIs agrupadas por recurso), EF Core directamente en los endpoints o en handlers chicos, DTOs para la entrada y la salida, y validación en el borde. Sin proyectos separados ni puertos para la persistencia.

Razones: no hay invariantes que proteger, la infraestructura no va a cambiar y el sistema es chico; las capas de Clean solo serían pasamanos. Si en el futuro aparecen reglas (cupos por curso, prerrequisitos, aprobaciones), se extrae un dominio para esa parte: es más fácil agregar estructura cuando se necesita que quitar la que sobra.

</details>

-----

## Siguiente lección

Terminaste el módulo. Practica con los [Ejercicios del módulo](Ejercicios.md) y vuelve al [índice](README.md).
