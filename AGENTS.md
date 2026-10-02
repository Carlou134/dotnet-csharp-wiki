# AGENTS.md: guía para agentes que editan esta wiki

Wiki personal de estudio de C# y .NET. El dueño aprende con estas notas y aporta **notas de clase** (guías de sesión con teoría y ejercicios) para que se revisen, corrijan e integren. Tu trabajo es mantener **una sola línea editorial**: mismo formato, mismo tono, misma exigencia técnica.

-----

## Reglas generales

* **Versión de referencia:** .NET 10 / C# 14. Si algo cambió entre versiones, explícalo en la sección "Según la versión".
* **Idioma de las lecciones:** español neutro con **tú** (nunca voseo ni usted). Términos técnicos en inglés en cursiva la primera vez: *boxing*, *endpoint*.
* **Conversación con el usuario:** español rioplatense (voseo), respuestas cortas, una pregunta a la vez.
* **No hagas commits** salvo que el usuario lo pida. Si los pide: *conventional commits* (`docs(scope): ...`), sin atribución a IA ni `Co-Authored-By`.
* **No inventes datos:** salidas de programas, tiempos, tamaños o conteos deben ser los que realmente produce el código. Si una salida depende de la cultura, indícalo ("Salida (con cultura `en-US`)").
* **Verifica antes de afirmar:** firmas de APIs, nombres de tipos, códigos de error (CSxxxx) y comportamiento. Si no estás seguro, suaviza la afirmación o investiga.

-----

## Estructura del repositorio

```
README.md                              índice general (tabla por bloque; actualízala al agregar módulos)
01-csharp-core-and-runtime/            NN-modulo/ con lecciones del lenguaje
02-architecture-and-design-patterns/   DDD, EDA, patrones, Clean Architecture...
03-aspnet-core-apis/                   diseño de APIs, minimal APIs, seguridad...
04-data-access-and-transactions/       EF Core, transacciones, concurrencia (planificado)
```

Cada módulo es una carpeta `NN-nombre-en-kebab-case/` con:

* `README.md`: introducción, "Antes de empezar", tabla "Orden de lectura", "El mapa completo en una mirada" (bloque de texto), "Cómo está armada cada lección", "Después de esta carpeta".
* `Glosario.md`: términos en orden alfabético, formato `**Término:** definición. ([Lección](enlace.md))`.
* `NN-Titulo de la leccion.md`: una lección por archivo.
* `Ejercicios.md`: solo si hay material de clase o práctica extra.

**Nombres de archivo:** número de dos dígitos + título en español **sin tildes, sin `#` ni símbolos** (`03-Value objects.md`, `03-Sintaxis moderna de CSharp.md`).

**Enlaces:** siempre relativos, con espacios codificados como `%20` (`[Agregados](04-Agregados.md)`, `[Value objects](03-Value%20objects.md)`). Comprueba que el archivo de destino existe.

**Separador entre secciones:** `-----` (cinco guiones).

-----

## Formato de una lección (13 secciones, en este orden)

```markdown
# Título

## En una frase
Una o dos oraciones, sin jerga.

-----

## Antes de empezar
Conviene que ya sepas: (lista con enlaces a lecciones previas)
Palabras nuevas (también están en el [Glosario](Glosario.md)): (lista **término:** definición)

-----

## El problema
Una situación concreta, con código, que muestra por qué existe la herramienta. Lista de consecuencias.

-----

## Cómo funciona
Subsecciones ### 1., ### 2.... Código chico y explicado. Tablas comparativas.
Diagramas de texto (```text) cuando aclaren un mecanismo (flujos, memoria, orden de ejecución).

-----

## Ejemplo completo
Programa que COMPILA tal cual (consola con top-level statements, o Program.cs de `dotnet new web`).
Después, la salida real en un bloque ```text y qué observar.

-----

## Errores comunes
**N. Título del error.**
Qué pasa: ... (con código de error CSxxxx o excepción si aplica)
Por qué: ...
Arreglo: ...

-----

## Según la versión de C#        (o "de .NET" en módulos de ASP.NET/arquitectura)
Lista: versión → qué apareció o cambió.

-----

## Cuándo sí y cuándo no
**Usa X cuando:** ... **No lo uses cuando:** ...

-----

## Resumen en 5 líneas
Lista numerada de exactamente 5 líneas.

-----

## Para profundizar
Uno o más <details><summary>Tema</summary> ... </details> con detalles avanzados.

-----

## En entrevista
### Respuesta corta (junior)
### Respuesta ampliada (semi-senior)
### Preguntas frecuentes de seguimiento
**1. Pregunta?** Respuesta breve.

-----

## Práctica
**Ejercicio 1.** Enunciado.
<details><summary>Solución</summary> código + explicación </details>
(uno o dos ejercicios)

-----

## Siguiente lección
[Título](archivo.md)   (en la última del módulo: enlace a Ejercicios.md y al README)
```

