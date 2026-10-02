# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Acumulador.** Valor que se va construyendo elemento a elemento en `Aggregate`.

**Agregación.** Reducir una secuencia a un solo valor: total, promedio, máximo, cantidad.

**Agrupamiento.** Reunir los elementos que comparten una clave (`GroupBy`).

**Aplanar.** Convertir una secuencia de colecciones en una sola secuencia (`SelectMany`).

**Árbol de expresión.** Representación de una lambda como datos, que proveedores como Entity Framework traducen a SQL. Lo usa `IQueryable<T>`.

**Clave de ordenamiento.** Valor por el que se ordena: `OrderBy(l => l.Titulo)`.

**Cuantificador.** Operador que responde sí o no sobre una secuencia: `Any`, `All`, `Contains`.

**Ejecución diferida.** La consulta no se ejecuta al definirla, sino al recorrerla, y de nuevo en cada recorrido.

**Ejecución inmediata.** Operadores que se ejecutan en el momento: los que devuelven un valor (`Count`, `First`, `Sum`) y los que materializan (`ToList`).

**Formato compuesto.** Marcadores como `{0,-10}` o `{1,8:N2}` para alinear y formatear columnas al imprimir.

**`IGrouping<TKey, TElement>`.** Un grupo producido por `GroupBy`: una clave (`Key`) y los elementos que la comparten.

**`IQueryable<T>`.** Variante de `IEnumerable<T>` cuyas consultas se traducen y ejecutan en el origen de datos (por ejemplo, SQL).

**Join (unión).** Combinar dos secuencias emparejando elementos con una clave común. `Join` es un inner join; `GroupJoin` permite un left join.

**LINQ (*Language-Integrated Query*).** Conjunto de operadores de consulta integrados en C#, implementados como métodos de extensión sobre `IEnumerable<T>`.

**Lookup.** Estructura tipo diccionario en la que cada clave tiene varios valores (`ToLookup`).

**Materializar.** Ejecutar una consulta y guardar el resultado en una colección (`ToList`, `ToArray`, `ToDictionary`).

**Método de extensión.** Método estático que se usa como si fuera de instancia. LINQ está hecho de métodos de extensión.

**Operador de consulta.** Cada operación de LINQ: `Where`, `Select`, `OrderBy`, `GroupBy`...

**Orden estable.** Los elementos con la misma clave conservan su orden original. `OrderBy` es estable.

**Paginación.** Mostrar los resultados por páginas con `Skip` y `Take`, siempre sobre una secuencia ordenada.

**Predicado.** Lambda que devuelve `bool` y decide si un elemento pasa un filtro.

**Proyección.** Transformar cada elemento en otro valor (`Select`).

**Sintaxis de consulta.** Forma de escribir LINQ con palabras clave parecidas a SQL: `from ... where ... select`.

**Sintaxis de método.** Forma de escribir LINQ encadenando llamadas: `.Where(...).Select(...)`.

**Tipo anónimo.** Objeto creado con `new { ... }` sin declarar una clase; sus propiedades son de solo lectura y se usa con `var`.
