# CLR, compilación y sistema de tipos

## En una frase
Roslyn traduce C# a CIL y metadatos; el CLR carga ese ensamblado, verifica tipos, administra memoria y convierte el código a instrucciones nativas.

-----

## Antes de empezar
Conviene que ya sepas: [Plataforma .NET y su evolución](05-Plataforma%20NET%20y%20evolucion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)): **CIL:** lenguaje intermedio común; **CTS:** reglas comunes de tipos; **CLS:** subconjunto para interoperabilidad entre lenguajes; **código administrado:** código ejecutado bajo servicios del CLR.

-----

## El problema
«C# compila directamente a código máquina» y «.NET interpreta C#» son explicaciones incompletas. Ocultan por qué una DLL contiene metadatos, por qué varios lenguajes comparten tipos y cuándo interviene el JIT.

-----

## Cómo funciona
### 1. Del fuente al ensamblado
```text
.cs -> Roslyn -> CIL + metadatos -> ensamblado .dll
                                      |
                                      v
                              CLR -> JIT -> CPU
```

### 2. Roslyn
Roslyn es la plataforma de compilación de C# y Visual Basic. Analiza sintaxis y semántica, informa diagnósticos y emite CIL y metadatos. También expone APIs usadas por analizadores y refactorizaciones.

### 3. CIL y ensamblados
El CIL es independiente del procesador. El ensamblado agrega manifiesto, referencias y descripción de tipos. El CLR puede cargarlo y el JIT compila los métodos necesarios para la arquitectura actual.

### 4. CLR y CoreCLR
CLR es el concepto y entorno de ejecución administrada. CoreCLR es la implementación usada por .NET moderno. Proporciona GC, excepciones, carga de ensamblados, hilos, JIT y servicios de diagnóstico.

### 5. CTS
El *Common Type System* define cómo se declaran y usan tipos. Por eso `int` de C# corresponde a `System.Int32` y puede consumirse desde otro lenguaje .NET.

### 6. CLS
La *Common Language Specification* es un subconjunto de reglas pensado para APIs públicas interoperables. Un tipo puede ser válido para el CTS y no ser CLS-compliant, como una API pública basada solo en enteros sin signo.

### 7. JIT frente a AOT
El JIT compila durante la ejecución y puede optimizar con información del entorno. Native AOT compila al publicar, reduce capacidades dinámicas y genera un artefacto específico de plataforma.

-----

## Ejemplo completo
```csharp
int quantity = 3;
Console.WriteLine(quantity.GetType().FullName);
Console.WriteLine(typeof(int) == typeof(System.Int32));
```

Salida:
```text
System.Int32
True
```

`int` es un alias del lenguaje para el tipo CTS `System.Int32`.

-----

## Errores comunes
**1. Decir que el CLR es un compilador.** Qué pasa: se confunden responsabilidades. Por qué: el CLR hospeda la ejecución; el JIT es uno de sus componentes. Arreglo: separa runtime, compilador de lenguaje y JIT.

**2. Usar MIL como nombre principal.** Qué pasa: se emplea un término impreciso. Por qué: los nombres habituales son CIL o IL; MSIL es histórico. Arreglo: usa CIL/IL.

**3. Confundir CTS con CLS.** Qué pasa: se cree que ambos definen lo mismo. Por qué: CTS define el sistema completo; CLS selecciona reglas interoperables. Arreglo: piensa «universo» frente a «subconjunto público».

-----

## Según la versión de .NET
- **.NET Framework:** usa su implementación histórica del CLR.
- **.NET Core y .NET 5+:** usan CoreCLR y comparten versión de producto con .NET.
- **.NET 7+:** Native AOT es una alternativa soportada para escenarios compatibles.
- **.NET 10:** mantiene JIT como modelo general y amplía escenarios AOT.

-----

## Cuándo sí y cuándo no
**Profundiza en el pipeline cuando:** diagnosticas carga, rendimiento, compatibilidad o generación de código. **No uses estos términos como decoración:** para explicar una regla de negocio suele bastar el nivel del lenguaje.

-----

## Resumen en 5 líneas
1. Roslyn compila C# a CIL y metadatos.
2. Un ensamblado contiene código intermedio y descripción de tipos.
3. El CLR administra la ejecución y el JIT produce código nativo.
4. El CTS permite compartir tipos entre lenguajes .NET.
5. El CLS define un subconjunto conveniente para APIs interoperables.

-----

## Para profundizar
<details><summary>¿Todo código .NET usa JIT?</summary>No. Native AOT genera código nativo al publicar, y algunas plataformas usan otros modos. El pipeline concreto depende del modelo de publicación.</details>

-----

## En entrevista
### Respuesta corta (junior)
C# se compila a CIL dentro de un ensamblado. El CLR lo carga y el JIT convierte los métodos a código máquina.

### Respuesta ampliada (semi-senior)
El ensamblado incluye CIL, metadatos y manifiesto. El CTS normaliza el modelo de tipos y el CLS facilita APIs públicas entre lenguajes. El CLR aporta GC, excepciones, carga y ejecución; Native AOT cambia la etapa de generación nativa.

### Preguntas frecuentes de seguimiento
**1. ¿`int` es distinto de `System.Int32`?** No, es un alias de C#.

**2. ¿CIL es código máquina?** No, todavía debe traducirse para la plataforma de ejecución.

-----

## Práctica
**Ejercicio 1.** Ordena Roslyn, CIL, CLR, JIT y CPU.
<details><summary>Solución</summary>Roslyn produce CIL; el CLR carga el ensamblado; el JIT traduce los métodos; la CPU ejecuta el código nativo.</details>

-----

## Siguiente lección
[SDK, CLI, proyectos y NuGet](07-SDK%20CLI%20proyectos%20y%20NuGet.md)
