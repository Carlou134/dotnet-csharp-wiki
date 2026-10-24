# Cobertura de código con Coverlet

## En una frase

La cobertura mide qué líneas y ramas de tu código se ejecutaron durante las pruebas; Coverlet la recolecta y ReportGenerator la convierte en un reporte HTML, pero un porcentaje alto solo dice que el código **corrió**, no que alguien verificó que hiciera lo correcto.

-----

## Antes de empezar

Conviene que ya sepas:

* Escribir y ejecutar pruebas con xUnit: [xUnit: Fact, Theory y aserciones](02-xUnit%20Fact%20Theory%20y%20aserciones.md).
* Herramientas de la CLI de .NET: [Introducción](../../01-csharp-core-and-runtime/00-introduccion/README.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Cobertura de código:** porcentaje del código que se ejecutó durante las pruebas.
* **Cobertura de líneas:** proporción de líneas ejecutables que corrieron al menos una vez.
* **Cobertura de ramas:** proporción de caminos de decisión (cada lado de un `if`, cada caso de un `switch`) que se recorrieron.
* **Coverlet:** herramienta de cobertura multiplataforma para .NET.
* **Cobertura (formato):** formato XML estándar de reportes de cobertura (`coverage.cobertura.xml`); no confundir con la palabra en español.
* **ReportGenerator:** herramienta que convierte reportes de cobertura en HTML y otros formatos.
* **Pruebas de mutación:** técnica que introduce errores a propósito en el código para comprobar si las pruebas los detectan.

-----

## El problema

El equipo decide que "la cobertura debe ser 90 %". Dos semanas después, el número está en 92 %. Una de las pruebas que lo logró:

```csharp
[Fact]
public void CalcularEnvio_VariosCasos()
{
    var calculadora = new CalculadoraEnvio();
    calculadora.CalcularEnvio(50m, 1m);
    calculadora.CalcularEnvio(150m, 30m);
    calculadora.CalcularEnvio(100m, 20m);
}
```

No tiene un solo `Assert`. Ejecuta todas las líneas y todas las ramas, así que la cobertura del método es completa. Si alguien cambia `>=` por `>`, la prueba sigue verde.

Además, para llegar al número, alguien marcó con `[ExcludeFromCodeCoverage]` el servicio de facturación "porque es difícil de probar".

Lo que pasa:

1. El porcentaje sube y la capacidad de detectar errores no cambia.
2. El código más riesgoso (facturación) queda fuera de la métrica justo por ser difícil.
3. La guía de clase mezclaba `coverlet.msbuild` y `coverlet.collector` con comandos de uno y otro, y nadie entendía por qué a veces no aparecía el porcentaje.

Las consecuencias:

* una métrica optimizada como objetivo deja de medir lo que importaba (ley de Goodhart);
* exclusiones que esconden riesgo;
* herramientas configuradas a medias.

-----

## Cómo funciona

### 1. Qué mide (y qué no)

```csharp
public decimal CalcularEnvio(decimal subtotal, decimal pesoKg)
{
    decimal costo = subtotal >= 100m ? 0m : 10m;   // rama A: ≥ 100 · rama B: < 100
    if (pesoKg > 20m)                              // rama C: > 20 · rama D: ≤ 20
    {
        costo += 15m;
    }
    return costo;
}
```

| Pruebas | Líneas | Ramas | ¿Detecta `>=` → `>`? |
| --- | --- | --- | --- |
| `(50, 1)` | Todas menos `costo += 15m` | B, D (2 de 4) | No |
| `(50, 1)` y `(150, 30)` | Todas | A, B, C, D | No (no prueba 100 exactos) |
| Las anteriores + `(100, 1)` **con asserts** | Todas | Todas | **Sí** |

La cobertura dice **qué no probaste**. No dice si lo que probaste está bien probado.

### 2. Dos formas de usar Coverlet

| | `coverlet.collector` | `coverlet.msbuild` |
| --- | --- | --- |
| Instalación | Viene en la plantilla `dotnet new xunit` | `dotnet add package coverlet.msbuild` |
| Comando | `dotnet test --collect:"XPlat Code Coverage"` | `dotnet test /p:CollectCoverage=true` |
| Resultado | `TestResults/{guid}/coverage.cobertura.xml` | `coverage.json` (o el formato que pidas) en la carpeta del proyecto, y **resumen en consola** |
| Configuración | Archivo `.runsettings` | Propiedades `/p:` |
| Pensado para | CI (Azure DevOps, GitHub Actions) | Uso local y umbrales en el build |

Usa uno. Si ejecutas `/p:CollectCoverage=true` con solo el *collector* instalado, no pasa nada: la propiedad la entiende el paquete MSBuild.

### 3. Del XML a un reporte que se lee

```text
dotnet tool install --global dotnet-reportgenerator-globaltool

dotnet test --collect:"XPlat Code Coverage"
reportgenerator -reports:"tests/**/TestResults/*/coverage.cobertura.xml" -targetdir:coverage-report -reporttypes:Html
```

* El *collector* crea una carpeta con un Guid por ejecución; el patrón con `*` y `**` la encuentra sin copiar rutas a mano.
* `coverage-report/index.html` muestra el resumen por ensamblado y clase, y cada archivo con las líneas en **verde** (cubiertas), **rojo** (no cubiertas) y **amarillo** (ramas cubiertas a medias).
* En la terminal, los argumentos con comodines van entre comillas rectas (`"`), no tipográficas (`"` `"`): estas últimas, frecuentes al copiar de documentos, rompen el comando.

### 4. Filtrar lo que se mide

```text
dotnet test /p:CollectCoverage=true /p:Include="[Tienda]*" /p:ExcludeByAttribute="GeneratedCodeAttribute"
```

| Filtro | Sintaxis | Ejemplo |
| --- | --- | --- |
| Incluir | `[Ensamblado]Namespace.Clase` con comodines | `[Tienda]*` solo el ensamblado de producción |
| Excluir | Igual | `[*]Tienda.Migrations.*` las migraciones generadas |
| Por atributo | Nombre del atributo | `ExcludeFromCodeCoverage`, `GeneratedCode` |
| Por archivo | Ruta con comodines | `**/Migrations/*.cs` |

```csharp
[ExcludeFromCodeCoverage(Justification = "Código generado por la herramienta X")]
public partial class ClienteGenerado { }
```

Excluye lo que **no es tuyo o no tiene lógica**: código generado, migraciones, `Program.cs` mínimo. No excluyas lo que es difícil de probar: ahí está el riesgo que la métrica debería mostrar.

### 5. Umbrales en el build

```text
dotnet test /p:CollectCoverage=true /p:Threshold=80 /p:ThresholdType=branch
```

Con `coverlet.msbuild`, si la cobertura de ramas baja de 80 %, `dotnet test` falla. Útil para que la cobertura **no baje** en cada *pull request*; peligroso como objetivo a alcanzar a cualquier costo (vuelve al problema).

### 6. ¿Cuánta cobertura?

No hay un número correcto universal. Referencias útiles:

* La **tendencia** importa más que el valor: que el código nuevo venga con pruebas y que el número no baje.
* La cobertura de **ramas** dice más que la de líneas.
* El 100 % rara vez compensa: las últimas líneas suelen ser manejo de errores improbables o código trivial.
* Una zona crítica (cálculo de precios, permisos) con 60 % preocupa más que un proyecto entero con 70 %.

Para medir si las pruebas **detectan errores**, existen las **pruebas de mutación** (Stryker.NET): cambian `>=` por `>`, `+` por `-`, etc., y cuentan cuántos "mutantes" sobreviven sin que ninguna prueba falle. La prueba sin asserts del problema dejaría vivos a todos.

-----

## Ejemplo completo

La solución `Tienda` de la [lección 1](01-Pruebas%20unitarias%20y%20TDD.md), con `CalculadoraEnvio` y sus pruebas. Primero, una versión con pruebas incompletas: deja en `CalculadoraEnvioTests` solo la prueba `CalcularEnvio_SubtotalMenorAlUmbral_CobraCostoBase`.

```text
cd Tienda
dotnet add tests/Tienda.Tests package coverlet.msbuild
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura /p:Include="[Tienda]Tienda.CalculadoraEnvio"
```

Coverlet imprime al final una tabla con la cobertura de líneas, ramas y métodos del módulo `Tienda`, y genera `tests/Tienda.Tests/coverage.cobertura.xml`. Convierte el XML en HTML:

```text
reportgenerator -reports:"tests/Tienda.Tests/coverage.cobertura.xml" -targetdir:coverage-report -reporttypes:Html
```

En `coverage-report/index.html`, en el archivo `CalculadoraEnvio.cs` verás:

```text
ArgumentOutOfRangeException.ThrowIfNegative(subtotal);       verde
decimal costo = subtotal >= UmbralEnvioGratis ? 0m : CostoBase;   amarillo: solo se recorrió la rama "< 100"
if (pesoKg > PesoMaximoSinRecargoKg)                            amarillo: solo se recorrió "≤ 20"
    costo += RecargoPorPeso;                                    rojo: nunca se ejecutó
return costo;                                                   verde
```

Restaura las seis pruebas de la lección 1 y vuelve a ejecutar ambos comandos: las líneas del método quedan en verde y sus ramas completas. Los porcentajes exactos de la tabla dependen de la versión del compilador y de Coverlet (el compilador genera ramas propias, por ejemplo en las validaciones), por eso no se copian aquí: lo que importa es **qué líneas y ramas** quedan sin cubrir.

Qué observar:

* Con una sola prueba, el reporte señala exactamente lo que faltaba: el envío gratis y el recargo por peso. Es la mejor forma de usar la cobertura: un mapa de lo que no probaste.
* Las líneas amarillas son ramas a medias: una condición que solo se evaluó en un sentido.
* `/p:Include` limitó la medición a la clase de producción; sin filtro, Coverlet mide todos los ensamblados que instrumenta (excepto el de pruebas).
* El reporte dice que las líneas corrieron, no que las pruebas tengan asserts. Eso lo revisas tú (o Stryker.NET).

-----

## Errores comunes

**1. Usar la cobertura como objetivo.**
Qué pasa: pruebas sin asserts y exclusiones para inflar el número.
Por qué: cuando una métrica se vuelve objetivo, la gente optimiza la métrica.
Arreglo: úsala como mapa de lo no probado y como freno a que baje; revisa la calidad de las pruebas en code review.

**2. Mezclar comandos de `collector` y `msbuild`.**
Qué pasa: `/p:CollectCoverage=true` no genera nada, o no aparece el resumen en consola.
Por qué: cada paquete entiende su propio mecanismo (`--collect` frente a propiedades MSBuild).
Arreglo: elige uno y usa sus comandos; en CI, normalmente el *collector*.

**3. Excluir lo difícil de probar.**
Qué pasa: el código más riesgoso desaparece de la métrica.
Por qué: `[ExcludeFromCodeCoverage]` se usó para mejorar el número, no para quitar código generado.
Arreglo: excluye solo código generado o sin lógica; lo difícil se rediseña para poder probarlo o se cubre con integración.

**4. Rutas fijas al XML del *collector*.**
Qué pasa: ReportGenerator no encuentra el archivo en la siguiente ejecución.
Por qué: el *collector* crea una carpeta con un Guid nuevo cada vez.
Arreglo: patrones con comodines (`**/TestResults/*/coverage.cobertura.xml`) o `--results-directory` fijo.

**5. Comillas tipográficas al copiar comandos.**
Qué pasa: `reportgenerator` interpreta mal el argumento o no encuentra archivos.
Por qué: `"` y `"` no son comillas para la terminal.
Arreglo: comillas rectas `"`, o sin comillas si la ruta no tiene espacios ni comodines.

-----

## Según la versión de .NET

* **2018:** Coverlet aparece como cobertura multiplataforma para .NET Core (`coverlet.msbuild`), alternativa a la cobertura solo de Visual Studio Enterprise.
* **2019:** `coverlet.collector`, integrado con `dotnet test --collect`; la plantilla `xunit` de .NET Core 3.0 lo incluye.
* **Visual Studio 2022 17.8 (2023):** la cobertura integrada, antes exclusiva de Enterprise, llega a las ediciones Community y Professional.
* **2024:** `Microsoft.Testing.Platform`, con `Microsoft.Testing.Extensions.CodeCoverage` como alternativa oficial.
* **.NET 10:** `dotnet test` puede ejecutarse sobre Microsoft.Testing.Platform; Coverlet sigue funcionando con VSTest.

-----

## Cuándo sí y cuándo no

**Usa la cobertura cuando:** buscas qué partes del código no tienen pruebas, revisas un *pull request* o quieres impedir que la cobertura baje en CI.

**Usa pruebas de mutación cuando:** quieres saber si las pruebas existentes detectan errores de verdad, sobre todo en el código crítico.

**No uses la cobertura cuando:** quieres juzgar la calidad de una suite o la de un desarrollador por un número.

-----

## Resumen en 5 líneas

1. La cobertura dice qué código corrió en las pruebas, no si se verificó bien.
2. Ramas dice más que líneas; el reporte HTML muestra rojo, amarillo y verde por línea.
3. `coverlet.collector` con `--collect` (CI) o `coverlet.msbuild` con `/p:CollectCoverage` (local, umbrales): uno de los dos.
4. ReportGenerator convierte el XML en HTML; usa comodines para encontrar el archivo.
5. Excluye solo código generado; usa la cobertura como mapa y freno, no como objetivo.

-----

## Para profundizar

<details>
<summary>Configurar el <em>collector</em> con <code>.runsettings</code></summary>

```xml
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat code coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Include>[Tienda]*</Include>
          <ExcludeByAttribute>GeneratedCodeAttribute,ExcludeFromCodeCoverageAttribute</ExcludeByAttribute>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

`dotnet test --collect:"XPlat Code Coverage" --settings coverlet.runsettings`. Es el equivalente de las propiedades `/p:` de `coverlet.msbuild`, versionado junto al código.

</details>

<details>
<summary>Stryker.NET en dos comandos</summary>

```text
dotnet tool install --global dotnet-stryker
cd tests/Tienda.Tests
dotnet stryker
```

Genera un reporte HTML con cada mutante: "eliminado" (alguna prueba falló, bien) o "sobreviviente" (ninguna prueba lo detectó). Es lento, porque ejecuta la suite una vez por mutante; úsalo sobre el código crítico, no en cada build.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La cobertura es el porcentaje de código que ejecutan las pruebas. En .NET se mide con Coverlet (`dotnet test --collect:"XPlat Code Coverage"`) y se ve en HTML con ReportGenerator. Un número alto no garantiza que las pruebas sean buenas.

### Respuesta ampliada (semi-senior)

Uso la cobertura como mapa de lo que no probé y como freno a que baje en CI, no como objetivo. Miro sobre todo la de ramas. En CI uso `coverlet.collector` con un `.runsettings` y ReportGenerator; localmente, `coverlet.msbuild` si quiero umbrales en el build. Excluyo solo código generado o sin lógica, nunca lo que es difícil de probar. Sé que la cobertura no ve si hay asserts: para medir la capacidad de detectar errores, uso pruebas de mutación con Stryker.NET en el código crítico.

### Preguntas frecuentes de seguimiento

**1. ¿100 % de cobertura significa que no hay errores?**
No. Significa que todo el código se ejecutó; no que se verificara.

**2. ¿Diferencia entre `coverlet.collector` y `coverlet.msbuild`?**
El primero se usa con `--collect` y genera XML en `TestResults`; el segundo con propiedades `/p:`, muestra resumen en consola y permite umbrales.

**3. ¿Qué excluirías de la cobertura?**
Código generado, migraciones y `Program.cs` mínimo; no la lógica difícil de probar.

**4. ¿Cómo sabes si tus pruebas detectan errores?**
Con pruebas de mutación, que introducen errores y cuentan cuántos pasan sin ser detectados.

-----

## Práctica

**Ejercicio 1.** La prueba del problema (sin asserts) da 100 % de cobertura al método. Escribe la cantidad mínima de pruebas con asserts que detecten estos dos cambios: `>=` → `>` en el umbral y la eliminación del recargo por peso.

<details>
<summary>Solución</summary>

```csharp
[Fact]
public void CalcularEnvio_SubtotalDeCienExactos_EsGratis() =>
    Assert.Equal(0m, new CalculadoraEnvio().CalcularEnvio(100m, 1m));

[Fact]
public void CalcularEnvio_PedidoPesado_SumaRecargo() =>
    Assert.Equal(25m, new CalculadoraEnvio().CalcularEnvio(50m, 30m));
```

Con `>`, la primera devuelve 10 y falla. Sin recargo, la segunda devuelve 10 y falla. Dos pruebas con asserts detectan lo que la prueba de "100 % de cobertura" no veía.

</details>

**Ejercicio 2.** Escribe el paso de GitHub Actions que ejecute las pruebas con cobertura y genere el reporte HTML como artefacto.

<details>
<summary>Solución</summary>

```yaml
- name: Pruebas con cobertura
  run: dotnet test --collect:"XPlat Code Coverage" --results-directory ./TestResults

- name: Reporte de cobertura
  run: |
    dotnet tool install --global dotnet-reportgenerator-globaltool
    reportgenerator -reports:"./TestResults/*/coverage.cobertura.xml" -targetdir:coverage-report -reporttypes:"Html;MarkdownSummaryGithub"

- name: Publicar reporte
  uses: actions/upload-artifact@v4
  with:
    name: coverage-report
    path: coverage-report
```

`--results-directory` fija la carpeta raíz y el comodín encuentra la subcarpeta con Guid. `MarkdownSummaryGithub` genera además un resumen en Markdown que puede mostrarse en la página de la ejecución.

</details>

-----

## Siguiente lección

Terminaste el módulo. Vuelve al [índice](README.md).