-----

## Cómo procesar notas de clase (flujo principal)

Cuando el usuario pegue una guía de sesión ("Continúa con este"):

1. **Revisa la guía con ojo crítico.** Enumera los errores técnicos y conceptuales con evidencia: código que no compila (con el código CSxxxx), excepciones en tiempo de ejecución, APIs deprecadas, afirmaciones exageradas ("desacoplamiento total", "mejora el rendimiento"), malas prácticas de seguridad. Explícalos al usuario al entregar el trabajo.
2. **Decide la ubicación.** Lenguaje → `01-...`; diseño/arquitectura → `02-...`; ASP.NET Core → `03-...`; datos → `04-...`. Si encaja en un módulo existente, amplíalo; si no, crea `NN-modulo-nuevo/`. Solo pregunta si hay una ambigüedad real.
3. **Escribe las lecciones** con el formato de 13 secciones. Completa lo que la guía no cubre pero hace falta para entender el tema (por ejemplo, la configuración necesaria para que un ejemplo funcione).
4. **Crea `Ejercicios.md`** con todos los ejercicios de la guía, **corregidos**:
   * Encabezado + una línea de requisitos previos.
   * `## Ejercicios guiados`: cada uno con **Objetivo**, **Contexto** (incluye qué estaba mal en la guía), **Instrucciones**, `<details>` con la solución completa y su salida, y **Qué observar**.
   * `## Retos`: **Misión**, **Pista** y `<details>` con la solución.
   * `## Checkpoint`: 4-6 preguntas para autoevaluarse.
   * Mantén los nombres en inglés que use la guía (`Order`, `Customer`) para que el usuario los reconozca.
5. **Actualiza los índices:** README y Glosario del módulo, "Siguiente lección" y "Después de esta carpeta" de los módulos vecinos, y la tabla del `README.md` raíz.

-----

## Calidad del código

* Debe compilar en .NET 10 sin cambios. Records para DTOs, eventos y value objects; `class` para entidades.
* Nada de `throw new Exception(...)` genérico: usa `ArgumentException`, `InvalidOperationException` o una excepción de dominio.
* Valida `null` antes de usar un valor; usa los *throw helpers* (`ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegativeOrZero`).
* `async` de punta a punta: nada de `.Result`, `.Wait()` ni `async void` (salvo eventos de UI).
* En ejemplos de consola con estado compartido, aclara si no es seguro con hilos.

### Trampas ya detectadas (no las repitas)

* `Equals` sin `GetHashCode` → CS0659; `==` en clases compara referencias.
* `IReadOnlyCollection<T> => _lista` se puede castear a `List<T>`: usa `_lista.AsReadOnly()`.
* Record posicional con `init` → `with` se salta la validación; `default(struct)` se salta el constructor.
* `Validator.TryValidateObject` no ve atributos en parámetros de records posicionales; MVC exige que estén en el parámetro (no `property:`).
* `Problem("texto")`: el primer parámetro es `detail`, no `title`. Usa argumentos con nombre.
* `TypedResults.Unauthorized()` devuelve `UnauthorizedHttpResult` (también `ForbidHttpResult`, `ProblemHttpResult`).
* Versionamiento: paquete `Asp.Versioning.Mvc`; `[ApiVersion]` en el controlador, `[MapToApiVersion]` en la acción; en enlaces y `CreatedAtAction` pasa `version = RouteData.Values["version"]`.
* Comprobar que un header existe no es autenticación: `AddAuthentication` + `RequireAuthorization`.
* Castear un `Delegate` guardado como `Action<T>` a `Func<T, Task>` → `InvalidCastException`. Una lambda `async` pasada a `Action<T>` es `async void`.
* ProblemDetails sigue el RFC 9457 (reemplaza al 7807).
* MediatR y MassTransit 9 tienen licencia comercial desde 2025: menciónalo si los recomiendas.

-----

## Antes de terminar, comprueba

- [ ] Las 13 secciones están, en orden, separadas por `-----`.
- [ ] Todo el código compila y las salidas mostradas son las reales.
- [ ] Los enlaces relativos apuntan a archivos existentes.
- [ ] Las palabras nuevas están en el `Glosario.md` del módulo.
- [ ] El README del módulo y el README raíz están actualizados.
- [ ] Le explicaste al usuario los errores encontrados en sus notas.
