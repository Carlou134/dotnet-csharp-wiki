# Pruebas unitarias

En esta carpeta aprendes a escribir pruebas que sirven: qué es una prueba unitaria y qué no, el ciclo de TDD, xUnit con `Fact`, `Theory` y aserciones precisas, dobles de prueba con Moq sin acoplar las pruebas a la implementación, y cobertura con Coverlet entendida como mapa y no como objetivo.

-----

## Antes de empezar

Conviene que ya tengas:

* C# con clases, interfaces, excepciones y `async`/`await`: [C# Core & Runtime](../../README.md).
* Inyección por constructor, para la lección de dobles: [Inyección de dependencias y ciclos de vida](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md).

Si aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Pruebas unitarias y TDD](01-Pruebas%20unitarias%20y%20TDD.md) | Qué es una unidad, pirámide de pruebas, AAA, FIRST, nombres, rojo-verde-refactor y casos límite | Clases y excepciones |
| [2. xUnit: Fact, Theory y aserciones](02-xUnit%20Fact%20Theory%20y%20aserciones.md) | `[Fact]`, `[Theory]`, `InlineData`, `MemberData`, aserciones, ciclo de vida, `Skip` y pruebas async | Lección 1 |
| [3. Dobles de prueba con Moq](03-Dobles%20de%20prueba%20con%20Moq.md) | Stubs, mocks y fakes; `Setup`, `Verify`; qué reemplazar y qué no; `NullLogger` y `FakeLogger` | xUnit, interfaces y DI |
| [4. Cobertura de código con Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md) | Líneas y ramas, `collector` frente a `msbuild`, ReportGenerator, filtros, umbrales y pruebas de mutación | xUnit |

-----

## El mapa completo en una mirada

```text
            /\    E2E: pocas
           /--\   Integración: base y HTTP reales
          /----\  Unitarias: muchas, milisegundos  ← esta carpeta
TDD:  🔴 prueba que falla → 🟢 código mínimo → 🔵 refactor (triangula con un segundo caso)

Prueba = Arrange → Act (una acción) → Assert (el resultado que define la regla)
xUnit:  [Fact] un escenario · [Theory] + [InlineData]/[MemberData] varios datos · una instancia por prueba
Dobles: stub (respuestas) · mock (interacciones) · fake (implementación simple)
        reemplaza lo externo; tu dominio va real; no mockees lo que no es tuyo
Cobertura: qué corrió ≠ qué se verificó → mapa y freno, no objetivo · mutación para medir detección
```

-----

## Cómo está armada cada lección

Todas las lecciones siguen las 13 secciones de la wiki y usan la misma solución `Tienda` (una biblioteca y su proyecto de pruebas xUnit). Los resúmenes de `dotnet test` se describen por cantidad de pruebas porque su formato cambia según la versión del SDK y el idioma del sistema.

-----

## Después de esta carpeta

El siguiente paso son las pruebas de integración: `WebApplicationFactory` para probar la API completa en memoria, bases reales con Testcontainers y pruebas de contrato. Para que las pruebas sean fáciles de escribir, repasa [Inyección de dependencias](../../03-aspnet-core-apis/04-web-api-servicios-y-documentacion/02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md) y [Código limpio](../../01-csharp-core-and-runtime/11-codigo-limpio/README.md).
