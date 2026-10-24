# Glosario: pruebas unitarias

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**AAA (Arrange, Act, Assert):** estructura de una prueba: preparar, ejecutar una acción y verificar el resultado. ([Pruebas y TDD](01-Pruebas%20unitarias%20y%20TDD.md))

**Cobertura (formato):** formato XML estándar de reportes de cobertura (`coverage.cobertura.xml`). ([Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md))

**Cobertura de código:** porcentaje del código que se ejecutó durante las pruebas. ([Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md))

**Cobertura de líneas:** proporción de líneas ejecutables que corrieron al menos una vez. ([Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md))

**Cobertura de ramas:** proporción de caminos de decisión (cada lado de un `if`, cada caso de un `switch`) que se recorrieron. ([Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md))

**Coverlet:** herramienta de cobertura multiplataforma para .NET, en variantes `collector` y `msbuild`. ([Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md))

**Doble de prueba (test double):** objeto que reemplaza a una dependencia real durante una prueba. ([Moq](03-Dobles%20de%20prueba%20con%20Moq.md))

**Fact:** prueba de xUnit sin parámetros. ([xUnit](02-xUnit%20Fact%20Theory%20y%20aserciones.md))

**Fake:** implementación funcional simplificada de una dependencia, como un repositorio en memoria. ([Moq](03-Dobles%20de%20prueba%20con%20Moq.md))

**Fixture:** objeto compartido entre pruebas para preparar algo costoso una sola vez (`IClassFixture<T>`). ([xUnit](02-xUnit%20Fact%20Theory%20y%20aserciones.md))

**InlineData / MemberData:** atributos que suministran datos a una `Theory`, en línea o desde un miembro de la clase. ([xUnit](02-xUnit%20Fact%20Theory%20y%20aserciones.md))

**Mock:** doble sobre el que se verifican interacciones: qué métodos se llamaron y con qué argumentos. ([Moq](03-Dobles%20de%20prueba%20con%20Moq.md))

**Moq:** biblioteca de .NET que crea stubs y mocks a partir de interfaces en tiempo de ejecución. ([Moq](03-Dobles%20de%20prueba%20con%20Moq.md))

**Prueba de integración:** prueba que ejercita varias piezas reales juntas, como base de datos o HTTP. ([Pruebas y TDD](01-Pruebas%20unitarias%20y%20TDD.md))

**Prueba omitida (skipped):** prueba que existe pero no se ejecuta; no cuenta como superada. ([xUnit](02-xUnit%20Fact%20Theory%20y%20aserciones.md))

**Prueba unitaria:** prueba automática, rápida y aislada de una unidad de comportamiento. ([Pruebas y TDD](01-Pruebas%20unitarias%20y%20TDD.md))

**Pruebas de mutación:** técnica que introduce errores a propósito para comprobar si las pruebas los detectan. ([Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md))

**Refactorizar:** cambiar la estructura del código sin cambiar su comportamiento. ([Pruebas y TDD](01-Pruebas%20unitarias%20y%20TDD.md))

**Regresión:** algo que funcionaba y dejó de funcionar por un cambio. ([Pruebas y TDD](01-Pruebas%20unitarias%20y%20TDD.md))

**ReportGenerator:** herramienta que convierte reportes de cobertura en HTML y otros formatos. ([Coverlet](04-Cobertura%20de%20codigo%20con%20Coverlet.md))

**Sistema bajo prueba (SUT):** clase o función que la prueba ejercita. ([Pruebas y TDD](01-Pruebas%20unitarias%20y%20TDD.md))

**Stub:** doble que devuelve respuestas preparadas; se verifica el resultado del SUT. ([Moq](03-Dobles%20de%20prueba%20con%20Moq.md))

**TDD (Test-Driven Development):** ciclo de escribir una prueba que falla, el código mínimo que la hace pasar y luego refactorizar. ([Pruebas y TDD](01-Pruebas%20unitarias%20y%20TDD.md))

**Theory:** prueba parametrizada de xUnit que debe pasar para todos los datos que recibe. ([xUnit](02-xUnit%20Fact%20Theory%20y%20aserciones.md))

**TheoryData&lt;T&gt;:** colección tipada de casos para `[MemberData]`. ([xUnit](02-xUnit%20Fact%20Theory%20y%20aserciones.md))

**Verificación de interacción:** comprobar que el SUT llamó, o no llamó, a una dependencia de cierta forma. ([Moq](03-Dobles%20de%20prueba%20con%20Moq.md))
